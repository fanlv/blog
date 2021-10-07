

内核占据的内存分为几个段，其边界保存在变量中。

* _text和_etext是代码段的起始和结束地址，包含了编译后的内核代码。
* 数据段位于_etext和_edata之间，保存了大部分内核变量。
* 初始化数据在内核启动过程结束后不再需要（例如，包含初始化为0的所有静态全局变量的BSS段）保存在最后一段，从_edata到_end。

**在内核初始化完成后，其中的大部分数据都可以从内存删除，给应用程序留出更多空间**。这一段内存区划分为更小的子区间，以控制哪些可以删除，哪些不能删除，但这对于我们现在的讨论没多大意义。

虽然用来划定段边界的变量定义在内核源代码（arch/x86/kernel/setup_32.c）中，但此时尚未赋值。这是因为不太可能。编译器在编译时间怎么能知道内核最终有多大？只有在目标文件链接完成后，才能知道确切的数值，接下来则打包为二进制文件。该操作是由arch/arch/vmlinux.ld.S控制的（对IA-32来说，该文件是arch/x86/vmlinux_32.ld.S），其中也划定了内核的内存布局。

/proc/iomem也提供了有关物理内存划分出的各个段的一些信息。

	[root@n227-005-021 fanlv ]$ cat /proc/iomem
	00000000-00000fff : Reserved
	00001000-0009fbff : System RAM
	0009fc00-0009ffff : Reserved
	000a0000-000bffff : PCI Bus 0000:00
	000c0000-000c95ff : Video ROM
	000c9800-000cbbff : Adapter ROM
	000f0000-000fffff : Reserved
	  000f0000-000fffff : System ROM
	00100000-bffdbfff : System RAM
	  01000000-01a02fff : Kernel code  （1 ~ 26 M）
	  01a03000-021241bf : Kernel data   （26M ~ 33M）
	  02573000-02611fff : Kernel bss     （37M ~ 38M）
	  25000000-34ffffff : Crash kernel
	bffdc000-bfffffff : Reserved



- 内核态页表, 系统初始化时就创建
    - swapper_pg_dir 指向内核顶级页目录 pgd
        - xxx_ident/kernel/fixmap_pgt 分别是直接映射/内核代码/固定映射的 xxx 级页表目录
    - 创建内核态页表
        - swapper_pg_dir 指向 init_top_pgt, 是 ELF 文件的全局变量, 因此再内存管理初始化之间就存在
        - init_top_pgt 先初始化了三项
            - 第一项指向 level3_ident_pgt (内核代码段的某个虚拟地址) 减去 __START_KERNEL_MAP (内核代码起始虚拟地址) 得到实际物理地址
            - 第二项也是指向 level3_ident_pgt
            - 第三项指向 level3_kernel_pgt 内核代码区
    - 初始化各页表项, 指向下一集目录
        - 页表覆盖范围较小, 内核代码 512MB, 直接映射区 1GB
        - 内核态也定义 mm_struct 指向 swapper_pg_dir
        - 初始化内核态页表, start_kernel→ setup_arch
            - load_cr3(swapper_pg_dir) 并刷新 TLB
            - 调用 init_mem_mapping→kernel_physical_mapping_init, 用 __va 将物理地址映射到虚拟地址, 再创建映射页表项
            - CPU 在保护模式下访问虚拟地址都必须通过 cr3, 系统只能照做
            - 在 load_cr3 之前, 通过 early_top_pgt 完成映射
- vmalloc 和 kmap_atomic
    - 内核的虚拟地址空间 vmalloc 区域用于映射
    - kmap_atomic 临时映射
        - 32 位, 调用 set_pte 通过内核页表临时映射
        - 64 位, 调用 page_address→lowmem_page_address 进行映射
- 内核态缺页异常
    - kmap_atomic 直接创建页表进行映射
    - vmalloc 只分配内核虚拟地址, 访问时触发缺页中断, 调用 do_page_fault→vmalloc_fault 用于关联内核页表项
- kmem_cache 和 kmalloc 用于保存内核数据结构, 不会被换出; 而内核 vmalloc 会被换出


内存管理(上)

- 内存管理包含: 物理内存管理; 虚拟内存管理; 两者的映射
- 除了内存管理模块, 其他都使用虚拟地址(包括内核)
- 虚拟内存空间包含: 内核空间(高地址); 用户空间(低地址)
- 用户空间从低到高布局为: 代码段; DATA 段; BSS 段(未初始化静态变量); 堆段; 内存映射段; 栈地址空间段
- 多个进程看到的用户空间是独立的
- 内核空间: 多个进程看到同一内核空间, 但内核栈每个进程不一样
- 内核代码也仅能访问内核空间
- 内核也有内核代码段, DATA 段, 和 BSS 段; 位于内核空间低地址
- 内核代码也是 ELF 格式


    - 在 load_elf_bianry 时做 vm_area_struct 与各区域的映射, 并将 elf 映射到内存, 将依赖 so 添加到内存映射
    - 所有进程看到的内核虚拟地址空间是同一个
    - 32 位系统, 前 896MB 为直接映射区(虚拟地址 - 3G = 物理地址)
        - 直接映射区也需要建立页表, 通过虚拟地址访问(除了内存管理模块)
        - 直接映射区组成: 1MB 启动时占用; 然后是内核代码/全局变量/BSS等,即 内核 ELF文件内容; 进程 task_struct 即内核栈也在其中
        - 896MB 也称为高端内存(指物理内存)
        - 剩余虚拟空间组成: 8MB 空余; 内核动态映射空间(动态分配内存, 映射放在内核页表中); 持久内存映射(储存物理页信息); 固定内存映射; 临时内存映射(例如为进程映射文件时使用)
    - 64 位系统: 8T 空余; 64T 直接映射区域; 32T(动态映射); 1T(物理页描述结构 struct page); 512MB(内核代码, 也采用直接映射)




- 物理内存组织方式
    - 每个物理页由 struct page 表示
    - 物理页连续, page 放入一个数组中, 称为平坦内存模型
    - 多个 CPU 通过总线访问内存, 称为 SMP 对称多处理器(采用平坦内存模型, 总线成为瓶颈)
    - 每个 CPU 都有本地内存, 访问内存不用总线, 称为 NUMA 非一致内存访问
    - 本地内存称为 NUMA 节点, 本地内存不足可以向其他节点申请
    - NUMA 采用非连续内存模型，页号不连续
    - 另外若内存支持热插拔，则采用稀疏内存模型
- 节点
    - 用 pglist_data 表示 NUMA 节点，多个节点信息保存在 node_data 数组中
    - pglist_data 包括 id，page 数组,起始页号, 总页数, 可用页数
    - 节点分为多个区域 zone, 包括 DMA; 直接映射区; 高端内存区; 可移动区(避免内存碎片)
- 区域 zone
    - 用 zone 表示; 包含第一个页页号; 区域总页数; 区域实际页数; 被伙伴系统管理的页数; 用 per_cpu_pageset 区分冷热页(热页, 被 CPU 缓存的页)
- 页
    - 用 struct page 表示, 有多种使用模式, 因此 page 结构体多由 union 组成
    - 使用一整个页: 1) 直接和虚拟地址映射(匿名页); 2) 与文件关联再与虚拟地址映射(内存映射文件)
        - page 记录: 标记用于内存映射; 指向该页的页表数; 换出页的链表; 复合页, 用于合成大页;
    - 分配小块内存:
        - Linux 采用 slab allocator 技术; 申请一整页, 分为多个小块存储池, 用队列维护其状态(较复杂)
        - slub allocator 更简单
        - slob allocator 用于嵌入式
        - page 记录: 第一个 slab 对象; 空闲列表; 待释放列表
- 页分配
    - 分配较大内存(页级别), 使用伙伴系统
    - Linux 把空闲页分组为 11 个页块链表, 链表管理大小不同的页块(页大小 2^i * 4KB)
    - 分配大页剩下的内存, 插入对应空闲链表
    - alloc_pages->alloc_pages_current 用 gfp 指定在哪个 zone 分配


- 页面换出
    - 触发换出:
    - 1) 分配内存时发现没有空闲; 调用 `get_page_from_freelist->node_reclaim->__node_reclaim->shrink_node`
    - 2) 内存管理主动换出, 由内核线程 kswapd 实现
    - kswapd 在内存不紧张时休眠, 在内存紧张时检测内存 调用 balance_pgdat->kswapd_shrink_node->shrink_node
    - 页面都挂在 lru 链表中, 页面有两种类型: 匿名页; 文件内存映射页
    - 每一类有两个列表: active 和 inactive 列表
    - 要换出时, 从 inactive 列表中找到最不活跃的页换出
    - 更新列表, shrink_list 先缩减 active 列表, 再缩减不活跃列表
    - 缩减不活跃列表时对页面进行回收:
        - 匿名页回收: 分配 swap, 将内存也写入文件系统
        - 文件内存映射页: 将内存中的文件修改写入文件中




- 申请小块内存用 brk; 申请大块内存或文件映射用 mmap
- mmap 映射文件, 由 fd 得到 struct file
    - 调用 ...->do_mmap
        - 调用 get_unmapped_area 找到一个可以进行映射的 vm_area_struct
        - 调用 mmap_region 进行映射
    - get_unmapped_area
        - 匿名映射: 找到前一个 vm_area_struct
        - 文件映射: 调用 file 中 file_operations 文件的相关操作, 最终也会调用到 get_unmapped_area
    - mmap_region
        - 通过 vm_area_struct 判断, 能否基于现有的块扩展(调用 vma_merge)
        - 若不能, 调用 kmem_cache_alloc 在 slub 中得到一个 vm_area_struct 并进行设置
        - 若是文件映射: 则调用 file_operations 的 mmap 将 vm_area_struct 的内存操作设置为文件系统对应操作(读写内存就是读写文件系统)
        - 通过 vma_link 将 vm_area_struct 插入红黑树
        - 若是文件映射, 调用 __vma_link_file 建立文件到内存的反映射
- 内存管理不直接分配内存, 在使用时才分配
- 用户态缺页异常, 触发缺页中断, 调用 do_page_default
- __do_page_fault 判断中断是否发生在内核
    - 若发生在内核, 调用 vmalloc_fault, 使用内核页表进行映射
    - 若不是, 找到对应 vm_area_struct 调用 handle_mm_fault
    - 得到多级页表地址 pgd 等
    - pgd 存在 task_struct.mm_struct.pgd 中
    - 全局页目录项 pgd 在创建进程 task_struct 时创建并初始化, 会调用 pgd_ctor 拷贝内核页表到进程的页表
- 进程被调度运行时, 通过 switch_mm_irqs_off->load_new_mm_cr3 切换内存上下文
- cr3 是 cpu 寄存器, 存储进程 pgd 的物理地址(load_new_mm_cr3 加载时通过直接内存映射进行转换)
- cpu 访问进程虚拟内存时, 从 cr3 得到 pgd 页表, 最后得到进程访问的物理地址
- 进程地址转换发生在用户态, 缺页时才进入内核态(调用__handle_mm_fault)
- __handle_mm_fault 调用 pud_alloc, pmd_alloc, handle_pte_fault 分配页表项
    - 若不存在 pte
        - 匿名页: 调用 do_anonymous_page 分配物理页 ①
        - 文件映射: 调用 do_fault ②
    - 若存在 pte, 调用 do_swap_page 换入内存 ③
    - ① 为匿名页分配内存
        - 调用 pte_alloc 分配 pte 页表项
        - 调用 ...->__alloc_pages_nodemask 分配物理页
        - mk_pte 页表项指向物理页; set_pte_at 插入页表项
    - ② 为文件映射分配内存 __do_fault
        - 以 ext4 为例, 调用 ext4_file_fault->filemap_fault
        - 文件映射一般有物理页作为缓存 find_get_page 找缓存页
        - 若有缓存页, 调用函数预读数据到内存
        - 若无缓存页, 调用 page_cache_read 分配一个, 加入 lru 队列, 调用 readpage 读数据: 调用 kmap_atomic 将物理内存映射到内核临时映射空间, 由内核读取文件, 再调用 kunmap_atomic 解映射
    - ③ do_swap_page
        - 先检查对应 swap 有没有缓存页
        - 没有, 读入 swap 文件(也是调用 readpage)
        - 调用 mk_pte; set_pet_at; swap_free(清理 swap)
- 避免每次都需要经过页表(存再内存中)访问内存
    - TLB 缓存部分页表项的副本


- 涉及三块内容:
    - 内存映射函数 vmalloc, kmap_atomic
    - 内核态页表存放位置和工作流程
    - 内核态缺页异常处理
- 内核态页表, 系统初始化时就创建
    - swapper_pg_dir 指向内核顶级页目录 pgd
        - xxx_ident/kernel/fixmap_pgt 分别是直接映射/内核代码/固定映射的 xxx 级页表目录
    - 创建内核态页表
        - swapper_pg_dir 指向 init_top_pgt, 是 ELF 文件的全局变量, 因此再内存管理初始化之间就存在
        - init_top_pgt 先初始化了三项
            - 第一项指向 level3_ident_pgt (内核代码段的某个虚拟地址) 减去 __START_KERNEL_MAP (内核代码起始虚拟地址) 得到实际物理地址
            - 第二项也是指向 level3_ident_pgt
            - 第三项指向 level3_kernel_pgt 内核代码区
    - 初始化各页表项, 指向下一集目录
        - 页表覆盖范围较小, 内核代码 512MB, 直接映射区 1GB
        - 内核态也定义 mm_struct 指向 swapper_pg_dir
        - 初始化内核态页表, start_kernel→ setup_arch
            - load_cr3(swapper_pg_dir) 并刷新 TLB
            - 调用 init_mem_mapping→kernel_physical_mapping_init, 用 __va 将物理地址映射到虚拟地址, 再创建映射页表项
            - CPU 在保护模式下访问虚拟地址都必须通过 cr3, 系统只能照做
            - 在 load_cr3 之前, 通过 early_top_pgt 完成映射
- vmalloc 和 kmap_atomic
    - 内核的虚拟地址空间 vmalloc 区域用于映射
    - kmap_atomic 临时映射
        - 32 位, 调用 set_pte 通过内核页表临时映射
        - 64 位, 调用 page_address→lowmem_page_address 进行映射
- 内核态缺页异常
    - kmap_atomic 直接创建页表进行映射
    - vmalloc 只分配内核虚拟地址, 访问时触发缺页中断, 调用 do_page_fault→vmalloc_fault 用于关联内核页表项
- kmem_cache 和 kmalloc 用于保存内核数据结构, 不会被换出; 而内核 vmalloc 会被换出





操作系统如何管理 物理地址和虚拟 地址 。

页表里面存的 什么东西

用户态分配内存

load_new_mm_cr3

cr3  存储

- kmem_cache 和 kmalloc 用于保存内核数据结构, 不会被换出; 而内核 vmalloc 会被换出
            - CPU 在保护模式下访问虚拟地址都必须通过 cr3, 系统只能照做
            - 调用 init_mem_mapping→kernel_physical_mapping_init, 用 __va 将物理地址映射到虚拟地址, 再创建映射页表项

            - load_cr3(swapper_pg_dir) 并刷新 TLB
        - 页表覆盖范围较小, 内核代码 512MB, 直接映射区 1GB
        - swapper_pg_dir 指向 init_top_pgt, 是 ELF 文件的全局变量, 因此再内存管理初始化之间就存在
    - swapper_pg_dir 指向内核顶级页目录 pgd
    - 内存映射函数 vmalloc, kmap_atomic
    - 内核态页表存放位置和工作流程
    - 内核态缺页异常处理

