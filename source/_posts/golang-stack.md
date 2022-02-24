---
title: 深入理解 Golang Stack
tags:
  - Backend
  - Golang
categories:
  - Golang
date: 2021.08.01 01:50:20
updated: 2023.10.09 11:50:20
---


<br/>


<!--
<img alt="cover" src="https://upload-images.jianshu.io/upload_images/12321605-b6543138cca8bb9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240">
-->


## 基础知识

### Linux 进程地址空间布局

我们知道`CPU`有实模式和保护模式，系统刚刚启动的时候是运行在实模式下，然后经过一系列初始化工作以后，`Linux`会把`CPU`的实模式改为保护模式（具体就是修改`CPU`的`CR0寄存器`相关标记位），在保护模式下，`CPU`访问的地址都是虚拟地址(逻辑地址)。`Linux` 为了每个进程维护了一个单独的虚拟地址空间，虚拟地址空间又分为“`用户空间`”和“`内核空间`”。 虚拟地址空间更多相关可以看[Linux内核虚拟地址空间](/2021/07/25/linux-mem/)这篇文章。

![Linux-Memory-X86-64.jpg](https://upload-images.jianshu.io/upload_images/12321605-2cdf1bedff166c2f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br/>

### Golang 栈内存在虚拟地址空间哪个区域

`Golang` 的内存管理是用的 `TCMalloc`（`Thread-Caching Malloc`）算法, 简单点说就是 `Golang` 是使用 `mmap` 函数去操作系统申请一大块内存，然后把内存按照 `8Byte~32KB` `67`个 `size` 类型的 `mspan`，每个 `mspan`按照它自身的属性 `Size Class` 的大小分割成若干个 `object`[（每个span默认是8K）](https://github.com/golang/go/blob/go1.16.6/src/runtime/sizeclasses.go)，因为分需要 `gc` 的 `mspan` 和不需要 `gc` 的 `mspan`（`Golang`的`stack`），所以一共有`134`种类型。

**上面说了 Golang 内存申请是用的 mmap，mmap申请的内存都在虚拟地址空间的“Memory Mapping Segment”，所有Golang 所有的内存使用（包括栈）都是在“Memory Mapping Segment”，并不是在传统应用地址空间划的栈区。**

写`Demo`个代码验证一下

	func main() {
		a := 8
		println("a address :  ", &a)
		time.Sleep(time.Hour)
	}
	
	// 这里加“-m”来check没有内存逃逸
	[root@n227-005-021 GoTest ]$ go build -gcflags "-N -l -m"  stack3.go 
	[root@n227-005-021 GoTest ]$ ./stack3
	a address :   0xc000070f68

我们可以看到变量 a 的地址是`0xc000070f68 `，是一个很小的地址，而虚拟地址空间的堆区，都是在最上面，地址一般都很大（比如 `00007ffd8fe4d000`这种），所以我们可以变量 `a` 肯定不是在堆区。我们可以进一步用 `pmap` 看进程的内存区域验证一下。
	
	[root@n227-005-021 fanlv ]$ pmap  162277
	162277:   ./stack3
	0000000000400000    400K r-x-- stack3
	0000000000464000    448K r---- stack3
	00000000004d4000     20K rw--- stack3
	00000000004d9000    200K rw---   [ anon ]
	000000c000000000  65536K rw---   [ anon ]
	00007f13d92f3000  36292K rw---   [ anon ]
	00007f13db664000 263680K -----   [ anon ]
	00007f13eb7e4000      4K rw---   [ anon ]
	00007f13eb7e5000 293564K -----   [ anon ]
	00007f13fd694000      4K rw---   [ anon ]
	00007f13fd695000  36692K -----   [ anon ]
	00007f13ffa6a000      4K rw---   [ anon ]
	00007f13ffa6b000   4580K -----   [ anon ]
	00007f13ffee4000      4K rw---   [ anon ]
	00007f13ffee5000    508K -----   [ anon ]
	00007f13fff64000    384K rw---   [ anon ]
	00007ffd8fe4d000    132K rw---   [ stack ]
	00007ffd8ff82000     12K r----   [ anon ]
	00007ffd8ff85000      8K r-x--   [ anon ]
	 total           702472K

`pmap` 我们看到`000000c000000000 `属于 `mmap` 映射的一个匿名的`page([anon])`，符合预期。


### 常用寄存器

| 寄存器 | 64位名称 | 32位名称 | 16位名称 | 用途                                                                                    |
|-----|-------|-------|-------|---------------------------------------------------------------------------------------|
| AX  | rax   | eax   | ax    | 函数返回值一般都存这个里面                                                                         |
| BX  | rbx   | ebx   | bx    | 基址(Base)寄存器，常做内存数据的指针, 或者说常以它为基址来访问内存.                                                |
| CX  | rcx   | ecx   | cx    | 计数器(Counter)寄存器，常做字符串和循环操作中的计数器, 或者用于保存函数调用第4个参数                                                      |
| DX  | rdx   | edx   | dx    | 数据(Data)寄存器，常用于乘、除法和 I/O 指针，或者用于保存函数调用第3个参数                                                       |
| SI  | rsi   | esi   | si    | 来源索引(Source Index)寄存器，常做内存数据指针和源字符串指针，或者用于保存函数调用第2个参数                                              |
| DI  | rdi   | edi   | di    | 目的索引(Destination Index)寄存器，或者用于保存函数调用第1个参数	，常做内存数据指针和目的字符串指针                                          |
| SP  | rsp   | esp   | sp    | 堆栈指针(Stack Point)寄存器，只做堆栈的栈顶指针; 不能用于算术运算与数据传送                                         |
| BP  | rbp   | ebp   | bp    | 基址指针(Base Point)寄存器，只做堆栈指针, 可以访问堆栈内任意地址, 经常用于中转 ESP 中的数据, 也常以它为基址来访问堆栈; 不能用于算术运算与数据传送 |
| IP  | rip   | eip   | ip    | 指令指针(Instruction Pointer)寄存器，总是指向下一条指令的地址; 所有已执行的指令都被它指向过.                            |

还有 R8 ~ R15 8个寄存器这里就不详细列出来了， R8用于保存函数调用5个参数，R9用于保存函数调用6个参数


虽然在`x86-64`架构下，增加了很多通用寄存器，使得调用惯例(`calling convention`)变为函数传参可以部分（最多6个）**使用寄存器直接传递**，但是在`Golang`中，**编译器强制规定函数的传参全部都用栈传递**，不使用寄存器传参。`go 1.17`以后好像已经开始可以用寄存器传参。

`Golang`不使用寄存器传参，应该还是为了使生成的伪汇编方便跨平台。这里扩展一个优化点，为了减少函数调用的开销小，可以尽量让函数内联。

内联的条件：1. 函数语法解析完，`token`数不超过 80. 2. `Interface` 的方法调用，不能内联。

### 什么是栈帧？

栈帧，也就是`stack frame`，其本质就是一种栈，只是这种栈专门用于保存**函数调用过程**中的各种信息（参数，返回地址，本地变量等）。栈帧有栈顶和栈底之分，其中栈顶的地址最低，栈底的地址最高用，`BP`（栈指针）指向栈底，`SP`(栈指针)指向栈顶的。

具体实现，我们看一个`Demo`

	#include <stdio.h>
	#include <stdlib.h>
	int add(int x, int y)
	{
	    return x + y;
	}
	int main()
	{
	    int p = add(2, 6);
	    printf("add:%d\n", p);
	    return 0;
	}


`clang add.c -o add.o` // 编译，Linux可以用 gcc add.c -o add.o
`otool -tvV add.o`  //导出汇编代码 Linux可以用 objdump -d -M at -S add.o
		
	add.o:
	(__TEXT,__text) section
	_add:
	0000000100003f20	pushq	%rbp
	0000000100003f21	movq	%rsp, %rbp
	0000000100003f24	movl	%edi, -0x4(%rbp)
	0000000100003f27	movl	%esi, -0x8(%rbp)
	0000000100003f2a	movl	-0x4(%rbp), %eax
	0000000100003f2d	addl	-0x8(%rbp), %eax
	0000000100003f30	popq	%rbp
	0000000100003f31	retq
	0000000100003f32	nopw	%cs:(%rax,%rax)
	0000000100003f3c	nopl	(%rax)
	_main:
	0000000100003f40	pushq	%rbp
	0000000100003f41	movq	%rsp, %rbp
	0000000100003f44	subq	$0x10, %rsp
	0000000100003f48	movl	$0x0, -0x4(%rbp)
	0000000100003f4f	movl	$0x2, %edi
	0000000100003f54	movl	$0x6, %esi
	0000000100003f59	callq	_add
	0000000100003f5e	movl	%eax, -0x8(%rbp)
	0000000100003f61	movl	-0x8(%rbp), %esi
	0000000100003f64	leaq	0x37(%rip), %rdi                ##literal pool for: "add:%d\n"
	0000000100003f6b	movb	$0x0, %al
	0000000100003f6d	callq	0x100003f80                     ##symbol stub for: _printf
	0000000100003f72	xorl	%ecx, %ecx
	0000000100003f74	movl	%eax, -0xc(%rbp)
	0000000100003f77	movl	%ecx, %eax
	0000000100003f79	addq	$0x10, %rsp
	0000000100003f7d	popq	%rbp
	0000000100003f7e	retq
	

我们接着`lldb add.o` `Debug`一下。我们在Add函数最开始的地方`0000000100003f20`打上断点，看下调用`add`过程栈的变化。我们执行` x/8xg $rbp `先打印下`rbp`，由下面可以知`rbp`的地址是`0x7ffeefbff580 `，`rbp`地址指向的内容是`0x00007ffeefbff590`

	(lldb) s
	Process 43672 stopped
	* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step into
	    frame #0: 0x0000000100003f20 add.o`add
	add.o`add:
	->  0x100003f20 <+0>: pushq  %rbp // 把rbp地址压栈
	    0x100003f21 <+1>: movq   %rsp, %rbp // rbp = rsp
	    0x100003f24 <+4>: movl   %edi, -0x4(%rbp) // $(rbp-4) = 2
	    0x100003f27 <+7>: movl   %esi, -0x8(%rbp) // $(rbp-8) = 6
	Target 0: (add.o) stopped.
	(lldb) x/8xg $rbp
	0x7ffeefbff580: 0x00007ffeefbff590 0x00007fff203fbf3d
	0x7ffeefbff590: 0x0000000000000000 0x0000000000000001
	
执行完`movl   %esi, -0x8(%rbp) `我们再`x/8xg $rbp `看下rbp地址，发现`rbp`地址变成了`0x7ffeefbff560 ` `rbp`指向了`0x00007ffeefbff580`
	
	(lldb) s
	Process 43672 stopped
	* thread #1, queue = 'com.apple.main-thread', stop reason = instruction step into
	    frame #0: 0x0000000100003f2a add.o`add + 10
	add.o`add:
	->  0x100003f2a <+10>: movl   -0x4(%rbp), %eax
	    0x100003f2d <+13>: addl   -0x8(%rbp), %eax
	    0x100003f30 <+16>: popq   %rbp
	    0x100003f31 <+17>: retq
	Target 0: (add.o) stopped.
	(lldb)  x/8xg $rbp
	0x7ffeefbff560: 0x00007ffeefbff580 0x0000000100003f5e
	0x7ffeefbff570: 0x00007ffeefbff590 0x0000000000011025
	
我们在看下 `$rbp-8`，发现 `$rbp-8`的位置数据存的是6 `$rbp-4`位置存的数据是`2` ，`$rbp`数据存的是main函数的`rbp`地址，`$rbp+8`的地址是 `add` 函数返回以后需要执行的代码地址`0x00003f5e `
	
	(lldb)  x/8xw $rbp-8
	0x7ffeefbff558: 0x00000006 0x00000002 0xefbff580 0x00007ffe
	0x7ffeefbff568: 0x00003f5e 0x00000001 0xefbff590 0x00007ffe

栈的图大致如下

![image.png](https://upload-images.jianshu.io/upload_images/12321605-2e4149f286f5ceb0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个栈帧的图里面我们主要关注几点。

1. 栈是由高地址像低地址扩张，比如`main`函数里面执行`subq $0x10, %rsp` 就是申请16个字节的栈空间，执行完`main`函数再调用`addq	$0x10, %rsp` 表示释放之前申请的`16`个字节栈空间。
2. 每次函数调用都会有一定的栈空间的开销，（老的`rbp`压栈、函数的返回地址压栈）
3. 函数执行完以后。会继续执行`return address`指向的代码。

**这里要理解栈帧是向下扩展的很重要。下面golang的stack扩容缩容判断的时候会用到**

### 什么是内存逃逸？

逃逸分析是一种确定指针动态范围的方法。简单来说就是分析在程序的哪些地方可以访问到该指针。编译器会根据变量是否被外部引用来决定是否逃逸：

1. 如果函数外部没有引用，则优先放到栈中；
2. 如果函数外部存在引用，则必定放到堆中；

**注意：go 在编译阶段确立逃逸，并不是在运行时。**

#### 逃逸场景（什么情况才分配到堆中）

1. 指针逃逸，`Go`可以返回局部变量指针，这其实是一个典型的变量逃逸案例，示例代码如下：

		func StudentRegister(name string, age int) *Student {
		    s := new(Student) //局部变量s逃逸到堆
		    s.Name = name
		    s.Age = age
		    return s
		}
		
2. 栈空间不足逃逸（空间开辟过大），下面代码`Slice()`函数中分配了一个`1000`个长度的切片，是否逃逸取决于栈空间是否足够大。如下：

		func Slice() { 
		    s := make([]int, 1000, 1000) // s 会分配在堆上
		
		    for index, _ := range s {
		        s[index] = index
		    }
		}

3. 动态类型逃逸（不确定长度大小）。很多函数参数为`interface`类型，比如`fmt.Println(a …interface{})`，编译期间很难确定其参数的具体类型，也能产生逃逸。

		func main() {
		    s := "Escape"
		    fmt.Println(s)
		}

4. 闭包引用对象逃逸

		func Fibonacci() func() int {
		    a, b := 0, 1
		    return func() int {
		        a, b = b, a+b
		        return a
		    }
		}


可以使用`go build -gcflags=-m`查看逃逸分析日志

## Golang 栈

### Golang 栈大小变更历史

`Go` 语言使用用户态线程 `Goroutine` 作为执行上下文，它的额外开销和默认栈大小都比线程小很多，然而 `Goroutine` 的栈内存空间和栈结构也在经过很多次变化：

[第一版提交sys·newproc 栈默认是4K](https://github.com/golang/go/commit/af58f17af936f8d88ccfed96b7a0e9953b4e6010)、[malg commit](https://github.com/golang/go/commit/a67258f3801b6aa218c8c2563f0a743b944e5946) （早期`Golang`的`runtime`代码是用c写的）


[第二次修改 4K -> 8K](https://github.com/golang/go/commit/408238e20bb794d91199c892c68a0989fc924d65)，主要是提高部分`encode`和`decode`的性能

	// 2013-10-03 go1.2rc2
	Significant runtime reductions:
	
	          amd64  386
	GoParse    -14%  -1%
	GobDecode  -12% -20%
	GobEncode  -64%  -1%
	JSONDecode  -9%  -4%
	JSONEncode -15%  -5%
	Template   -17% -14%

	Benchmark graphs at
	http://swtch.com/~rsc/gostackamd64.html
	http://swtch.com/~rsc/gostack386.html


[第三次修改 8K -> 4K](https://github.com/golang/go/commit/1665b006a57099d7bdf5c9f1277784d36b7168d9)

	// 2014-02-27 go1.3beta1
	runtime: grow stack by copying
	
	On stack overflow, if all frames on the stack are
	copyable, we copy the frames to a new stack twice
	as large as the old one.  During GC, if a G is using
	less than 1/4 of its stack, copy the stack to a stack
	half its size.


[第四次修改 4K -> 8K](https://github.com/golang/go/commit/6aee29648fce3af20507787035ae22d06d75d39b)

	// 2014-05-20 go1.3beta2
	runtime: switch default stack size back to 8kB
	
	The move from 4kB to 8kB in Go 1.2 was to eliminate many stack split hot spots.
	
	The move back to 4kB was predicated on copying stacks eliminating
	the potential for hot spots.
	
	Unfortunately, the fact that stacks do not copy 100% of the time means
	that hot spots can still happen under the right conditions, and the slowdown
	is worse now than it was in Go 1.2. There is a real program in issue 8030 that
	sees about a 30x slowdown: it has a reflect call near the top of the stack
	which inhibits any stack copying on that segment.
	
	Go back to 8kB until stack copying can be used 100% of the time.
	
	Fixes issue 8030.


[最后、将最小栈内存从8K降低到了2KB](https://github.com/golang/go/commit/6c934238c93f8f60775409f1ab410ce9c9ea2357) ，主要是为了节省内存空间使用

	// 2014-09-17 go1.4beta1
	runtime: change minimum stack size to 2K.
	
	It will be 8K on windows because it needs 4K for the OS.
	Similarly, plan9 will be 4K.
	
	On linux/amd64, reduces size of 100,000 goroutines
	from ~819MB to ~245MB.
	
	Update issue 7514




### 分段栈

分段栈是 `Go` 语言在 `v1.2` 版本之前的实现，所有 `Goroutine` 在栈扩容的时候都会调用[runtime·newstack:go1.2](https://github.com/golang/go/blob/go1.2/src/pkg/runtime/stack.c#L196) 分配的内存为[StackMin + StackSystem](https://github.com/golang/go/blob/go1.2/src/pkg/runtime/stack.h#L79) 表示，在 `v1.2` 版本中为`StackMin` `8KB`：


	// Called from runtime·newstackcall or from runtime·morestack when a new
	// stack segment is needed.  Allocate a new stack big enough for
	// m->moreframesize bytes, copy m->moreargsize bytes to the new frame,
	// and then act as though runtime·lessstack called the function at
	// m->morepc.
	void
	runtime·newstack(void)
	{
	   ...........
		// gp->status is usually Grunning, but it could be Gsyscall if a stack split
		// happens during a function call inside entersyscall.
		gp = m->curg;
		oldstatus = gp->status;
	
		framesize = m->moreframesize;
		argsize = m->moreargsize;
		gp->status = Gwaiting;
		gp->waitreason = "stack split";
		newstackcall = framesize==1;
		if(newstackcall)
			framesize = 0;
	   
	   
		if(newstackcall && m->morebuf.sp - sizeof(Stktop) - argsize - 32 > gp->stackguard) {
		   .........
		} else {
			// 这里计算栈的空间大小
			// allocate new segment.
			framesize += argsize;
			framesize += StackExtra;	// room for more functions, Stktop.
			if(framesize < StackMin)
				framesize = StackMin; // 栈最小8K
			framesize += StackSystem; // StackSystem  Window-64 是4K，plan9 是9，其他平台是0
			gp->stacksize += framesize;
			if(gp->stacksize > runtime·maxstacksize) { // maxstacksize x64是 1G，x32是250m
				runtime·printf("runtime: goroutine stack exceeds %D-byte limit\n", (uint64)runtime·maxstacksize);
				runtime·throw("stack overflow");
			}
			stk = runtime·stackalloc(framesize);
			top = (Stktop*)(stk+framesize-sizeof(*top));
			free = framesize;
		}
	}	
	
	..............
	
	
	void*
	runtime·stackalloc(uint32 n)
	{
		uint32 pos;
		void *v;
	
		if(g != m->g0)
			runtime·throw("stackalloc not on scheduler stack");
	
		if(n == FixedStack || m->mallocing || m->gcing) {
			if(n != FixedStack) {
				runtime·printf("stackalloc: in malloc, size=%d want %d\n", FixedStack, n);
				runtime·throw("stackalloc");
			}
			if(m->stackcachecnt == 0)
				stackcacherefill();
			pos = m->stackcachepos;
			pos = (pos - 1) % StackCacheSize;
			v = m->stackcache[pos];
			m->stackcachepos = pos;
			m->stackcachecnt--;
			m->stackinuse++;
			return v;
		}
		
		// https://github.com/golang/go/blob/go1.2/src/pkg/runtime/malloc.goc#L34
		// 这里调用 runtime·mallocgc 去申请内存，指定内存不需要GC
		return runtime·mallocgc(n, 0, FlagNoProfiling|FlagNoGC|FlagNoZero|FlagNoInvokeGC);
	}
	
	
	
如果通过该方法申请的内存大小为固定的 `8KB` 或者满足其他的条件，运行时会在全局的栈缓存链表中找到空闲的内存块并作为新 `Goroutine` 的栈空间返回；在其余情况下，栈内存空间会从堆上申请一块合适的内存。

当 `Goroutine` 调用的函数层级或者局部变量需要的越来越多时，运行时会调用 [runtime.morestack:go1.2](https://github.com/golang/go/blob/go1.2/src/pkg/runtime/asm_amd64.s#L195) 和 [runtime.newstack:go1.2](https://github.com/golang/go/blob/go1.2/src/pkg/runtime/stack.c#L196) 创建一个新的栈空间，这些栈空间虽然不连续，但是当前 `Goroutine` 的多个栈空间会以链表的形式串联起来，运行时会通过指针找到连续的栈片段：


一旦 `Goroutine` 申请的栈空间不在被需要，运行时会调用 [runtime.lessstack:go1.2](https://github.com/golang/go/blob/go1.2/src/pkg/runtime/asm_amd64.s#L370) 和 [runtime.oldstack:go1.2](https://github.com/golang/go/blob/go1.2/src/pkg/runtime/stack.c#L132) 释放不再使用的内存空间。

分段栈机制虽然能够按需为当前 `Goroutine` 分配内存并且及时减少内存的占用，但是它也存在两个比较大的问题：

1. 如果当前 `Goroutine` 的栈几乎充满，那么任意的函数调用都会触发栈扩容，当函数返回后又会触发栈的收缩，如果在一个循环中调用函数，栈的分配和释放就会造成巨大的额外开销，**这被称为热分裂问题（Hot split）**；
2. 一旦 `Goroutine` 使用的内存越过了分段栈的扩缩容阈值，运行时会触发栈的扩容和缩容，带来额外的工作量；



### 连续栈

#### 什么是连续栈

连续栈可以解决分段栈中存在的两个问题，其核心原理是每当程序的栈空间不足时，初始化一片更大的栈空间并将原栈中的所有值都迁移到新栈中，新的局部变量或者函数调用就有充足的内存空间。使用连续栈机制时，栈空间不足导致的扩容会经历以下几个步骤：

1. 在内存空间中分配更大的栈内存空间；
2. 将旧栈中的所有内容复制到新栈中；
3. 将指向旧栈对应变量的指针重新指向新栈；
4. 销毁并回收旧栈的内存空间；

在扩容的过程中，最重要的是调整指针的第三步，这一步能够保证指向栈的指针的正确性，因为栈中的所有变量内存都会发生变化，所以原本指向栈中变量的指针也需要调整。我们在前面提到过经过逃逸分析的 `Go` 语言程序的遵循以下不变性 —— 指向栈对象的指针不能存在于堆中，所以指向栈中变量的指针只能在栈上，我们只需要调整栈中的所有变量就可以保证内存的安全了。


#### NOSPLIT的自动检测

我们在编写函数时，编译出汇编代码会发现，在一些函数的执行代码中，编译器很智能的加上了`NOSPLIT`标记。这个标记可以禁用栈溢出检测`prolog`，即该函数运行不会导致栈分裂，由于不需要再照常执行栈溢出检测，所以会提升一些函数性能。这是如何做到的

	// cmd/internal/obj/s390x/objz.go
	
	if p.Mark&LEAF != 0 && autosize < objabi.StackSmall {
		// A leaf function with a small stack can be marked
		// NOSPLIT, avoiding a stack check.
		p.From.Sym.Set(obj.AttrNoSplit, true)
	}

当函数处于调用链的叶子节点，且栈帧小于`StackSmall`字节时，则自动标记为`NOSPLIT`。 `x86`架构处理与之类似

自动标记为`NOSPLIT`的函数，链接器就会知道该函数最多还会使用`StackLimit`字节空间，不需要栈分裂。

备注：用户也可以使用`//go:nosplit`强制指定`NOSPLIT`属性，但如果函数实际真的溢出了，则会在编译期就报错`nosplit stack overflow`

	$ GOOS=linux GOARCH=amd64 go build -gcflags="-N -l" main.go
	#command-line-arguments
	main.add: nosplit stack overflow
		744	assumed on entry to main.add (nosplit)
		-79264	after main.add (nosplit) uses 80008


<br/>

#### Goroutine的创建

程序中执行 `go func(){} `创建`Goroutine`的时候，runtime会执行`newproc1`，这里会先去`_p_.gFree`列表拿，如果没有空闲的g就会调用`malg` 去new一个。

	// https://github.com/golang/go/blob/go1.16.6/src/runtime/proc.go#L4065

	func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
	   ..............
	   
		newg := gfget(_p_) // 先去P的free list拿 没有的话就调用malg new一个g
		if newg == nil {
			newg = malg(_StackMin)
			casgstatus(newg, _Gidle, _Gdead)
			allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
		}
	}

`new` 一个新的`Goroutine`的时候，会调用`stackalloc`来申请栈空间。

	//https://github.com/golang/go/blob/go1.16.6/src/runtime/proc.go#L3987
	// Allocate a new g, with a stack big enough for stacksize bytes.
	func malg(stacksize int32) *g {
		newg := new(g)
		if stacksize >= 0 {
			// round2 是求2的指数，比如传 6 返回 8
			// _StackSystem linux是0、plan9 是512 、 Windows-x64 是4k
			stacksize = round2(_StackSystem + stacksize)
			systemstack(func() {//切换到 G0 为 newg 初始化栈内存
				newg.stack = stackalloc(uint32(stacksize))
			})
			// 设置 stackguard0 ，用来判断是否要进行栈扩容
			newg.stackguard0 = newg.stack.lo + _StackGuard
			newg.stackguard1 = ^uintptr(0)
			// Clear the bottom word of the stack. We record g
			// there on gsignal stack during VDSO on ARM and ARM64.
			*(*uintptr)(unsafe.Pointer(newg.stack.lo)) = 0
		}
		return newg
	}
	
每一个`Goroutine`的`g->stackguard`0都被设置为指向`stack.lo `+ `StackGuard`的位置。所以每一个函数在真正执行前都会将`SP`和`stackguard0`进行比较。

<br/>

#### 栈的初始化
	
	
	// Number of orders that get caching. Order 0 is FixedStack
	// and each successive order is twice as large.
	// We want to cache 2KB, 4KB, 8KB, and 16KB stacks. Larger stacks
	// will be allocated directly.
	// Since FixedStack is different on different systems, we
	// must vary NumStackOrders to keep the same maximum cached size.
	//   OS               | FixedStack | NumStackOrders
	//   -----------------+------------+---------------
	//   linux/darwin/bsd | 2KB        | 4
	//   windows/32       | 4KB        | 3
	//   windows/64       | 8KB        | 2
	//   plan9            | 4KB        | 3
	_NumStackOrders = 4 - sys.PtrSize/4*sys.GoosWindows - 1*sys.GoosPlan9

	
	// 全局的栈缓存，分配 32KB以下内存
	var stackpool [_NumStackOrders]struct {
		item stackpoolItem
		_    [cpu.CacheLinePadSize - unsafe.Sizeof(stackpoolItem{})%cpu.CacheLinePadSize]byte // CacheLine对齐，防止Flase Sharding的问题
	}
	
	//go:notinheap
	type stackpoolItem struct {
		mu   mutex
		span mSpanList 
	}
	
	// 全局的栈缓存，分配 32KB 以上内存 
	// heapAddrBits 48 , pageShift 13
	var stackLarge struct {
		lock mutex
		free [heapAddrBits - pageShift]mSpanList // free lists by log_2(s.npages)
	}
	
	func stackinit() {
		if _StackCacheSize&_PageMask != 0 {
			throw("cache size must be a multiple of page size")
		}
		for i := range stackpool {
			stackpool[i].item.span.init()
			lockInit(&stackpool[i].item.mu, lockRankStackpool)
		}
		for i := range stackLarge.free {
			stackLarge.free[i].init()
			lockInit(&stackLarge.lock, lockRankStackLarge)
		}
	}

在执行栈初始化的时候会初始化两个全局变量 `stackpool` 和 `stackLarge`。`stackpool` 可以分配小于 `32KB `的内存，`stackLarge` 用来分配大于 `32KB` 的栈空间。


<br/>

#### 栈内存分配

	// https://github.com/golang/go/blob/go1.16.6/src/runtime/stack.go#L327

	// Per-P, per order stack segment cache size.
	_StackCacheSize = 32 * 1024
	// stackalloc allocates an n byte stack.
	//
	// stackalloc must run on the system stack because it uses per-P
	// resources and must not split the stack.
	//
	//go:systemstack
	func stackalloc(n uint32) stack {
		// Stackalloc must be called on scheduler stack, so that we
		// never try to grow the stack during the code that stackalloc runs.
		// Doing so would cause a deadlock (issue 1547).
		thisg := getg() // 必须是g0
		if thisg != thisg.m.g0 {
			throw("stackalloc not on scheduler stack")
		}
		if n&(n-1) != 0 {
			throw("stack size not a power of 2")
		}
		if stackDebug >= 1 {
			print("stackalloc ", n, "\n")
		}
	
		if debug.efence != 0 || stackFromSystem != 0 {
			n = uint32(alignUp(uintptr(n), physPageSize))
			v := sysAlloc(uintptr(n), &memstats.stacks_sys)
			if v == nil {
				throw("out of memory (stackalloc)")
			}
			return stack{uintptr(v), uintptr(v) + uintptr(n)}
		}
	
		// Small stacks are allocated with a fixed-size free-list allocator.
		// If we need a stack of a bigger size, we fall back on allocating
		// a dedicated span.
		var v unsafe.Pointer
		// _FixedStack: 2K、_NumStackOrders：4 、_StackCacheSize = 32 * 1024
		if n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {// 小于32K
			order := uint8(0)
			n2 := n
			// 大于 2048 ,那么 for 循环 将 n2 除 2,直到 n 小于等于 2048
			for n2 > _FixedStack {
				order++
				n2 >>= 1
			}
			var x gclinkptr
			//preemptoff != "", 在 GC 的时候会进行设置,表示如果在 GC 那么从 stackpool 分配
			// thisg.m.p = 0 会在系统调用和 改变 P 的个数的时候调用,如果发生,那么也从 stackpool 分配
			if stackNoCache != 0 || thisg.m.p == 0 || thisg.m.preemptoff != "" {
				// thisg.m.p == 0 can happen in the guts of exitsyscall
				// or procresize. Just get a stack from the global pool.
				// Also don't touch stackcache during gc
				// as it's flushed concurrently.
				lock(&stackpool[order].item.mu)
				x = stackpoolalloc(order)// 从 stackpool 分配

				unlock(&stackpool[order].item.mu)
			} else {
			
				// 从 P 的 mcache 分配内存
				c := thisg.m.p.ptr().mcache
				x = c.stackcache[order].list
				if x.ptr() == nil {
					// 从堆上申请一片内存空间填充到stackcache中
					stackcacherefill(c, order)
					x = c.stackcache[order].list
				}
				c.stackcache[order].list = x.ptr().next // 移除链表的头节点
				c.stackcache[order].size -= uintptr(n)
			}
			v = unsafe.Pointer(x) // 获取到分配的span内存块
		} else {
			// 申请的内存空间过大，从 runtime.stackLarge 中检查是否有剩余的空间
			var s *mspan
			// 计算需要分配多少个 span 页， 8KB 为一页
			npage := uintptr(n) >> _PageShift
			log2npage := stacklog2(npage)
	
			// Try to get a stack from the large stack cache.
			lock(&stackLarge.lock)
			// 如果 stackLarge 对应的链表不为空
			if !stackLarge.free[log2npage].isEmpty() {
				//获取链表的头节点，并将其从链表中移除
				s = stackLarge.free[log2npage].first
				stackLarge.free[log2npage].remove(s)
			}
			unlock(&stackLarge.lock)
	
			lockWithRankMayAcquire(&mheap_.lock, lockRankMheap)
			//这里是stackLarge为空的情况
			if s == nil {
				// 从堆上申请新的内存 span
				// Allocate a new stack from the heap.
				s = mheap_.allocManual(npage, spanAllocStack)
				if s == nil {
					throw("out of memory")
				}
				// OpenBSD 6.4+ 系统需要做额外处理
				osStackAlloc(s)
				s.elemsize = uintptr(n)
			}
			v = unsafe.Pointer(s.base())
		}
	
		..........
		
		return stack{uintptr(v), uintptr(v) + uintptr(n)}
	}
	



**小于32KB的栈内存分配**

小栈指大小为 `2K/4K/8K/16K` 的栈，在分配的时候，会根据大小计算不同的 `order` 值，如果栈大小是 `2K`，那么 `order` 就是 `0，4K` 对应 `order` 就是 `1`，以此类推。这样一方面可以减少不同 `Goroutine` 获取不同栈大小的锁冲突，另一方面可以预先缓存对应大小的 `span` ，以便快速获取。

`thisg.m.p == 0`可能发生在系统调用 `exitsyscall` 或改变 `P` 的个数 `procresize` 时，`thisg.m.preemptoff != ""`会发生在 `GC` 的时候。也就是说在发生在系统调用 `exitsyscall` 或改变 P 的个数在变动，亦或是在 `GC` 的时候，会从 `stackpool` 分配栈空间，否则从 `mcache` 中获取。

在 `stackpoolalloc` 函数中会去找 `stackpool` 对应 `order` 下标的 `span` 链表的头节点，如果不为空，那么直接将头节点的属性 `manualFreeList` 指向的节点从链表中移除，并返回；

如果 `list.first`为空，那么调用 `mheap_`的 `allocManual` 函数从堆中分配 `mspan`

从 `allocManual` 函数会分配 `32KB` 大小的内存块，分配好新的 `span` 之后会根据 `elemsize` 大小将 `32KB` 内存进行切割，然后通过单向链表串起来并将最后一块内存地址赋值给 `manualFreeList` 。


	func stackpoolalloc(order uint8) gclinkptr {
		list := &stackpool[order].item.span
		s := list.first
		lockWithRankMayAcquire(&mheap_.lock, lockRankMheap)
		if s == nil {
			// no free stacks. Allocate another span worth.
			// 从堆上分配 mspan
	        // _StackCacheSize = 32 * 1024
			s = mheap_.allocManual(_StackCacheSize>>_PageShift, &memstats.stacks_inuse)
			if s == nil {
				throw("out of memory")
			}
			// 刚分配的 span 里面分配对象个数肯定为 0
			if s.allocCount != 0 {
				throw("bad allocCount")
			}
			if s.manualFreeList.ptr() != nil {
				throw("bad manualFreeList")
			}
			//OpenBSD 6.4+ 系统需要做额外处理
			osStackAlloc(s)
			// Linux 中 _FixedStack = 2048
			s.elemsize = _FixedStack << order
			//_StackCacheSize =  32 * 1024
			// 这里是将 32KB 大小的内存块分成了elemsize大小块，用单向链表进行连接
			// 最后 s.manualFreeList 指向的是这块内存的尾部
			for i := uintptr(0); i < _StackCacheSize; i += s.elemsize {
				x := gclinkptr(s.base() + i)
				x.ptr().next = s.manualFreeList
				s.manualFreeList = x
			}
			// 插入到 list 链表头部
			list.insert(s)
		}
		x := s.manualFreeList
		// 代表被分配完毕
		if x.ptr() == nil {
			throw("span has no free stacks")
		}
		// 将 manualFreeList 往后移动一个单位
		s.manualFreeList = x.ptr().next
		// 统计被分配的内存块
		s.allocCount++
		// 因为分配的时候第一个内存块是 nil
		// 所以当指针为nil 的时候代表被分配完毕
		// 那么需要将该对象从 list 的头节点移除
		if s.manualFreeList.ptr() == nil {
			// all stacks in s are allocated.
			list.remove(s)
		}
		return x
	}

如果 `mcache` 对应的 `stackcache` 获取不到，那么调用 `stackcacherefill` 从堆上申请一片内存空间填充到`stackcache`中。
	
`stackcacherefill` 函数会调用 `stackpoolalloc` 从 `stackpool` 中获取一半的空间组装成 `list` 链表，然后放入到 `stackcache` 数组中。

	
	func stackcacherefill(c *mcache, order uint8) { 
		var list gclinkptr
		var size uintptr
		lock(&stackpool[order].item.mu)
		//_StackCacheSize = 32 * 1024
		// 将 stackpool 分配的内存组成一个单向链表 list
		for size < _StackCacheSize/2 {
			x := stackpoolalloc(order)
			x.ptr().next = list
			list = x
			// _FixedStack = 2048
			size += _FixedStack << order
		}
		unlock(&stackpool[order].item.mu)
		c.stackcache[order].list = list
		c.stackcache[order].size = size
	}

**大于等于32KB的栈内存分配**

对于大栈内存分配，运行时会查看 `stackLarge` 中是否有剩余的空间，如果不存在剩余空间，它也会调用 `mheap_.allocManual` 从堆上申请新的内存。

<br/>

#### 栈的扩容

`Go` 语言中的执行栈由 `runtime.stack` 表示，该结构体中只包含两个字段，分别表示栈的顶部和栈的底部，每个栈结构体都表示范围为 `[lo, hi) `的内存空间：

	type stack struct {
		lo uintptr
		hi uintptr
	}

栈的结构虽然非常简单，但是想要理解 `Goroutine` 栈的实现原理，还是需要我们从编译期间和运行时两个阶段入手：

1. 编译器会在编译阶段会通过 `cmd/internal/obj/x86.stacksplit` 在调用函数前插入 `runtime.morestack` 或者 `runtime.morestack_noctxt` 函数；
2. 运行时在创建新的 `Goroutine` 时会在 `runtime.malg` 中调用 `runtime.stackalloc` 申请新的栈内存，并在编译器插入的 `runtime.morestack` 中检查栈空间是否充足；


需要注意的是，`Go` 语言的编译器不会为所有的函数插入 `runtime.morestack`，它只会在必要时插入指令以减少运行时的额外开销，编译指令 `nosplit` 可以跳过栈溢出的检查，虽然这能降低一些开销，不过固定大小的栈也存在溢出的风险。本节将分别分析栈的初始化、创建 `Goroutine` 时栈的分配、编译器和运行时协作完成的栈扩容以及当栈空间利用率不足时的缩容过程。

在 `Goroutine` 中会通过 `stackguard0` 来判断是否要进行栈增长：

* `stackguard0`：`stack.lo` + `StackGuard`, 用于`stack overlow`的检测；
* `StackGuard`：保护区大小，常量`Linux`上为 [928 字节](https://github.com/golang/go/blob/go1.16.6/src/cmd/internal/objabi/stack.go#L21)；
* `StackSmall`：常量大小为 `128` 字节，用于小函数调用的优化；
* `StackBig`：常量大小为 `4096` 字节；

![image.png](https://upload-images.jianshu.io/upload_images/12321605-8dd04e53b3137274.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![image.png](https://upload-images.jianshu.io/upload_images/12321605-59ca6d23d703f42f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

需要注意的是，由于栈是由高地址向低地址增长的，所以对比的时候，都是小于才执行扩容。

当执行栈扩容时，会在内存空间中分配更大的栈内存空间，然后将旧栈中的所有内容复制到新栈中，并修改指向旧栈对应变量的指针重新指向新栈，最后销毁并回收旧栈的内存空间，从而实现栈的动态扩容。

**具体代码实现**

[runtime.morestack_noctxt](https://github.com/golang/go/blob/go1.16.6/src/runtime/asm_amd64.s#L416) 是用汇编实现的，它会调用到 `runtime·morestack`，下面我们看看它的实现：
	
	TEXT runtime·morestack(SB),NOSPLIT,$0-0
		// Cannot grow scheduler stack (m->g0).
		// 无法增长调度器的栈(m->g0)
		get_tls(CX)
		MOVQ	g(CX), BX
		MOVQ	g_m(BX), BX
		MOVQ	m_g0(BX), SI
		CMPQ	g(CX), SI
		JNE	3(PC)
		CALL	runtime·badmorestackg0(SB)
		CALL	runtime·abort(SB)
		// 省略signal stack、morebuf和sched的处理
		...
		// Call newstack on m->g0's stack.
		// 在 m->g0 栈上调用 newstack.
		MOVQ	m_g0(BX), BX
		MOVQ	BX, g(CX)
		MOVQ	(g_sched+gobuf_sp)(BX), SP
		CALL	runtime·newstack(SB)
		CALL	runtime·abort(SB)	// 如果 newstack 返回则崩溃 crash if newstack returns
		RET


[runtime·morestack](https://github.com/golang/go/blob/go1.16.6/src/runtime/asm_amd64.s#L416) 做完校验和赋值操作后会切换到 `G0` 调用 [runtime·newstack](https://github.com/golang/go/blob/go1.16.6/src/runtime/stack.go#L938)来完成扩容的操作。
	
	func newstack() {
		thisg := getg() 
	
		gp := thisg.m.curg
		 
		// 初始化寄存器相关变量
		morebuf := thisg.m.morebuf
		thisg.m.morebuf.pc = 0
		thisg.m.morebuf.lr = 0
		thisg.m.morebuf.sp = 0
		thisg.m.morebuf.g = 0
		...
		// 校验是否被抢占
		preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt
	 
		// 如果被抢占
		if preempt {
			// 校验是否可以安全的被抢占
			// 如果 M 上有锁
			// 如果正在进行内存分配
			// 如果明确禁止抢占
			// 如果 P 的状态不是 running
			// 那么就不执行抢占了
			if !canPreemptM(thisg.m) {
				// 到这里表示不能被抢占？
				// Let the goroutine keep running for now.
				// gp->preempt is set, so it will be preempted next time.
				gp.stackguard0 = gp.stack.lo + _StackGuard
				// 触发调度器的调度
				gogo(&gp.sched) // never return
			}
		}
	
		if gp.stack.lo == 0 {
			throw("missing stack in newstack")
		}
		// 寄存器 sp
		sp := gp.sched.sp
		if sys.ArchFamily == sys.AMD64 || sys.ArchFamily == sys.I386 || sys.ArchFamily == sys.WASM {
			// The call to morestack cost a word.
			sp -= sys.PtrSize
		} 
		...
		if preempt {
			//需要收缩栈
			if gp.preemptShrink { 
				gp.preemptShrink = false
				shrinkstack(gp)
			}
			// 被 runtime.suspendG 函数挂起
			if gp.preemptStop {
				// 被动让出当前处理器的控制权
				preemptPark(gp) // never returns
			}
	 
			//主动让出当前处理器的控制权
			gopreempt_m(gp) // never return
		}
	 
		// 计算新的栈空间是原来的两倍
		oldsize := gp.stack.hi - gp.stack.lo
		newsize := oldsize * 2 
		... 
		//将 Goroutine 切换至 _Gcopystack 状态
		casgstatus(gp, _Grunning, _Gcopystack)
	 
		//开始栈拷贝
		copystack(gp, newsize) 
		casgstatus(gp, _Gcopystack, _Grunning)
		gogo(&gp.sched)
	}

<br/>

#### 栈拷贝

在开始执行栈拷贝之前会先计算新栈的大小是原来的两倍，然后将 `Goroutine` 状态切换至 `_Gcopystack` 状态。
	
	func copystack(gp *g, newsize uintptr) { 
		old := gp.stack 
		// 当前已使用的栈空间大小
		used := old.hi - gp.sched.sp
	 
		//分配新的栈空间
		new := stackalloc(uint32(newsize))
		...
	 
		// 计算调整的幅度
		var adjinfo adjustinfo
		adjinfo.old = old
		// 新栈和旧栈的幅度来控制指针的移动
		adjinfo.delta = new.hi - old.hi
	 
		// 调整 sudogs, 必要时与 channel 操作同步
		ncopy := used
		if !gp.activeStackChans {
			...
			adjustsudogs(gp, &adjinfo)
		} else {
			// 到这里代表有被阻塞的 G 在当前 G 的channel 中，所以要防止并发操作，需要获取 channel 的锁
			 
			// 在所有 sudog 中找到地址最大的指针
			adjinfo.sghi = findsghi(gp, old) 
			// 对所有 sudog 关联的 channel 上锁，然后调整指针，并且复制 sudog 指向的部分旧栈的数据到新的栈上
			ncopy -= syncadjustsudogs(gp, used, &adjinfo)
		} 
		// 将源栈中的整片内存拷贝到新的栈中
		memmove(unsafe.Pointer(new.hi-ncopy), unsafe.Pointer(old.hi-ncopy), ncopy)
		// 继续调整栈中 txt、defer、panic 位置的指针
		adjustctxt(gp, &adjinfo)
		adjustdefers(gp, &adjinfo)
		adjustpanics(gp, &adjinfo)
		if adjinfo.sghi != 0 {
			adjinfo.sghi += adjinfo.delta
		} 
		// 将 G 上的栈引用切换成新栈
		gp.stack = new
		gp.stackguard0 = new.lo + _StackGuard // NOTE: might clobber a preempt request
		gp.sched.sp = new.hi - used
		gp.stktopsp += adjinfo.delta
	 
		// 在新栈重调整指针
		gentraceback(^uintptr(0), ^uintptr(0), 0, gp, 0, nil, 0x7fffffff, adjustframe, noescape(unsafe.Pointer(&adjinfo)), 0)
	 
		if stackPoisonCopy != 0 {
			fillstack(old, 0xfc)
		}
		//释放原始栈的内存空间
		stackfree(old)
	}

 
1. `copystack` 首先会计算一下使用栈空间大小，那么在进行栈复制的时候只需要复制已使用的空间就好了；
2. 然后调用 `stackalloc` 函数从堆上分配一片内存块；
3. 然后对比新旧栈的 `hi` 的值计算出两块内存之间的差值 `delta`，这个 `delta` 会在调用 `adjustsudogs`、`adjustctxt` 等函数的时候判断旧栈的内存指针位置，然后加上 `delta` 然后就获取到了新栈的指针位置，这样就可以将指针也调整到新栈了；
4. 调用 `memmove` 将源栈中的整片内存拷贝到新的栈中；
5. 然后继续调用调整指针的函数继续调整栈中 `txt`、`defer`、`panic` 位置的指针；
6. 接下来将 `G` 上的栈引用切换成新栈；
7. 最后调用 `stackfree` 释放原始栈的内存空间；

![image.png](https://upload-images.jianshu.io/upload_images/12321605-82ea33f8a92282b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br/>

#### 栈的收缩

栈的收缩发生在 `GC` 时对栈进行扫描的阶段：

	func scanstack(gp *g, gcw *gcWork) {
		... 
		// 进行栈收缩
		shrinkstack(gp)
		...
	}

	func shrinkstack(gp *g) {
		...
		oldsize := gp.stack.hi - gp.stack.lo
		newsize := oldsize / 2 
		// 当收缩后的大小小于最小的栈的大小时，不再进行收缩
		if newsize < _FixedStack {
			return
		}
		avail := gp.stack.hi - gp.stack.lo
		// 计算当前正在使用的栈数量，如果 gp 使用的当前栈少于四分之一，则对栈进行收缩
		// 当前使用的栈包括到 SP 的所有内容以及栈保护空间，以确保有 nosplit 功能的空间
		if used := gp.stack.hi - gp.sched.sp + _StackLimit; used >= avail/4 {
			return
		}
		// 将旧栈拷贝到新收缩后的栈上
		copystack(gp, newsize)
	}
	
新栈的大小会缩小至原来的一半，如果小于 `_FixedStack` （`2KB`）那么不再进行收缩。除此之外还会计算一下当前栈的使用情况是否不足 `1/4 `，如果使用超过 `1/4 `那么也不会进行收缩。

最后判断确定要进行收缩则调用 `copystack` 函数进行栈拷贝的逻辑。

<br/>

#### 关于Golang栈思考产生几个有趣的实验

<br/>

**Demo0：Golang栈会自动扩容，是不是永远不会栈溢出？**

	func f(i int) int {
		if i == 0 || i == 1 {
			return i
		}
		return f(i - 1)
	}
	func main() {
		println(f(100000000))
	}

执行这个函数，程序会报“`stack overflow`”的`excepttion`，具体错误日志如下：

	➜  GoTest git:(master) ✗ go run stack.go
	runtime: goroutine stack exceeds 1000000000-byte limit
	runtime: sp=0xc0200e0390 stack=[0xc0200e0000, 0xc0400e0000]
	fatal error: stack overflow
	
	runtime stack:
	runtime.throw(0x1074923, 0xe)
		/Users/fanlv/.g/go/src/runtime/panic.go:1117 +0x72
	runtime.newstack()
		/Users/fanlv/.g/go/src/runtime/stack.go:1069 +0x7ed
	runtime.morestack()
		/Users/fanlv/.g/go/src/runtime/asm_amd64.s:458 +0x8f

这个错误是在`newstack`函数中抛出来的

	if newsize > maxstacksize || newsize > maxstackceiling {
		if maxstacksize < maxstackceiling {
			print("runtime: goroutine stack exceeds ", maxstacksize, "-byte limit\n")
		} else {
			print("runtime: goroutine stack exceeds ", maxstackceiling, "-byte limit\n")
		}
		print("runtime: sp=", hex(sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n")
		throw("stack overflow")
	}

有上面可知，当新的栈大小超过了`maxstacksize`就会抛出"`stack overflow`"的异常。`maxstacksize`是在`runtime.main`中设置的。`64位` 系统下栈的最大值`1GB`、`32位`系统是`250MB`

	// Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
	// Using decimal instead of binary GB and MB because
	// they look nicer in the stack overflow failure message.
	if sys.PtrSize == 8 {
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}

<br/>

**Demo1：验证栈扩容以后地址变了，对我们是不是写代码的时候会有影响？**

	func f(i int) int {
		if i == 0 || i == 1 {
			return i
		}
		return f(i - 1)
	}
	
	func main() {	
		println("demo3: ",demo3())
	}
	
	func demo1() {
		var xDemo1 uint64
		xAddr := uintptr(unsafe.Pointer(&xDemo1))
		println("demo1 before stack copy xDemo1 : ", xDemo1, " xDemo1 pointer: ", &xDemo1)
	
		f(10000000)
	
		xPointer := (*uint64)(unsafe.Pointer(xAddr))
		atomic.AddUint64(xPointer, 1)
		println("demo1 after stack copy xDemo1 : ", xDemo1, " xDemo1 pointer:", &xDemo1)
	}


编译执行上面的代码，输入结果如下，由下面内容可以知道栈扩容过程中，栈上变量的地址的确会发生改变。

	➜  GoTest git:(master) ✗ go build -gcflags "-N -l -m"  stack.go
	➜  GoTest git:(master) ✗ ./stack
	demo1 before stack copy xDemo1 :  0  xDemo1 pointer:  0xc000044740
	demo1 after stack copy xDemo1 :  0  xDemo1 pointer: 0xc04015ff40
	
<br/>

**Demo2：变量逃逸到堆上的情况**

上面是变量没有逃逸的情况，我们构造个变量逃逸到堆上的`case`（其实用`fmt.Println`打印`x变量`就导致`x逃逸`，笔者最开始使用`fmt.Println`打印`x变量`发现栈扩容的情况下，`x地址`也一直不会变），代码如下：
	
	// f 和 main 函数代码如demo1
	func demo2() {
		var xDemo2 uint64
		println("demo2 before stack copy xDemo2 : ", xDemo2, " xDemo2 pointer: ", &xDemo2)
		f(10000000)
		atomic.AddUint64(&xDemo2, 1)
		println("demo2 after stack copy xDemo2 : ", xDemo2, " xDemo2 pointer:", &xDemo2)
	}


`demo2`中，我们直接用取地址的方式去调用`atomic.AddUint64(&xDemo2, 1)`，看编译日志可以知道`xDemo2`这个变量逃逸到堆上了。逃逸 到堆上的变量地址是不会变的。这样也符合预期。

	➜  GoTest git:(master) ✗ go build -gcflags "-N -l -m"  stack.go
	#command-line-arguments
	./stack.go:36:6: moved to heap: xDemo2
	➜  GoTest git:(master) ✗ ./stack
	demo2 before stack copy xDemo2 :  0  xDemo2 pointer:  0xc0000180b8
	demo2 after stack copy xDemo2 :  1  xDemo2 pointer: 0xc0000180b8
	
<br/>

**Demo3：go run方式去运行代码**

`demo3` 中，我们直接返回变量的地址，这个时候我们知道，变量肯定已经逃逸到堆上。我们用`go run`的方式去跑下面代码，地址会变吗？

	func demo3() *uint64 {
		var xDemo3 uint64 = 8
		println("demo3 before stack copy xDemo3 : ", xDemo3, " xDemo3 pointer: ", &xDemo3)
		f(1000)
		println("demo3 after stack copy xDemo3 : ", xDemo3, " xDemo3 pointer:", &xDemo3)
		return &xDemo3
	}


输入日志如下，我们发现逃逸到堆上的变量地址也变了，这个是为什么？

	➜  GoTest git:(master) ✗ go run stack.go
	demo3 before stack copy xDemo3 :  0  xDemo3 pointer:  0xc000044770
	demo3 after stack copy xDemo3 :  0  xDemo3 pointer: 0xc000117f70

其实执行`go run`的时候，它就是想编译在运行，执行`go build`的时候没有“`禁止内联`”，所以`demo3`函数就发生了内联，所以变量也不会逃逸到堆上了。我们可以`go build -gcflags "-m"  stack.go`看下，输出日志如下。`can inline demo3 `，这个时候`demo3`已经内联了。

	➜  GoTest git:(master) ✗ go build -gcflags "-m"  stack.go
	#command-line-arguments
	./stack.go:37:6: can inline demo3
	./stack.go:14:7: inlining call to demo3
	./stack.go:38:6: moved to heap: xDemo3

虽然逃逸分析日志还有`moved to heap: xDemo3 `，但是其实这个时候已经没有逃逸了，具体我们可以导出汇编代码 `otool -tvV stack >> ~/Desktop/stack.s` 看下。可以看到main里面已经没有`demo3`的调用了。`xDemo3`这边变量还是在栈上`$0x8, 0x10(%rsp)`

		
	_main.main:
	000000000105c920	movq	%gs:0x30, %rcx
	000000000105c929	cmpq	0x10(%rcx), %rsp
	000000000105c92d	jbe	0x105ca21
	000000000105c933	subq	$0x20, %rsp
	000000000105c937	movq	%rbp, 0x18(%rsp)
	000000000105c93c	leaq	0x18(%rsp), %rbp
	000000000105c941	movq	$0x8, 0x10(%rsp)
	000000000105c94a	callq	_runtime.printlock
	000000000105c94f	leaq	0x18e96(%rip), %rax
	000000000105c956	movq	%rax, (%rsp)
	000000000105c95a	movq	$0x22, 0x8(%rsp)
	000000000105c963	callq	_runtime.printstring
	000000000105c968	movq	0x10(%rsp), %rax
	000000000105c96d	movq	%rax, (%rsp)
	000000000105c971	callq	_runtime.printuint
	000000000105c976	leaq	0x16ccd(%rip), %rax
	000000000105c97d	movq	%rax, (%rsp)
	000000000105c981	movq	$0x13, 0x8(%rsp)
	000000000105c98a	callq	_runtime.printstring
	000000000105c98f	leaq	0x10(%rsp), %rax
	000000000105c994	movq	%rax, (%rsp)
	000000000105c998	callq	_runtime.printpointer
	000000000105c99d	nopl	(%rax)
	000000000105c9a0	callq	_runtime.printnl
	000000000105c9a5	callq	_runtime.printunlock
	000000000105c9aa	movq	$0x3e8, (%rsp)  
	000000000105c9b2	callq	_main.f
	000000000105c9b7	callq	_runtime.printlock
	000000000105c9bc	leaq	0x18bf7(%rip), %rax
	000000000105c9c3	movq	%rax, (%rsp)
	000000000105c9c7	movq	$0x21, 0x8(%rsp)
	000000000105c9d0	callq	_runtime.printstring
	000000000105c9d5	movq	0x10(%rsp), %rax
	000000000105c9da	movq	%rax, (%rsp)
	000000000105c9de	nop
	000000000105c9e0	callq	_runtime.printuint
	000000000105c9e5	leaq	0x16b62(%rip), %rax
	000000000105c9ec	movq	%rax, (%rsp)
	000000000105c9f0	movq	$0x12, 0x8(%rsp)
	000000000105c9f9	callq	_runtime.printstring
	000000000105c9fe	leaq	0x10(%rsp), %rax
	000000000105ca03	movq	%rax, (%rsp)
	000000000105ca07	callq	_runtime.printpointer
	000000000105ca0c	callq	_runtime.printnl
	000000000105ca11	callq	_runtime.printunlock
	000000000105ca16	movq	0x18(%rsp), %rbp
	000000000105ca1b	addq	$0x20, %rsp
	000000000105ca1f	nop
	000000000105ca20	retq
	000000000105ca21	callq	_runtime.morestack_noctxt
	000000000105ca26	jmp	_main.main

## 参考

[https://www.cnblogs.com/luozhiyun/p/14619585.html](https://www.cnblogs.com/luozhiyun/p/14619585.html)

[http://www.huamo.online/2019/06/25/%E6%B7%B1%E5%85%A5%E7%A0%94%E7%A9%B6goroutine%E6%A0%88/](http://www.huamo.online/2019/06/25/%E6%B7%B1%E5%85%A5%E7%A0%94%E7%A9%B6goroutine%E6%A0%88/)

[https://www.cnblogs.com/shijingxiang/articles/12200355.html
](https://www.cnblogs.com/shijingxiang/articles/12200355.html)

[https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-stack-management/](https://www.cnblogs.com/shijingxiang/articles/12200355.html)