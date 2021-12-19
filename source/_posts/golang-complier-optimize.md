---
title: Golang 编译器优化那些事
tags:
  - Backend
  - Golang
categories:
  - Golang
date: 2021-12-18 12:13:14
updated: 2023-12-18 12:13:14
---

# Golang 编译器优化那些事


## 一、背景

去年写了一篇 [Golang Memory Model](https://fanlv.wiki/2020/06/09/golang-memory-model/) 文章。当时在文章里面贴了验证一个线程可见性问题`Demo`代码如下：

	func main() {
		running := true
		go func() {
			println("start thread1")
			count := 1
			for running {
				count++
			}
			println("end thread1: count =", count) // 这句代码永远执行不到为什么？
		}()
		go func() {
			println("start thread2")
			for {
				running = false
			}
		}()
		time.Sleep(time.Hour)
	}

	
	
今年`8`月份的时候私下跟一个朋友做技术交流的时候，朋友指出我这个`Demo`里面`thread1`不会结束，是因为`running = false`这句代码被编译器优化掉了，不是因为线程可见性的问题导致不会结束。因为最近几个月一直都在忙公司新项目（`9、10`双月自己就主动打了7天黑工，不是卷，只是单纯想把负责的项目做好），总之就是事情比较多，人一直处于超负荷状态，所以也没闲心去研究这些事情了。一拖就是几个月，一眨眼马上`2021`年都要过去了，想着再不研究下，这个技术债不知道要拖到什么时候去了，所以这两个周末花时间看了下。

先说结论，上面的`for`中的`running = false` 的确是被优化掉了，`thread2`的`for`循环汇编代码如下（[完整汇编代码点我](https://godbolt.org/z/qPGnKh3v8)）：

	main_func2_pc61:
	        PCDATA  $1, $-1   // Golang 垃圾回收器相关的指令，这里可以无视
	        JMP     main_func2_pc64
	        NOP


由上面汇编代码可以看到，`for { running = false }`, 直接被优化成了一个无限`JUMP`的死循环，没有其他指令，即使加上了 `-N -l` 禁止内联和编译器优化结果也一样，只是一条`JUMP`指令变成了四条（[完整汇编点我](https://godbolt.org/z/hrGjWW8Wd)）。

	        JMP     main_func2_pc71
	main_func2_pc71:
	        PCDATA  $1, $-1
	        JMP     main_func2_pc73
	main_func2_pc73:
	        JMP     main_func2_pc75
	main_func2_pc75:
	        JMP     main_func2_pc71


看到编译器这个汇编代码，我第一反应是感觉这个不符合直觉，是不是`Golang`编译器的`Bug`？

然后我用`C`写了个类似的 [Demo](https://godbolt.org/z/foME4ahYh)，分别用 [GCC](https://godbolt.org/z/foME4ahYh) 和 [LLVM](https://godbolt.org/z/PdYz6zo9v) 去编译代码，发现在 `-O1`的优化级别下，编译器都是会把`for`循环里面的变量赋值给优化掉，具体汇编代码如下：

	void thread_test1( void *ptr) {
	    printf("thread_test1 start\n");
	    while (1) {
	        counter++;
	    }
	}
	
	// 汇编代码如下
	thread_test1:
	        subq    $8, %rsp
	        movl    $.LC0, %edi
	        call    puts
	.L2:
	        jmp     .L2
					

这也证明了这个优化并不是`Golang`编译器的`BUG`，那为什么会这样优化？这里先按下不表，我们来复习下编译原理相关的基础知识。

## 二、基础知识

### 2.1 编译系统

![image.png](https://upload-images.jianshu.io/upload_images/12321605-44302a94fbdcc7a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一般一个程序从源码翻译成可执行目标文件需要经过“`预处理`”、“`编译`”、“`汇编`”、”`链接`“四个阶段。

我们这里主要关注的是“`编译器`”做的事情。

### 2.2 编译器工作流程

![image.png](https://upload-images.jianshu.io/upload_images/12321605-89087d945e04e645.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里我们主要关注编译器代码优化流程。

### 2.3 中间代码(Intermediate Representation - IR)

中间代码（`Intermediate Representation`）也叫`IR`，是处于源代码和目标代码之间的一种表示形式。我们倾向于使用`IR`有两个原因。

1. 是很多解释型的语言，可以直接执行`IR`，比如`Python`和`Java`。这样的话，编译器生成`IR`以后就完成任务了，没有必要生成最终的汇编代码。
2. 我们生成代码的时候，需要做大量的优化工作。而很多优化工作没有必要基于汇编代码来做，而是可以基于`IR`，用统一的算法来完成。

像`GCC`、`LLVM`这种编译器，可以支持`N`种不同的源语言，并可以为`M`个不同`CPU`架构生成机器码，如果没有`IR`，直接由源语言直接生成真实的机器代码，这个工作量是巨大的。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-34f1f5d76e859b3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有了`IR`可以让编译器的工作更好的模块化，编译器前端不用再关注机器细节，编译器后端也不用关注编程语言的细节。这种实现会更加合理一些。

`IR`基于抽象层次划分，可以分为`HIR`、`MIR`、`LIR`。

`IR`数据数据结构常见的有几种：类似三地址指令（`Three Address Code - TAC`）线性结构、树结构、有向无环图（`Directed Acyclic Graph - DAG`）、程序依赖图（`Program Dependence Graph，PDG`）

这里不是重点，不再过多阐述，感兴趣朋友可以自己去深入了解下。

### 2.4 静态单赋值形式(Static Single Assignment - SSA)

静态单赋值（`SSA`），是`IR`的一种设计范式，它要求一个变量只能被赋值一次。举例来说
	
	 y := 1
	 y := 2
	 x := y

从上面的描述所知，第一行赋值行为是不需要的，因为y在第二行被二度赋值，`y`的数值在第三行被使用，一个程序通常会进行定义可达性分析（`reaching definition analysis`）来测定它。在`SSA`下，将会变成下列的形式：

	 y1 := 1
	 y2 := 2
	 x1 := y2

使用SSA的好处：

1. 当每个变量只有一个定值时，数据流分析和优化算法可以变的更简单。
2. 如果一个变量有`N`个使用和`M`个定值（占了程序中大约`N+M`条指令），表示定值-使用链所需要的空间（和时间）和`N*M`成正比，即成平方增大，对于几乎所有的实际程序，`SSA`形式的大小和原始程序成线性关系。
3. `SSA`形式中，变量的使用和定值可以与控制流程图的必经节点结构以一有用的方式联系起来，从而简化诸如冲突图构建这样的算法。
4. 源程序中同一个变量的不相关使用在`SSA`形式中变成了不同的变量，从而删除了他们来了之间不必要联系。

更多可以参考 [静态单赋值形式 - WIKI](https://zh.wikipedia.org/wiki/%E9%9D%99%E6%80%81%E5%8D%95%E8%B5%8B%E5%80%BC%E5%BD%A2%E5%BC%8F)

静态单赋值形式引入了一个`φ`节点(音标`/faɪ/`)，又叫`Phi`节点，这个节点的作用，就是根据控制流提供的信息，来确定选择两个值中的哪一个。`Phi`运算是`SSA`格式的`IR`中必然会采用的一种运算，用来从多个可能的数据流分支中选择一个值。

现代语言用于优化的`IR`，很多都是基于`SSA`的了，比如`Golang`编译器、`Java`的`JIT`编译器、`JavaScript` 的`V8`编译器，以及 `LLVM` 等。


### 2.5 常见的代码优化方法

常见的代码优化可以分为两个维度：

**第一个分类维度，是机器无关的优化与机器相关的优化**。机器无关的优化与硬件特征无关，比如把常数值在编译期计算出来（常数折叠）。而机器相关的优化则需要利用某硬件特有的特征，比如`SIMD`指令可以在一条指令里完成多个数据的计算。

**第二个分类维度，是优化的范围**。本地优化是针对一个基本块中的代码，全局优化是针对整个函数（或过程），过程间优化则能够跨越多个函数（或过程）做优化。

**本地优化**主要是针对一个`Block`内的优化，常见的优化手段有`常数折叠（Constant Folding）`、`常数传播（Constant Propagation）`、`代数简化（Algebra Simplification）`、`公共子表达式消除（Common Subexpression Elimination，CSE）`等等

**全局优化**是针对整个函数（或过程），全局优化中会把基本的`Block`构建成一个`控制流图（Control Flow Graph - CFG）`，然后基于`CFG`做`控制流分析（Control-Flow Analysis-CFA）`。`控制流分析`是帮助我们建立对程序执行过程的理解，比如哪里是程序入口，哪里是出口，哪些语句构成了一个基本块，基本块之间跳转关系，从而可以帮助我们做一些`活跃性分析`，对一些不可达的`Block`，会做`死码消除`。


### 2.6 Go Compiler - gc

`Go Compiler`（下文用`gc`表示）在`1.5`版本的时候实现了`自举`[WIKI](https://en.wikipedia.org/wiki/Bootstrapping_(compilers))。编译器相关源代码主要在 [src/cmd/compile](https://github.com/golang/go/tree/release-branch.go1.16/src/cmd/compile) 目录下。

在`Go Compiler`的 [README.md](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/README.md) 中介绍了`gc`主要分为`4`个模块：

1. `Parsing`，词法分析和语法分析，词法分析器代码主要在`scanner.go`中，语法分析器代码主要在`parser.go`中，`Go`的`AST`的节点定义在`nodes.go`中。
2. `Type-checking and AST transformations`，语义分析（类型检查和`AST`变换），语义分析的代码主要在`typecheck.go`中，主要做`类型检查`、`名称消解（Name Resolution）`、`类型推导`、`内联优化`、`逃逸分析`等。
3. `Generic SSA`，生成`SSA`格式的`IR`，`gc`的`IR`是基于控制流图`CFG`的。一个函数会被分成多个基本块，基本块中包含了一行行的指令。`Go SSA`中有三个比较重要的概念，分别是 [Value](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/value.go#L18) 、 [Block](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/block.go#L12)、[Func](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/func.go#L26)。
	* `Value` 是`SSA`的最主要构造单元，它可以定义一次、使用多次。在定义一个`Value`的时候，需要一个标识符`ID`作为名称、产生该`Value`的操作码（[Op](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/op.go#L19)）、一个类型（[Type](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/types/type.go#L118)，就是代码中`<>`里面的值），以及一些参数。
	* `Block`，基本块有三种：简单（`Plain`）基本块，它只有一个后继基本块；退出（`Exit`）基本块，它的最后一个指令是一个返回指令；还有`if`基本块，它有一个控制值，并且它会根据该值是`true`还是`false`，跳转到不同的基本块。
	* `Func`，函数是由多个基本块构成的。它必须有一个入口基本块（`Entry Block`），但可以有`0`到多个退出基本块，就像一个`Go`函数允许包含多个`Return`语句一样。
4. Generating machine code，生成机器码，主要代码在 [genssa](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/gc/ssa.go#L6296) 中。需要说的是，`gc`生成的汇编代码是`Plan9`汇编，是一种“伪汇编”，它是一种半抽象的汇编代码。在生成特定`CPU`的机器码的时候，它还会做一些转换。值得一说的是`gc`并没有做`指令排序`的工作。
  
`Go compiler SSA`更多可以看`SSA`官方 [README.MD](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/README.md)


### 三、问题分析

为了方便验证问题，我简化了一下[Demo](https://godbolt.org/z/TTofx3Ma5) 代码如下：
	
	var counter = 1
	
	func main() {
		go thread1()
		for counter < 2 {}
		println("finish , counter = %d", counter)
	}
	
	func thread1() {
		for {
			counter++
		}
	}

然后通过设置`GOSSAFUNC`环境变量来控制`SSA`调试输出

	GOSSAFUNC=main.thread1+ go build -gcflags="-N -l" ./demo.go

我们可以看到控制台输出如下，完整的`SSA`结果[点我查看](./ssa1.html)

	thread1 func() 
	  b1:
	    v1 = InitMem <mem> DEAD // Init函数使用的内存
	    v2 = SP <uintptr> DEAD // 栈指针
	    v3 = SB <uintptr> DEAD // 栈底指针
	    v4 = Addr <*int> {"".counter} v3 DEAD // 全局变量 counter 地址
	    v7 = Const64 <int> [1] DEAD // 常量"1"
	    v9 = Addr <*int> {"".counter} v3 DEAD // 全局变量 counter 地址
	    Plain -> b2  // 跳转到 b2 
	  b2: <- b1 b4
	    v12 = Phi <mem> v1 v10 DEAD // b1 过来的 v12 = v1 , b4 过来的 v12 = v10
	    Plain -> b3 // 跳转到 b3
	  b3: <- b2
	    v5 = Copy <mem> v12 DEAD  // 把 v12 的值拷贝到 v5
	    v6 = Load <int> v4 v5 DEAD // 读取 v4 地址中的数据到 v5 中
	    v8 = Add64 <int> v6 v7 DEAD / v8 = v6 + v7
	    v10 = Store <mem> {int} v9 v8 v5 DEAD // v9 = v8 使用的内存地址是 v5
	    Plain -> b4
	  b4: <- b3
	    Plain -> b2
	  b5: DEAD
	    v11 = Unknown <mem> DEAD
	    Ret v11
	  pass number lines begin

这个是`gc`生成的`SSA`形式的`IR`，简单说下这个`IR`相关含义：

1. `b1~b5`就是上文说的`Block`，表示基本块，不同基本快之间会数据流转。
2. `v1~v12`就是上文说的`Value`。`Value`的`=` 右边第一个是 [Op](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/op.go#L19)，`<>`里面表示的是`Value`的类型 [Type](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/types/type.go#L118)。
3. `b2: <- b1 b4`表示，数据可以从`b1`或者`b4`流转到`b2`
4. `v12 = Phi <mem> v1 v10` 表示`v12`取`v1`和`v10`中的一个值（具体需要看是从`b1`流入的还是`b2`流入的）

在 [SSA的结果中](./ssa1.html) 我们看到有很多 [Pass](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/compile.go#L426) ，每一个`Pass`对应这一个`IR`代码优化过程，比如`opt deadcode`是做`死码消除`优化、`generic cse` 是消除公共子表达式的算法等等。


可以看到所有`Block`中的`Value`在执行`Pass`优化前就已经标记为`Dead`了，然后`opt deadcode`优化的时候，清除掉了所有标记为`Dead`的`Value`。在 [SSA的结果中](./ssa1.html) 到最后生成汇编的时候，就只剩下`4`个`JUMP`指令了，也说就是变成一个空循环了。

	b1 00003 (+13) JMP 4
	b2 00004 (+14) JMP 5
	b3 00005 (14) JMP 6
	b4 00006 (14) JMP 4

到这里，我们知道`for`循环里面代码被优化为空循环，是因为编译器认为`for`循环里面的代码都是`死码`，接着我们要看编译器为什么要把`for`循环里面的`Counter++`标记为`Dead`，由上面[SSA的结果中](./ssa1.html) 结果我们知道，在`start`之前编译器就认为所有`Value`是`死码`，所以后面的`Pass`优化，不是我们关注的重点，可以先不用管。

在此之前，我们先看下这个`4`个`Block`的是如何生成的。我在`gc`的`ssa.go`源码中找到了`for`转换为`Block`的[相关代码](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/gc/ssa.go#L1449)，

	// OFOR: for Ninit; Left; Right { Nbody }
	// cond (Left); body (Nbody); incr (Right)
	//
	// OFORUNTIL: for Ninit; Left; Right; List { Nbody }
	// => body: { Nbody }; incr: Right; if Left { lateincr: List; goto body }; end:
	bCond := s.f.NewBlock(ssa.BlockPlain) // b2
	bBody := s.f.NewBlock(ssa.BlockPlain) // b3
	bIncr := s.f.NewBlock(ssa.BlockPlain) // b4
	bEnd := s.f.NewBlock(ssa.BlockPlain)  // b5


由代码可以知，`for`循环生成了`bCond`、`bBody`、`bIncr`、`bEnd`四个`Block`，因为我们`for`是个死循环，永远不会结束，所以它是可以被优化掉的，转对应的`CFG`如下图如下：

![image.png](https://upload-images.jianshu.io/upload_images/12321605-61c1541e6ac07077.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


再来看`gc`是如何判断一个`Block`中的`Value`是`Live`还是`Dead`的，我找到 [相关代码](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/deadcode.go#L56) 如下，


	// liveValues returns the live values in f and a list of values that are eligible
	// to be statements in reversed data flow order.
	// The second result is used to help conserve statement boundaries for debugging.
	// reachable is a map from block ID to whether the block is reachable.
	// The caller should call f.retDeadcodeLive(live) and f.retDeadcodeLiveOrderStmts(liveOrderStmts)
	// when they are done with the return values.
	func liveValues(f *Func, reachable []bool) (live []bool, liveOrderStmts []*Value) {
		// ..... 初始化 live []bool ，默认全部是Dead/false
		// ..... 如果是寄存器分配函数，全部设置为存活
		// ..... 内联相关代码，我们先无视
		
		// Find all live values
		q := f.Cache.deadcode.q[:0]
		defer func() { f.Cache.deadcode.q = q }()

		// Starting set: all control values of reachable blocks are live.
		// Calls are live (because callee can observe the memory state).
		for _, b := range f.Blocks {
			if !reachable[b.ID] { 
			// 如果 Block 不可达，则认为这个 Block下面所有的 Value 都是 Dead Value
			// 我们这个Demo场景，b5 下面的 Value 都是 Dead Value
				continue
			}
			for _, v := range b.ControlValues() {
				if !live[v.ID] {
					live[v.ID] = true
					q = append(q, v)
					if v.Pos.IsStmt() != src.PosNotStmt {
						liveOrderStmts = append(liveOrderStmts, v)
					}
				}
			}
			for _, v := range b.Values {
				if (opcodeTable[v.Op].call || opcodeTable[v.Op].hasSideEffects) && !live[v.ID] {
					live[v.ID] = true
					q = append(q, v)
					if v.Pos.IsStmt() != src.PosNotStmt {
						liveOrderStmts = append(liveOrderStmts, v)
					}
				}
				// ... 内联相关代码先忽略
			}
		}

		// Compute transitive closure of live values.
		for len(q) > 0 {
			// pop a reachable value
			v := q[len(q)-1]
			q = q[:len(q)-1]
			for i, x := range v.Args { 
			   // 存活的 Value 依赖的 Value 也必须是存活状态
			   // 假设 v11 (15) = StaticLECall <mem> {AuxCall{"".emptyFunc()}} v10
			   // v11 如果是存活的状态, 那么 v10 也就应该是存活状态
				if v.Op == OpPhi && !reachable[v.Block.Preds[i].b.ID] {
					continue
				}
				if !live[x.ID] {
					live[x.ID] = true
					q = append(q, x) // push
					if x.Pos.IsStmt() != src.PosNotStmt {
						liveOrderStmts = append(liveOrderStmts, x)
					}
				}
			}
		}

		return
	}












