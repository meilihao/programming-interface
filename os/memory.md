# 内存
Linux像多数现代内核一样,采用了虚拟内存管理技术. 该技术利用了大多数程序的一个典型特征,即访问局部性(locality of reference),以求高效使用 CPU 和 RAM(物理内存)资源. 大多数程序都展现了两种类型的局部性:
- 空间局部性(Spatial locality)

	指程序倾向于访问在最近访问过的内存地址附近的内存(由于指令是顺序执行的,且有时会按顺序处理数据结构).
- 时间局部性(Temporal locality)

	指程序倾向于在不久的将来再次访问最近刚访问过的内存地址(由于循环).

虚拟内存的规划之一是将每个程序使用的内存切割成小型的、固定大小的`页(page)`单元. 相应地,将 RAM 划分成一系列与虚存页尺寸相同的页帧. 任一时刻,每个程序仅有部分页需要驻留在物理内存页帧中, 这些页构成了所谓驻留集(resident set). 程序未使用的页拷贝保存在交换区(swap area)内—这是磁盘空间中的保留区域,作为计算机 RAM 的补充—
仅在需要时才会载入物理内存. 若进程欲访问的页面目前并未驻留在物理内存中,将会发生页面错误(page fault),内核即刻挂起进程的执行,同时从磁盘中将该页面载入内存.

> 程序可调用 sysconf(_SC_PAGESIZE)来获取系统虚拟内存的页面大小.

为支持这一组织方式,内核需要为每个进程维护一张页表(page table). 该页表描述了每页在进程虚拟地址空间(virtual address space)中的位置(可为进程所用的所有虚拟内存页面的集合). 页表中的每个条目要么指出一个虚拟页面在 RAM 中的所在位置,要么
表明其当前驻留在磁盘上.

在进程虚拟地址空间中,并非所有的地址范围都需要页表条目. 通常情况下,由于可能存在大段的虚拟地址空间并未投入使用,故而也无必要为其维护相应的页表条目. 若进程试
图访问的地址并无页表条目与之对应,那么进程将收到一个 SIGSEGV 信号.

由于内核能够为进程分配和释放页(和页表条目),所以进程的有效虚拟地址范围在其生命周期中可以发生变化:
- 栈向下增长超出之前曾达到的位置
- 当在堆中分配或释放内存时,通过调用 brk()、sbrk()或 malloc 函数族来提升 program break 的位置
- 当调用 shmat()连接 System V 共享内存区时,或者当调用 shmdt()脱离共享内存区时
- 当调用 mmap()创建内存映射时,或者当调用 munmap()解除内存映射时

> 虚拟内存的实现需要硬件中分页内存管理单元(PMMU)的支持. PMMU 把要访问的每个虚拟内存地址转换成相应的物理内存地址,当特定虚拟内存地址所对应的页没有驻留于 RAM 中时,将以页面错误通知内核.

虚拟内存有点:
- 进程隔离
- 共享内存
- 便于实现内存保护机制

	可对不同进程的页表条目(共享ram时)进行标记,以表示相关页面内容是可读、可写、可执行亦或是这些保护措施的组合

- 程序员和编译器, 链接器之类的工具不用关心程序的内存布局

## 内存管理
参考:
- [基本分页存储管理](https://www.jianshu.com/p/ee2d0b912d05)

![](/misc/img/os/7dd9039e4ad2f6433aa09c14ede92991.jpg)

内存管理:
- 连续分配

	不实用: 32 位环境下，虚拟地址空间共 4GB. 如果分成 4KB 一个页，那就是 1M 个页, 每个页表项需要 4 个字节来存储，那么整个 4GB 空间的映射就需要 4MB 的内存来存储映射表, 如果每个进程都有自己的映射表，100 个进程就需要 400MB 的内存, 对内存的消耗实在是太大了, 更别说 64bit 环境了.

- 不连续分配

	- 分页
	- 分段
	- 段页结合

目前的主流是分页, 但x86 cpu非主流设计的分页是构建在分段的基础上, 因此x86是段页结合.

### 分段机制
分段机制下的虚拟地址由两部分组成: 段选择子和段内偏移量.

段选择子就保存在段寄存器中, 段选择子里面最重要的是段号，用作段表的索引. 段表里面保存的是这个段的基地址、段的界限和特权等级等.

虚拟地址中的段内偏移量应该位于 0 和段界限之间, 如果段内偏移量是合法的，就将段基地址加上段内偏移量得到物理内存地址.

在 Linux 里面，段表全称段描述符表（segment descriptors），放在全局描述符表 GDT（Global Descriptor Table）里面，会有下面的宏来初始化段描述符表里面的表项:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/desc_defs.h#L23
#define GDT_ENTRY_INIT(flags, base, limit)			\
	{							\
		.limit0		= (u16) (limit),		\
		.limit1		= ((limit) >> 16) & 0x0F,	\
		.base0		= (u16) (base),			\
		.base1		= ((base) >> 16) & 0xFF,	\
		.base2		= ((base) >> 24) & 0xFF,	\
		.type		= (flags & 0x0f),		\
		.s		= (flags >> 4) & 0x01,		\
		.dpl		= (flags >> 5) & 0x03,		\
		.p		= (flags >> 7) & 0x01,		\
		.avl		= (flags >> 12) & 0x01,		\
		.l		= (flags >> 13) & 0x01,		\
		.d		= (flags >> 14) & 0x01,		\
		.g		= (flags >> 15) & 0x01,		\
	}
```

一个段表项由段基地址 base、段界限 limit，还有一些标识符组成:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/kernel/cpu/common.c#L113
DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
#ifdef CONFIG_X86_64
	/*
	 * We need valid kernel segments for data and code in long mode too
	 * IRET will check the segment types  kkeil 2000/10/28
	 * Also sysret mandates a special GDT layout
	 *
	 * TLS descriptors are currently at a different place compared to i386.
	 * Hopefully nobody expects them at a fixed place (Wine?)
	 */
	[GDT_ENTRY_KERNEL32_CS]		= GDT_ENTRY_INIT(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_CS]		= GDT_ENTRY_INIT(0xa09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_DS]		= GDT_ENTRY_INIT(0xc093, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER32_CS]	= GDT_ENTRY_INIT(0xc0fb, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_DS]	= GDT_ENTRY_INIT(0xc0f3, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_CS]	= GDT_ENTRY_INIT(0xa0fb, 0, 0xfffff),
#else
	[GDT_ENTRY_KERNEL_CS]		= GDT_ENTRY_INIT(0xc09a, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_DS]		= GDT_ENTRY_INIT(0xc092, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_CS]	= GDT_ENTRY_INIT(0xc0fa, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_DS]	= GDT_ENTRY_INIT(0xc0f2, 0, 0xfffff),
	/*
	 * Segments used for calling PnP BIOS have byte granularity.
	 * They code segments and data segments have fixed 64k limits,
	 * the transfer segment sizes are set at run time.
	 */
	/* 32-bit code */
	[GDT_ENTRY_PNPBIOS_CS32]	= GDT_ENTRY_INIT(0x409a, 0, 0xffff),
	/* 16-bit code */
	[GDT_ENTRY_PNPBIOS_CS16]	= GDT_ENTRY_INIT(0x009a, 0, 0xffff),
	/* 16-bit data */
	[GDT_ENTRY_PNPBIOS_DS]		= GDT_ENTRY_INIT(0x0092, 0, 0xffff),
	/* 16-bit data */
	[GDT_ENTRY_PNPBIOS_TS1]		= GDT_ENTRY_INIT(0x0092, 0, 0),
	/* 16-bit data */
	[GDT_ENTRY_PNPBIOS_TS2]		= GDT_ENTRY_INIT(0x0092, 0, 0),
	/*
	 * The APM segments have byte granularity and their bases
	 * are set at run time.  All have 64k limits.
	 */
	/* 32-bit code */
	[GDT_ENTRY_APMBIOS_BASE]	= GDT_ENTRY_INIT(0x409a, 0, 0xffff),
	/* 16-bit code */
	[GDT_ENTRY_APMBIOS_BASE+1]	= GDT_ENTRY_INIT(0x009a, 0, 0xffff),
	/* data */
	[GDT_ENTRY_APMBIOS_BASE+2]	= GDT_ENTRY_INIT(0x4092, 0, 0xffff),

	[GDT_ENTRY_ESPFIX_SS]		= GDT_ENTRY_INIT(0xc092, 0, 0xfffff),
	[GDT_ENTRY_PERCPU]		= GDT_ENTRY_INIT(0xc092, 0, 0xfffff),
	GDT_STACK_CANARY_INIT
#endif
} };
EXPORT_PER_CPU_SYMBOL_GPL(gdt_page);
```

这里面对于 64 位的和 32 位的，都定义了内核代码段、内核数据段、用户代码段和用户数据段.

另外，还会定义下面四个段选择子，指向上面的段描述符表项. 内核初始化的时候，启动第一个用户态的进程，就是将这四个值赋值给段寄存器.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/segment.h#L134
/*
 * Segment selector values corresponding to the above entries:
 */

#define __KERNEL_CS			(GDT_ENTRY_KERNEL_CS*8)
#define __KERNEL_DS			(GDT_ENTRY_KERNEL_DS*8)
#define __USER_DS			(GDT_ENTRY_DEFAULT_USER_DS*8 + 3)
#define __USER_CS			(GDT_ENTRY_DEFAULT_USER_CS*8 + 3)
```

通过分析现，所有的段的起始地址都是一样的，都是 0. 所以，在 Linux 操作系统中，并没有使用到全部的分段功能, 而是用分段做权限审核，例如用户态 DPL 是 3，内核态 DPL 是 0, 当用户态试图访问内核态的时候，会因为权限不足而报错.

### 分页机制
其实 Linux 倾向于另外一种从虚拟地址到物理地址的转换方式，称为分页（Paging）. 对于物理内存，操作系统把它分成一块一块大小相同的页，这样更方便管理，例如有的内存页面长时间不用了，可以暂时写到硬盘上，称为换出. 一旦需要的时候，再加载进来，叫做换入. 这样可以扩大可用物理内存的大小，提高物理内存的利用率. 这个换入和换出都是以页为单位的. 页面的大小一般为 4KB. 为了能够定位和访问每个页，需要有个页表，保存每个页的起始地址，再加上在页内的偏移量，组成线性地址，就能对于内存中的每个位置进行访问了.

虚拟地址分为两部分: 页号和页内偏移, 页号作为页表的索引，页表包含物理页每页所在物理内存的基地址. 这个基地址与页内偏移的组合就形成了物理内存地址.

在32bit上, 会用前 10 位定位到页目录表中的一项, 将这一项对应的页表取出来共 1k 项，再用中间 10 位定位到页表中的一项，将这一项对应的存放数据的页取出来，再用最后 12 位(range=0~4k)定位到页中的具体位置访问数据, 这样加起来正好 32 位即4B.

如果这样的话，映射 4GB 地址空间就需要`4KB + 4MB(一级页表size+二级页表size = 1K*4B + 1k*1k*4B)`的内存, 还变多了? 但结合实际情况, 根据局部性原理可知，很多时候，进程在一段时间内只需要访问某几个页面就可以正常运行了, 没必要让所有页面都常驻内存, 因此使用多级页表可节省空间.

![](/misc/img/os/b6960eb0a7eea008d33f8e0c4facc8b8.jpg)

对于 64 位的系统，两级肯定不够了，而是使用四级目录，分别是全局页目录项 PGD（Page Global Directory）、上层页目录项 PUD（Page Upper Directory）、中间页目录项 PMD（Page Middle Directory）和页表项 PTE（Page Table Entry）.
![](/misc/img/os/42eff3e7574ac8ce2501210e25cd2c0b.jpg)

> 32bit 的页目录在CR3寄存器中, 保存的是页目录的指针.

## 内存分配
Linux内核内存管理的一项重要工作就是如何在频繁申请释放内存的情况下，避免碎片的产生. Linux采用伙伴系统解决外部碎片的问题，采用slab解决内部碎片的问题.

### 伙伴系统
参考:
- [操作系统（2020）_清华大学](https://www.bilibili.com/video/BV1x7411T7mh?p=36)

伙伴系统的宗旨就是用最小的内存块来满足内核的对于内存的请求. 其基本思想很简单. 内存被分成含有很多页面的大块, 每一块都是2个页面大小的方幂. 如果找不到想要的块, 一个大块会被分成两部分, 这两部分彼此就成为伙伴. 其中一半被用来分配, 而另一半则空闲. 这些块在以后分配的过程中会继续被二分直至产生一个所需大小的块. 当一个块被最终释放时, 其伙伴将被检测出来, 如果伙伴也空闲则合并两者.

伙伴系统中可用内存块的大小是2^k字节, (L<=K<=U, 2^L表示分配的最小块大小, 2^U是分配的最大块的大小, 通常2^U是整个内存大小).

假设用户向系统申请大小为 n 的存储空间，若 2k-1 < n <= 2k，此时就需要查看可利用空间表中大小为 2k的链表中有没有可利用的空间结点：
- 如果该链表不为 NULL，可以直接采用头插法从头部取出一个结点，提供给用户使用；
- 如果大小为 2k的链表为 NULL，就需要依次查看比2k大的链表，找到后从链表中删除，截取相应大小的空间给用户使用，剩余的空间，根据大小插入到相应的链表中

## 内存分页
内存分页状态:
- free : 有效的, 可立刻分配
- inactive clean : 没有活跃使用, 内容符合磁盘上的内容, 因为它已经被写回或自读以来没有改变
- inactive dirty : 没有活跃使用, 但是自从磁盘上读出来后, 分页内容已被修改, 且还未被回写
- active : 在活跃使用中, 并不能作为可释放的候选

> /proc/meminfo存储了整个系统的内存分配概况.

脏页相关的内核配置:
![](/misc/img/arch/vm_dirty.png)

## OutOfMemory(内存不足) 和 OOM killer
当发生一个次要页错误(minor page fault), 但又没有空闲的分页可使用时, 内核将尝试回收内存来满足请求. 如何不能及时回收足够的内存, 将出现OutOfMemory的情况.

默认情况下, 系统会调用OOM killer选择杀死一个或多个进程来是否内存, 以满足请求.

kernel会在`/proc/${pid}/oom_score`下保存一个运行的不良分数(badness score), 较高的分数的进程最有可能被OOM killer杀掉. 但也有进程是免杀的, 比如kernel本身和init.

> 通过`/proc/${pid}/oom_score_adj`可调整oom_score.
> oom_score的计算来源: 虚拟内存大小, 进程包括所有子进程累计的虚拟内存大小, nice值(正数得到一个较高分数), 总共运行时间(较长时间减少分数), 运行的用户(root进程得到轻微的保护), 进程可直接访问内存(减少分数).

## Translation Lookaside Buffer(TLB)
TLB是特别的cpu缓存, 通过缓存进程最近使用的分页映射来加速地址转换(虚拟地址 -> 物理地址).

进程使用的内存越多, 分页表就越大. 同时进程在分页表上查询一个分页映射的开销也会很大. 因此tlb可很好地解决这个问题.

当上下文切换时, 为了将进程调度出来, 内核必须经常刷新tlb条目.

> tlb可通过`x86info -a`查看.
> linux通过hugepage来支持大号的分页. x86_64支持分页大小: 4K, 2M, 4M, 1G. 进程使用mmap 系统调用或shmat和shmget系统调用来请求大页.

## transparent huge pages(THP)

> thp参数在`/sys/kernel/mm/transparent_hugepage`. enabled被设为always, madvise时, khugepaged进程会自动启动; 为never时, 自动关闭.

## 内存同页合并
合并完全相同的page, 依赖于:
- ksm : 实际扫描内存和合并分页
- ksmtuned : 控制ksm是否扫描内存和如何积极地扫描内存

## 内存映射
调用系统函数 mmap()的进程,会在其虚拟地址空间中创建一个新的内存映射. 映射分为两类:
1. 文件映射:将文件的部分区域映射入调用进程的虚拟内存. 映射一旦完成,对文件映射
内容的访问则转化为对相应内存区域的字节操作. 映射页面会按需自动从文件中加载.
1. 与文件无关的匿名映射,其映射页面的内容会被初始化为 0.

由某一进程所映射的内存可以与其他进程的映射共享. 达成共享的方式有二:
1. 两个进程都针对某一文件的相同部分加以映射
1. 是由 fork()创建的子进程自父进程处继承映射

当两个或多个进程共享的页面相同时,进程之一对页面内容的改动是否为其他进程所见呢?这
取决于创建映射时所传入的标志参数. 若传入标志为私有,则某进程对映射内容的修改对于其
他进程是不可见的,而且这些改动也不会真地落实到文件上;若传入标志为共享,对映射内容
的修改就会为其他进程所见,并且这些修改也会造成对文件的改动. 内存映射用途很多,其中
包括:以可执行文件的相应段来初始化进程的文本段、内存(内容填充为 0)分配、文件 I/O(即
映射内存 I/O)以及进程间通信(通过共享映射).

## swap
### swappiness
当kernel想释放一个分页时, 由两种选择:
1. 从进程的内存中换出一个分页(swap_tendency >= 100)
1. 从cache中丢弃一个分页(swap_tendency < 100)

```
// mappend_ratio : 物理内存使用的百分比
// distree : 衡量内核在释放内存时所需开销(0~100, 初始是0)
swap_tendency = mappend_ratio/2 + distree + vm_swappiness
```

在linux里面，swappiness的值的大小对如何使用swap分区是有着很大的联系的.两个极端: swappiness=0的时候表示最大限度使用物理内存，然后才是 swap空间, 会增加文件系统开销 ;swappiness＝100的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面. 简单的理解就是内存在使用到100-vm.swappiness时, 就开始出现有交换分区的使用. 对于ubuntu的默认设置，这个值等于60，建议修改为10. 具体这样做：

1. 查看你的系统里面的swappiness

       $ cat /proc/sys/vm/swappiness

2. 修改swappiness值为10

       $ sudo sysctl vm.swappiness=10

 这只是临时性的修改，在你重启系统后会恢复默认的60，所以，还要做一步：

       $ sudo gedit /etc/sysctl.conf

 在这个文档的最后加上这样一行:

       vm.swappiness=10

改成10后感觉开关机,运行都慢了许多,调为50即可.

swap分区:
- `<=4G`: 内存的2倍
- `>4G&&<=16G`: 内存大小
- `>16G`: 不设置swap

> db server建议使用大内存且关闭swap
> 永久禁用swap: 修改`/etcd/fstab`
> 临时禁用swap: `swapoff -a`

## 内存分配
一般情况下,C 程序使用 malloc 函数族在堆上分配和释放内存. 较之 brk()和 sbrk(),这些函数具备不少优点:
1. 属于 C 语言标准的一部分
1. 更易于在多线程程序中使用
1. 接口简单,允许分配小块内存
1. 允许随意释放内存块,它们被维护于一张空闲内存列表中,在后续内存分配调用时循环使用

`malloc()`会在堆上分配参数 size 字节大小的内存,并返回指向新分配内存起始位置处的指针(返回类型为 void*),其所分配的内存**未经初始化**.

> malloc()返回内存块所采用的字节对齐方式. 在大多数硬件架构上, 这实际意味着 malloc 是基于 8 字节或 16 字节边界来分配内存的, 这样适宜于高效访问任何类型的 C 语言数据结构.

一般情况下,free()并不降低 program break 的位置,而是将这块内存填加到空闲内存列表中,供后续的malloc()函数循环使用:
- 被释放的内存块通常会位于堆的中间,而非堆的顶部,因而降低 porgram break 是不可能的.
- 它最大限度地减少了程序必须执行的 sbrk()调用次数.
- 在大多数情况下,降低 program break 的位置不会对那些分配大量内存的程序有多少帮助,因为它们通常倾向于持有已分配内存或是反复释放和重新分配内存,而非释放
所有内存后再持续运行一段时间.

> 仅当堆顶空闲内存(是连续的)"足够"大的时候,free()函数的 glibc 实现会调用 sbrk()来降低 program break 的地址,至于"足够"与否则取决于 malloc 函数包行为的控制参数(128 KB 为典型值).

free()将内存块置于空闲列表之上时,是如何知晓内存块大小的: 当 malloc()分配内存块时,会额外分配几个字节来存放记录这块内存大小的整数值. 该整数位于内存块的起始处,而实际返回给调用者的
内存地址恰好位于这一长度记录字节之后.

内存分配应准守的规则:
1. 分配一块内存后,应当小心谨慎,不要改变这块内存范围外的任何内容 : 防止要free的内存大小被改变
1. 释放同一块已分配内存超过一次是错误的. Linux 上的 glibc 库经常报出分段错误(SIGSEGV 信号) : 防止已重新分配的内存被free
1. 若非经由 malloc 函数包中函数所返回的指针,绝不能在调用 free()函数时使用
1. 在编写需要长时间运行的程序时,出于各种目的,如果需要反复分配内存,那么应当确保释放所有已使用完毕的内存, 否则会memory leak

### malloc选项
- tcmalloc
- jemalloc

### 在堆上分配内存的其他方法
用 calloc()和 realloc()分配内存. 与 malloc()不同,calloc()会将已分配的内存初始化为 0.
使用 calloc()或 realloc()分配的内存应使用 free()来释放.

分配对齐的内存(在于分配内存时,起始地址要与 2 的整数次幂边界对齐): memalign()和 posix_memalign().

### 在栈上分配内存:alloca()
alloca()分配内存的速度要快于 malloc(),因为编译器将 alloca()作为内联代码处理,并通过直接调整堆栈指针来实
现. 此外,alloca()也不需要维护空闲内存块列表, 且由 alloca()分配的内存随栈帧的移除而自动释放.