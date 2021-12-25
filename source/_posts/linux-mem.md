---
title: Linux内核虚拟地址空间
tags:
  - Linux
  - System
  - Memory
categories:
  - System
updated: 2023.08.09 11:50:20
date: 2021.07.25 21:16:01
cover: 1
---

<br/>

## x86-32位虚拟地址空间

就我们所知，`Linux`内核一般将处理器的虚拟地址空间划分为两个部分。底部比较大的部分用于用户进程，顶部则专用于内核。虽然（在两个用户进程之间的）上下文切换期间会改变下半部分，**但虚拟地址空间的内核部分总是保持不变**。

<br/>

`Linux`将虚拟地址空间划分为：`0~3G`为用户空间，`3~4G`为内核空间

<img alt="cover" src="https://upload-images.jianshu.io/upload_images/12321605-cb891da74fe7ebaa.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240">

<!-- ![Linux-Memory-X86-32](https://upload-images.jianshu.io/upload_images/12321605-b62889a2b3c2eb38.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240) -->


[点我查看原图](https://raw.githubusercontent.com/fanlv/blog/9f9a531bcf4876759ef216060175dba9702415f2/backup/Content/Foundation/images/Linux%20-%20Memory%20-%20X86-32.jpg)

<br/>

### 用户地址空间

<br/>

#### 保留区 - 0x08048000

位于虚拟地址空间的最低部分，未赋予物理地址。任何对它的引用都是非法的，用于捕捉使用空指针和小整型值指针引用内存的异常情况。

它并不是一个单一的内存区域，而是对地址空间中受到操作系统保护而禁止用户进程访问的地址区域的总称。大多数操作系统中，极小的地址通常都是不允许访问的，如`NULL`。`C语言`将无效指针赋值为0也是出于这种考虑，因为0地址上正常情况下不会存放有效的可访问数据。

因为“历史原因”前面 `128.28125MB` 属于保留空间。

[On Linux, why does the text segment start at 0x08048000? What is stored below that address?](https://www.quora.com/On-Linux-why-does-the-text-segment-start-at-0x08048000-What-is-stored-below-that-address)

[Where are “the kernel stack”, “Frames for C run-time startup functions”, and “Frame for main()” in the memory layout of a program?](https://unix.stackexchange.com/questions/466389/where-are-the-kernel-stack-frames-for-c-run-time-startup-functions-and-fr)

<br/>

#### 代码段(text)

代码段也称正文段或文本段，通常用于存放程序执行代码(即`CPU`执行的机器指令)。一般`C语言`执行语句都编译成机器代码保存在代码段。通常代码段是可共享的，因此频繁执行的程序只需要在内存中拥有一份拷贝即可。代码段通常属于只读，以防止其他程序意外地修改其指令(对该段的写操作将导致段错误)。某些架构也允许代码段为可写，即允许修改程序。

代码段指令根据程序设计流程依次执行，对于顺序指令，只会执行一次(每个进程)；若有反复，则需使用跳转指令；若进行递归，则需要借助栈来实现。

代码段指令中包括操作码和操作对象(或对象地址引用)。若操作对象是立即数(具体数值)，将直接包含在代码中；若是局部数据，将在栈区分配空间，然后引用该数据地址；若位于`BSS`段和数据段，同样引用该数据地址。

代码段最容易受优化措施影响。

<br/>

#### 数据段(Data)

数据段通常用于存放程序中已初始化且初值不为`0`的全局变量和静态局部变量。数据段属于静态内存分配(静态存储区)，可读可写。

数据段保存在目标文件中(在嵌入式系统里一般固化在镜像文件中)，其内容由程序初始化。例如，对于全局变量`int gVar = 10`，必须在目标文件数据段中保存`10`这个数据，然后在程序加载时复制到相应的内存。

数据段与`BSS段`的区别如下： 

* `BSS段`不占用物理文件尺寸，但占用内存空间；数据段占用物理文件，也占用内存空间。对于大型数组如`int ar0[10000] = {1, 2, 3, ...}`和`int ar1[10000]`，`ar1`放在`BSS段`，只记录共有`10000*4`个字节需要初始化为`0`，而不是像`ar0`那样记录每个数据`1、2、3...`，此时`BSS`为目标文件所节省的磁盘空间相当可观。

* 当程序读取数据段的数据时，系统会出发缺页故障，从而分配相应的物理内存；当程序读取`BSS段`的数据时，内核会将其转到一个全零页面，不会发生缺页故障，也不会为其分配相应的物理内存。

运行时数据段和`BSS段`的整个区段通常称为数据区。某些资料中“数据段”指代数据段 + `BSS段` + 堆。

<br/>

#### BSS段

`BSS(Block Started by Symbol)`段中通常存放程序中以下符号：

* 未初始化的全局变量和静态局部变量
* 初始值为`0`的全局变量和静态局部变量(依赖于编译器实现)
* 未定义且初值不为`0`的符号(该初值即`common block`的大小)

`C语言`中，未显式初始化的静态分配变量被初始化为`0`(算术类型)或空指针(指针类型)。由于程序加载时，`BSS`会被操作系统清零，所以未赋初值或初值为`0`的全局变量都在`BSS`中。`BSS段`仅为未初始化的静态分配变量预留位置，在目标文件中并不占据空间，这样可减少目标文件体积。但程序运行时需为变量分配内存空间，故目标文件必须记录所有未初始化的静态分配变量大小总和(通过`start_bss`和`end_bss`地址写入机器代码)。当加载器(`loader`)加载程序时，将为`BSS段`分配的内存初始化为0。在嵌入式软件中，进入`main()`函数之前`BSS段`被`C`运行时系统映射到初始化为全零的内存(效率较高)。

注意，尽管均放置于`BSS段`，但初值为`0`的全局变量是强符号，而未初始化的全局变量是弱符号。若其他地方已定义同名的强符号(初值可能非 `0`)，则弱符号与之链接时不会引起重定义错误，但运行时的初值可能并非期望值(会被强符号覆盖)。因此，定义全局变量时，若只有本文件使用，则尽量使用`static`关键字修饰；否则需要为全局变量定义赋初值(哪怕`0`值)，保证该变量为强符号，以便链接时发现变量名冲突，而不是被未知值覆盖。

某些编译器将未初始化的全局变量保存在`common段`，链接时再将其放入`BSS段`。在编译阶段可通过`-fno-common`选项来禁止将未初始化的全局变量放入`common段`。

此外，由于目标文件不含`BSS段`，故程序烧入`存储器(Flash)`后`BSS段`地址空间内容未知。`U-Boot`启动过程中，将`U-Boot`的`Stage2`代码(通常位于`lib_xxxx/board.c`文件)搬迁(拷贝)到`SDRAM`空间后必须人为添加清零`BSS段`的代码，而不可依赖于`Stage2`代码中变量定义时赋`0`值。


>【扩展阅读】BSS历史
>
>`BSS`(`Block Started by Symbol`，以符号开始的块)一词最初是UA-SAP汇编器(`United Aircraft Symbolic Assembly Program`)中的伪指令，用于为符号预留一块内存空间。该汇编器由美国联合航空公司于`20世纪50`年代中期为`IBM 704`大型机所开发。
>
>后来该词被作为关键字引入到了`IBM 709`和`7090/94`机型上的标准汇编器`FAP(Fortran Assembly Program)`，用于定义符号并且为该符号预留指定字数的未初始化空间块。
>
>在采用段式内存管理的架构中(如`Intel 80x86`系统)，`BSS段`通常指用来存放程序中未初始化全局变量的一块内存区域，该段变量只有名称和大小却没有值。程序开始时由系统初始化清零。
>
>`BSS段`不包含数据，仅维护开始和结束地址，以便内存能在运行时被有效地清零。`BSS`所需的运行时空间由目标文件记录，但`BSS`并不占用目标文件内的实际空间，即`BSS节段`应用程序的二进制映象文件中并不存在。

<br/>

#### 堆(heap)

堆用于存放进程运行时动态分配的内存段，可动态扩张或缩减。堆中内容是匿名的，不能按名字直接访问，只能通过指针间接访问。当进程调用`malloc(C)/new(C++)`等函数分配内存时，新分配的内存动态添加到堆上(扩张)；当调用`free(C)/delete(C++)`等函数释放内存时，被释放的内存从堆中剔除(缩减) 。

分配的堆内存是经过字节对齐的空间，以适合原子操作。堆管理器通过链表管理每个申请的内存，由于堆申请和释放是无序的，最终会产生内存碎片。堆内存一般由应用程序分配释放，回收的内存可供重新使用。若程序员不释放，程序结束时操作系统可能会自动回收。

堆的末端由`break`指针标识，当堆管理器需要更多内存时，可通过系统调用`brk()`和`sbrk()`来移动`break`指针以扩张堆，一般由系统自动调用。

使用堆时经常出现两种问题：

1. 释放或改写仍在使用的内存(“内存破坏”)；
2. 未释放不再使用的内存(“内存泄漏”)。当释放次数少于申请次数时，可能已造成内存泄漏。泄漏的内存往往比忘记释放的数据结构更大，因为所分配的内存通常会圆整为下个大于申请数量的2的幂次(如申请`212B`，会圆整为`256B`)。

注意，堆不同于数据结构中的”堆”，其行为类似链表。


>【扩展阅读】栈和堆的区别
>
>①管理方式：栈由编译器自动管理；堆由程序员控制，使用方便，但易产生内存泄露。
>
>②生长方向：栈向低地址扩展(即”向下生长”)，是连续的内存区域；堆向高地址扩展(即”向上生长”)，是不连续的内存区域。这是由于系统用链表来存储空闲内存地址，自然不连续，而链表从低地址向高地址遍历。
>
>③空间大小：栈顶地址和栈的最大容量由系统预先规定(通常默认`2M`或`10M`)；堆的大小则受限于计算机系统中有效的虚拟内存，`32`位`Linux`系统中堆内存可达`2.9G`空间。
>
>④存储内容：栈在函数调用时，首先压入主调函数中下条指令(函数调用语句的下条可执行语句)的地址，然后是函数实参，然后是被调函数的局部变量。本次调用结束后，局部变量先出栈，然后是参数，最后栈顶指针指向最开始存的指令地址，程序由该点继续运行下条可执行语句。堆通常在头部用一个字节存放其大小，堆用于存储生存期与函数调用无关的数据，具体内容由程序员安排。
>
>⑤分配方式：栈可静态分配或动态分配。静态分配由编译器完成，如局部变量的分配。动态分配由`alloca`函数在栈上申请空间，用完后自动释放。堆只能动态分配且手工释放。
>
>⑥分配效率：栈由计算机底层提供支持：分配专门的寄存器存放栈地址，压栈出栈由专门的指令执行，因此效率较高。堆由函数库提供，机制复杂，效率比栈低得多。`Windows`系统中`VirtualAlloc`可直接在进程地址空间中分配一块内存，快速且灵活。
>
>⑦分配后系统响应：只要栈剩余空间大于所申请空间，系统将为程序提供内存，否则报告异常提示栈溢出。
>
>操作系统为堆维护一个记录空闲内存地址的链表。当系统收到程序的内存分配申请时，会遍历该链表寻找第一个空间大于所申请空间的堆结点，然后将该结点从空闲结点链表中删除，并将该结点空间分配给程序。若无足够大小的空间(可能由于内存碎片太多)，有可能调用系统功能去增加程序数据段的内存空间，以便有机会分到足够大小的内存，然后进行返回。，大多数系统会在该内存空间首地址处记录本次分配的内存大小，供后续的释放函数(如`free/delete`)正确释放本内存空间。
>
> 此外，由于找到的堆结点大小不一定正好等于申请的大小，系统会自动将多余的部分重新放入空闲链表中。
> 
>⑧碎片问题：栈不会存在碎片问题，因为栈是先进后出的队列，内存块弹出栈之前，在其上面的后进的栈内容已弹出。而频繁申请释放操作会造成堆内存空间的不连续，从而造成大量碎片，使程序效率降低。
>
> 可见，堆容易造成内存碎片；由于没有专门的系统支持，效率很低；由于可能引发用户态和内核态切换，内存申请的代价更为昂贵。所以栈在程序中应用最广泛，函数调用也利用栈来完成，调用过程中的参数、返回地址、栈基指针和局部变量等都采用栈的方式存放。所以，建议尽量使用栈，仅在分配大量或大块内存空间时使用堆。
>
> 使用栈和堆时应避免越界发生，否则可能程序崩溃或破坏程序堆、栈结构，产生意想不到的后果。

<br/>

#### 内存映射段(mmap)

此处，内核将硬盘文件的内容直接映射到内存, 任何应用程序都可通过`Linux`的`mmap()`系统调用或`Windows`的`CreateFileMapping()/MapViewOfFile()`请求这种映射。内存映射是一种方便高效的文件`I/O`方式， 因而被用于装载动态共享库。用户也可创建匿名内存映射，该映射没有对应的文件, 可用于存放程序数据。在`Linux`中，若通过`malloc()`请求一大块内存，`C运行库`将创建一个匿名内存映射，而不使用堆内存。”大块” 意味着比阈值 `MMAP_THRESHOLD`还大，缺省为`128KB`，可通过`mallopt()`调整。

**PS:  内存映射端在Linux 2.6.7以前是向上增长的，在2.6.7之后改为向下增长**。

更多可以查看`《深入理解操作系统内核》4.2.1章节`

<br/>

#### 栈(stack)

栈又称堆栈，由编译器自动分配释放，行为类似数据结构中的栈(先进后出)。堆栈主要有三个用途：

* 为函数内部声明的非静态局部变量(`C语言`中称“自动变量”)提供存储空间。
* 记录函数调用过程相关的维护性信息，称为栈帧(`Stack Frame`)或过程活动记录(`Procedure Activation Record`)。它包括函数返回地址，不适合装入寄存器的函数参数及一些寄存器值的保存。除递归调用外，堆栈并非必需。因为编译时可获知局部变量，参数和返回地址所需空间，并将其分配于BSS段。
临时存储区，用于暂存长算术表达式部分计算结果或`alloca()`函数分配的栈内内存。
* 持续地重用栈空间有助于使活跃的栈内存保持在CPU缓存中，从而加速访问。进程中的每个线程都有属于自己的栈。向栈中不断压入数据时，若超出其容量就会耗尽栈对应的内存区域，从而触发一个页错误。此时若栈的大小低于堆栈最大值`RLIMIT_STACK`(通常是8M)，则栈会动态增长，程序继续运行。映射的栈区扩展到所需大小后，不再收缩。

`Linux`中`ulimit -s`命令可查看和设置堆栈最大值，当程序使用的堆栈超过该值时, 发生栈溢出(`Stack Overflow`)，程序收到一个段错误(`Segmentation Fault`)。注意，调高堆栈容量可能会增加内存开销和启动时间。

堆栈既可向下增长(向内存低地址)也可向上增长, 这依赖于具体的实现。本文所述堆栈向下增长。

堆栈的大小在运行时由内核动态调整。

<br/>

### 内核地址空间

<br/>

#### 直接映射区（896M）

所谓的直接映射区，就是这一块空间是连续的，和物理内存是非常简单的映射关系，其实就是虚拟内存地址减去 `3G`，就得到物理内存的位置。

	
	__pa(vaddr) 返回与虚拟地址 vaddr 相关的物理地址；
	__va(paddr) 则计算出对应于物理地址 paddr 的虚拟地址。

	// PAGE_OFFSET => 3G  0x0c0000000
    #define __va(x)      ((void *)((unsigned long)(x)+PAGE_OFFSET))
    #define __pa(x)    __phys_addr((unsigned long)(x))
    #define __phys_addr(x)    __phys_addr_nodebug(x)
    #define __phys_addr_nodebug(x)  ((x) - PAGE_OFFSET)

这 `896M` 还需要仔细分解。在系统启动的时候，物理内存的前 `1M` 已经被占用了，从 `1M` 开始加载内核代码段，然后就是内核的`全局变量`、`BSS` 等，也是 `ELF` 里面涵盖的。这样内核的代码段，全局变量，`BSS` 也就会被映射到 `3G` 后的虚拟地址空间里面。具体的物理内存布局可以查看。


	cat /proc/iomem
	.....
	
	00100000-bffdbfff : System RAM
	  01000000-01a02fff : Kernel code
	  01a03000-021241bf : Kernel data
	  02573000-02611fff : Kernel bss
	  25000000-34ffffff : Crash kernel
	
	.....

在内核运行的过程中，如果碰到系统调用创建进程，会创建 `task_struct` 这样的实例，内核的进程管理代码会将实例创建在 `3G` 至 `3G+896M` 的虚拟空间中，当然也会被放在物理内存里面的前 `896M` 里面，相应的页表也会被创建。

在内核运行的过程中，会涉及内核栈的分配，内核的进程管理的代码会将内核栈创建在 `3G` 至 `3G+896M` 的虚拟空间中，当然也就会被放在物理内存里面的前 `896M` 里面，相应的页表也会被创建。

[进程空间管理](https://time.geekbang.org/column/article/95715)

<br/>

#### 高端内存 - HIGH_MEMORY

`x86-32`下特有的（`x64下没有这个东西`），因为内核虚拟空间只有`1G`无法管理全部的内存空间。

当内核想访问高于`896MB`物理地址内存时，从`0xF8000000 ~ 0xFFFFFFFF`地址空间范围内找一段相应大小空闲的逻辑地址空间，借用一会。借用这段逻辑地址空间，建立映射到想访问的那段物理内存（即填充内核`PTE`页面表），临时用一会，用完后归还。这样别人也可以借用这段地址空间访问其他物理内存，实现了使用有限的地址空间，访问所有所有物理内存。如下图。

从上面的描述，我们可以知道高端内存的最基本思想：借一段地址空间，建立临时地址映射，用完后释放，达到这段地址空间可以循环使用，访问所有物理内存。

看到这里，不禁有人会问：万一有内核进程或模块一直占用某段逻辑地址空间不释放，怎么办？若真的出现的这种情况，则内核的高端内存地址空间越来越紧张，若都被占用不释放，则没有建立映射到物理内存都无法访问了。

[Linux内核高端内存](http://ilinuxkernel.com/?p=1013)

<br/>

#### VMALLOC_OFFSET

系统会在`low memory`和`VMALLOC`区域留8M，防止访问越界。因此假如理论上`vmalloc size`有`300M`，实际可用的也是只有`292M`。

	include/asm-x86/pgtable_32.h 
	#define VMALLOC_OFFSET (8*1024*1024)


这个缺口可用作针对任何内核故障的保护措施。如果访问越界地址（即无意地访问物理上不存在的内存区），则访问失败并生成一个异常，报告该错误。如果`vmalloc`区域紧接着直接映射，那么访问将成功而不会注意到错误。在稳定运行的情况下，肯定不需要这个额外的保护措施，但它对开发尚未成熟的新内核特性是有用的。

`《深入理解Linux内核》3.4`

<br/>

#### VMALLOC

虚拟内存中连续、但物理内存中不连续的内存区，可以在`vmalloc`区域分配。该机制通常用于用户过程，内核自身会试图尽力避免非连续的物理地址。内核通常会成功，因为大部分大的内存块都在启动时分配给内核，那时内存的碎片尚不严重。但在已经运行了很长时间的系统上，在内核需要物理内存时，就可能出现可用空间不连续的情况。此类情况，主要出现在动态加载模块时。


	include/asm-x86/pgtable_32.h 
	#define VMALLOC_START (((unsigned long) high_memory + \ 
	 2*VMALLOC_OFFSET-1) & ~(VMALLOC_OFFSET-1)) 
	#ifdef CONFIG_HIGHMEM 
	#define VMALLOC_END (PKMAP_BASE-2*PAGE_SIZE) 
	#else 
	#define VMALLOC_END (FIXADDR_START-2*PAGE_SIZE) 
	#endif

`vmalloc`区域的起始地址，取决于在直接映射物理内存时，使用了多少虚拟地址空间内存（因此也依赖于上文的`high_memory`变量）。内核还考虑到下述事实，即两个区域之间有至少为`VMALLOC_OFFSET`的一个缺口，而且`vmalloc`区域从可被`VMALLOC_OFFSET`整除的地址开始。

`VMALLOC_START` 到 `VMALLOC_END` 之间称为内核动态映射空间，也即内核想像用户态进程一样 `malloc` 申请内存，在内核里面可以使用 `vmalloc`。假设物理内存里面，`896M` 到 `1.5G` 之间已经被用户态进程占用了，并且映射关系放在了进程的页表中，内核 `vmalloc` 的时候，只能从分配物理内存 `1.5G` 开始，就需要使用这一段的虚拟地址进行映射，映射关系放在专门给内核自己用的页表里面。

使用`vmalloc`的最著名的实例是内核对模块的实现。因为模块可能在任何时候加载，如果模块数据比较多，那么无法保证有足够的连续内存可用，特别是在系统已经运行了比较长时间的情况下。如果能够用小块内存拼接出足够的内存，那么使用`vmalloc`可以规避该问题。

内核中还有大约`400`处地方调用了`vmalloc`，特别是在设备和声音驱动程序中。

因为用于`vmalloc`的内存页总是必须映射在内核地址空间中，因此使用`ZONE_HIGHMEM`内存域的页要优于其他内存域。这使得内核可以节省更宝贵的较低端内存域，而又不会带来额外的坏处。因此，`vmalloc`（连同其他映射函数在`3.5.8节`讨论）是内核出于自身的目的（并非因为用户空间应用程序）使用高端内存页的少数情形之一。

`《深入理解Linux内核》3.4`

[进程空间管理](https://time.geekbang.org/column/article/95715)

<br/>

#### 持久映射

内核专门为此留出一块线性空间，从 `PKMAP_BASE` 到 `FIXADDR_START` ，用于映射高端内存。在 `2.6内核`上，这个地址范围是 `4G-8M` 到 `4G-4M` 之间。这个空间起叫”`内核永久映射空间`”或者”`永久内核映射空间`”。这个空间和其它空间使用同样的页目录表，对于内核来说，就是 `swapper_pg_dir`，对普通进程来说，通过 `CR3 寄存器`指向。通常情况下，这个空间是 `4M` 大小，因此仅仅需要一个页表即可，内核通过来 `pkmap_page_table` 寻找这个页表。通过 `kmap()`，可以把一个 `page` 映射到这个空间来。由于这个空间是 `4M`大小，最多能同时映射 `1024` 个 `page`。因此，对于不使用的的 `page`，及应该时从这个空间释放掉（也就是解除映射关系），通过 `kunmap() `，可以把一个 `page` 对应的线性地址从这个空间释放出来。
 
	 include/asm-x86/highmem.h 
	#define LAST_PKMAP 1024 
	#define PKMAP_BASE ( (FIXADDR_BOOT_START -PAGE_SIZE*(LAST_PKMAP + 1)) & PMD_MASK )
  
如果需要将高端页帧长期映射（作为持久映射）到内核地址空间中，必须使用`kmap`函数。需要映射的页用指向`page`的指针指定，作为该函数的参数。该函数在有必要时创建一个映射（即，如果该页确实是高端页），并返回数据的地址。

如果没有启用高端支持，该函数的任务就比较简单。在这种情况下，所有页都可以直接访问，因此只需要返回页的地址，无需显式创建一个映射。

如果确实存在高端页，情况会比较复杂。类似于`vmalloc`，内核首先必须建立高端页和所映射到的地址之间的关联。还必须在虚拟地址空间中分配一个区域以映射页帧，最后，内核必须记录该虚拟区域的哪些部分在使用中，哪些仍然是空闲的。

`pkmap_count`（在`mm/highmem.c`定义）是一容量为`LAST_PKMAP`的整数数组，其中每个元素都对应于一个持久映射页。它实际上是被映射页的一个使用计数器，语义不太常见。该计数器计算了内核使用该页的次数加`1`。如果计数器值为`2`，则内核中只有一处使用该映射页。计数器值为`5`表示有`4`处使用。一般地说，计数器值为`n`代表内核中有`n-1`处使用该页。

和通常的使用计数器一样，`0`意味着相关的页没有使用。计数器值`1`有特殊语义。这表示该位置关联的页已经映射，但由于`CPU`的`TLB`没有更新而无法使用，此时访问该页，或者失败，或者会访问到一个不正确的地址。

内核利用下列数据结构，来建立物理内存页的page实例与其在虚似内存区中位置之间的关联：

	mm/highmem.c 
	struct page_address_map { 
	 struct page *page; 
	 void *virtual; 
	 struct list_head list; 
	};
	
该结构用于建立`page→virtual`的映射（该结构由此得名）。`page`是一个指向全局`mem_map`数组中的`page`实例的指针，`virtual`指定了该页在内核虚拟地址空间中分配的位置。

刚才描述的`kmap`函数不能用于中断处理程序，因为它可能进入睡眠状态。如果`pkmap`数组中没有空闲位置，该函数会进入睡眠状态，直至情形有所改善。因此内核提供了一个备选的映射函数，其执行是原子的，逻辑上称为`kmap_atomic`。该函数的一个主要优点是它比普通的`kmap`快速。但它不能用于可能进入睡眠的代码。因此，它对于很快就需要一个临时页的简短代码，是非常理想的。`kmap_atomic`的定义在`IA-32`、`PPC`、`Sparc32`上是特定于体系结构的，但这3种实现只有非常细微的差别。其原型是相同的。

`《深入理解Linux内核》3.5.8`

[Linux内核高端内存](http://ilinuxkernel.com/?p=1013)

 <br/>

#### 固定映射/临时映射

`FIXADDR_START` 到 `FIXADDR_TOP`(`0xFFFF F000`) 的空间，称为固定映射区域，主要用于满足特殊需求。

这块空间具有如下特点：
（1）每个 `CPU` 占用一块空间
（2）在每个 `CPU` 占用的那块空间中，又分为多个小空间，每个小空间大小是 `1` 个 `page`，每个小空间用于一个目的，这些目的定义在 `kmap_types.h` 中的 `km_type` 中。

当要进行一次固定映射的时候，需要指定映射的目的，根据映射目的，可以找到对应的小空间，然后把这个空间的地址作为映射地址。这意味着一次固定映射会导致以前的映射被覆盖。通过 `kmap_atomic()` 可实现固定映射。

固定映射是与物理地址空间中的固定页关联的虚拟地址空间项，但具体关联的页帧可以自由选择。它与通过固定公式与物理内存关联的直接映射页相反，虚拟固定映射地址与物理内存位置之间的关联可以自行定义，关联建立后内核总是会注意到的。

最后一个内存段由固定映射占据。这些地址指向物理内存中的随机位置。相对于内核空间起始处的线性映射，在该映射内部的虚拟地址和物理地址之间的关联不是预设的，而可以自由定义，但定义后不能改变。固定映射区域会一直延伸到虚拟地址空间顶端。

	include/asm-x86/fixmap_32.h 
	#define __FIXADDR_TOP 0xfffff000 
	#define FIXADDR_TOP ((unsigned long)__FIXADDR_TOP) 
	#define __FIXADDR_SIZE (__end_of_permanent_fixed_addresses << PAGE_SHIFT) 
	#define FIXADDR_START (FIXADDR_TOP -__FIXADDR_SIZE)


固定映射地址的优点在于，在编译时对此类地址的处理类似于常数，内核一启动即为其分配了物理地址。此类地址的解引用比普通指针要快速。内核会确保在上下文切换期间，对应于固定映射的页表项不会从TLB刷出，因此在访问固定映射的内存时，总是通过TLB高速缓存取得对应的物理地址。

对每个固定映射地址都会创建一个常数，加入到`fixed_addresses`枚举值列表中。


	include/asm-x86/fixmap_32.h 
	enum fixed_addresses { 
	 FIX_HOLE, 
	 FIX_VDSO, 
	 FIX_DBGP_BASE, 
	 FIX_EARLYCON_MEM_BASE, 
	#ifdef CONFIG_X86_LOCAL_APIC 
	 FIX_APIC_BASE, /* 本地CPU APIC信息，在SMP系统上需要 */ 
	#endif 
	... 
	#ifdef CONFIG_HIGHMEM 
	 FIX_KMAP_BEGIN, /* 保留的页表项，用于临时内核映射 */ 
	 FIX_KMAP_END = FIX_KMAP_BEGIN+(KM_TYPE_NR*NR_CPUS)-1, 
	#endif 
	... 
	 FIX_WP_TEST, 
	 __end_of_fixed_addresses 
	};

内核提供了`fix_to_virt`函数，用于计算固定映射常数的虚拟地址。

	include/asm-x86/fixmap_32.h 
	static __always_inline unsigned long fix_to_virt(const unsigned int idx) 
	{ 
	 if (idx >= __end_of_fixed_addresses) 
	 __this_fixmap_does_not_exist(); 
	 return __fix_to_virt(idx); 
	}

编译器优化机制会完全消除if语句，因为该函数定义为内联函数，而且其参数都是常数。这样的优化是有必要的，否则固定映射地址实际上并不优于普通指针。形式上的检查确保了所需的固定映射地址在有效区域中。`__end_of_fixed_adresses`是`fixed_addresses`的最后一个成员，定义了最大的可能数字。如果内核访问的是无效地址，则调用伪函数`__this_fixmap_does_not_exist`（没有定义）。在内核链接时，这会导致错误信息，表明由于存在未定义符号而无法生成映像文件。因此，此种内核故障在编译时即可检测，而不会在运行时出现。

在引用有效的固定映射地址时，if语句中的比较总是会通过。由于比较的两个操作数都是常数，该条件判断语句实际上不会执行，在编译优化的过程中会直接消除。

`__fix_to_virt`定义为宏。由于`fix_to_virt`是内联函数，其实现代码会直接复制到查询固定映射地址的代码处。该宏定义如下：

	include/asm-x86/fixmap_32.h 
	#define __fix_to_virt(x) (FIXADDR_TOP -((x) << PAGE_SHIFT)) 

从顶部开始（不是按照常理从底部开始），内核回退n页，以确定第n个固定映射项的虚拟地址。这个计算同样也只使用了常数，编译器能够在编译时计算结果。根据上文提到的内存划分，地址空间中对应的虚拟地址尚未用于其他用途。固定映射虚拟地址与物理内存页之间的关联是由`set_fixmap(fixmap, page_nr)`和`set_fixmap_nocache`建立的（未讨论后者的实现）。这两个函数只是将页表中的对应项与物理内存中的一页关联起来。不同于`set_fixmap`，`set_fixmap_nocache`在必要情况下，会停用所涉及页帧的硬件高速缓存。


在最后一个区域可以通过 `kmap_atomic` 实现临时内核映射。假设用户态的进程要映射一个文件到内存中，先要映射用户态进程空间的一段虚拟地址到物理内存，然后将文件内容写入这个物理内存供用户态进程访问。给用户态进程分配物理内存页可以通过 `alloc_pages()`，分配完毕后，按说将用户态进程虚拟地址和物理内存的映射关系放在用户态进程的页表中，就完事大吉了。这个时候，用户态进程可以通过用户态的虚拟地址，也即 `0` 至 `3G` 的部分，经过页表映射后访问物理内存，并不需要内核态的虚拟地址里面也划出一块来，映射到这个物理内存页。但是如果要把文件内容写入物理内存，这件事情要内核来干了，这就只好通过 `kmap_atomic` 做一个临时映射，写入物理内存完毕后，再 `kunmap_atomic` 来解映射即可。



`《深入理解Linux内核》3.5.8`

[Linux内核高端内存](http://ilinuxkernel.com/?p=1013)、

[进程空间管理](https://time.geekbang.org/column/article/95715)

<br/>


### 物理内存

<br/>


#### 0~1M

前`4 KiB`是第一个页帧，一般会忽略，因为通常保留给`BIOS`使用。接下来的`640 KiB`原则上是可用的，但也不用于内核加载。其原因是，该区域之后紧邻的区域由系统保留，用于映射各种`ROM`（通常是`系统BIOS`和`显卡ROM`）。不可能向映射`ROM`的区域写入数据。但内核总是会装载到一个连续的内存区中，如果要从`4 KiB`处作为起始位置来装载内核映像，则要求内核必须小于`640 KiB`。

`《深入理解Linux内核》3.4.2`

<br/>

#### ZONE-DMA、ZONE_NORMAL、ZONE_HIGHMEM

在x86架构中内存有三种区域：`ZONE_DMA`，`ZONE_NORMAL`，`ZONE_HIGHMEM`，不同类型的区域适合不同需要。在32位系统中结构中，`1G`(内核空间)/`3G`(用户空间) 地址空间划分时，三种类型的区域如下:

`ZONE_DMA`              内存开始的`16MB`

`ZONE_NORMAL`       `16MB`~`896MB`

`ZONE_HIGHMEM`     `896MB` ~ 结束


* `ZONE-DMA` (`16M`)
	它是低内存的一块区域,这块区域由标准工业架构(`Industry Standard Architecture`)设备使用，适合`DMA`内存。这部分区域大小和CPU架构有关，在`x86架构`中，该部分区域大小 限制为`16MB`。
	
	该区域的物理页面专门供`I/O`设备的`DMA`使用。之所以需要单独管理DMA的物理页面，是因为DMA使用物理地址访问内存，不经过`MMU`，并且需要连续的缓冲区，所以为了能够提供物理上连续的缓冲区，必须从物理地址空间专门划分一段区域用于`DMA`。
	
	`DMA` 技术就是我们在主板上放一块独立的芯片。在进行内存和 `I/O `设备的数据传输的时候，我们不再通过 `CPU` 来控制数据传输，而直接通过 `DMA` 控制器（`DMA Controller`，简称 `DMAC`）。这块芯片，我们可以认为它其实就是一个协处理器（`Co-Processor`）。

* `ZONE_NORMAL`(`16~896M`)

	`ZONE_NORMAL`的范围是`16M~896M`，该区域的物理页面是内核能够直接使用的。属于直接映射区

* `ZONE_HIGHMEM`（`896M~结束`）

	是系统中剩下的可用内存,但因为内核的地址空间有限,这部分内存不直接映射到内核。


[Linux内核高端内存](http://ilinuxkernel.com/?p=1013)

<br/>

#### 临时内存页表、内核镜像

内核启动过程中，存在一个实模式保护模式的切换过程。在`Linux`启动的最初阶段，内核刚刚被装入内存时，分页功能还未启用，此时是直接存取物理地址的（或者说线性地址就等于物理地址）。但初始化完成后，内核也需要有自己的虚拟地址空间（`1个G大小`），该虚拟地址空间的地址映射关系，会被作为模版拷贝到其他进程的内核地址空间中。

临时内核页表只用来映射物理地址的前8M空间内容。目的是允许`CPU`在实模式（直接存取物理地址）和保护模式（根据虚拟地址映射）之间切换的过程中，都能对这前`8M`的地址进行访问。（假如内核使用的全部内存可以存放在`8M`的空间里，因为一个页表可以映射`4M`的地址，所以`8M`的空间需要两个页表，也就是需要两个页目录项。这两张页表我们称为临时内核页表`pg0`和`pg1`。（页表的作用，参见地址映射））


`Linux Kernel` 有自己專屬的 `Page Directory` 及 `Page Table`
在系統初始化時，會先建立 2個 `Page Table` -- 包含 `2048`個 `Page`，共 `8MB` 的記憶體空間
這 `8MB` 是 `Linux` 開機最少需要的記憶體大小，而且保留給 `Kernel` 使用

`Kernel Page Global Directory` 是以變數 `swapper_pg_dir` 表示其資料結構可視為含有 `1024` 個元素的 `pgd_t`型態的陣列實體記憶體位址為 `0x00101000`

`Kernel Page Table` (第 `0`及第 `1`個 Table) 是以變數 `pg0` 及 `pg1` 表示其資料結構可視為含有 `1024`個元素的 `pte_t` 型態的陣列實體記憶體位址分別為 `0x00102000` 及 `0x00103000`

`Linux Page` 初始化的動作定義於 `arch/i386/kernel/head.S`因為開機時僅需要 `8MB`，所以只要初始化 `2`個 `Page Table` 便可，即 `pg0` 及 `pg1`其餘的 `Page Table` 均填入 `0` 的值

又為了讓這 `2`個 `Page Table` 可以被 `Real Mode` 及 `Protect Mode` 所存取`Kernel Page Global Directory` 中的第 `0`、第 `768`個 Entry以及第 `1`、第 `769`個 `Entry`分別會設為相同的 `Page Table` 的實體記憶體位址，如圖所示


初始化完成後，得到以下的結果：

	swapper_pg_dir[0] = swapper_pg_dir[768] = 0x00102007
	swapper_pg_dir[1] = swapper_pg_dir[769] = 0x00103007
	
`pg0` 加上 `pg1` 定址到實體記憶體 `0x00000000` - `0x007FFFFF`，共 `8MB` 的分頁

根據 `pgd_t` 及 `pte_t` 的欄位格式可得知：

1. `bit` `0-11` 為 `0x007`，表示 `Enable` 旗號 `Present`、`Read/Write`、`User/Supervisor`（详见下面#页表描述符章节）
2. `bit` `12-31` 為 `Base Address`

`《深入理解Linux虚拟内存管理》`、

[Linux内存管理总结-系统初始化](https://blog.csdn.net/fullofwindandsnow/article/details/8565512)

[Linux Memory Paging - 3](http://parrotshen.blogspot.com/2008/01/test.html)

[linux启动过程中建立临时页表](https://www.cnblogs.com/4a8a08f09d37b73795649038408b5f33/p/10154324.html)

[linux内存源码分析 - 页表的初始化](https://www.cnblogs.com/tolimit/p/4585803.html)

<br/>

## x86-64位虚拟地址空间

![Linux-Memory-X86-64.jpg](https://upload-images.jianshu.io/upload_images/12321605-2cdf1bedff166c2f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[点我查看原图](https://raw.githubusercontent.com/fanlv/fanlv.github.io/master/Content/Foundation/images/Linux-Memory-X86-64.jpg)

<br/>

### ZONE_DMA、ZONE_DMA32、ZONE_NORMAL

* `ZONE_DMA` 标记适合`DMA`的内存域。该区域的长度依赖于处理器类型。在`IA-32`计算机上，一般的限制是`16 MiB`，这是由古老的ISA设备强加的边界。但更现代的计算机也可能受这一限制的影响。
* `ZONE_DMA32` 标记了使用`32位`地址字可寻址、适合`DMA`的内存域。显然，只有在`64位`系统上，两种`DMA`内存域才有差别。在`32位`计算机上，本内存域是空的，即长度为`0 MiB`。在`Alpha`和`AMD64`系统上，该内存域的长度可能从`0`到`4GiB`。
* `ZONE_NORMAL`标记了可直接映射到内核段的普通内存域。这是在所有体系结构上保证都会存在的唯一内存域，但无法保证该地址范围对应了实际的物理内存。例如，如果`AMD64`系统有`2 GiB`内存，那么所有内存都属于`ZONE_DMA32`范围，而`ZONE_NORMAL`则为空。


<br/>

## 其他基础知识

<br/>

### 页表描述符（page table descriptor）

![image.png](https://upload-images.jianshu.io/upload_images/12321605-c58943107fd21df0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* P（`Present`） - 为`1`表明该`page`存在于当前物理内存中，为`0`则`PTE`的其他部分都失去意义了，不用看了，直接触发`page fault`。`P`位为`0`的PTE也不会有对应的`TLB entry`，因为早在P位由`1`变为`0`的时候，对应的`TLB`就已经被`flush`掉了。
* G （`Global`）- 这个标志位在这篇文章中有介绍，主要是用于`context switch`的时候不用`flush`掉`kernel`对应的`TLB`，所以这个标志位在`TLB entry`中也是存在的。
* A（`Access`） - 当这个`page`被访问（读/写）过后，硬件将该位置`1`，`TLB`只会缓存`access`的值为`1`的`page`对应的映射关系。软件可将该位置`0`，然后对应的`TLB`将会被`flush`掉。这样，软件可以统计出每个`page`被访问的次数，作为内存不足时，判断该`page`是否应该被回收的参考。
* D （`Dirty`）- 这个标志位只对`file backed`的`page`有意义，对`anonymous`的`page`是没有意义的。当`page`被写入后，硬件将该位置`1`，表明该`page`的内容比外部`disk/flash`对应部分要新，当系统内存不足，要将该`page`回收的时候，需首先将其内容`flush`到外部存储。之后软件将该标志位清0。
* R/W（`Read/Write`） - 置为`1`表示该`page`是`writable`的，置为`0`则是`readonly`，对只读的`page`进行写操作会触发`page fault`。
* U/S（`User/Supervisor`） - 置为`0`表示只有`supervisor`（比如操作系统中的`kernel`）才可访问该`page`，置为`1`表示`user`也可以访问。
* PCD（`Page Cache Disabled`）- 置为`1`表示`disable`，即该`page`中的内容是不可以被`cache`的。如果置为`0`（`enable`），还要看`CR0寄存器`中的`CD位`这个总控开关是否也是`0`。
* PWT （`Page Write Through`）- 置为`1`表示该`page`对应的`cache`部分采用`write through`的方式，否则采用`write back`。

**64位页面描述符：**

![image.png](https://upload-images.jianshu.io/upload_images/12321605-e919d40d408e8c17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/12321605-204714b4708fda1d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




<br/>

### system.map

每次编译内核时，都生成一个文件System.map并保存在源代码目录下。除了所有其他（全局）变量、内核定义的函数和例程的地址，该文件还包括图3-11给出的常数的值。

[x86-debian-10.10.0-System.map](https://raw.githubusercontent.com/fanlv/blog/main/backup/Content/Foundation/x86-debian-10.10.0-System.map-4.19.0-17-686-pae)

[x64-Debian-9-System.map](https://raw.githubusercontent.com/fanlv/blog/main/backup/Content/Foundation/x64-System.map-4.14.81.bm.15-amd64)

	fanlv@debian:~$ head -10 /boot/System.map-4.19.0-17-686-pae
	000001c9 A kexec_control_code_size
	01000000 A phys_startup_32
	c1000000 T _stext
	c1000000 T _text
	c1000000 T startup_32
	c100009b W xen_entry
	c10000a0 T start_cpu0
	c10000b0 T startup_32_smp
	c1000218 T verify_cpu
	c1000314 T pvh_start_xen



符号类型：大写为全局符号，小写为局部符号
A：该符号的值是不能改变的，等于`const`
B：该符号来自于未初始化代码段`bss段`
C: 该符号是通用的，通用的符号指未初始化的数据。当链接时，多个通用符号可能对应一个名称，如果该符号在某一个位置定义，这个通用符号被当做未定义的引用。不明白，内核中也没有该类型的符号
D: 该符号位于初始化的数据段
G: 位于初始化数据段，专门对应小的数据对象，比如`global int x`,对应的大数据对象为 数组类型等
I： 到其他符号的间接引用，是对于`a.out`文件的`GNU`扩展，使用非常少
N：调试符号
R：只读代码段的符号
S：`BSS段`（未初始化数据段）的小对象符号
T：代码段符号，全局函数，`t`为局部函数
U：未定义的符号
V：该符号是一个`weak object`，当其连接到为定义的对象上上，该符号的值变为`0`
W： 类似于`V`
—： 该符号是`a.out`文件中的一个`stabs symbol`，获取调试信息
？： 未知类型的符号

[system.map文件详解](https://blog.csdn.net/chun_1959/article/details/45786769)

<br/>

### (N)UMA 模型中的内存组织

有两种类型计算机，分别以不同的方法管理物理内存

1. `UMA`计算机（一致内存访问，`uniform memory access`）将可用内存以连续方式组织起来（可能有小的缺口）。SMP系统中的每个处理器访问各个内存区都是同样快。
	* SMP(`Symmetric Multi-Processor`)所谓对称多处理器结构，是指服务器中多个`CPU`对称工作，无主次或从属关系。各`CPU`共享相同的物理内存，每个`CPU`访问内存中的任何地址所需时间是相同的，因此**SMP也被称为一致存储器访问结构(UMA：Uniform Memory Access)**
2. `NUMA`计算机（非一致内存访问，`non-uniform memory access`）总是多处理器计算机。系统的各个`CPU`都有本地内存，可支持特别快速的访问。各个处理器之间通过总线连接起来，以支持对其他`CPU`的本地内存的访问，当然比访问本地内存慢些。

PS：还有个MPP(`Massive Parallel Processing`)，这里不做讨论。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-7a5a953fd520ed72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



**现在我们用的OS基本都是NUMA内存模式**，可以通过`lscpu` 查看`numa`相关信息

	dp-xxx(xxx@default:prod):message#lscpu
	Architecture:          x86_64
	CPU op-mode(s):        32-bit, 64-bit
	Byte Order:            Little Endian
	CPU(s):                92
	On-line CPU(s) list:   0,1
	Off-line CPU(s) list:  2-91
	Thread(s) per core:    0
	Core(s) per socket:    23
	Socket(s):             2
	NUMA node(s):          2
	Vendor ID:             GenuineIntel
	CPU family:            6
	Model:                 85
	Model name:            Intel(R) Xeon(R) Platinum 8260 CPU @ 2.40GHz
	Stepping:              7
	CPU MHz:               2399.998
	BogoMIPS:              4799.99
	Virtualization:        VT-x
	Hypervisor vendor:     KVM
	Virtualization type:   full
	L1d cache:             32K
	L1i cache:             32K
	L2 cache:              4096K
	L3 cache:              16384K
	NUMA node0 CPU(s):     0-45
	NUMA node1 CPU(s):     46-91

和 `numactl --hardware` 查看

	[root@n227-005-021 fanlv ]$ numactl --hardware
	available: 1 nodes (0)
	node 0 cpus: 0 1 2 3 4 5 6 7
	node 0 size: 15787 MB
	node 0 free: 2025 MB
	node distances:
	node   0
	  0:  10
	 
**NUMA特性禁用**

	一、检查OS是否开启NUMA
	#numactl --hardware
	二、Linux OS层面禁用NUMA
	1、修改 grub.conf
	#vi /boot/grub/grub.conf
	#/* Copyright 2010, Oracle. All rights reserved. */
	 
	default=0
	timeout=5
	hiddenmenu
	foreground=000000
	background=ffffff
	splashimage=(hd0,0)/boot/grub/oracle.xpm.gz
	 
	title Trying_C0D0_as_HD0
	root (hd0,0)
	kernel /boot/vmlinuz-2.6.18-128.1.16.0.1.el5 root=LABEL=DBSYS ro bootarea=dbsys rhgb quiet console=ttyS0,115200n8 console=tty1 crashkernel=128M@16M numa=off
	initrd /boot/initrd-2.6.18-128.1.16.0.1.el5.img
	 
	2、重启Linux操作系统
	#/sbin/reboot
	 
	3、确认OS层面禁用NUMA是否成功
	#cat /proc/cmdline
	root=LABEL=DBSYS ro bootarea=dbsys rhgb quiet console=ttyS0,115200n8 console=tty1 crashkernel=128M@16M numa=off

<br/>

### 三种内存模型

<br/>

#### 什么是内存模型？

这里的内存模型，是指`Linux`内核用怎么样的方式去管理物理内存，一个物理内存页（`4k`），内核会用一个[page（64Byte）](https://github.com/torvalds/linux/blob/master/include/linux/mm_types.h#L70)（类似物理页的`meta`）去记录`Physics Page Number`相关信息，下面几种内存模型是讲如何存储这个物理机的`meta`信息（`page`），保证内核能快速根据`PFN/PPN`找到`Page`，也可以根据`Page`快速算出`PFN`。

`Linux`内存模型发展经历了三个模式，分别为`FLATMEM`、`DISCONTIGMEM`、`SPARSEMEM`。

PS：这里说下PFN和PPN是一个东西。具体可以看[这个PPT第4页](https://compas.cs.stonybrook.edu/~nhonarmand/courses/fa17/cse306/slides/06-paging.pdf)
 
<br/>

#### FLATMEM (flat memory model)

`FLATMEM`内存模型是`Linux`最早使用的内存模型，那时计算机的内存通常不大。`Linux`会使用一个`struct page mem_map[x]`的数组根据PFN去依次存放所有的`strcut page`，且`mem_map`也位于内核空间的线性映射区，所以根据`PFN(页帧号)`即可轻松的找到目标页帧的`strcut page`。
	
	#define __pfn_to_page(pfn)	(mem_map + ((pfn) - ARCH_PFN_OFFSET))
	#define __page_to_pfn(page)	((unsigned long)((page) - mem_map) + \
					 ARCH_PFN_OFFSET)

而对于FLATMEM来说，如果其管理的的物理内存本身是连续的还好说，如果不连续的话，那么中间一部分物理地址是没有对应的物理内存的，形象的说就像是一个个洞（`hole`），这会浪费`mem_map`数组本身占用的内存空间。

![image.png](https://upload-images.jianshu.io/upload_images/12321605-c8c7c21d83278669.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


<br/>

#### DISCONTIGMEM (discontiguous memory model)

对于物理地址空间不存在空洞(`holes`)的计算机来说，`FLATMEM`无疑是最优解。可物理地址中若是存在空洞的话，`FLATMEM`就显得格外的浪费内存，因为`FLATMEM`会在`mem_map`数组中为所有的物理地址都创建一个`struct page`，即使大块的物理地址是空洞，即不存在物理内存。可是为这些空洞这些`struct page`完全是没有必要的。

那什么情况下物理内存是不连续的？那就要说到后来出现的`NUMA`。为了有效的管理NUMA模式下的物理内存，一种被称为不连续内存模型(`discontiguous memory model`)的实现于`1999年`被引入`Linux`系统。

**DISCONTIGMEM是个稍纵即逝的内存模型，在SPARSEMEM出现后即被完全替代**，且当前的`Linux kernel`默认都是使用`SPARSEMEM`，所以介绍`DISCONTIGMEM`的意义不大。

	//PS. 这段源码在Linux最新的代码中已经找不到
	#define __pfn_to_page(pfn)            \
	({    unsigned long __pfn = (pfn);        \
	    unsigned long __nid = arch_pfn_to_nid(__pfn);  \
	    NODE_DATA(__nid)->node_mem_map + arch_local_page_offset(__pfn, __nid);\
	})
	
	#define __page_to_pfn(pg)                        \
	({    const struct page *__pg = (pg);                    \
	    struct pglist_data *__pgdat = NODE_DATA(page_to_nid(__pg));    \
	    (unsigned long)(__pg - __pgdat->node_mem_map) +            \
	     __pgdat->node_start_pfn;                    \
	})
	
![image.png](https://upload-images.jianshu.io/upload_images/12321605-c5734ee77b6e12f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
<br/>

#### SPARSEMEM (sparse memory model)

稀疏内存模型是当前内核默认的选择，从`2005年`被提出后沿用至今，但中间经过几次优化，包括：`CONFIG_SPARSEMEM_VMEMMAP`和`CONFIG_SPARSEMEM_EXTREME`的引入，这两个配置通常是被打开的。
	

**`CONFIG_SPARSEMEM_VMEMMAP:`**

	/* memmap is virtually contiguous.  */
	#define __pfn_to_page(pfn)	(vmemmap + (pfn))
	#define __page_to_pfn(page)	(unsigned long)((page) - vmemmap)

	// /linux/arch/x86/include/asm/pgtable_64.h
	#define vmemmap ((struct page *)VMEMMAP_START)  



PS：引入 `vmemmap` 的核心思想就是用空间（虚拟地址空间）换时间，**2008年以后，SPARSEMEM_VMEMMAP 成为 x86-64 唯一支持的内存模型**，因为它只比`FLAGMEM`开销稍微大一点点，但是比`DISCONTIGMEM`要高效的多。详见[这里](https://lwn.net/Articles/789304/)


![image.png](https://upload-images.jianshu.io/upload_images/12321605-26470483d3f7872a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image.png](https://upload-images.jianshu.io/upload_images/12321605-953e1dc0f94b6573.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

关于这三种内存模型，网上很多文章讲的很混乱，推荐看这篇：
[内存模型「memory model」](https://chasinglulu.github.io/2019/05/29/%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E3%80%8Cmemory-model%E3%80%8D/)

[Reducing page structures for huge pages](https://lwn.net/Articles/839737/)

<br/>

### CPU寻址方式

实模式和保护模式都是`CPU`的工作模式，而`CPU`的工作模式是指`CPU`的寻址方式、寄存器大小等用来反应`CPU`在该环境下如何工作的概念。

<br/>

#### 实模式

实模式的“实”体现在程序中用到的地址都是**真实的物理地址**，“段基址:段内偏移地址”产生的逻辑地址就是物理地址，即程序员可见的地址完全是真实的内存地址。

在实模式下，内存寻址方式和`8086`相同，由`16位`段寄存器的内容乘以`16`（左移`4`位）作为段基址，加上`16`位段偏移地址形成`20`位的物理地址，最大寻址空间`1MB`，最大分段`64KB`。可以使用`32位`指令，即`32位`的`x86 CPU`也可以兼容实模式，此时的实模式相当于高速的`8086`（**32位CPU的实模式可以使用32位下的资源**）。在3`2位CPU`下，**系统复位或加电时都是以实模式启动，然后再切换为保护模式**。在实模式下，所有的段都是可以读、写和可执行的。

`8086CPU`的实模式开创性地提出了地址分段的概念，改变了在它之前的`CPU`只能“硬编码”，程序无法重定位的缺点。然而实模式还是有很多缺陷，其中最主要的是实模式的安全隐患。在实模式下，用户程序和操作系统拥有同等权利，因为实模式下没有特权级。此外，程序可以随意修改自己的段基址，加上实模式下对地址的访问就是实实在在的物理地址，因此程序可以随意修改任意物理地址，甚至包括操作系统所在的内存，这给操作系统带来极大的安全问题。

<br/>

#### 保护模式

保护模式下，`CPU`访问的所有地址都是逻辑地址（段寄存器都为`0`的话，逻辑地址就是虚拟地址），`CPU`会通过"`分段`"或者“`分页`“方式来查寻到对应的物理地址。

在保护模式下，全部 `32` 条地址线有效，可寻址高达 `4 GB` 的物理地址空间。扩充的存储器**段式管理机制**和可选的**页式管理机制**，不仅为存储器共享和保护提供了硬件支持，而且为实现虚拟存储器提供了硬件支持，支持多任务，能够快速地进行任务切换和保护任务环境。四个特权级和完善的特权检查机制，既能实现资源共享又能保证代码和数据的安全及任务的隔离。

**总的来说，我们现在的系统CPU都是在保护模式下，并且用的是页管理机制（查页表方式）来访问内存。**

**CPU寻址相关的寄存器：**

控制寄存器（`CR0~CR3`）用于控制和确定处理器的操作模式以及当前执行任务的特性。4个控制寄存器都是32位的。

* `CR0`：**含有控制CPU操作模式和状态的标识**
* `CR1`：保留不用
* `CR2`：存储导致页错误的线性地址
* `CR3`：**含有页目录表的物理内存基址**


**CR0中的保护控制位：**

**PE：CR0的位0是启用保护（Protection Enable）标志。（CR0的最低位）**

当设置该位时即开启了保护模式，当复位时即进入实地址模式。这个标志仅开启段级保护，而没有启用分页机制。若要启用分页机制，那么`PE`和`PG`都要置位。

**PG：CR0的位31是分页（Paging）标志。（CR0的最高位）**

当设置该位时即开启了分页机制，当复位时则禁止分页机制，此时所有线性地址等同于物理地址。

注意，在开启这个标志之前必须已经开启PE标志，否则`CPU`会产生一个一般保护性异常。

改变`PG`位的代码必须在线性地址空间和物理地址空间中具有相同地址，这部分具有相同地址的代码在分页和未分页世界之间起着桥梁的作用。

如果`PE=0、PG=0`，处理器工作在实地址模式下。（兼容早期的实模式操作系统）

如果`PE=1、PG=0`，处理器工作在无分页机制的段保护模式下（兼容段式管理的操作系统）

如果`PE=1、PG=1`，处理器工作在段页式保护模式下

在系统刚上电时，处理器被复位成`PE=0`和`PG=0`（即实模式状态），以允许引导代码在启用分段和分页机制之前能够初始化这些寄存器和数据结构。

<br/>

#### PAE - 32位系统如何突破4G限制？

在`x86 CPU`中，只有`32位`地址总线，也就意味着只有`4G`地址空间。为了实现在`32位`系统中使用更多内存，`Intel CPU` 提供了 `PAE` (`Pyhsical Address Extensions`)机制，这样可以让操作系统使用超过`4G`的物理内存。

`PAE`机制的打开，需要设置`CR0`、`CR4`控制寄存器和`IA32_EFER` `MSR` 寄存器，设置值`CR0.PG=1`，`CR4.PAE=1` 和 `IA32_EFER.LME=0`。 但`PAE`机制打开后，`MMU`会将`32位`线性地址转换为`52位`物理地址，尽快物理地址是`52位`（4PB），但线性地址仍然为`32位`，即进程可使用的物理内存不超过`4GB`。

<br/>

#### Linux 如何从实模式切换到保护模式

在使用启动装载器（如`LILO`、`GRUB`等）将内核载入物理内存之后，将通过跳转语句，将控制流切换到内存中适当的位置，来调用`arch/x86/boot/header.S`中的汇编语言“函数”`setup`。这是可能的，因为`setup`函数总是位于目标文件中的同
一位置。

该代码执行下列任务，这需要许多汇编代码。

1. 它检查内核是否加载到内存中正确的位置。为此，它使用一个4字节的特征标记，该标记在编
译时集成到内核映像中，并且总是位于物理内存中一个不变的正确位置。
2. 它确定系统内存的大小。
3. 初始化显卡。
4. 将内核映像移动到内存中的某个位置，使得在后续的解压缩期间，映像不会自阻其路。
5. **将CPU切换到保护模式**。

具体实现，内核会调用 [`protected_mode_jump`](https://github.com/torvalds/linux/blob/5bfc75d92efd494db37f5c4c173d3639d4772966/arch/x86/boot/pmjump.S#L24) 把CPU从实模式切换到保护模式。核心代码如下

	movl	%cr0, %edx
	orb	$X86_CR0_PE, %dl	#Protected mode
	movl	%edx, %cr0

<br/>

### 分页模式


<br/>

#### 内核视角 

由上面我们知道，`CPU`读取数据过程是先去`CR3`寄存器拿到`pdg`，然后然后查一级一级页表，最终拿到物理地址的高`40位`基地址。`CR3寄存器`和每个程序的页表内容都是由`Linux`内核来维护的。
<br/>

#### 应用程序页表

`Linux`内核通过一个被称为进程描述符的 [task_struct](https://github.com/torvalds/linux/blob/master/include/linux/sched.h#L661) 结构体来管理进程，这个结构体包含了一个进程所需的所有信息。程序内存相关的都存在 [mm_struct](https://github.com/torvalds/linux/blob/master/include/linux/mm_types.h#L394) 中 `mm_struct` 有一个`pgd_t * pgd;`就是最顶级的目录的地址。内核在做程序切换的时候会调用 `pick_next_task` -> `context_switch` -> [`switch_mm_irqs_off`](https://github.com/torvalds/linux/blob/9269d27e519ae9a89be8d288f59d1ec573b0c686/arch/x86/mm/tlb.c#L428) -> [`load_new_mm_cr3`](https://github.com/torvalds/linux/blob/9269d27e519ae9a89be8d288f59d1ec573b0c686/arch/x86/mm/tlb.c#L269) -> 


这里需要注意两点。第一点，`cr3` 里面存放当前进程的顶级 `pgd`，这个是硬件的要求。`cr3` 里面需要存放 `pgd` 在物理内存的地址，不能是虚拟地址。因而 `load_new_mm_cr3` 里面会使用 `__pa`，将 `mm_struct` 里面的成员变量 `pgd`（`mm_struct` 里面存的都是虚拟地址）变为物理地址，才能加载到 `cr3` 里面去。

第二点，用户进程在运行的过程中，访问虚拟内存中的数据，会被 `cr3` 里面指向的页表转换为物理地址后，才在物理内存中访问数据，这个过程都是在用户态运行的，地址转换的过程无需进入内核态。

	static void load_new_mm_cr3(pgd_t *pgdir, u16 new_asid, bool need_flush)
	{
		unsigned long new_mm_cr3;
	
		if (need_flush) {
			invalidate_user_asid(new_asid);
			new_mm_cr3 = build_cr3(pgdir, new_asid);
		} else {
			new_mm_cr3 = build_cr3_noflush(pgdir, new_asid);
		}
	
		/*
		 * Caution: many callers of this function expect
		 * that load_cr3() is serializing and orders TLB
		 * fills with respect to the mm_cpumask writes.
		 */
		write_cr3(new_mm_cr3);
	}


`write_cr3 `相对就比较简单，一个`mov`汇编指令来设置`cr3`寄存器的值

	static inline void write_cr3(unsigned long x)
	{
		PVOP_ALT_VCALL1(mmu.write_cr3, x,
				"mov %%rdi, %%cr3", ALT_NOT(X86_FEATURE_XENPV));
	}

<br/>

#### 内核页表

和用户态页表不同，在系统初始化的时候，我们就要创建内核页表了。[内核页表定义如下](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/pgtable_64.h#L19)：

	extern p4d_t level4_kernel_pgt[512];
	extern p4d_t level4_ident_pgt[512];
	extern pud_t level3_kernel_pgt[512];
	extern pud_t level3_ident_pgt[512];
	extern pmd_t level2_kernel_pgt[512];
	extern pmd_t level2_fixmap_pgt[512];
	extern pmd_t level2_ident_pgt[512];
	extern pte_t level1_fixmap_pgt[512 * FIXMAP_PMD_NUM];
	extern pgd_t init_top_pgt[];
	
		
	struct mm_struct init_mm = {
	  .mm_rb    = RB_ROOT,
	  .pgd    = swapper_pg_dir,
	  .mm_users  = ATOMIC_INIT(2),
	  .mm_count  = ATOMIC_INIT(1),
	  .mmap_sem  = __RWSEM_INITIALIZER(init_mm.mmap_sem),
	  .page_table_lock =  __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),
	  .mmlist    = LIST_HEAD_INIT(init_mm.mmlist),
	  .user_ns  = &init_user_ns,
	  INIT_MM_CONTEXT(init_mm)
	};

定义完了内核页表，接下来是初始化内核页表，在系统启动的时候 `start_kernel` 会调用 `setup_arch`。

	void __init setup_arch(char **cmdline_p)
	{
	#ifdef CONFIG_X86_32
		memcpy(&boot_cpu_data, &new_cpu_data, sizeof(new_cpu_data));
	
		/*
		 * copy kernel address range established so far and switch
		 * to the proper swapper page table
		 */
		clone_pgd_range(swapper_pg_dir     + KERNEL_PGD_BOUNDARY,
				initial_page_table + KERNEL_PGD_BOUNDARY,
				KERNEL_PGD_PTRS);
	
		load_cr3(swapper_pg_dir);
		.......
	}
	
	static inline void load_cr3(pgd_t *pgdir)
	{
		write_cr3(__sme_pa(pgdir));
	}

内核的页表初始化是在 `arch\x86\kernel\head_64.S` 中。这段代码比较难看懂，占不去深究。大概知道内核是怎么维护页表的就好。


<br/>

#### 硬件视角

<br/>

##### CPU翻译虚拟地址到物理地址过程

![image](https://upload-images.jianshu.io/upload_images/12321605-16b2286da60c1b08.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<br/>

##### CPU读取数据过程

![image](https://upload-images.jianshu.io/upload_images/12321605-ff93072e983e0e68.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](https://upload-images.jianshu.io/upload_images/12321605-877f131a5206033e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




<br/>

## 参考

**《深入理解Linux虚拟内存管理》**

**《深入理解操作系统》**

**《深入理解Linux内核》**

http://ilinuxkernel.com/?p=1013

https://jekton.github.io/2018/11/18/linux-page-table-setup/

https://blog.csdn.net/fullofwindandsnow/article/details/8565512

https://www.cnblogs.com/tolimit/p/4585803.html

https://www.cnblogs.com/4a8a08f09d37b73795649038408b5f33/p/10154324.html

http://parrotshen.blogspot.com/2008/01/test.html

https://zhuanlan.zhihu.com/p/67053210

https://zhuanlan.zhihu.com/p/220068494

https://toutiao.io/posts/6r7pjh/preview

https://www.cnblogs.com/chenwb89/p/operating_system_002.html

http://www.wowotech.net/memory_management/memory_model.html

https://chasinglulu.github.io/2019/05/29/%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E3%80%8Cmemory-model%E3%80%8D/

https://elinux.org/images/b/b0/Introduction_to_Memory_Management_in_Linux.pdf

https://pdfs.semanticscholar.org/5b92/9e20c9203232ac8aefbab9d905499f4bde25.pdf