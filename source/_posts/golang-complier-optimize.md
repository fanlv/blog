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



## 一、背景

去年写了一篇 [Golang Memory Model](https://fanlv.wiki/2020/06/09/golang-memory-model/) 文章。当时在文章里面贴了验证一个线程可见性问题`Demo`，具体代码如下：

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

	
	
今年`8`月份的时候跟一个朋友做技术交流的时候，朋友指出我这个`Demo`里面`thread1`不会结束，是因为`running = false`这句代码被编译器优化掉了，不是因为线程可见性的问题导致不会结束，[当时转汇编](https://godbolt.org/z/ba35MsGeY) 看了下，发现的确是被优化的掉了，但是一直没想清楚是为什么，后面因为一直都在忙公司新项目（`9、10`双月周末自己就主动打了`7`天黑工，不是我卷，只是简单想把负责的项目做好），事情比较多，人一直处于超负荷状态，所以也没闲心去研究这些事情了。一拖就是几个月，一眨眼马上`2021`年都要过去了，想着再不研究下，这个技术债不知道要拖到什么时候去了，所以这两周末花时间看了下。

先说结论，上面的`for`中的`running = false` 的确是被优化掉了，`thread2`的`for`循环汇编代码如下（[完整汇编代码点我](https://godbolt.org/z/ba35MsGeY)）：

	        CALL    runtime.printlock(SB)
	        LEAQ    go.string."start thread2\n"(SB), AX
	        MOVQ    AX, (SP)
	        MOVQ    $14, 8(SP)
	        CALL    runtime.printstring(SB)
	        NOP
	        CALL    runtime.printunlock(SB)
	        JMP     main_func2_pc71
	main_func2_pc71:
	        PCDATA  $1, $-1 // Golang 垃圾回收器相关的指令，这里可以无视
	        JMP     main_func2_pc73
	main_func2_pc73:
	        JMP     main_func2_pc75
	main_func2_pc75:
	        JMP     main_func2_pc71


由上面汇编代码可以看到，`for { running = false }`, 直接被优化成了`4`条`JMP`的死循环。


看到编译器这个汇编代码，我第一反应是感觉这个不符合直觉（因为优化后的代码改变了程序含义），是不是`Go `编译器的`Bug`？

为了验证是不是`Go`编译器`BUG`，然后我用`C`写了个类似的 [Demo](https://godbolt.org/z/foME4ahYh)，分别用 [GCC](https://godbolt.org/z/ETrdf8d77) 和 [LLVM](https://godbolt.org/z/Ea758P863) 去测试，发现在 `-O1`的优化级别下，编译器都会把`for`循环里面的变量赋值给优化掉，具体汇编代码如下：

	void thread_test1( void *ptr) {
	    while (1) {
	        counter++;
	    }
	}
	
	// 汇编代码如下
	thread_test1:
	.L2:
	        jmp     .L2
					

从 [GCC](https://godbolt.org/z/ETrdf8d77) 和 [LLVM](https://godbolt.org/z/Ea758P863) 结果来看，这个优化并不是`Go`编译器特有的优化，那为什么会这样优化？ 我们接着往下看。

## 二、基础知识

### 2.1 编译系统

![image.png](https://upload-images.jianshu.io/upload_images/12321605-44302a94fbdcc7a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一般一个程序从源码翻译成可执行目标文件需要经过“`预处理`”、“`编译`”、“`汇编`”、”`链接`“四个阶段。

我们这里主要关注的是“`编译器`”做的事情。

### 2.2 编译器工作流程

![image.png](https://upload-images.jianshu.io/upload_images/12321605-89087d945e04e645.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在我们这个上下文场景中，我们主要关注编译器代码优化流程。

### 2.3 中间代码(Intermediate Representation - IR)

中间代码（`Intermediate Representation`）也叫`IR`，是处于源代码和目标代码之间的一种表示形式。我们倾向于使用`IR`有两个原因。

1. 是很多解释型的语言，可以直接执行`IR`，比如`Python`和`Java`。这样的话，编译器生成`IR`以后就完成任务了，没有必要生成最终的汇编代码。
2. 我们生成代码的时候，需要做大量的优化工作。而很多优化工作没有必要基于汇编代码来做，而是可以基于`IR`，用统一的算法来完成。

像`GCC`、`LLVM`这种编译器，可以支持`N`种不同的源语言，并可以为`M`个不同机器码，如果没有`IR`，直接由源语言直接生成真实的机器代码，这个工作量是巨大的。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-34f1f5d76e859b3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

有了`IR`可以让编译器的工作更好的**模块化**，编译器前端不用再关注机器细节，编译器后端也不用关注编程语言的细节。这种实现会更加合理一些。

`IR`基于抽象层次划分，可以分为`HIR`、`MIR`、`LIR`。

`IR`数据结构常见的有几种：类似三地址指令（`Three Address Code - TAC`）线性结构、树结构、有向无环图（`Directed Acyclic Graph - DAG`）、程序依赖图（`Program Dependence Graph，PDG`）


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

静态单赋值形式引入了一个`φ`节点(`/faɪ/`又叫`Phi`)，这个节点的作用，就是根据控制流提供的信息，来确定选择两个值中的哪一个。`Phi`运算是`SSA`格式的`IR`中必然会采用的一种运算，用来从多个可能的数据流分支中选择一个值。

**现代语言用于优化的 IR，很多都是基于 SSA 的了，比如 Golang 编译器、Java 的 JIT 编译器、JavaScript的 V8 编译器，以及 LLVM 等。**


### 2.5 编译器常用的循环优化算法

**第一种：归纳变量优化（Induction Variable Optimization`）**

看下面这个循环，其中的变量`j`是由循环变量派生出来的，这种变量叫做该循环的归纳变量。归纳变量的变化是很有规律的，因此可以尝试做**强度折减优化**。示例代码中的乘法可以由加法替代。

	int j = 0;
	for (int i = 1; i < 100; i++) {
	    j = 2*i;  //2*i可以替换成j+2
	}
	return j;
	
**第二种：边界检查消除（Unnecessary Bounds-checking Elimination）**

当引用一个数组成员的时候，通常要检查下标是否越界。在循环里面，如果每次都要检查的话，代价就会相当高（例如做多个数组的向量运算的时候）。如果编译器能够确定，在循环中使用的数组下标（通常是循环变量或者基于循环变量的归纳变量）不会越界，那就可以消除掉边界检查的代码，从而大大提高性能。

**第三种：循环展开（Loop Unrolling）**

把循环次数减少，但在每一次循环里，完成原来多次循环的工作量。比如：

	for (int i = 0; i< 100; i++){
	  sum = sum + i;
	}
	
优化后可以变成：

	for (int i = 0; i< 100; i+=5){
	  sum = sum + i;
	  sum = sum + i + 1;
	  sum = sum + i + 2;
	  sum = sum + i + 3;
	  sum = sum + i + 4;
	}
	
进一步，循环体内的`5`条语句就可以优化成`1`条语句：`sum = sum + i*5 + 10;`。

减少循环次数，本身就能减少循环条件的执行次数。同时，它还会增加一个基本块中的指令数量，从而为指令排序的优化算法创造机会。指令排序会在下一讲中介绍。

**第四种：循环向量化（Loop Vectorization）**

在循环展开的基础上，我们有机会把多次计算优化成一个向量计算。比如，如果要循环`16`万次，对一个包含了`16`万个整数的数组做汇总，就可以变成循环`1`万次，每次用向量化的指令计算`16`个整数。

**第五种：重组（Reassociation）**

在循环结构中，使用代数简化和重组，能获得更大的收益。比如，如下对数组的循环操作，其中数组`a[i,j]`的地址是`a+i*N+j`。但这个运算每次循环就要计算一次，一共要计算`M*N`次。但其实，这个地址表达式的前半截`a+i*N`不需要每次都在内循环里计算，只要在外循环计算就行了。


	for (i = 0; i< M; i++){
	  for (j = 0; j<N; j++){
	    a[i,j] = b + a[i,j];
	  }
	}
	
优化后的代码相当于：
	
	for (i = 0; i< M; i++){
	  t=a+i*N;
	  for (j = 0; j<N; j++){
	    *(t+j) = b + *(t+j);
	  }
	}
	
**第六种：循环不变代码外提（Loop-Invariant Code Motion，LICM）**

在循环结构中，如果发现有些代码其实跟循环无关，那就应该提到循环外面去，避免一次次重复计算。

**第七种：代码提升（Code Hoisting，或 Expression Hoisting）**

在下面的`if`结构中，`then`块和`else`块都有`z=x+y`这个语句，它可以提到`if`语句的外面。

	  if (x > y)
	    ...
	    z = x + y
	    ...
	  }
	  else{
	    z = x + y
	    ...
	  }
	  
这样变换以后，至少代码量会降低。但是，如果这个`if`结构是在循环里面，那么可以继续借助**循环不变代码外提优化**，把`z=x+y`从循环体中提出来，从而降低计算量。

	z = x + y
	for(int i = 0; i < 10000; i++){
	  if (x > y)
	    ...
	  }
	  else{
	    ...
	  }
	}
	
**第八种：激进的死代码优化**	

	int testFunc()  {
	    int k =0;
	    while (k<100) {
	        k++;
	    }
	    return 1;
	}


在某些优化场景下，因为循环理解的代码并不影响返回结果，编译器会直接优化掉循环的代码，只剩一个`return 1;`

还有一些场景，激进的死代码删除算法会删除没有输出的无线循环，从而改变程序的含义。因为在原来的程序不产生任何输出的情况下，删除这种无线循环后，程序会执行改循环之后的语句，而这些语句有可能产生输出。在许多环境下，这被认为是不可接受的（详见《编译原理-虎书》`19.5`章节）。



### 2.6 窥孔优化

窥孔优化，就是通过一个滑动窗口（窥孔）扫描`IR`、`汇编代码`或者是`机器码`，每次扫描`n`行，然后检查窗口中的指令，看是否可以用更快或者更短的指令来替换窗口中的指令序列。这个窗口可以沿着代码不断的滑动，从而发现所有代码中的优化机会。窺孔优化技术并不要求在窺孔中的代码一定是连续的。窺孔优化的特点是每一次改进又可以产生新的优化机会。

窺孔优化常用的优化场景有：

1. 消除冗余的加载和保存指令。
2. 消除不可达代码。
3. 控制流优化。
4. 代数化简和强度消减。
5. 使用机器特有的指令。


### 2.7 寄存器使用

代码生成的关键问题之一是决定哪个值放在寄存器里面，**寄存器是目标机上运行速度最快的计算单元**，但是我们通常没有足够的寄存器来存放所有的值。没有存放在寄存器中的值必须存放在内存中。只涉及寄存器运算分量的指令比那些涉及内运算分量的指令运行的快，因此，有效利用寄存器非常重要。

寄存器的使用经常被分解为两个子问题：

1. 寄存器分配：对于原程序中的每个点，我们选择一组将被存放在寄存器中的变量。
2. 寄存器指派：我们指定一个变量存放在哪个寄存器中。


当然，很少有“能够将所有的数据在所有时间内都保存在寄存器”的情况，因此精心的使用好寄存器，控制好对那些不在寄存器中的变量的访问很重要。有个几个需要特别考虑的问题：

1. **尽可能将程序执行中最频繁的变量分配到寄存器中**。
2. 尽可能高效的访问那些不在寄存器中的变量。
3. 使“记账”使用的寄存器（例如，为管理变量对存储器的访问而保留的寄存器）个数尽可能的少，以便能用更多寄存器来容纳变量的值。
4. 尽可能提高过程调用和相关操作的效率，如进入和退出作用域，以便使他们的开销减至最小。

参与寄存器分配常见的有以下对象：

* 栈指针（frame pointer），指向运行栈当前过程的栈帧开始处。
* 动态链（dynamic link）
* 静态链 （static link）
* 全局偏移表指针（global offset table pointer）
* 参数，参数由当前活跃过程传递给被调用过程
* 返回值，当前活跃过程调用的过程返回的结果
* 频繁使用的变量，最频繁使用的局部（**也可能是非局部的或者全局**）变量
* 临时变量，在表达式求之期间和其他较短活跃期内使用和计算出的临时值。




### 2.8 Go编译器基础

`Go Compiler`（下文用`gc`表示）在`1.5`版本的时候实现了 [自举](https://en.wikipedia.org/wiki/Bootstrapping_(compilers))。编译器相关源代码主要在 [src/cmd/compile](https://github.com/golang/go/tree/release-branch.go1.16/src/cmd/compile) 目录下。

在`gc`的 [README.md](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/README.md) 中介绍了`gc`主要分为`4`个模块：

1. `Parsing`，词法分析和语法分析，词法分析器代码主要在`scanner.go`中，语法分析器代码主要在`parser.go`中，`Go`的`AST`的节点定义在`nodes.go`中。
2. `Type-checking and AST transformations`，语义分析（类型检查和`AST`变换），语义分析的代码在`typecheck.go`中，主要做`类型检查`、`名称消解（Name Resolution）`、`类型推导`、`内联优化`、`逃逸分析`等。
3. `Generic SSA`，生成`SSA`格式的`IR`，`gc`的`IR`是基于控制流图`CFG`的。一个函数会被分成多个`基本块`，`基本块`中包含了一行行的`指令`。`Go SSA`中有三个比较重要的概念，分别是 [Value](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/value.go#L18) 、 [Block](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/block.go#L12)、[Func](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/func.go#L26)。
	* `Value` 是`SSA`的最主要构造单元，它可以**定义一次、使用多次**。在定义一个`Value`的时候，需要一个标识符`ID`作为名称、产生该`Value`的操作码（[Op](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/op.go#L19)）、一个类型（[Type](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/types/type.go#L118)，就是代码中`<>`里面的值），以及一些参数。
	* `Block`，基本块有三种：简单（`Plain`）基本块，它只有一个后继基本块；退出（`Exit`）基本块，它的最后一个指令是一个返回指令；还有`if`基本块，它有一个`控制值`，并且它会根据该值是`true`还是`false`，跳转到不同的基本块。
	* `Func`，函数是由多个基本块构成的。它必须有一个入口基本块（`Entry Block`），但可以有`0`到多个退出基本块，就像一个`Go`函数允许包含多个`Return`语句一样。
4. `Generating machine code`，生成机器码，主要代码在 [cmd/compile/internal/gc/ssa.go](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/gc/ssa.go#L6296) 中。需要说的是，`gc`生成的汇编代码是`Plan9`汇编，是一种`伪汇编`，它是一种半抽象的汇编代码。在生成特定`CPU`的机器码的时候，它还会做一些转换，值得一提的是`gc`并没有做`指令排序`的工作。
  
`Go compiler SSA`更多介绍，可以看`gc SSA `官方 [README.MD](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/README.md)

`gc`提供了一个生成可视化`SSA`的`IR`的选项，只要我们在执行`go build`设置环境变量`GOSSAFUNC`为想打印`IR`的函数名，`gc`会生成一个`ssa.html`文件。`ssa.html`文件会记录了编译器为了优化我们代码所经过的所有步骤和每个步骤的耗时。如果想在控制台也输出相关代码，可以在函数名后面加上`+`。[具体代码可看](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/gc/main.go#L525)

假如我要看`mian`包下面函数名为`thread1`的`IR`优化过程，可以执行如下命令

	GOSSAFUNC=main.thread1+ go build -gcflags="-N -l" ./demo.go


## 三、Go编译器For循环优化分析

### 3.1 Demo验证

为了方便验证问题，我简化了一下[Demo](https://godbolt.org/z/TTofx3Ma5) 代码如下：
	
	var counter = 1
	
	func main() {
		go thread1()
		for counter < 2 {}
		println("finish , counter = ", counter)
	}
	
	func thread1() {
		for {
			counter++
		}
	}

然后`go build`时候设置`GOSSAFUNC`环境变量来查看`SSA IR`输出

	GOSSAFUNC=main.thread1+ go build -gcflags="-N -l" ./demo.go

我们可以看到控制台输出如下，（完整的`SSA`结果[点我查看](./ssa1.html)）

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
	  b5: DEAD // 这个 Block 不可达，所以直接标记为 Dead 了
	    v11 = Unknown <mem> DEAD
	    Ret v11
	  pass number lines begin

这个是`Go Compiler`生成的`SSA`形式的`IR`，简单说下这个`IR`相关含义：

1. `b1~b5`就是上文说的`Block`，表示基本块，不同基本块之间会跳转。
2. `v1~v12`就是上文说的`Value`。`Value`的`=` 右边第一个是 [Op](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/op.go#L19)，`<>`里面表示的是`Value`的类型 [Type](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/types/type.go#L118)。所有的`Value`后面都有`DEAD`标记，表示这`Value`已经是`死码`了。
3. `b2: <- b1 b4`表示，数据可以从`b1`或者`b4`流转到`b2`
4. `v12 = Phi <mem> v1 v10` 表示`v12`取`v1`和`v10`中的一个值（具体需要看是从`b1`流入的还是`b2`流入的）

在 [SSA的结果中](./ssa1.html) 我们看到有很多 [Pass](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/compile.go#L426) ，每一个`Pass`对应这一个`IR`代码优化过程，比如`deadcode`是做`死码消除`优化、`cse` 是消除公共子表达式的算法等等。

可以看到所有`Block`中的`Value`在执行`Pass`优化前就已经标记为`Dead`了，然后`opt deadcode`优化的时候，清除掉了所有标记为`Dead`的`Value`。在 [SSA的结果中](./ssa1.html) 到最后生成汇编的时候，就只剩下`4`个`JUMP`指令了，也说就是变成一个空循环了。

	b1 00003 (+13) JMP 4
	b2 00004 (+14) JMP 5
	b3 00005 (14) JMP 6
	b4 00006 (14) JMP 4

看到这里，我们知道编译器认为`for`循环里面的代码都是`死码`，所以把`for`循环中的代码优化掉了，接着我们要看编译器为什么认为`for`循环里面的代码是`死码`。

由上面 [SSA的结果中](./ssa1.html) 输出我们知道，在`Pass`之前编译器就已经认为所有`Value`是`Dead`了，所以后面的`Pass`优化逻辑，不是我们关注的重点，可以先不用管。我们要看的是编译器如何判断一个`Value`是否存活。

在此之前，我们先看下这个`4`个`Block`的是如何生成的。我在`Go Compiler`的`ssa.go`源码中找到了`for`转换为`Block`的[相关代码](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/gc/ssa.go#L1449)，

	// OFOR: for Ninit; Left; Right { Nbody }
	// cond (Left); body (Nbody); incr (Right)
	//
	// OFORUNTIL: for Ninit; Left; Right; List { Nbody }
	// => body: { Nbody }; incr: Right; if Left { lateincr: List; goto body }; end:
	bCond := s.f.NewBlock(ssa.BlockPlain) // b2
	bBody := s.f.NewBlock(ssa.BlockPlain) // b3
	bIncr := s.f.NewBlock(ssa.BlockPlain) // b4
	bEnd := s.f.NewBlock(ssa.BlockPlain)  // b5


由代码可以知，`for`循环生成了`bCond`、`bBody`、`bIncr`、`bEnd`四个`Block`，转对应的`CFG`如下图如下：

![image.png](https://upload-images.jianshu.io/upload_images/12321605-61c1541e6ac07077.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


再来看`Go Compiler`是如何判断一个`Block`中的`Value`是`Live`还是`Dead`的，我在 [compile/internal/ssa/deadcode.go](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/deadcode.go#L56) 找到了如下代码：


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
				if v.Op == OpPhi && !reachable[v.Block.Preds[i].b.ID] {
					continue
				}
				
				// 存活的 Value 依赖的 Value 也必须是存活状态
			   // 假设 v11 (15) = StaticLECall <mem> {AuxCall{"".emptyFunc()}} v10
			   // v11 如果是存活的状态, 那么 v10 也就应该是存活状态，依次类推
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


`liveValues`函数主要做三件事

1. 遍历所有`reachable`（可达）的`Block`的`控制语句（ControlValues）`，有的话直接设置该`ControlValue`为`Live=Ture`，然后把存活的`Value`放到到一个队列中去，在第三步会用到。
2. 遍历所有`reachable`（可达）的`Block`的`Value`，如果`Value`的`Op`在 [opcodeTable](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/opGen.go#L2907) 对象是一个`函数调用(call=true)`就认为这个`Value`是存活的，或者如果`Value`的`Op`对象的`hasSideEffects`为`true`，也认为是存活的。`hasSideEffects`我翻了下`Go`的`Commit`是为了修复`Go 1.8` [sync/atomic loop elided](https://github.com/golang/go/issues/19182) 的`Bug`加入的，主要是 `Atomic` 相关的几个方法会设置`hasSideEffects`为`True`，具体可以看[add opcode flag hasSideEffects for do-not-remove](https://go-review.googlesource.com/c/go/+/37333/) 这个`commit`。这一步里面存活的`Value`也会放到队列中去。
3. 从队列依次取出存活的`Value`，然后遍历`Value`依赖的参数，设置参数`Value`的`Live=Ture`，依次类推，算出所有存活的`Value`。

我们再看下我们上面代码中`b1~b4`中所有`Value`的`op`，依次是`InitMem`、`SP`、`SB`、`Addr`、`Const64`、`Phi`、`Copy`、`Load`、`Add64`、`Store`，在 [opcodeTable](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/ssa/opGen.go#L2907) 中查了下，`call`和`hasSideEffects`都没有设置，默认都是`false`，所以最后所有`Block`中的`Value`都标记为`Dead`了。


### 3.2 添加控制语句、函数调用验证

上面说了，如果`Block`中有`控制语句（ControlValues）`或者`函数调用`，`Go Compiler`就会认为这个`Value`是存活的，我们分别用两个`Demo`验证下。

先看`控制语句`场景，我加了个`flag`变量，代码如下：（[完整代码](https://godbolt.org/z/b9sKePKWv)）

	var flag = true
	
	func thread1() {
		for flag { // 虽然 flag 一直是 true，然而 go 编译器并没有优化掉这个变量
			counter++
		}
	}

我们再执行一下`GOSSAFUNC=main.thread1+ go build -gcflags="-N -l" ./demo2.go`看一下`SSA`的`IR`[完整代码](./ssa2.html)

	thread1 func()
	  b1:
	    v1 = InitMem <mem>
	    v2 = SP <uintptr> DEAD
	    v3 = SB <uintptr>
	    v4 = Addr <*bool> {"".flag} v3
	    v7 = Addr <*int> {"".counter} v3
	    v10 = Const64 <int> [1]
	    v12 = Addr <*int> {"".counter} v3
	    Plain -> b2
	  b2: <- b1 b4
	    v5 = Phi <mem> v1 v13
	    v6 = Load <bool> v4 v5  // v6 依赖了 v4、v5，所以 v4、v5 也是存活的，依次类推
	    If v6 -> b3 b5 (likely) // v6 是 ControlValue ，所以 v6 是存活的
	  b3: <- b2
	    v8 = Copy <mem> v5
	    v9 = Load <int> v7 v8
	    v11 = Add64 <int> v9 v10
	    v13 = Store <mem> {int} v12 v11 v8
	    Plain -> b4
	  b4: <- b3
	    Plain -> b2
	  b5: <- b2
	    v14 = Copy <mem> v5
	    Ret v14



因为 [ssa.BlockIf](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/gc/ssa.go#L3031) 和 [ssa.BlockRet](https://github.com/golang/go/blob/release-branch.go1.16/src/cmd/compile/internal/gc/ssa.go#L1647) 都设置了`ControlValue`，所以 `v6`和`v14`会认为是存活的，然后 `v6`和`v14`的`Args`又间接依赖了其他所有的`Value`，所以其他所有`Value`也都标记为存活了（上面这些`Value`输出的时候末尾没有再标记为`Dead`了）。

因为`for`循环中的的代码没有被优化掉，所以这个时候循环里面的逻辑已经正常，运行函数，也能正常结束了（注意这里并不代表没有可见性问题），具体生成的汇编代码如下。


	        JMP     thread1_pc2
	thread1_pc2:
	        CMPB    "".flag(SB), $0 // 判断 flag 是否等于 0
	        JNE     thread1_pc13 // 不等于 0 跳转到 thread1_pc13
	        JMP     thread1_pc24 // 等于 0 跳转到 thread1_pc24
	thread1_pc13:
	        INCQ    "".counter(SB)  // counter++ （内存操作）
	        JMP     thread1_pc22 // 跳转到 thread1_pc22
	thread1_pc22:
	        JMP     thread1_pc2
	thread1_pc24:
	        RET    


再来看下**函数调用**场景，我在函数中加了个 [空函数调用](https://godbolt.org/z/oE75sT6jo) 测试了下一下（注意要加`-l`禁止内联），结果也一样。`for`循环的`body`代码没有再被优化掉。

	func thread1() {
		for {
			emptyFunc()
			counter++
		}
	}
	
	func emptyFunc() {}

	thread1_pc27:
	        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	        PCDATA  $1, $0
	        NOP
	        CALL    "".emptyFunc(SB)
	        INCQ    "".counter(SB)
	        JMP     thread1_pc27

### 3.3 总结

基于上面分析，我们可以知道在满足以下两个条件时，`Go`编译器会才会优化掉`for`循环的`body`中相关的`Value`

1. 循环永远不会结束，或者说没有控制语句。
2. `body`中没有函数调用。



## 四、进一步思考

上面，我们分析了`gc`代码，知道了编译器是怎么一步步把`for`循环中`body`的代码优化掉的。但是我们并不知道编译器基于什么的考量来做这个优化的。查阅了下`Go`编译器的各种资料，也没有找到支撑相关优化算法的资料。

`gc`这个研究走进死胡同了，这个时候，我把眼光转向了`GCC`，文章最开始说了，我用 [GCC](https://godbolt.org/z/foME4ahYh) 和 [LLVM](https://godbolt.org/z/PdYz6zo9v) 测试相关代码的时候，有一样的优化效果。而且`GCC`的优化参数，比`Go`编译器丰富很多（`Go`编译器相关的只有一个`-N`参数，`GCC`可以自己指定各种[优化算法](https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html)）。


### 4.1 GCC 优化分析

`C`测试代码如下：
		
	void thread_test1 (void *ptr);
	int counter = 1;
	
	int main()
	{
	    pthread_t thread1;
	    int res = pthread_create(&thread1, NULL, (void *)&thread_test1, NULL);
	    if (res != 0) {
	        printf("pthread_create failed\n");
	        return 0;
	    }
	    
	    while (counter<2) {}
	    printf("finish, counter = %d\n",counter);
	}
	
	void thread_test1( void *ptr) {
	    printf("thread_test1 start\n");
	    while (1) {
	        counter++;
	    }
	}


我们先不加优化标记（或者用`-O0`）看下 [生成的编译代码](https://godbolt.org/z/W383ocKGe) 如下：

	thread_test1:
	        pushq   %rbp
	        movq    %rsp, %rbp
	        subq    $16, %rsp
	        movq    %rdi, -8(%rbp)
	        movl    $.LC2, %edi
	        call    puts
	.L7:
	        movl    counter(%rip), %eax // 读取内存中的值给 eax 寄存器
	        addl    $1, %eax // eax 寄存器的值 +1
	        movl    %eax, counter(%rip) // 把寄存器的值回写到内存
	        jmp     .L7  // 跳转到 .L7


我们看到，在不指定优化级别的情况下，`GCC`是不会优化`for`循环中的代码的。 如果加上了`-O1`优化级别的话，`for`循环直接变成一个无限循环的`jmp`指令，`thread_test1`[汇编代码](https://godbolt.org/z/foME4ahYh) 会变成如下：

	thread_test1:
	        subq    $8, %rsp
	        movl    $.LC0, %edi
	        call    puts
	.L2:
	        jmp     .L2


加个`flag`控制变量测试再测试一下，[代码如下](https://godbolt.org/z/8P3Ea8rhv)：

	int flag = 1;
	void thread_test1( void *ptr) {
	    printf("thread_test1 start\n");
	    while (flag) {
	        counter++;
	    }
	}
	
	thread_test1:
        subq    $8, %rsp
        movl    $.LC0, %edi
        call    puts
        cmpl    $0, flag(%rip) // 判断 flag == 0
        je      .L1  // 等于0跳转到.L1
	.L3: 
        jmp     .L3 // 不等于 0 .L3 无限循环 
    .L1:  // 函数返回
        addq    $8, %rsp
        ret



我们看到`GCC`还是比`Go Compiler`聪明一点，知道`flag`变量一直`1`，所以只执行了一次比较（`cmpl`）然后依然是一个`jmp`的死循环。


我们把`flag` 换成`counter`再试下，[新生成的代码如下](https://godbolt.org/z/WhYxhPYr9)：
	
	void thread_test1( void *ptr) {
	    while (counter) {
	        counter++;
	    }
	}
	
	thread_test1:
	        subq    $8, %rsp
	        movl    $.LC0, %edi
	        call    puts
	        movl    counter(%rip), %eax // 读取 counter 内存值给 eax
	        testl   %eax, %eax // 判断 eax 是否为 0
	        je      .L1 // eax = 0 跳转到 .L1 函数结束
	.L3:
	        addl    $1, %eax // eax = eax+1
	        jne     .L3 // eax 不等于 0，就跳转到 .L3，一直到 EAX 溢出，才跳出循环
	        movl    $0, counter(%rip) // 设置 counter = 0
	.L1:
	        addq    $8, %rsp
	        ret
	// eax 是 32 位寄存器，CPU 主频按 2GHZ 算，每个时钟信号周期为0.5纳秒
	// 不考虑流水线和指令拆分，eax 寄存器溢出只要 4294967296*2*0.5ns= 4.2s


这一次优化结果比较有趣，从内存读取`counter`的值以后，直接赋值给了`eax`寄存器，然后`for`循环里面不停的对`eax`做加`1`操作，然后在循环之后（`counter`溢出）的代码直接设置`counter=0`（编译器认为只有`counter`为`0`的时候`for`循环才会结束）。

这里我们发现了一个比较关键的优化，在没有指定优化级别的时候，每次循环的时候，都会从内存中读出`counter`赋值给`eax`，然后对`eax`加`1`，最后把`eax`值回写到内存`counter`地址。但是在`-O1`优化级别下，如上代码所示，编译器把读取`counter`赋值给`eax`的操作提到循环外层来了，循环里面只是简单的对`eax`做了加`1`操作。这个优化也符合我们上面说的“**编译器会尽可能将程序执行中最频繁的变量分配到寄存器中**”。


上面在一个循环内，不停的对寄存器做加`1`操作，这个算是无意义的操作，编译器没有优化掉我比较意外。

	.L3:
	        addl    $1, %eax 
	        jne     .L3 
	        movl    $0, counter(%rip) 

所以，我用`-O2`优化级别测试了下，看下了[输出代码](https://godbolt.org/z/jPPPace56)如下：

	thread_test1:
	        subq    $8, %rsp
	        movl    $.LC0, %edi
	        call    puts
	        movl    counter(%rip), %eax // 读取 counter 内存值给 eax
	        testl   %eax, %eax // 判断 eax == 0
	        je      .L1 // eax = 0 的话跳转到 .L1 （函数结束）
	        movl    $0, counter(%rip) // eax != 0, 执行当前代码，设置 counter = 0 ， 然后继续执行 .L1 （函数结束）
	.L1:
	        addq    $8, %rsp
	        ret

上面，我们看到整个循环直接被优化掉了（对程序结果没有改变），这个也是符合预期的。



### 4.2 问题本质


再回来 [看这个代码](https://godbolt.org/z/1P3E19dch)，在`-O1`的级别下生成代码如下：

	void thread_test1( void *ptr) {
	    while (1) {
	        counter++;
	    }
	}

	thread_test1:
	.L2:
	        jmp     .L2

我们知道，在没有优化的场景下（`-O0`），每次循环的都会从内存读取`counter`值给`eax`，然后对`eax+1`，最后把`eax`值回写到内存。

（以下只是个人猜测）因为`counter`是在循环里面`频繁使用的变量`，`-O1`的场景下，编译器会把`counter`操作，优化到寄存器，优化以后的伪代码如下：

	thread_test1:
	 movl    counter(%rip), %eax // 读取 counter 内存值给 eax
	.L2:
		     addl    $1, %eax
	        jmp     .L2

编译器进一步优化的时候比如做`窺孔优化`的时候，发现循环中做`eax+1`操作是没有意义的（因为没有读取`eax`的地方），所以最终就只优化成一条` jmp .L2`指令了。



## 五、扩展阅读

问题一：编译下面这段代码，为什么`go build -gcflags "-N" demo.go` 和 `go build demo.go`程序运行的结果不一样

	type Foo struct{}
	
	func main() {
		fooA()
		fooB()
	}
	
	func fooA() {
		f1 := &Foo{}
		f2 := &Foo{}
	
		fmt.Println("f1 = ", f1, " f2 = ", f2)
		//println("f1 = ", f1, " f2 = ", f2)
		println(f1 == f2)
	
	}
	
	func fooB() {
		f1 := &Foo{}
		f2 := &Foo{}
	
		fmt.Println(f1 == f2)
	}
	
	
问题二：为什么下面`a`、`b` 两个值不相等

	package main
	
	import "fmt"
	
	const s = "123456789"
	
	func main() {
		var a byte = (1 << len(s)) / 128 // 编译器计算
		var b byte = (1 << len(s[:])) / 128 // 运行时计算
	
		fmt.Println(a, b)
	}


## 六、参考资料

[《编译原理》 - 龙书](https://book.douban.com/subject/3296317/)

[《现代编译原理》-  虎书](https://book.douban.com/subject/1806974/)

[《高级编译器设计与实现》 - 鲸书](https://book.douban.com/subject/1400374/)

[《编译原理实战》](https://time.geekbang.org/column/intro/100052801?tab=intro)

[Introduction to the Go compiler](https://github.com/golang/go/blob/master/src/cmd/compile/README.md)

[Go compiler: SSA optimization rules description language](https://quasilyte.dev/blog/post/go_ssa_rules/)

[Introduction to the Go compiler's SSA backend](https://github.com/golang/go/blob/master/src/cmd/compile/internal/ssa/README.md)

[How to read GoLang static single-assignment (SSA)](https://sitano.github.io/2018/03/18/howto-read-gossa/)
