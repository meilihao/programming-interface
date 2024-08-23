# 内存
ref:
- [替 swap 辩护：常见的误解](https://farseerfc.me/zhs/in-defence-of-swap.html)

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

	被共享的是物理内存,并不是虚拟内存
- 便于实现内存保护机制

	可对不同进程的页表条目(共享ram时)进行标记,以表示相关页面内容是可读、可写、可执行亦或是这些保护措施的组合

- 程序员和编译器, 链接器之类的工具不用关心程序的内存布局

## 概念
内存的通道数，实际上是一种内存的带宽加速技术, 使得带宽倍增. 通常pc是双通道, 服务器可能是3/4/8通道.
![](/misc/img/os/FNdSBT.png)

双通道，就是在cpu里设计两组内存控制器，这两个内存控制器可相互独立工作，每个控制器控制一个内存通道.


内存Interleave: 把内存打散了平均分配在多根DIMM上，进行交错，从根本上让多通道利用起来，也叫做Channel Interleaving（和Rank interleaving不同）, 需要设置biso开启.

> 相同的, ssd,pcie也有类似的通道概念.

## 内存模式
ref:
- [**深入理解 Linux 物理内存分配全链路实现**](https://www.51cto.com/article/743324.html)
- [kmalloc分配物理内存与高端内存映射--Linux内存管理(十八) ](https://www.cnblogs.com/linhaostudy/p/10183797.html)

Linux支持FLATMEM、DISCONTIGMEM和SPARSEMEM等多种模式,还有一种SPARSEMEM_VMEMMAP是优化版,系统有足够的资源时启用,一般为前三种.

FLATMEM也就是FlatMemory,平面内存模式, DISCONTIGMEM和SPARSEMEM适用于NUMA或者内存热插拔的情形.

Flat Memory模式适用于大多数情形,它把内存看作是连续的,即使中间有hole,也会将hole计算在物理页内。由于管理内存需要利用数据结构记录所有物理页的使用情况,所以它的劣势是当可用的物理内
存段中存在较大比例的hole时,会浪费一些空间.

DISCONTIGMEM模式(Discontiguous Memory,非连续内存模式) = FLATMEM + 把每个内部连续的内存段分开管理而不计算hole.

```c
// https://elixir.bootlin.com/linux/v6.6.28/source/include/linux/mmzone.h#L1265
/*
 * On NUMA machines, each NUMA node would have a pg_data_t to describe
 * it's memory layout. On UMA machines there is a single pglist_data which
 * describes the whole memory.
 *
 * Memory statistics and page replacement data structures are maintained on a
 * per-zone basis.
 */
typedef struct pglist_data {
	/*
	 * node_zones contains just the zones for THIS node. Not all of the
	 * zones may be populated, but it is the full list. It is referenced by
	 * this node's node_zonelists as well as other node's node_zonelists.
	 */
	struct zone node_zones[MAX_NR_ZONES]; // 节点包含的zone的集合

	/*
	 * node_zonelists contains references to all zones in all nodes.
	 * Generally the first zones will be references to this node's
	 * node_zones.
	 */
	struct zonelist node_zonelists[MAX_ZONELISTS];

	int nr_zones; /* number of populated zones in this node */
#ifdef CONFIG_FLATMEM	/* means !SPARSEMEM */
	struct page *node_mem_map; // 节点包含的物理内存页对应的page结构体集合
#ifdef CONFIG_PAGE_EXTENSION
	struct page_ext *node_page_ext;
#endif
#endif
#if defined(CONFIG_MEMORY_HOTPLUG) || defined(CONFIG_DEFERRED_STRUCT_PAGE_INIT)
	/*
	 * Must be held any time you expect node_start_pfn,
	 * node_present_pages, node_spanned_pages or nr_zones to stay constant.
	 * Also synchronizes pgdat->first_deferred_pfn during deferred page
	 * init.
	 *
	 * pgdat_resize_lock() and pgdat_resize_unlock() are provided to
	 * manipulate node_size_lock without checking for CONFIG_MEMORY_HOTPLUG
	 * or CONFIG_DEFERRED_STRUCT_PAGE_INIT.
	 *
	 * Nests above zone->lock and zone->span_seqlock
	 */
	spinlock_t node_size_lock;
#endif
	unsigned long node_start_pfn; // 节点包含的物理内存的起始页页框号
	unsigned long node_present_pages; /* total number of physical pages */ // 节点包含的物理内存页数=node_spanned_pages - hole所占页数
	unsigned long node_spanned_pages; /* total size of physical page
					     range, including holes */ // 节点包含的物理内存区间, 包含中间的hole
	int node_id;
	wait_queue_head_t kswapd_wait;
	wait_queue_head_t pfmemalloc_wait;

	/* workqueues for throttling reclaim for different reasons. */
	wait_queue_head_t reclaim_wait[NR_VMSCAN_THROTTLE];

	atomic_t nr_writeback_throttled;/* nr of writeback-throttled tasks */
	unsigned long nr_reclaim_start;	/* nr pages written while throttled
					 * when throttling started. */
#ifdef CONFIG_MEMORY_HOTPLUG
	struct mutex kswapd_lock;
#endif
	struct task_struct *kswapd;	/* Protected by kswapd_lock */
	int kswapd_order;
	enum zone_type kswapd_highest_zoneidx;

	int kswapd_failures;		/* Number of 'reclaimed == 0' runs */

#ifdef CONFIG_COMPACTION
	int kcompactd_max_order;
	enum zone_type kcompactd_highest_zoneidx;
	wait_queue_head_t kcompactd_wait;
	struct task_struct *kcompactd;
	bool proactive_compact_trigger;
#endif
	/*
	 * This is a per-node reserve of pages that are not available
	 * to userspace allocations.
	 */
	unsigned long		totalreserve_pages;

#ifdef CONFIG_NUMA
	/*
	 * node reclaim becomes active if more unmapped pages exist.
	 */
	unsigned long		min_unmapped_pages;
	unsigned long		min_slab_pages;
#endif /* CONFIG_NUMA */

	/* Write-intensive fields used by page reclaim */
	CACHELINE_PADDING(_pad1_);

#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
	/*
	 * If memory initialisation on large machines is deferred then this
	 * is the first PFN that needs to be initialised.
	 */
	unsigned long first_deferred_pfn;
#endif /* CONFIG_DEFERRED_STRUCT_PAGE_INIT */

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
	struct deferred_split deferred_split_queue;
#endif

#ifdef CONFIG_NUMA_BALANCING
	/* start time in ms of current promote rate limit period */
	unsigned int nbp_rl_start;
	/* number of promote candidate pages at start time of current rate limit period */
	unsigned long nbp_rl_nr_cand;
	/* promote threshold in ms */
	unsigned int nbp_threshold;
	/* start time in ms of current promote threshold adjustment period */
	unsigned int nbp_th_start;
	/*
	 * number of promote candidate pages at start time of current promote
	 * threshold adjustment period
	 */
	unsigned long nbp_th_nr_cand;
#endif
	/* Fields commonly accessed by the page reclaim scanner */

	/*
	 * NOTE: THIS IS UNUSED IF MEMCG IS ENABLED.
	 *
	 * Use mem_cgroup_lruvec() to look up lruvecs.
	 */
	struct lruvec		__lruvec;

	unsigned long		flags;

#ifdef CONFIG_LRU_GEN
	/* kswap mm walk data */
	struct lru_gen_mm_walk mm_walk;
	/* lru_gen_folio list */
	struct lru_gen_memcg memcg_lru;
#endif

	CACHELINE_PADDING(_pad2_);

	/* Per-node vmstats */
	struct per_cpu_nodestat __percpu *per_cpu_nodestats;
	atomic_long_t		vm_stat[NR_VM_NODE_STAT_ITEMS];
#ifdef CONFIG_NUMA
	struct memory_tier __rcu *memtier;
#endif
#ifdef CONFIG_MEMORY_FAILURE
	struct memory_failure_stats mf_stats;
#endif
} pg_data_t;
```

内核得到的内存不一定是连续的,也不一定全部由内核使用。系统初始化阶段,内核会使用某些机制(如bios e820/uefi GetMemoryMap)获得系统当前的内存配置情况, linux可通过/sys/firmware/memmap或dmesg的"BIOS-provided physical RAM map"查看.

e820描述的内存段共有6种类型,但只有E820_RAM和E820_RESERVED_KERN两种属于log中的usable,其他的几种并不归内核管理, 其他各段都有特殊的用途.

内核提供了memblock机制来管理可用内存, 每一段内存都有一个memblock_region对象与之对应。node的pglist_data结构体初始化的时候,calculate_node_totalpages函数会根据两段内存和各zone的边界计算得到node_spanned_pages和node_present_pages.

之后在alloc_node_mem_map函数中,根据node_start_pfn和node_spanned_pages的值计算出最终需要管理的内存的页数(主要是对
齐调整)。页数就是该节点page对象的个数,据此申请内存,将内存地址赋值给node_mem_map字段,page对象就存在这里. node的page对象本质是一个数组,这个数组对应
着Flat Memory模式的整个平面物理内存,所以内存的hole也需要有page结构体与之对应.

节点的内存根据使用情况划分为多个区域,每个区域都对应一个zone对象. zone最多分为ZONE_DMA、ZONE_DMA32、ZONE_NORMAL、ZONE_HIGHMEM、ZONE_MOVABLE和
ZONE_DEVICE六种。ZONE_DMA32用在64位系统上 , ZONE_MOVABLE是为了减少内存碎片化引入的虚拟zone.

ZONE_DMA的内存区间为低16M,ZONE_NORMAL的内存区间为16M到max_low_pfn页框的地址,ZONE_HIGHMEM的内存区间为max_low_pfn 到 max_pfn.

max_pfn等于最大的页框号,不考虑预留HighMemory的情况下,如果它大于MAXMEM_PFN(MAXMEM对应的页框) ,
max_low_pfn就等于MAXMEM_PFN。如果它小于MAXMEM_PFN,max_low_pfn则等于max_pfn,这样就没有ZONE_HIGHMEM了.

page是zone的下一级单位,属于zone。但所有的page对象,都以数组的形式统一放在了pglist_data的node_mem_map字段指向的内存中.

除了碎片化,还要考虑效率,内核在不同阶段采用的内存管理方式也不同,主要分为memblock和buddy系统两种.

并不是所有的usable部分内存都直接归内核管理. 除了memblock和slab系统外,还有一个更高的级别是启动程序,比如
grub。grub可以通过参数(比如“mem=2048M”即高于2048M的部分不归内核管理)来限制内核可以管理的内存上限. 同样,memblock也可以扣留一部分内存,剩下的部分才轮到buddy系统管理.

> grub扣下的内存可给修改grub的人用的,这部分内存可以在程序中映射、独享.

扣除了grub预留的内存后,所有用途为usable的内存块都会默认进入可用内存块数组(简称memory数组),有一些模块也会选择预留一部分内存供其使用,这些块会进入被预留的内存块数组(简称reserve数组).

内核以memblock_region结构体表示一块,它的base和size字段分别表示块的起始地址和大小. 块以数组的形式进行管理,该数组由memblock_type结构体的regions字段表示。内核共有两个memblock_type对象(也可以理解为有两个由内存块组成的数组),一个表示可用的内存,一个表示被预留的内存,分别由memblock结构体的memory和reserved字段表示.

memblock函数表:
- memblock_add: 将内存块加入memory数组
- memblock_remove: 将内存块从memory数组中删除, 很少使用
- memblock_reserve: 预留内存块, 加入reserve数组
- memblock_free: 释放内存块, 从reserve数组删除

memblock是内存管理的第一个阶段,也是初级阶段,buddy系统会接替它的工作继续内存管理。它与buddy系统交接是在mem_init函数
中完成的,标志为after_bootmem变量置为1。mem_init函数会调用set_highmem_pages_init和memblock_free_all分别释放highmem和
lowmem到buddy系统.

块被加入reserve数组的时候并未从memory数组删除,只有在memory数组中,且不在reserve数组中的块才会进入buddy系统. 与
grub一样,被预留的内存块也是给预留内存块的模块使用的,模块自行负责内存的映射。所以,buddy系统管理的内存是经过grub和
memblock"克扣"后剩下的部分.

> 伙伴系统只会维护空闲的块,已经分配出去的块不再属于伙伴系统,只有被释放后才会重新进入伙伴系统.

> memblock预留内存时必须在memblock切换到buddy系统之前调用memblock_reserve

内核的buddy(伙伴)系统也是以块来管理内存的,不过块的大小并不是任意的,块以页为单位,仅有2^0
,2^1, ...2^10,共11(MAX_ORDER)种(1页,2页......1024页,4K,8K...4M)大小,在内核中以阶(order)来表示块的大小,所以阶有0, 1...10共11种。内存申请优先从小块开始是自然的,释放一个块的时候,如果它的伙伴也处于空闲状态,那么就将二者合并成一个更大的块。合并后
的块,如果它的伙伴也空闲,就继续合并,依次类推。另外,块包含的内存必须是连续的,不能有洞(hole).

页(page)的上一级是zone,所以块是由zone来记录的,zone的free_area字段是一个大小等于MAX_ORDER的free_area结构体类型的
数组,数组的每一个元素对应伙伴系统当前所有大小为某个阶的块,阶的值等于数组下标. free_area的free_list字段也是一个数组,数组的
每一个元素都是同种迁移(MIGRATE)类型的块组成的链表的头,它的nr_free字段表示空闲的块的数量.

这样,zone内同阶同迁移类型的块都在同一个链表上,比如阶等于2、迁移类型为type(比如MIGRATE_MOVABLE)的块,都在`zone-
>free_area[2]. free_list[type]`链表上,另外,阶等于2的块的数量为`zone->free_area[2]. nr_free个`.

一个块可以用它的第一个页框和阶表示,即page和order。page的page_type字段的PG_buddy标志清零(不是置位)时表示它代表的块在
伙伴系统中,private字段的值则表示块的阶。需要强调的是,只有块的第一个页框的page对象才有这个对应关系,中间页框的标志对伙伴
系统而言没有意义.

page的order函数:
- PageBuddy: page表示的块是否在buddy中
- page_order: page表示的块的阶
- set_page_order: 设置page的page_type和private, 块纳入buddy
- rmv_page_order: 设置page的page_type和private, 块从buddy删除

申请/释放page函数:
- alloc_page/alloc_pages: 申请一个或多个页
- free_page/free_pages: 释放一个或多个页
- `__free_page/__free_pages`: 释放一个或多个页

> XXX_page都是通过XXX_pages实现的,只不过传递给后者的order参数为0.

> `free_XXX 和 __free_XXX` 是有区别的, 与alloc_XXX对应的是`__free_XXX`,而不是前者(有点怪,可能是历史原因)。`__free_XXX`的参数为page和order,而free_XXX的参数为虚拟地址addr和order,addr经过计算得到page,然后调用`__free_XXX`继续释放内
存。由addr计算得到page,只有映射到直接映射区的Low Memory才可以满足,所以free_XXX只能用于直接映射区的LowMemory

所有的多页申请或释放内存的宏(XXX_pages)都有一个参数order表示申请的内存块的大小,也就是说申请的内存块只能是一整
块,比如申请32页内存,得到的块order为5,申请33页内存,得到的块order为6,共64页,这确实是一种浪费. 那申请33页内存,先申请
32页,再申请1页是否可行呢?某些情况下并不可行,申请一个完整的块,得到的是连续的33页内存,分多次申请得到的内存不一定是连续
的,要求连续内存的场景不能采用这种策略.

如果程序的应用场景并不要求连续内存,应该优先多次使用alloc_page,而不是alloc_pages,多次申请几个分散的
页,比一次申请多个连续的页,对整个系统要友善得多. 某些malloc lib(比如jemalloc)通常会从内核申请一大块内存再在应用侧自己管理内存.

alloc_pages最终调用`__alloc_pages_nodemask`实现,后者是伙伴系统页分配的核心.

gfp_mask是alloc_pages传递的第一个参数,它是按位解释的,包含了优先选择的zone和内存分配的行为信息,可以由多种标志组合而成:
```c
// https://elixir.bootlin.com/linux/v6.6.28/source/include/linux/gfp_types.h#L28
/* Plain integer GFP bitmasks. Do not use this directly. */
#define ___GFP_DMA		0x01u // DMA zone
#define ___GFP_HIGHMEM		0x02u // 最高为HIGHMEM zone
#define ___GFP_DMA32		0x04u
#define ___GFP_MOVABLE		0x08u
#define ___GFP_RECLAIMABLE	0x10u
#define ___GFP_HIGH		0x20u // 可使用紧急内存池
#define ___GFP_IO		0x40u // 可启动物理IO操作
#define ___GFP_FS		0x80u // 可使用文件系统的操作
#define ___GFP_ZERO		0x100u // 清零
/* 0x200u unused */
#define ___GFP_DIRECT_RECLAIM	0x400u // 可直接进入页面回收, 需要时进入睡眠
#define ___GFP_KSWAPD_RECLAIM	0x800u // 可以唤醒kswapd进行页面回收, kswapd是负责页面回收的内核线程
#define ___GFP_WRITE		0x1000u
#define ___GFP_NOWARN		0x2000u
#define ___GFP_RETRY_MAYFAIL	0x4000u
#define ___GFP_NOFAIL		0x8000u
#define ___GFP_NORETRY		0x10000u
#define ___GFP_MEMALLOC		0x20000u // 最高优先级地分配内存
#define ___GFP_COMP		0x40000u
#define ___GFP_NOMEMALLOC	0x80000u // 若未置位, 可次高优先级地分配内存
#define ___GFP_HARDWALL		0x100000u
#define ___GFP_THISNODE		0x200000u
#define ___GFP_ACCOUNT		0x400000u
#define ___GFP_ZEROTAGS		0x800000u
#ifdef CONFIG_KASAN_HW_TAGS
#define ___GFP_SKIP_ZERO	0x1000000u
#define ___GFP_SKIP_KASAN	0x2000000u
#else
#define ___GFP_SKIP_ZERO	0
#define ___GFP_SKIP_KASAN	0
#endif
#ifdef CONFIG_LOCKDEP
#define ___GFP_NOLOCKDEP	0x4000000u
#else
#define ___GFP_NOLOCKDEP	0
#endif

// https://elixir.bootlin.com/linux/v6.6.28/source/include/linux/gfp_types.h#L325
/**
 * DOC: Useful GFP flag combinations
 *
 * Useful GFP flag combinations
 * ----------------------------
 *
 * Useful GFP flag combinations that are commonly used. It is recommended
 * that subsystems start with one of these combinations and then set/clear
 * %__GFP_FOO flags as necessary.
 *
 * %GFP_ATOMIC users can not sleep and need the allocation to succeed. A lower
 * watermark is applied to allow access to "atomic reserves".
 * The current implementation doesn't support NMI and few other strict
 * non-preemptive contexts (e.g. raw_spin_lock). The same applies to %GFP_NOWAIT.
 *
 * %GFP_KERNEL is typical for kernel-internal allocations. The caller requires
 * %ZONE_NORMAL or a lower zone for direct access but can direct reclaim.
 *
 * %GFP_KERNEL_ACCOUNT is the same as GFP_KERNEL, except the allocation is
 * accounted to kmemcg.
 *
 * %GFP_NOWAIT is for kernel allocations that should not stall for direct
 * reclaim, start physical IO or use any filesystem callback.
 *
 * %GFP_NOIO will use direct reclaim to discard clean pages or slab pages
 * that do not require the starting of any physical IO.
 * Please try to avoid using this flag directly and instead use
 * memalloc_noio_{save,restore} to mark the whole scope which cannot
 * perform any IO with a short explanation why. All allocation requests
 * will inherit GFP_NOIO implicitly.
 *
 * %GFP_NOFS will use direct reclaim but will not use any filesystem interfaces.
 * Please try to avoid using this flag directly and instead use
 * memalloc_nofs_{save,restore} to mark the whole scope which cannot/shouldn't
 * recurse into the FS layer with a short explanation why. All allocation
 * requests will inherit GFP_NOFS implicitly.
 *
 * %GFP_USER is for userspace allocations that also need to be directly
 * accessibly by the kernel or hardware. It is typically used by hardware
 * for buffers that are mapped to userspace (e.g. graphics) that hardware
 * still must DMA to. cpuset limits are enforced for these allocations.
 *
 * %GFP_DMA exists for historical reasons and should be avoided where possible.
 * The flags indicates that the caller requires that the lowest zone be
 * used (%ZONE_DMA or 16M on x86-64). Ideally, this would be removed but
 * it would require careful auditing as some users really require it and
 * others use the flag to avoid lowmem reserves in %ZONE_DMA and treat the
 * lowest zone as a type of emergency reserve.
 *
 * %GFP_DMA32 is similar to %GFP_DMA except that the caller requires a 32-bit
 * address. Note that kmalloc(..., GFP_DMA32) does not return DMA32 memory
 * because the DMA32 kmalloc cache array is not implemented.
 * (Reason: there is no such user in kernel).
 *
 * %GFP_HIGHUSER is for userspace allocations that may be mapped to userspace,
 * do not need to be directly accessible by the kernel but that cannot
 * move once in use. An example may be a hardware allocation that maps
 * data directly into userspace but has no addressing limitations.
 *
 * %GFP_HIGHUSER_MOVABLE is for userspace allocations that the kernel does not
 * need direct access to but can use kmap() when access is required. They
 * are expected to be movable via page reclaim or page migration. Typically,
 * pages on the LRU would also be allocated with %GFP_HIGHUSER_MOVABLE.
 *
 * %GFP_TRANSHUGE and %GFP_TRANSHUGE_LIGHT are used for THP allocations. They
 * are compound allocations that will generally fail quickly if memory is not
 * available and will not wake kswapd/kcompactd on failure. The _LIGHT
 * version does not attempt reclaim/compaction at all and is by default used
 * in page fault path, while the non-light is used by khugepaged.
 */
#define GFP_ATOMIC	(__GFP_HIGH|__GFP_KSWAPD_RECLAIM)
#define GFP_KERNEL	(__GFP_RECLAIM | __GFP_IO | __GFP_FS)
#define GFP_KERNEL_ACCOUNT (GFP_KERNEL | __GFP_ACCOUNT)
#define GFP_NOWAIT	(__GFP_KSWAPD_RECLAIM)
#define GFP_NOIO	(__GFP_RECLAIM)
#define GFP_NOFS	(__GFP_RECLAIM | __GFP_IO)
#define GFP_USER	(__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
#define GFP_DMA		__GFP_DMA
#define GFP_DMA32	__GFP_DMA32
#define GFP_HIGHUSER	(GFP_USER | __GFP_HIGHMEM)
#define GFP_HIGHUSER_MOVABLE	(GFP_HIGHUSER | __GFP_MOVABLE | __GFP_SKIP_KASAN)
#define GFP_TRANSHUGE_LIGHT	((GFP_HIGHUSER_MOVABLE | __GFP_COMP | \
			 __GFP_NOMEMALLOC | __GFP_NOWARN) & ~__GFP_RECLAIM)
#define GFP_TRANSHUGE	(GFP_TRANSHUGE_LIGHT | __GFP_DIRECT_RECLAIM)
```
> `___GFP_DIRECT_RECLAIM`被置位的情况下,会睡眠,不允许睡眠的情景中不可使用,GFP_KERNEL隐含了`___GFP_DIRECT_RECLAIM`,慎重使用

`__alloc_pages_nodemask`先调用prepare_alloc_pages,根据参数计算得到ac、alloc_mask和alloc_flags,先以此调用get_page_from_freelist分配块,如果失败,调用`__alloc_pages_slowpath`,后者会根据gfp_mask
计算新的alloc_flags,重新调用get_page_from_freelist.

get_page_from_freelist会从用户期望的zone(ac->high_zoneidx)开始向下遍历,例如用户期望分配ZONE_HIGHMEM的内存,函数会按
照ZONE_HIGHMEM、ZONE_NORMAL和ZONE_DMA的优先级分配内存:
1. zone_watermark_fast判断当前zone是否能够分配该内存块,由内存块的order和zone的水位线(watermark)决定。

zone的水位线由它的_watermark字段表示,该字段是个数组,分别对应WMARK_MIN、WMARK_LOW和WMARK_HIGH三种情况下的水位
线,它们的值默认在模块初始化的时候设定,也可以运行时改变。水位线表示zone需要预留的内存页数,内核针对不同紧急程度的
内存需求有不同的策略,它会预留一部分内存应对紧急需求,根据zone分配内存后是否满足水位线的最低要求来决定是否从该zone分配
内存.

紧急程度共分为四个等级,分别是默认等级、ALLOC_HIGH、ALLOC_HARDER和ALLOC_OOM,各等级对应的最低剩余内存页数
分别为`watermark、watermark/2、watermark*3/8和watermark/4`。例如,zone的watermark值为32,当前zone剩余空闲内存为40页,申请的内存块的order为4,分配了内存块之后剩余24页(40-16);如果是默认等级,需要剩余32页,分配失败;如果是ALLOC_HIGH,只需剩余16页
(32/2),分配成功。

紧急程度和进程优先级与gfp_mask的对应关系:
- ALLOC_HIGH: `__GFP_HIGH`
- ALLOC_HARDER: `__GFP_ATOMIC`且!`__GFP_NOMEMALLOC`或者实时进程且不在中断上下文
- ALLOC_OOM: Out of Memory
- ALLOC_NO_WATERMARKS: `__GFP_MEMALLOC`
- 默认: 其他

1. 除了以上四个等级,还有一个额外的ALLOC_NO_WATERMARKS,它表示内存申请不受watermark的限
制
1. 找到了合适的zone或者在ALLOC_NO_WATERMARKS置位的情况下,调用rmqueue分配块,rmqueue分两种情况处理

第一种情况,order等于0(只申请一页),内核并不立即使用`zone->free_area[0]. free_list[migratetype]`链表上的块,而是调用
rmqueue_pcplist函数先查询per_cpu_pages结构体(简称pcp)维护的链表是否可以满足内存申请。pcp对象由zone->pageset的pcp字段表示,
其中zone->pageset是每cpu变量.

pclist如MIGRATE_UNMOVABLE、MIGRATE_RECLAIMABLE和MIGRATE_MOVABLE。它们相当于几个order为0的块的缓存池,内核
直接从池中分配块,如果池中的资源不足,则向伙伴系统申请,一次申请pcp->batch块。很显然,它的设计是基于其他模块申请一页内存的
次数很多的假设,这样一次从伙伴系统申请多个order为0的块缓存下来,就不需要每次都经过伙伴系统,有利于提高内存申请的效率.

第二种情况,order大于0,调用__rmqueue从伙伴系统申请块(实际上order等于0时,如果pcp池中的资源不足也会调用
`__rmqueue`), 它从当前的order开始向上查询`zone->free_area[order].free_list[migratetype]`链表,如果某个order上的链表不
为空,则分配成功。如果最终使用的order(记为order_bigger)大于申请的块的order,则拆分使用的块,生成大小等于order+1, order+2,...,
order_bigger-1的块各一个,大小等于order的块两个(推论2),其中一个返回给用户.

## 线性空间划分
32位空间划分:  用户[0, 0xC0000000), 内核[0xC0000000, 0xFFFFFFFF), 可通过Memory split编译选项修改. 它划分的方式决定了内核使
用 线 性 地 址 的 起 始 点 , PAGE_OFFSET , 该 宏 在 x86 上 等 同 于CONFIG_PAGE_OFFSET,后者就由选择的划分方式确定. 

> PAGE_OFFSET是内核线性地址空间的起始,并不是物理内存的开始,物理内存映射到该空间即可访问

X86_64 上 PAGE_OFFSET 在 五 级 页 表 使 能 的 情 况 下 , 值 等 于0xff11000000000000,否则等于0xffff888000000000.

### 32位
内核会将一部分内存(小于896M)直接映射到线性空间,这部分就是Low Memory,超出的部分必须另做映射才能访问,叫High Memory. 是否使能High Memory由宏CONFIG_HIGHMEM标示.

CONFIG_HIGHMEM 宏 使 能 的 情 况 下 包 含 两 种 情 况 , HIGHMEM4G和HIGHMEM64G,内存小于4G的时候HIGHMEM4G设
为真,内存大于4G的时候HIGHMEM64G设为真,会使能PAE;另外即 使 内 存 小 于 896M , 也 可 以 将 其 设 为 真 , 通 过 参 数 设 置
highmem_pages的值,内核也会留出相应的页作为High Memory.

线性空间变量表:
- high_memory: low memory和high memory的分界,  线性地址一般是`high_memory=max_low_pfn<<PAGE_SIZE+PAGE_OFFSET`
- PAGE_SIZE: 表示一页内存使用的二进制位数, 一般是12
- PAGE_OFFSET: 内核线性空间的起始地址
- FIXADDR_TOP: 固定映射的结尾位置
-  FIXADDR_START: 固定映射的开始位置
- VMALLOC_END: 动态映射区的结尾
- VMALLOC_START: 动态映射区的开始

1G的线性地址空间 [0xC0000000, 0xFFFFFFFF],最高的4K不用, 由FIXADDR_TOP开始向下算起,FIXADDR的TOP和START之间是固
定映射区。PKMAP_BASE和FIXADDR之间是永久映射区和CPU Entry区 ( 存 放 cpu_entry_area 对 象 ) 。 VMALLOC_END 和
VMALLOC_START之间是动态映射区,之前介绍的ioremap获取的虚拟内存区间就属于这个区。接下来8M的hole没有使用,目的是越界检
测,_VMALLOC_RESERVE默认为128M减去8M,实际上动态映射区为120M.

,high_memory是计算得来的,它和PAGE_OFFSET之间的间隔可能更小,但它和VMALLOC_START之间的8M是固定的,也就是说,动态映射区可能比120M大.

以上几个区域是必需的,1G的空间减去这些剩下约896M,这就是896这个数字的来源,以MAXMEM表示,该区间称为直接映射区.

线性地址用户空间和内核空间比例为3∶1,3G的用户空间每个进程独立拥有,互不影响,也就是说进程负责维护自己的这部分页表,称
为用户页表 ,它只需要映射自己使用的那部分内存,并不需要维护整套用户页表. 内核空间的1G则不同,基本所有进程共有,也就是说它们的页表的内核部分(内核页表)很多情况下是
相同的,属于公共部分. 

#### 直接映射区
直接映射区理论上最大为MAXMEM,所谓的直接映射非常简单 , 映 射 后 的 虚 拟 地 址 va 等 于 物 理 地 址 pa 加
PAGE_OFFSET(0xC0000000)。物理地址从最小页pfn=0开始,依次按照pa+PAGE_OFFSET → va的方式映射到该区间.

从pfn=0开始,一直映射到没有多余空间或者没有物理内存为止。所以,如果系统本身内存不足MAXMEM(896M),且没有特意预留
High Memory的情况下,全部的物理内存(不包括MMIO)都会映射到直接映射区。如果系统内存大于896M,或者预留了High Memory,内
存不能(不会)全部映射到该区.

直 接 映 射 区 是 唯 一 的 Low Memory 映 射 的 区 域 , 页 框 0 到 页 框max_low_pfn映射到该区间。映射一旦完成,系统运行期间不会改
变,这是相对于其他区的优势;从下面几个区的分析中会看到,High Memory映射区域在运行期间是可以改变的,所以系统中需要一直稳定
存在的结构体和数据只能使用直接映射区的内存,比如page对象.

#### 动态映射区
VMALLOC_START和VMALLOC_END之间的120M空间为动态映射区,它是内核线性空间中最灵活的一个区。其他几个区都有固定的
角色,它们不能满足的需求,都可以由动态映射区来满足,常见的ioremap、mmap一般都需要使用它。
使用动态映射区需要申请一段属于该区域的线性区间,内核提供了get_vm_area函数族来满足该需求,它们的区别在于参数不同,但最
终都通过调用__get_vm_area_node函数实现。

内核提供了vm_struct结构体来表示获取的线性空间及其映射情况.

```c
// https://elixir.bootlin.com/linux/v6.6.28/source/include/linux/vmalloc.h#L49
struct vm_struct {
	struct vm_struct	*next; // 下一个vm_struct, 组成链表
	void			*addr; // 线性空间的起始地址
	unsigned long		size; // 大小
	unsigned long		flags;
	struct page		**pages; // page对象组成的动态数组
#ifdef CONFIG_HAVE_ARCH_HUGE_VMALLOC
	unsigned int		page_order;
#endif
	unsigned int		nr_pages; // 包含的页数
	phys_addr_t		phys_addr; // 对应的物理地址
	const void		*caller;
};

struct vmap_area {
	unsigned long va_start;
	unsigned long va_end;

	struct rb_node rb_node;         /* address sorted rbtree */
	struct list_head list;          /* address sorted list */

	/*
	 * The following two variables can be packed, because
	 * a vmap_area object can be either:
	 *    1) in "free" tree (root is free_vmap_area_root)
	 *    2) or "busy" tree (root is vmap_area_root)
	 */
	union {
		unsigned long subtree_max_size; /* in "free" tree */
		struct vm_struct *vm;           /* in "busy" tree */
	};
	unsigned long flags; /* mark type of vm_map_ram area */
};
```

在`__get_vm_area_node`的参数中,start和end表示用户希望的目标线性区间所在的范围,get_vm_area传递的参数为VMALLOC_START
和VMALLOC_END。内核会将已使用的动态映射区的线性区间记录下来 , 每 一 个 区 间 以 vmap_area 结 构 体 表 示 。 vmap_area 结 构 体 除 了va_start和va_end字段表示区间的起始外,还有两个组织各个vmap_area对象的字段:rb_node和list.

一 个 vmap_area 对 象 既 在 以 vmap_area_root 为 根 的 红 黑 树 中(rb_node),又以list字段链接,rb_node字段的作用是快速查找,list
字段则是根据查找的结果继续遍历(各区间是按照顺序链接的).

`__get_vm_area_node`会查找一个没有被占用的合适区间,如果找到则将该区间记录到红黑树和链表中,然后利用得到的vmap_area对象
给vm_struct对象赋值并返回。vm_struct结构体是其他模块可见的,vmap_area结构体是动态映射区内部使用的。

申请到vm_struct线性区间后,就可以将物理内存映射到该区域, ioremap使用的是ioremap_page_range,有的模块使用map_vm_area,
remap_vmalloc_range等.

最后,free_vm_area和remove_vm_area可以用来取消映射并释放申请的区间,free_vm_area还会释放vm_struct对象所占用的内存.

#### 永久映射区
永 久 映 射 区 的 线 性 地 址 起 于 PKMAP_BASE , 大 小 为LAST_PKMAP个页,开启了PAE的情况下为512,否则为1024,也就
是一页页中级目录表示的大小(2M或者4M).

内核提供了kmap函数将一页内存映射到该区,kunmap函数用来取消映射。kmap的参数为page结构体指针,如果对应的页不属于High Memory,则直接返回页对应的虚拟地址vaddr(等于paddr+PAGE_OFFSET),否则
内核会在该线性区内寻址一个未被占用的页,作为虚拟地址,并根据page和虚拟地址建立映射。

传递的参数决定了kmap一次只映射一页,也就相对于永久映射区被分成了以页为单位的“坑”,内核利用数组来管理一个个“坑”,
pkmap_count数组就是用来记录“坑”的使用次数的,使用次数为0的表示未被占用。

kmap可能会引起睡眠,不可以在中断等环境中使用.

#### 固定映射区
固定映射区起始于FIXADDR_TOT_START,终止于FIXADDR_TOP,分为多个小区间,它们的边界由fixed_addresses定义,每一个小区间都有特定的用途,如ACPI等.

所有的小区间中,有一个比较特殊,它有一个特殊的名字叫临时映 射 区 。 它 在 固 定 映 射 区 的 偏 移 量 为 FIX_KMAP_BEGIN 和
FIX_KMAP_END,区间大小等于KM_TYPE_NR*NR_CPUS页.

从它的大小可以看出,每个CPU都有属于自己的临时映射区,大小为KM_TYPE_NR页。内核提供了kmap_atomic、kmap_atomic_pfn和
kmap_atomic_prot函数来映射物理内存到该区域,将获得的物理内存的 page 对 象 的 指 针 或 者 pfn 作 为 参 数 传 递 至 这 些 函 数 即 可 ,
kunmap_atomic函数用来取消映射.

与永久映射区一样,如果对应的物理页不属于High Memory,则直接返回页对应的虚拟地址vaddr(等于paddr + PAGE_OFFSET),否
则内核会找到一页未使用的线性地址,并建立映射.

临时映射区的区间很小,每个CPU只占几十页,它的管理也很简单。内核使用一个每CPU变量__kmap_atomic_idx来记录当前已使用的
页的号码,映射成功变量加1,取消映射变量减1。这意味着小于该变量的号码对应的页都被认为是已经映射的。另外,取消映射的时候,
内核直接取消变量当前值对应的映射,并不是传递进来的虚拟地址对应的映射,所以保证二者一致是程序员的责任。

临时映射区的管理设计得如此简单,优点是快速,kmap_atomic比kmap快很多。它的另一个优点是不会睡眠,kmap会睡眠,所以它的适
应性更广.

临时映射区的使用是有限制的: 。kmap_atomic会调用preempt_disable禁内核抢占,kunmap_atomic来重新使能它,所以基本上只能用来临时存放一些数据,数据处理完毕立即释放.

### mmap
mmap使用的不是内核线性空间,是用户线性空间. 

mmap用来将文件或设备映射进内存,映射之后不需要再使用read/write等操作文件,可以像访问普通内存一样访问它们,可以明显
地提高访问效率。mmap用来建立映射,munmap用来取消映射.

```c
#include <sys/mman.h>

void *mmap(void addr[.length], size_t length, int prot, int flags,
          int fd, off_t offset);
int munmap(void addr[.length], size_t length);
```
> mmap的offset必须是物理页大小`(sysconf(_SC_PAGE_SIZE))`的整数倍

mmap将文件或设备(fd)从offset开始, 到offset+length结束的区间映射到addr开始的虚拟内存中,并将addr返
回,映射后的内容访问权限由prot决定.

prot不能与文件的打开标志冲突,一般是PROT_NONE或者读写执行三种标志的组合:
- PROT_NONE: 映射的内容不可访问
- PROT_EXEC: 可执行
- PROT_READ: 可读
- PROT_WRITE: 可写

flags 用 来 传 递 映 射 标 示 , 常 见 的 是 MAP_PRIVATE 和MAP_SHARED的其中一种,和以下多种标志的组合:
-  MAP_PRIVATE: 私有映射
- MAP_SHARED: 共享映射
- MAP_FIXED: 严格按照mmap的addr表示的虚拟地址进行映射
- MAP_ANONYMOUS: 匿名映射, 映射不与任何文件关联
- MAP_LOCKED: 锁定映射区的页面, 防止被交换出内存

私有和共享是相对于其他进程来说的,MAP_SHARED对映射区的更新对其他映射同一区域的进程可见,所做的更新也会体现在文件
中,可以使用msync来控制何时写回文件。MAP_PRIVATE采用的是写时复制策略,映射区的更新对其他映射同一区域的进程不可见,所做
的更新也不会写回文件.

以上对prot和flags的描述,并不适用于所有文件,比如有些文件只接受MAP_PRIVATE,不接受MAP_SHARED,或
者相反。应用程序并不能随意选择参数,普通文件一般在不违反自身权限的情况下可以自由选择,设备文件等特殊文件可以接受的参数由
驱动决定.

mmap使用的线性区间管理与内核的线性区间管理是不同的,内核的线性区间是进程共享的,mmap使用的线性区间则是进程自己的用户
线性空间,在不考虑进程间关系的情况下,进程的用户线性空间是独立的.

内核以vm_area_struct结构体表示一个用户线性空间的区域.
```c
// https://elixir.bootlin.com/linux/v6.6.28/source/include/linux/mm_types.h#L565
/*
 * This struct describes a virtual memory area. There is one of these
 * per VM-area/task. A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
	/* The first cache line has the info for VMA tree walking. */

	union {
		struct {
			/* VMA covers [vm_start; vm_end) addresses within mm */
			unsigned long vm_start; // 线性区间的开始
			unsigned long vm_end; // 结束.  实际区间=[vm_start, vm_end)
		};
#ifdef CONFIG_PER_VMA_LOCK
		struct rcu_head vm_rcu;	/* Used for deferred freeing. */
#endif
	};

	struct mm_struct *vm_mm;	/* The address space we belong to. */
	pgprot_t vm_page_prot;          /* Access permissions of this VMA. */ // 映射区域的保护方式

	/*
	 * Flags, see mm.h.
	 * To modify use vm_flags_{init|reset|set|clear|mod} functions.
	 */
	union {
		const vm_flags_t vm_flags; // 映射的标志
		vm_flags_t __private __vm_flags;
	};

#ifdef CONFIG_PER_VMA_LOCK
	/*
	 * Can only be written (using WRITE_ONCE()) while holding both:
	 *  - mmap_lock (in write mode)
	 *  - vm_lock->lock (in write mode)
	 * Can be read reliably while holding one of:
	 *  - mmap_lock (in read or write mode)
	 *  - vm_lock->lock (in read or write mode)
	 * Can be read unreliably (using READ_ONCE()) for pessimistic bailout
	 * while holding nothing (except RCU to keep the VMA struct allocated).
	 *
	 * This sequence counter is explicitly allowed to overflow; sequence
	 * counter reuse can only lead to occasional unnecessary use of the
	 * slowpath.
	 */
	int vm_lock_seq;
	struct vma_lock *vm_lock;

	/* Flag to indicate areas detached from the mm->mm_mt tree */
	bool detached;
#endif

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap interval tree.
	 *
	 */
	struct {
		struct rb_node rb; // 将它插入红黑树中
		unsigned long rb_subtree_last;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	struct list_head anon_vma_chain; /* Serialized by mmap_lock &
					  * page_table_lock */
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */

	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops; // 映射的相关操作

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units */ // 在文件内的偏移量
	struct file * vm_file;		/* File we map to (can be NULL). */ // 映射文件对应的file对象
	void * vm_private_data;		/* was vm_pte (shared mem) */

#ifdef CONFIG_ANON_VMA_NAME
	/*
	 * For private and shared anonymous mappings, a pointer to a null
	 * terminated string containing the name given to the vma, or NULL if
	 * unnamed. Serialized by mmap_lock. Use anon_vma_name to access.
	 */
	struct anon_vma_name *anon_name;
#endif
#ifdef CONFIG_SWAP
	atomic_long_t swap_readahead_info;
#endif
#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
#ifdef CONFIG_NUMA_BALANCING
	struct vma_numab_state *numab_state;	/* NUMA Balancing state */
#endif
	struct vm_userfaultfd_ctx vm_userfaultfd_ctx;
} __randomize_layout;
```

既 然进程 要管 理自己的线 性区间分配,就需 要管理它所有 的vm_area_struct对象。进程的mm_struct结构体有两个字段完成该任务,
mmap字段是进程的vm_area_struct对象组成的链表的头,mm_rb字段是所有vm_area_struct对象组成的红黑树的根。内核使用链表来遍历对象 , 使 用 红 黑 树 来 查 找 符 合 条 件 的 对 象 。 vm_area_struct 结 构 体 的
vm_next和vm_prev字段用来实现链表,vm_rb字段用来实现红黑树.

从应用的角度看,mmap成功返回就意味着映射完成,但实际上从驱动的角度并不一定如此。很多驱动在mmap中并未实现内存映射,应
用程序访问该地址时触发异常后,才会做最终的映射工作,这就用到了vm_ops字段.

mmap使用的系统调用与平台有关,但最终都调用ksys_mmap_pgoff函数实现。需要说明的是,mmap传递的offset参数会
在调用的过程中转换成页数(offset/MMAP_PAGE_UNIT),一般由glibc完成,所以内核获得的参数已经是以页为单位的(pgoff可以理解
为page offset)。

ksys_mmap_pgoff调用vm_mmap_pgoff,经过参数检查后,最终调用do_mmap函数完成mmap.

do_mmap逻辑:
第一步当然是做合法性检查和调整,映射的长度len(区间大小)被调整为页对齐,所以mmap返回后,实际映射的长度可能比要求的要大。比如某个mmap调用中,offset和length等于0和0x50,实际映射为0
和0x1000.

检查完毕就开始找一个合适的线性区间,也就是找“坑”,通过get_unmapped_area 实 现 . get_unmapped_area优先调用文件的get_unmapped_area操作,如果文件没有定义,则使用当前进程的get_unmapped_area函数(current->mm->get_unmapped_area),该函数与平台有关. 如果参数addr等于0,内核会根据当前进程的mm_struct对象的mm_rb字段指向的进程所有的vm_area_struct对象组成的红黑树查找一段合适的区域,如果addr不等于0,且(addr, addr + len)没有被占用,则返回addr,否则与addr等于0等同。如果参数flags的MAP_FIXED标示
被置位,则直接返回addr.

> 在MAP_FIXED被置位的情况下, 不检查区间是否被占用就直接返回的原因: 为MAP_FIXED有个隐含属性,就是“抢地盘”. 如果得到的区间已被占用,内核会在mmap_region函数中取消该区间
的映射,将区间让出来. 所以MAP_FIXED并不推荐使用, 但有些情况下,它却是必需的, 比如brk.

> MAP_FIXED的另外一个版本MAP_FIXED_NOREPLACE会友好很多。如果指定的区间已经被占用,MAP_FIXED_NOREPLACE会直接返回错误(EEXIST)

然后计算vm_flags,可以看到,如果文件没有打开写权限,VM_MAYWRITE和VM_SHARED标示会被清除;如果文件没有打开读权限,则直接返回EACCES。实际
上,在一般情况下共享隐含着“写”,私有隐含着“读”。当然,从代码逻辑上看,即使文件没有打开写权限,MAP_SHARED也可以成功,但是映射的页并没有写权限`[vm_flags = calc_vm_prot_bits(prot) | (...)`,
因为文件没有打开写权限,prot也就没有PROT_WRITE标志,所以vm_flags也不会有VM_WRITE.

完成了以上准备工作,就可以调用mmap_region进行映射.

mmap_region主要做三件事,初始化vm_area_struct对象、完成映射、将对象插入链表和红黑树。不考虑匿名映射,mmap最终是靠驱动提供的文件的mmap操作实现的(file->f_op->mmap),也
就是说mmap究竟产生了什么效果最终是由驱动决定的.

驱动完成映射的方式一般有以下几类:
1. 驱动有属于自己的物理内存,多为MMIO,直接调用内核函数完成映射
1. 第二类与第一类类似, 但物理内存不是现成的, 需要先申请内存然后做映射
1. 第三类,驱动的mmap并不提供映射操作,由异常触发实际映射动作

映射后的页面的访问权限(保护标志)由mmap的prot参数决定. 在mmap_region函数中,参考了prot和flags处理得到的vm_flags经过计算得到`(vma->vm_page_prot=vm_get_page_prot(vm_flags))`,vma->vm_page_prot就是最终用来设置页访问权限的。在完成映射的最后过程中(vm_iomap_memory等),vma->vm_page_prot被用来设置页表的属性.

mmap返回后,实际上并没有完成内存映射的动作,返回的只是没有物理内存与之对应的虚拟地址. 稍后访问该地址会导致内存访问异常, 内核处理该异常则会回调驱动的vm_operations_struct的fault操作,
驱动的fault操作中,一般需要申请物理内存赋值给vm_fault的page字段并完成自身的逻辑,内核会完成虚拟内存和物理内存的映射。

必须使用brk的场景:
一个程序开始执行后,进程的堆内存的起始位置(start)就确定了。进程申请堆内存时,如果当前的堆内存可以满足申请,则返回虚
拟地址给进程,否则调用brk来扩大当前堆内存。内存释放的时候,也可以调用brk来缩小堆内存,进程当前的堆内存为[start, sbrk(0)]。
start不变,堆内存连续(虚拟地址),那么要想改变堆内存的大小,则只能改变程序间断点,而且用户空间必须计算新的程序间断点
后再调用brk。内核接收到的参数就是程序间断点表示的地址,这个地址是不允许改变的,否则堆内存就会混乱了,MAP_FIXED就派上用
场了。

最后,在内核中,brk并没有调用mmap来实现,但它实际上是一个简单的mmap.

几种不同的mmap映射方式:
使用文件映射,也就是非匿名映射,最终的映射由文件定义的mmap操作实现,有些文件在mmap函数中完成了最终的映射,接下来
可以正常访问内存(情况1),但很多文件仅仅在mmap函数中定义了`vma->vm_ops`,并没有映射内存,接下来访问内存会导致缺页异常,
进而调用`vma->vm_ops->fault`完成实际的映射.

情况1中 , 使用MAP_SHARED和MAP_PRIVATE映射的内存,得到的结果也不同,MAP_PRIVATE方式映射的物理页没有写权限,稍
后写内存会导致异常,这是COW的一种情况,MAP_SHARED方式映射的内存可读可写. 注意,“没有写权限”意思是暂时没有, 但是可能“可以有”。

匿名映射,映射后的内存访问与文件无关,使用MAP_SHARED方式映射时,内核会调用shmem_zero_setup函数创建一个文件来映射
内存,mmap返回后,并没有完整的内存映射,接下来的内存访问会导致 缺 页 异 常 , 最 终 由 vma->vm_ops->fault 完 成 映 射 。 使 用
MAP_PRIVATE方式映射并没有直接或者间接的文件操作,也就是没有内存映射,也没有vma->vm_ops,内存访问会导致缺页异常,由
do_anonymos_page函数完成内存映射.

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

分段与分页的优缺点 from 内存管理角度:
1. 从表示方式和状态确定角度考虑

	分页容易用位图表示
2. 内存碎片

	由于段的长度大小不一，更容易产生内存碎片.

	通过修改页表的方式, 就能让连续的虚拟页面映射到非连续的物理页面
3. 从内存和硬盘的数据交换效率考虑

	分段写回硬盘的时间也不同，有的段需要时间长，有的段需要时间短，硬盘的空间分配也会有上面第二点同样的问题，这样会导致系统性能抖动. 如果每次交换一个页，则没有这些问题
4. 段最大的问题是使得虚拟内存地址空间，难于实施

其实现在所有的商用操作系统都使用了分页模式管理内存.

### 内存区
物理内存分成三个逻辑区，分别为硬件区，内核区，应用区 from Cosmos(未知linux如何处理):
- 硬件区: 它占用物理内存低端区域，地址区间为 0~32MB, 这个内存区域是给硬件使用的

	虚拟地址主要依赖于 CPU 中的 MMU，但有很多外部硬件能直接和内存交换数据，常见的有 DMA，并且它只能访问低于 24MB 的物理内存, 这就导致了很多内存页不能随便分配给这些设备
- 内核区，内核也要使用内存，但是内核同样也是运行在虚拟地址空间，就需要有一段物理内存空间和内核的虚拟地址空间是线性映射关系
- 应用区，这个区域主是给应用用户态程序使用. 应用程序使用虚拟地址空间，一开始并不会为应用一次性分配完所需的所有物理内存，而是按需分配，即应用用到一页就分配一个页

如果访问到一个没有与物理内存页建立映射关系的虚拟内存页，这时候 CPU 就会产生缺页异常。最终这个缺页异常由操作系统处理，操作系统会分配一个物理内存页，并建好映射关系.

这是因为这种情况往往分配的是单个页面，所以为了给单个页面提供快捷的内存请求服务，就需要把离散的单页、或者是内核自身需要建好页表才可以访问的页面，统统收归到用户区.

linux内存区:
Linux 内核中也有区([zone](https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/mmzone.h#L811))的逻辑概念，因为硬件的限制，Linux 内核不能对所有的物理内存页统一对待，所以就把属性相同物理内存页面，归结到了一个区中.

不同硬件平台，区的划分也不一样。比如在 32 位的 x86 平台中，一些使用 DMA 的设备只能访问 0~16MB 的物理空间，因此将 0~16MB 划分为 DMA 区. 高内存区则适用于要访问的物理地址空间大于虚拟地址空间，Linux 内核不能建立直接映射的情况。除开这两个内存区，物理内存中剩余的页面就划分到常规内存区了, 可能还有防止内存碎片化的MOVABLE区和支持设备热插拔的DEVICE区. 有的平台没有 DMA 区，64 位的 x86 平台则没有高内存区.

> 见`cat /proc/zoneinfo |grep Node`

分配的时候，会先按请求的 migratetype 从对应的 page 结构块中寻找，如果不成功，才会从其他 migratetype 的 page 结构块中分配。这样做是为了让内存页迁移更加高效，可以有效降低内存碎片:
```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/mmzone.h#L45
enum migratetype {
	MIGRATE_UNMOVABLE, // 不可移动
	MIGRATE_MOVABLE, //可移动
	MIGRATE_RECLAIMABLE,
	MIGRATE_PCPTYPES, 	/* the number of types on the pcp lists */ // 属于pcp list的
	MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,
#ifdef CONFIG_CMA
	/*
	 * MIGRATE_CMA migration type is designed to mimic the way
	 * ZONE_MOVABLE works.  Only movable pages can be allocated
	 * from MIGRATE_CMA pageblocks and page allocator never
	 * implicitly change migration type of MIGRATE_CMA pageblock.
	 *
	 * The way to use it is to change migratetype of a range of
	 * pageblocks to MIGRATE_CMA which can be done by
	 * __free_pageblock_cma() function.
	 */
	MIGRATE_CMA, // 属于CMA区的
#endif
#ifdef CONFIG_MEMORY_ISOLATION
	MIGRATE_ISOLATE,	/* can't allocate from here */
#endif
	MIGRATE_TYPES
};

//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/mmzone.h#L111
//页面空闲链表头
struct free_area {
	struct list_head	free_list[MIGRATE_TYPES];
	unsigned long		nr_free;
};

struct zone {
	/* Read-mostly fields */

	/* zone watermarks, access with *_wmark_pages(zone) macros */
	unsigned long _watermark[NR_WMARK]; // _watermark 表示内存页面总量的水位线有 min, low, high 三种状态，可以作为启动内存页面回收的判断标准
	unsigned long watermark_boost;

	unsigned long nr_reserved_highatomic; //预留的内存页面数

	/*
	 * We don't know if the memory that we're going to allocate will be
	 * freeable or/and it will be released eventually, so to avoid totally
	 * wasting several GB of ram we must reserve some of the lower zone
	 * memory (otherwise we risk to run OOM on the lower zones despite
	 * there being tons of freeable ram on the higher zones).  This array is
	 * recalculated at runtime if the sysctl_lowmem_reserve_ratio sysctl
	 * changes.
	 */
	long lowmem_reserve[MAX_NR_ZONES];

#ifdef CONFIG_NUMA
	int node; //内存区属于哪个内存节点 
#endif
	struct pglist_data	*zone_pgdat;
	struct per_cpu_pages	__percpu *per_cpu_pageset;
	struct per_cpu_zonestat	__percpu *per_cpu_zonestats;
	/*
	 * the high and batch values are copied to individual pagesets for
	 * faster access
	 */
	int pageset_high;
	int pageset_batch;

#ifndef CONFIG_SPARSEMEM
	/*
	 * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
	 * In SPARSEMEM, this map is stored in struct mem_section
	 */
	unsigned long		*pageblock_flags;
#endif /* CONFIG_SPARSEMEM */

	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
	unsigned long		zone_start_pfn; // //内存区开始的page结构数组的开始下标

	/*
	 * spanned_pages is the total pages spanned by the zone, including
	 * holes, which is calculated as:
	 * 	spanned_pages = zone_end_pfn - zone_start_pfn;
	 *
	 * present_pages is physical pages existing within the zone, which
	 * is calculated as:
	 *	present_pages = spanned_pages - absent_pages(pages in holes);
	 *
	 * present_early_pages is present pages existing within the zone
	 * located on memory available since early boot, excluding hotplugged
	 * memory.
	 *
	 * managed_pages is present pages managed by the buddy system, which
	 * is calculated as (reserved_pages includes pages allocated by the
	 * bootmem allocator):
	 *	managed_pages = present_pages - reserved_pages;
	 *
	 * cma pages is present pages that are assigned for CMA use
	 * (MIGRATE_CMA).
	 *
	 * So present_pages may be used by memory hotplug or memory power
	 * management logic to figure out unmanaged pages by checking
	 * (present_pages - managed_pages). And managed_pages should be used
	 * by page allocator and vm scanner to calculate all kinds of watermarks
	 * and thresholds.
	 *
	 * Locking rules:
	 *
	 * zone_start_pfn and spanned_pages are protected by span_seqlock.
	 * It is a seqlock because it has to be read outside of zone->lock,
	 * and it is done in the main allocator path.  But, it is written
	 * quite infrequently.
	 *
	 * The span_seq lock is declared along with zone->lock because it is
	 * frequently read in proximity to zone->lock.  It's good to
	 * give them a chance of being in the same cacheline.
	 *
	 * Write access to present_pages at runtime should be protected by
	 * mem_hotplug_begin/done(). Any reader who can't tolerant drift of
	 * present_pages should use get_online_mems() to get a stable value.
	 */
	atomic_long_t		managed_pages;
	unsigned long		spanned_pages; //该内存区总的页面数
	unsigned long		present_pages; //该内存区存在的的页面数. 因为一些内存区中存在内存空洞，空洞对应的 page 结构不能用
#if defined(CONFIG_MEMORY_HOTPLUG)
	unsigned long		present_early_pages;
#endif
#ifdef CONFIG_CMA
	unsigned long		cma_pages;
#endif

	const char		*name; // 名称

#ifdef CONFIG_MEMORY_ISOLATION
	/*
	 * Number of isolated pageblock. It is used to solve incorrect
	 * freepage counting problem due to racy retrieving migratetype
	 * of pageblock. Protected by zone->lock.
	 */
	unsigned long		nr_isolate_pageblock;
#endif

#ifdef CONFIG_MEMORY_HOTPLUG
	/* see spanned/present_pages for more description */
	seqlock_t		span_seqlock;
#endif

	int initialized;

	/* Write-intensive fields used from the page allocator */
	CACHELINE_PADDING(_pad1_);

	/* free areas of different sizes */
	struct free_area	free_area[MAX_ORDER + 1]; //挂在页面page的链表. 这个数组就是用于实现伙伴系统的, 其中 MAX_ORDER 的值默认为 11，分别表示挂载地址连续的 page 结构数目为 1，2，4，8，16，32..., 最大为 2048. 该数组将具有相同迁移类型的 page 结构尽可能地分组，有的页面可以迁移，有的不可以迁移，同一类型的所有相同 order 的 page 结构，就构成了一组 page 结构块

#ifdef CONFIG_UNACCEPTED_MEMORY
	/* Pages to be accepted. All pages on the list are MAX_ORDER */
	struct list_head	unaccepted_pages;
#endif

	/* zone flags, see below */
	unsigned long		flags; //内存区的标志

	/* Primarily protects free_area */
	spinlock_t		lock; // 保护free_area的自旋锁

	/* Write-intensive fields used by compaction and vmstats. */
	CACHELINE_PADDING(_pad2_);

	/*
	 * When free pages are below this point, additional steps are taken
	 * when reading the number of free pages to avoid per-cpu counter
	 * drift allowing watermarks to be breached
	 */
	unsigned long percpu_drift_mark;

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* pfn where compaction free scanner should start */
	unsigned long		compact_cached_free_pfn;
	/* pfn where compaction migration scanner should start */
	unsigned long		compact_cached_migrate_pfn[ASYNC_AND_SYNC];
	unsigned long		compact_init_migrate_pfn;
	unsigned long		compact_init_free_pfn;
#endif

#ifdef CONFIG_COMPACTION
	/*
	 * On compaction failure, 1<<compact_defer_shift compactions
	 * are skipped before trying again. The number attempted since
	 * last failure is tracked with compact_considered.
	 * compact_order_failed is the minimum compaction failed order.
	 */
	unsigned int		compact_considered;
	unsigned int		compact_defer_shift;
	int			compact_order_failed;
#endif

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* Set to true when the PG_migrate_skip bits should be cleared */
	bool			compact_blockskip_flush;
#endif

	bool			contiguous;

	CACHELINE_PADDING(_pad3_);
	/* Zone statistics */
	atomic_long_t		vm_stat[NR_VM_ZONE_STAT_ITEMS];
	atomic_long_t		vm_numa_event[NR_VM_NUMA_EVENT_ITEMS];
} ____cacheline_internodealigned_in_smp;
```

在很多服务器和大型计算机上，如果物理内存是分布式的，由多个计算节点组成，那么每个 CPU 核都会有自己的本地内存，CPU 在访问它的本地内存的时候就比较快，访问其他 CPU 核内存的时候就比较慢，这种体系结构被称为 Non-Uniform Memory Access（NUMA）.

Linux 对 NUMA 进行了抽象，它可以将一整块连续物理内存的划分成几个内存节点，也可以把不是连续的物理内存当成真正的 NUMA.

```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/mmzone.h#L1175
enum {
	ZONELIST_FALLBACK,	/* zonelist with fallback */
#ifdef CONFIG_NUMA
	/*
	 * The NUMA zonelists are doubled because we need zonelists that
	 * restrict the allocations to a single node for __GFP_THISNODE.
	 */
	ZONELIST_NOFALLBACK,	/* zonelist without fallback (__GFP_THISNODE) */
#endif
	MAX_ZONELISTS
};

/*
 * This struct contains information about a zone in a zonelist. It is stored
 * here to avoid dereferences into large structures and lookups of tables
 */
struct zoneref {
	struct zone *zone;	/* Pointer to actual zone */ //内存区指针
	int zone_idx;		/* zone_idx(zoneref->zone) */ //内存区对应的索引
};

/*
 * One allocation request operates on a zonelist. A zonelist
 * is a list of zones, the first one is the 'goal' of the
 * allocation, the other zones are fallback zones, in decreasing
 * priority.
 *
 * To speed the reading of the zonelist, the zonerefs contain the zone index
 * of the entry being read. Helper functions to access information given
 * a struct zoneref are
 *
 * zonelist_zone()	- Return the struct zone * for an entry in _zonerefs
 * zonelist_zone_idx()	- Return the index of the zone for an entry
 * zonelist_node_idx()	- Return the index of the node for an entry
 */
struct zonelist {
	struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
};

//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/mmzone.h#L1254
/*
 * On NUMA machines, each NUMA node would have a pg_data_t to describe
 * it's memory layout. On UMA machines there is a single pglist_data which
 * describes the whole memory.
 *
 * Memory statistics and page replacement data structures are maintained on a
 * per-zone basis.
 */
//内存节点
//pglist_data 结构中包含了 zonelist 数组。第一个 zonelist 类型的元素指向本节点内的 zone 数组，第二个 zonelist 类型的元素指向其它节点的 zone 数组，而一个 zone 结构中的 free_area 数组中又挂载着 page 结构。这样在本节点中分配不到内存页面的时候，就会到其它节点中分配内存页面
//当计算机不是 NUMA 时，这时 Linux 就只创建一个节点
typedef struct pglist_data {
	/*
	 * node_zones contains just the zones for THIS node. Not all of the
	 * zones may be populated, but it is the full list. It is referenced by
	 * this node's node_zonelists as well as other node's node_zonelists.
	 */
	struct zone node_zones[MAX_NR_ZONES];  //定一个内存区数组，最大为6个zone元素

	/*
	 * node_zonelists contains references to all zones in all nodes.
	 * Generally the first zones will be references to this node's
	 * node_zones.
	 */
	struct zonelist node_zonelists[MAX_ZONELISTS]; //两个zonelist，一个是指向本节点的的内存区，另一个指向由本节点分配不到内存时可选的备用内存区

	int nr_zones; /* number of populated zones in this node */ //本节点有多少个内存区
#ifdef CONFIG_FLATMEM	/* means !SPARSEMEM */
	struct page *node_mem_map;
#ifdef CONFIG_PAGE_EXTENSION
	struct page_ext *node_page_ext;
#endif
#endif
#if defined(CONFIG_MEMORY_HOTPLUG) || defined(CONFIG_DEFERRED_STRUCT_PAGE_INIT)
	/*
	 * Must be held any time you expect node_start_pfn,
	 * node_present_pages, node_spanned_pages or nr_zones to stay constant.
	 * Also synchronizes pgdat->first_deferred_pfn during deferred page
	 * init.
	 *
	 * pgdat_resize_lock() and pgdat_resize_unlock() are provided to
	 * manipulate node_size_lock without checking for CONFIG_MEMORY_HOTPLUG
	 * or CONFIG_DEFERRED_STRUCT_PAGE_INIT.
	 *
	 * Nests above zone->lock and zone->span_seqlock
	 */
	spinlock_t node_size_lock;
#endif
	unsigned long node_start_pfn; //本节点开始的page索引号
	unsigned long node_present_pages; /* total number of physical pages */ //本节点有多少个可用的页面 
	unsigned long node_spanned_pages; /* total size of physical page 
					     range, including holes */ //本节点有多少个可用的页面包含内存空洞 
	int node_id;  //节点id
	//交换内存页面相关的字段
	wait_queue_head_t kswapd_wait;
	wait_queue_head_t pfmemalloc_wait;

	/* workqueues for throttling reclaim for different reasons. */
	wait_queue_head_t reclaim_wait[NR_VMSCAN_THROTTLE];

	atomic_t nr_writeback_throttled;/* nr of writeback-throttled tasks */
	unsigned long nr_reclaim_start;	/* nr pages written while throttled
					 * when throttling started. */
#ifdef CONFIG_MEMORY_HOTPLUG
	struct mutex kswapd_lock;
#endif
	struct task_struct *kswapd;	/* Protected by kswapd_lock */
	int kswapd_order;
	enum zone_type kswapd_highest_zoneidx;

	int kswapd_failures;		/* Number of 'reclaimed == 0' runs */

#ifdef CONFIG_COMPACTION
	int kcompactd_max_order;
	enum zone_type kcompactd_highest_zoneidx;
	wait_queue_head_t kcompactd_wait;
	struct task_struct *kcompactd;
	bool proactive_compact_trigger;
#endif
	/*
	 * This is a per-node reserve of pages that are not available
	 * to userspace allocations.
	 */
	unsigned long		totalreserve_pages; //本节点保留的内存页面

#ifdef CONFIG_NUMA
	/*
	 * node reclaim becomes active if more unmapped pages exist.
	 */
	unsigned long		min_unmapped_pages;
	unsigned long		min_slab_pages;
#endif /* CONFIG_NUMA */

	/* Write-intensive fields used by page reclaim */
	CACHELINE_PADDING(_pad1_);

#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
	/*
	 * If memory initialisation on large machines is deferred then this
	 * is the first PFN that needs to be initialised.
	 */
	unsigned long first_deferred_pfn;
#endif /* CONFIG_DEFERRED_STRUCT_PAGE_INIT */

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
	struct deferred_split deferred_split_queue;
#endif

#ifdef CONFIG_NUMA_BALANCING
	/* start time in ms of current promote rate limit period */
	unsigned int nbp_rl_start;
	/* number of promote candidate pages at start time of current rate limit period */
	unsigned long nbp_rl_nr_cand;
	/* promote threshold in ms */
	unsigned int nbp_threshold;
	/* start time in ms of current promote threshold adjustment period */
	unsigned int nbp_th_start;
	/*
	 * number of promote candidate pages at start time of current promote
	 * threshold adjustment period
	 */
	unsigned long nbp_th_nr_cand;
#endif
	/* Fields commonly accessed by the page reclaim scanner */

	/*
	 * NOTE: THIS IS UNUSED IF MEMCG IS ENABLED.
	 *
	 * Use mem_cgroup_lruvec() to look up lruvecs.
	 */
	struct lruvec		__lruvec;

	unsigned long		flags;

#ifdef CONFIG_LRU_GEN
	/* kswap mm walk data */
	struct lru_gen_mm_walk mm_walk;
	/* lru_gen_folio list */
	struct lru_gen_memcg memcg_lru;
#endif

	CACHELINE_PADDING(_pad2_);

	/* Per-node vmstats */
	struct per_cpu_nodestat __percpu *per_cpu_nodestats;
	atomic_long_t		vm_stat[NR_VM_NODE_STAT_ITEMS];
#ifdef CONFIG_NUMA
	struct memory_tier __rcu *memtier;
#endif
#ifdef CONFIG_MEMORY_FAILURE
	struct memory_failure_stats mf_stats;
#endif
} pg_data_t;

//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/mmzone.h#L716
enum zone_type {
	/*
	 * ZONE_DMA and ZONE_DMA32 are used when there are peripherals not able
	 * to DMA to all of the addressable memory (ZONE_NORMAL).
	 * On architectures where this area covers the whole 32 bit address
	 * space ZONE_DMA32 is used. ZONE_DMA is left for the ones with smaller
	 * DMA addressing constraints. This distinction is important as a 32bit
	 * DMA mask is assumed when ZONE_DMA32 is defined. Some 64-bit
	 * platforms may need both zones as they support peripherals with
	 * different DMA addressing limitations.
	 */
#ifdef CONFIG_ZONE_DMA
	ZONE_DMA,
#endif
#ifdef CONFIG_ZONE_DMA32
	ZONE_DMA32,
#endif
	/*
	 * Normal addressable memory is in ZONE_NORMAL. DMA operations can be
	 * performed on pages in ZONE_NORMAL if the DMA devices support
	 * transfers to all addressable memory.
	 */
	ZONE_NORMAL,
#ifdef CONFIG_HIGHMEM
	/*
	 * A memory area that is only addressable by the kernel through
	 * mapping portions into its own address space. This is for example
	 * used by i386 to allow the kernel to address the memory beyond
	 * 900MB. The kernel will set up special mappings (page
	 * table entries on i386) for each page that the kernel needs to
	 * access.
	 */
	ZONE_HIGHMEM,
#endif
	/*
	 * ZONE_MOVABLE is similar to ZONE_NORMAL, except that it contains
	 * movable pages with few exceptional cases described below. Main use
	 * cases for ZONE_MOVABLE are to make memory offlining/unplug more
	 * likely to succeed, and to locally limit unmovable allocations - e.g.,
	 * to increase the number of THP/huge pages. Notable special cases are:
	 *
	 * 1. Pinned pages: (long-term) pinning of movable pages might
	 *    essentially turn such pages unmovable. Therefore, we do not allow
	 *    pinning long-term pages in ZONE_MOVABLE. When pages are pinned and
	 *    faulted, they come from the right zone right away. However, it is
	 *    still possible that address space already has pages in
	 *    ZONE_MOVABLE at the time when pages are pinned (i.e. user has
	 *    touches that memory before pinning). In such case we migrate them
	 *    to a different zone. When migration fails - pinning fails.
	 * 2. memblock allocations: kernelcore/movablecore setups might create
	 *    situations where ZONE_MOVABLE contains unmovable allocations
	 *    after boot. Memory offlining and allocations fail early.
	 * 3. Memory holes: kernelcore/movablecore setups might create very rare
	 *    situations where ZONE_MOVABLE contains memory holes after boot,
	 *    for example, if we have sections that are only partially
	 *    populated. Memory offlining and allocations fail early.
	 * 4. PG_hwpoison pages: while poisoned pages can be skipped during
	 *    memory offlining, such pages cannot be allocated.
	 * 5. Unmovable PG_offline pages: in paravirtualized environments,
	 *    hotplugged memory blocks might only partially be managed by the
	 *    buddy (e.g., via XEN-balloon, Hyper-V balloon, virtio-mem). The
	 *    parts not manged by the buddy are unmovable PG_offline pages. In
	 *    some cases (virtio-mem), such pages can be skipped during
	 *    memory offlining, however, cannot be moved/allocated. These
	 *    techniques might use alloc_contig_range() to hide previously
	 *    exposed pages from the buddy again (e.g., to implement some sort
	 *    of memory unplug in virtio-mem).
	 * 6. ZERO_PAGE(0), kernelcore/movablecore setups might create
	 *    situations where ZERO_PAGE(0) which is allocated differently
	 *    on different platforms may end up in a movable zone. ZERO_PAGE(0)
	 *    cannot be migrated.
	 * 7. Memory-hotplug: when using memmap_on_memory and onlining the
	 *    memory to the MOVABLE zone, the vmemmap pages are also placed in
	 *    such zone. Such pages cannot be really moved around as they are
	 *    self-stored in the range, but they are treated as movable when
	 *    the range they describe is about to be offlined.
	 *
	 * In general, no unmovable allocations that degrade memory offlining
	 * should end up in ZONE_MOVABLE. Allocators (like alloc_contig_range())
	 * have to expect that migrating pages in ZONE_MOVABLE can fail (even
	 * if has_unmovable_pages() states that there are no unmovable pages,
	 * there can be false negatives).
	 */
	ZONE_MOVABLE,
#ifdef CONFIG_ZONE_DEVICE
	ZONE_DEVICE,
#endif
	__MAX_NR_ZONES

};
```

分配页面过程:
首先要找到内存节点，接着找到内存区，然后合适的空闲链表，最后在其中找到页的 page 结构，完成物理内存页面的分配

![分配内存页面接口](/misc/img/mem/9a33d0da55dfdd7dabdeb461af671418.jpg)

上图中，虚线框中为接口函数，下面则是分配内存页面的核心实现，所有的接口函数都会调用到 alloc_pages 函数，而这个函数最终会调用 `__alloc_pages_nodemask` 函数完成内存页面的分配.
- [alloc_pages](https://elixir.bootlin.com/linux/v6.5.1/source/mm/mempolicy.c#L2280)

	gfp_t 类型的 gfp: 用其中位的状态表示请求分配不同的内存区的内存页面，以及分配内存页面的不同方式

	最终要调用 `__alloc_pages` 函数
- [`__alloc_pages`](https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L4441)

	1. 准备分配页面的参数
	2. 进入快速分配路径
	3. 若快速分配路径没有分配到页面，就进入慢速分配路径

	```
	/*
	 * This is the 'heart' of the zoned buddy allocator.
	 */
	struct page *__alloc_pages(gfp_t gfp, unsigned int order, int preferred_nid,
								nodemask_t *nodemask)
	{
		struct page *page;
		unsigned int alloc_flags = ALLOC_WMARK_LOW;
		gfp_t alloc_gfp; /* The gfp_t that was actually used for allocation */
		struct alloc_context ac = { };

		/*
		 * There are several places where we assume that the order value is sane
		 * so bail out early if the request is out of bound.
		 */
		//分配页面的order大于等于最大的order直接返回NULL
		if (WARN_ON_ONCE_GFP(order > MAX_ORDER, gfp))
			return NULL;

		gfp &= gfp_allowed_mask;
		/*
		 * Apply scoped allocation constraints. This is mainly about GFP_NOFS
		 * resp. GFP_NOIO which has to be inherited for all allocation requests
		 * from a particular context which has been marked by
		 * memalloc_no{fs,io}_{save,restore}. And PF_MEMALLOC_PIN which ensures
		 * movable zones are not used during allocation.
		 */
		gfp = current_gfp_context(gfp);
		alloc_gfp = gfp;
		//准备分配页面的参数放在ac变量中
		if (!prepare_alloc_pages(gfp, order, preferred_nid, nodemask, &ac,
				&alloc_gfp, &alloc_flags))
			return NULL;

		/*
		 * Forbid the first pass from falling back to types that fragment
		 * memory until all local zones are considered.
		 */
		alloc_flags |= alloc_flags_nofragment(ac.preferred_zoneref->zone, gfp);

		/* First allocation attempt */
		//进入快速分配路径
		page = get_page_from_freelist(alloc_gfp, order, alloc_flags, &ac);
		if (likely(page))
			goto out;

		alloc_gfp = gfp;
		ac.spread_dirty_pages = false;

		/*
		 * Restore the original nodemask if it was potentially replaced with
		 * &cpuset_current_mems_allowed to optimize the fast-path attempt.
		 */
		ac.nodemask = nodemask;
		//进入慢速分配路径
		page = __alloc_pages_slowpath(alloc_gfp, order, &ac);

	out:
		if (memcg_kmem_online() && (gfp & __GFP_ACCOUNT) && page &&
		    unlikely(__memcg_kmem_charge_page(page, gfp, order) != 0)) {
			__free_pages(page, order);
			page = NULL;
		}

		trace_mm_page_alloc(page, order, alloc_gfp, ac.migratetype);
		kmsan_alloc_page(page, order, alloc_gfp);

		return page;
	}
	EXPORT_SYMBOL(__alloc_pages);

	// https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L4226
	static inline bool prepare_alloc_pages(gfp_t gfp_mask, unsigned int order,
		int preferred_nid, nodemask_t *nodemask,
		struct alloc_context *ac, gfp_t *alloc_gfp,
		unsigned int *alloc_flags)
	{
		//从哪个内存区分配内存
		ac->highest_zoneidx = gfp_zone(gfp_mask);
		//根据节点id计算出zone的指针
		ac->zonelist = node_zonelist(preferred_nid, gfp_mask);
		ac->nodemask = nodemask;
		//计算出free_area中的migratetype值，比如如分配的掩码为GFP_KERNEL，那么其类型为MIGRATE_UNMOVABLE；
		ac->migratetype = gfp_migratetype(gfp_mask);

		if (cpusets_enabled()) {
			*alloc_gfp |= __GFP_HARDWALL;
			/*
			 * When we are in the interrupt context, it is irrelevant
			 * to the current task context. It means that any node ok.
			 */
			if (in_task() && !ac->nodemask)
				ac->nodemask = &cpuset_current_mems_allowed;
			else
				*alloc_flags |= ALLOC_CPUSET;
		}

		might_alloc(gfp_mask);

		if (should_fail_alloc_page(gfp_mask, order))
			return false;
		//处理CMA相关的分配选项
		*alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, *alloc_flags);

		/* Dirty zone balancing only done in the fast path */
		ac->spread_dirty_pages = (gfp_mask & __GFP_WRITE);

		/*
		 * The preferred zone is used for statistics but crucially it is
		 * also used as the starting point for the zonelist iterator. It
		 * may get reset for allocations that ignore memory policies.
		 */
		//搜索nodemask表示的节点中可用的zone保存在preferred_zoneref
		ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
						ac->highest_zoneidx, ac->nodemask);

		return true;
	}

	//https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L3094 
	/*
	 * get_page_from_freelist goes through the zonelist trying to allocate
	 * a page.
	 */
	//遍历所有的候选内存区，然后针对每个内存区检查水位线，是不是执行内存回收机制，当一切检查通过之后，就开始调用 rmqueue 函数执行内存页面分配
	static struct page *
	get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
							const struct alloc_context *ac)
	{
		struct zoneref *z;
		struct zone *zone;
		struct pglist_data *last_pgdat = NULL;
		bool last_pgdat_dirty_ok = false;
		bool no_fallback;

	retry:
		/*
		 * Scan zonelist, looking for a zone with enough free.
		 * See also cpuset_node_allowed() comment in kernel/cgroup/cpuset.c.
		 */
		no_fallback = alloc_flags & ALLOC_NOFRAGMENT;
		z = ac->preferred_zoneref;
		//遍历ac->preferred_zoneref中每个内存区
		for_next_zone_zonelist_nodemask(zone, z, ac->highest_zoneidx,
						ac->nodemask) {
			struct page *page;
			unsigned long mark;

			if (cpusets_enabled() &&
				(alloc_flags & ALLOC_CPUSET) &&
				!__cpuset_zone_allowed(zone, gfp_mask))
					continue;
			/*
			 * When allocating a page cache page for writing, we
			 * want to get it from a node that is within its dirty
			 * limit, such that no single node holds more than its
			 * proportional share of globally allowed dirty pages.
			 * The dirty limits take into account the node's
			 * lowmem reserves and high watermark so that kswapd
			 * should be able to balance it without having to
			 * write pages from its LRU list.
			 *
			 * XXX: For now, allow allocations to potentially
			 * exceed the per-node dirty limit in the slowpath
			 * (spread_dirty_pages unset) before going into reclaim,
			 * which is important when on a NUMA setup the allowed
			 * nodes are together not big enough to reach the
			 * global limit.  The proper fix for these situations
			 * will require awareness of nodes in the
			 * dirty-throttling and the flusher threads.
			 */
			if (ac->spread_dirty_pages) {
				if (last_pgdat != zone->zone_pgdat) {
					last_pgdat = zone->zone_pgdat;
					last_pgdat_dirty_ok = node_dirty_ok(zone->zone_pgdat);
				}

				if (!last_pgdat_dirty_ok)
					continue;
			}

			if (no_fallback && nr_online_nodes > 1 &&
			    zone != ac->preferred_zoneref->zone) {
				int local_nid;

				/*
				 * If moving to a remote node, retry but allow
				 * fragmenting fallbacks. Locality is more important
				 * than fragmentation avoidance.
				 */
				local_nid = zone_to_nid(ac->preferred_zoneref->zone);
				if (zone_to_nid(zone) != local_nid) {
					alloc_flags &= ~ALLOC_NOFRAGMENT;
					goto retry;
				}
			}

			//查看内存水位线
			mark = wmark_pages(zone, alloc_flags & ALLOC_WMARK_MASK);
			//检查内存区中空闲内存是否在水位线之上
			if (!zone_watermark_fast(zone, order, mark,
					       ac->highest_zoneidx, alloc_flags,
					       gfp_mask)) {
				int ret;

				if (has_unaccepted_memory()) {
					if (try_to_accept_memory(zone, order))
						goto try_this_zone;
				}

	#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
				/*
				 * Watermark failed for this zone, but see if we can
				 * grow this zone if it contains deferred pages.
				 */
				if (deferred_pages_enabled()) {
					if (_deferred_grow_zone(zone, order))
						goto try_this_zone;
				}
	#endif
				/* Checked here to keep the fast path fast */
				BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK);
				if (alloc_flags & ALLOC_NO_WATERMARKS)
					goto try_this_zone;

				if (!node_reclaim_enabled() ||
				    !zone_allows_reclaim(ac->preferred_zoneref->zone, zone))
					continue;
				//当前内存区的内存结点需要做内存回收吗
				ret = node_reclaim(zone->zone_pgdat, gfp_mask, order);
				switch (ret) {
				//快速分配路径不处理页面回收的问题
				case NODE_RECLAIM_NOSCAN:
					/* did not scan */
					continue;
				case NODE_RECLAIM_FULL:
					/* scanned but unreclaimable */
					continue;
				default:
					/* did we reclaim enough */
					//根据分配的order数量判断内存区的水位线是否满足要求
					if (zone_watermark_ok(zone, order, mark,
						ac->highest_zoneidx, alloc_flags))
						//如果可以可就从这个内存区开始分配
						goto try_this_zone;

					continue;
				}
			}

	try_this_zone:
			//真正分配内存页面
			page = rmqueue(ac->preferred_zoneref->zone, zone, order,
					gfp_mask, alloc_flags, ac->migratetype);
			if (page) {
				//清除一些标志或者设置联合页等等
				prep_new_page(page, order, gfp_mask, alloc_flags);

				/*
				 * If this is a high-order atomic allocation then check
				 * if the pageblock should be reserved for the future
				 */
				if (unlikely(alloc_flags & ALLOC_HIGHATOMIC))
					reserve_highatomic_pageblock(page, zone, order);

				return page;
			} else {
				if (has_unaccepted_memory()) {
					if (try_to_accept_memory(zone, order))
						goto try_this_zone;
				}

	#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
				/* Try again if zone has deferred pages */
				if (deferred_pages_enabled()) {
					if (_deferred_grow_zone(zone, order))
						goto try_this_zone;
				}
	#endif
			}
		}

		/*
		 * It's possible on a UMA machine to get through all zones that are
		 * fragmented. If avoiding fragmentation, reset and try again.
		 */
		if (no_fallback) {
			alloc_flags &= ~ALLOC_NOFRAGMENT;
			goto retry;
		}

		return NULL;
	}

	//https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L3949
	static inline struct page *
	__alloc_pages_slowpath(gfp_t gfp_mask, unsigned int order,
							struct alloc_context *ac)
	{
		bool can_direct_reclaim = gfp_mask & __GFP_DIRECT_RECLAIM;
		const bool costly_order = order > PAGE_ALLOC_COSTLY_ORDER;
		struct page *page = NULL;
		unsigned int alloc_flags;
		unsigned long did_some_progress;
		enum compact_priority compact_priority;
		enum compact_result compact_result;
		int compaction_retries;
		int no_progress_loops;
		unsigned int cpuset_mems_cookie;
		unsigned int zonelist_iter_cookie;
		int reserve_flags;

	restart:
		compaction_retries = 0;
		no_progress_loops = 0;
		compact_priority = DEF_COMPACT_PRIORITY;
		cpuset_mems_cookie = read_mems_allowed_begin();
		zonelist_iter_cookie = zonelist_iter_begin();

		/*
		 * The fast path uses conservative alloc_flags to succeed only until
		 * kswapd needs to be woken up, and to avoid the cost of setting up
		 * alloc_flags precisely. So we do that now.
		 */
		alloc_flags = gfp_to_alloc_flags(gfp_mask, order);

		/*
		 * We need to recalculate the starting point for the zonelist iterator
		 * because we might have used different nodemask in the fast path, or
		 * there was a cpuset modification and we are retrying - otherwise we
		 * could end up iterating over non-eligible zones endlessly.
		 */
		ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
						ac->highest_zoneidx, ac->nodemask);
		if (!ac->preferred_zoneref->zone)
			goto nopage;

		/*
		 * Check for insane configurations where the cpuset doesn't contain
		 * any suitable zone to satisfy the request - e.g. non-movable
		 * GFP_HIGHUSER allocations from MOVABLE nodes only.
		 */
		if (cpusets_insane_config() && (gfp_mask & __GFP_HARDWALL)) {
			struct zoneref *z = first_zones_zonelist(ac->zonelist,
						ac->highest_zoneidx,
						&cpuset_current_mems_allowed);
			if (!z->zone)
				goto nopage;
		}
		//唤醒所有交换内存的线程
		if (alloc_flags & ALLOC_KSWAPD)
			wake_all_kswapds(order, gfp_mask, ac);

		/*
		 * The adjusted alloc_flags might result in immediate success, so try
		 * that first
		 */
		//依然调用快速分配路径入口函数尝试分配内存页面
		page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
		if (page)
			goto got_pg;

		/*
		 * For costly allocations, try direct compaction first, as it's likely
		 * that we have enough base pages and don't need to reclaim. For non-
		 * movable high-order allocations, do that as well, as compaction will
		 * try prevent permanent fragmentation by migrating from blocks of the
		 * same migratetype.
		 * Don't try this for allocations that are allowed to ignore
		 * watermarks, as the ALLOC_NO_WATERMARKS attempt didn't yet happen.
		 */
		if (can_direct_reclaim &&
				(costly_order ||
				   (order > 0 && ac->migratetype != MIGRATE_MOVABLE))
				&& !gfp_pfmemalloc_allowed(gfp_mask)) {
			page = __alloc_pages_direct_compact(gfp_mask, order,
							alloc_flags, ac,
							INIT_COMPACT_PRIORITY,
							&compact_result);
			if (page)
				goto got_pg;

			/*
			 * Checks for costly allocations with __GFP_NORETRY, which
			 * includes some THP page fault allocations
			 */
			if (costly_order && (gfp_mask & __GFP_NORETRY)) {
				/*
				 * If allocating entire pageblock(s) and compaction
				 * failed because all zones are below low watermarks
				 * or is prohibited because it recently failed at this
				 * order, fail immediately unless the allocator has
				 * requested compaction and reclaim retry.
				 *
				 * Reclaim is
				 *  - potentially very expensive because zones are far
				 *    below their low watermarks or this is part of very
				 *    bursty high order allocations,
				 *  - not guaranteed to help because isolate_freepages()
				 *    may not iterate over freed pages as part of its
				 *    linear scan, and
				 *  - unlikely to make entire pageblocks free on its
				 *    own.
				 */
				if (compact_result == COMPACT_SKIPPED ||
				    compact_result == COMPACT_DEFERRED)
					goto nopage;

				/*
				 * Looks like reclaim/compaction is worth trying, but
				 * sync compaction could be very expensive, so keep
				 * using async compaction.
				 */
				compact_priority = INIT_COMPACT_PRIORITY;
			}
		}

	retry:
		/* Ensure kswapd doesn't accidentally go to sleep as long as we loop */
		if (alloc_flags & ALLOC_KSWAPD)
			wake_all_kswapds(order, gfp_mask, ac);

		reserve_flags = __gfp_pfmemalloc_flags(gfp_mask);
		if (reserve_flags)
			alloc_flags = gfp_to_alloc_flags_cma(gfp_mask, reserve_flags) |
						  (alloc_flags & ALLOC_KSWAPD);

		/*
		 * Reset the nodemask and zonelist iterators if memory policies can be
		 * ignored. These allocations are high priority and system rather than
		 * user oriented.
		 */
		if (!(alloc_flags & ALLOC_CPUSET) || reserve_flags) {
			ac->nodemask = NULL;
			ac->preferred_zoneref = first_zones_zonelist(ac->zonelist,
						ac->highest_zoneidx, ac->nodemask);
		}

		/* Attempt with potentially adjusted zonelist and alloc_flags */
		page = get_page_from_freelist(gfp_mask, order, alloc_flags, ac);
		if (page)
			goto got_pg;

		/* Caller is not willing to reclaim, we can't balance anything */
		if (!can_direct_reclaim)
			goto nopage;

		/* Avoid recursion of direct reclaim */
		if (current->flags & PF_MEMALLOC)
			goto nopage;

		/* Try direct reclaim and then allocating */
		//尝试直接回收内存并且再分配内存页面
		page = __alloc_pages_direct_reclaim(gfp_mask, order, alloc_flags, ac,
								&did_some_progress);
		if (page)
			goto got_pg;

		/* Try direct compaction and then allocating */
		//尝试直接压缩内存并且再分配内存页面
		page = __alloc_pages_direct_compact(gfp_mask, order, alloc_flags, ac,
						compact_priority, &compact_result);
		if (page)
			goto got_pg;

		/* Do not loop if specifically requested */
		if (gfp_mask & __GFP_NORETRY)
			goto nopage;

		/*
		 * Do not retry costly high order allocations unless they are
		 * __GFP_RETRY_MAYFAIL
		 */
		if (costly_order && !(gfp_mask & __GFP_RETRY_MAYFAIL))
			goto nopage;

		//检查对于给定的分配请求，重试回收是否有意义
		if (should_reclaim_retry(gfp_mask, order, ac, alloc_flags,
					 did_some_progress > 0, &no_progress_loops))
			goto retry;

		/*
		 * It doesn't make any sense to retry for the compaction if the order-0
		 * reclaim is not able to make any progress because the current
		 * implementation of the compaction depends on the sufficient amount
		 * of free memory (see __compaction_suitable)
		 */
		//检查对于给定的分配请求，重试压缩是否有意义
		if (did_some_progress > 0 &&
				should_compact_retry(ac, order, alloc_flags,
					compact_result, &compact_priority,
					&compaction_retries))
			goto retry;


		/*
		 * Deal with possible cpuset update races or zonelist updates to avoid
		 * a unnecessary OOM kill.
		 */
		if (check_retry_cpuset(cpuset_mems_cookie, ac) ||
		    check_retry_zonelist(zonelist_iter_cookie))
			goto restart;

		/* Reclaim has failed us, start killing things */
		//回收、压缩内存已经失败了，开始尝试杀死进程，回收内存页面
		page = __alloc_pages_may_oom(gfp_mask, order, ac, &did_some_progress);
		if (page)
			goto got_pg;

		/* Avoid allocations with no watermarks from looping endlessly */
		if (tsk_is_oom_victim(current) &&
		    (alloc_flags & ALLOC_OOM ||
		     (gfp_mask & __GFP_NOMEMALLOC)))
			goto nopage;

		/* Retry as long as the OOM killer is making progress */
		if (did_some_progress) {
			no_progress_loops = 0;
			goto retry;
		}

	nopage:
		/*
		 * Deal with possible cpuset update races or zonelist updates to avoid
		 * a unnecessary OOM kill.
		 */
		if (check_retry_cpuset(cpuset_mems_cookie, ac) ||
		    check_retry_zonelist(zonelist_iter_cookie))
			goto restart;

		/*
		 * Make sure that __GFP_NOFAIL request doesn't leak out and make sure
		 * we always retry
		 */
		if (gfp_mask & __GFP_NOFAIL) {
			/*
			 * All existing users of the __GFP_NOFAIL are blockable, so warn
			 * of any new users that actually require GFP_NOWAIT
			 */
			if (WARN_ON_ONCE_GFP(!can_direct_reclaim, gfp_mask))
				goto fail;

			/*
			 * PF_MEMALLOC request from this context is rather bizarre
			 * because we cannot reclaim anything and only can loop waiting
			 * for somebody to do a work for us
			 */
			WARN_ON_ONCE_GFP(current->flags & PF_MEMALLOC, gfp_mask);

			/*
			 * non failing costly orders are a hard requirement which we
			 * are not prepared for much so let's warn about these users
			 * so that we can identify them and convert them to something
			 * else.
			 */
			WARN_ON_ONCE_GFP(costly_order, gfp_mask);

			/*
			 * Help non-failing allocations by giving some access to memory
			 * reserves normally used for high priority non-blocking
			 * allocations but do not use ALLOC_NO_WATERMARKS because this
			 * could deplete whole memory reserves which would just make
			 * the situation worse.
			 */
			page = __alloc_pages_cpuset_fallback(gfp_mask, order, ALLOC_MIN_RESERVE, ac);
			if (page)
				goto got_pg;

			cond_resched();
			goto retry;
		}
	fail:
		warn_alloc(gfp_mask, ac->nodemask,
				"page allocation failure: order:%u", order);
	got_pg:
		return page;
	}

	//https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L2117
	/*
	 * Do the hard work of removing an element from the buddy allocator.
	 * Call me with the zone->lock already held.
	 */
	static __always_inline struct page *
	__rmqueue(struct zone *zone, unsigned int order, int migratetype,
							unsigned int alloc_flags)
	{
		struct page *page;

		if (IS_ENABLED(CONFIG_CMA)) {
			/*
			 * Balance movable allocations between regular and CMA areas by
			 * allocating from CMA when over half of the zone's free memory
			 * is in the CMA area.
			 */
			if (alloc_flags & ALLOC_CMA &&
			    zone_page_state(zone, NR_FREE_CMA_PAGES) >
			    zone_page_state(zone, NR_FREE_PAGES) / 2) {
				page = __rmqueue_cma_fallback(zone, order);
				if (page)
					return page;
			}
		}
	retry:
		//从free_area中分配
		page = __rmqueue_smallest(zone, order, migratetype);
		if (unlikely(!page)) {
			if (alloc_flags & ALLOC_CMA)
				page = __rmqueue_cma_fallback(zone, order);

			if (!page && __rmqueue_fallback(zone, order, migratetype,
									alloc_flags))
				goto retry;
		}
		return page;
	}

	//https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L3949
	/*
	 * Do not instrument rmqueue() with KMSAN. This function may call
	 * __msan_poison_alloca() through a call to set_pfnblock_flags_mask().
	 * If __msan_poison_alloca() attempts to allocate pages for the stack depot, it
	 * may call rmqueue() again, which will result in a deadlock.
	 */
	__no_sanitize_memory
	static inline
	struct page *rmqueue(struct zone *preferred_zone,
				struct zone *zone, unsigned int order,
				gfp_t gfp_flags, unsigned int alloc_flags,
				int migratetype)
	{
		struct page *page;

		/*
		 * We most definitely don't want callers attempting to
		 * allocate greater than order-1 page units with __GFP_NOFAIL.
		 */
		WARN_ON_ONCE((gfp_flags & __GFP_NOFAIL) && (order > 1));

		if (likely(pcp_allowed_order(order))) {
			/*
			 * MIGRATE_MOVABLE pcplist could have the pages on CMA area and
			 * we need to skip it when CMA area isn't allowed.
			 */
			if (!IS_ENABLED(CONFIG_CMA) || alloc_flags & ALLOC_CMA ||
					migratetype != MIGRATE_MOVABLE) {
				page = rmqueue_pcplist(preferred_zone, zone, order,
						migratetype, alloc_flags);
				if (likely(page))
					goto out;
			}
		}

		page = rmqueue_buddy(preferred_zone, zone, order, alloc_flags,
								migratetype);

	out:
		/* Separate test+clear to avoid unnecessary atomics */
		if ((alloc_flags & ALLOC_KSWAPD) &&
		    unlikely(test_bit(ZONE_BOOSTED_WATERMARK, &zone->flags))) {
			clear_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
			wakeup_kswapd(zone, 0, 0, zone_idx(zone));
		}

		VM_BUG_ON_PAGE(page && bad_range(zone, page), page);
		return page;
	}

	// https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L2671
	static __always_inline
	struct page *rmqueue_buddy(struct zone *preferred_zone, struct zone *zone,
				   unsigned int order, unsigned int alloc_flags,
				   int migratetype)
	{
		struct page *page;
		unsigned long flags;

		do {
			page = NULL;
			spin_lock_irqsave(&zone->lock, flags);
			/*
			 * order-0 request can reach here when the pcplist is skipped
			 * due to non-CMA allocation context. HIGHATOMIC area is
			 * reserved for high-order atomic allocation, so order-0
			 * request should skip it.
			 */
			if (alloc_flags & ALLOC_HIGHATOMIC)
				page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);
			if (!page) {
				page = __rmqueue(zone, order, migratetype, alloc_flags);

				/*
				 * If the allocation fails, allow OOM handling access
				 * to HIGHATOMIC reserves as failing now is worse than
				 * failing a high-order atomic allocation in the
				 * future.
				 */
				if (!page && (alloc_flags & ALLOC_OOM))
					page = __rmqueue_smallest(zone, order, MIGRATE_HIGHATOMIC);

				if (!page) {
					spin_unlock_irqrestore(&zone->lock, flags);
					return NULL;
				}
			}
			__mod_zone_freepage_state(zone, -(1 << order),
						  get_pcppage_migratetype(page));
			spin_unlock_irqrestore(&zone->lock, flags);
		} while (check_new_pages(page, order));

		__count_zid_vm_events(PGALLOC, page_zonenum(page), 1 << order);
		zone_statistics(preferred_zone, zone, 1);

		return page;
	}

	/* Remove page from the per-cpu list, caller must protect the list */
	static inline
	struct page *__rmqueue_pcplist(struct zone *zone, unsigned int order,
				int migratetype,
				unsigned int alloc_flags,
				struct per_cpu_pages *pcp,
				struct list_head *list)
	{
		struct page *page;

		do {
			if (list_empty(list)) {
				int batch = READ_ONCE(pcp->batch);
				int alloced;

				/*
				 * Scale batch relative to order if batch implies
				 * free pages can be stored on the PCP. Batch can
				 * be 1 for small zones or for boot pagesets which
				 * should never store free pages as the pages may
				 * belong to arbitrary zones.
				 */
				if (batch > 1)
					batch = max(batch >> order, 2);
				//如果list为空，就从这个内存区中分配一部分页面到pcp中来
				alloced = rmqueue_bulk(zone, order,
						batch, list,
						migratetype, alloc_flags);

				pcp->count += alloced << order;
				if (unlikely(list_empty(list)))
					return NULL;
			}

			//获取list上第一个page结构
			page = list_first_entry(list, struct page, pcp_list);
			//脱链
			list_del(&page->pcp_list);
			//减少pcp页面计数
			pcp->count -= 1 << order;
		} while (check_new_pages(page, order));

		return page;
	}

	/* Lock and remove page from the per-cpu list */
	static struct page *rmqueue_pcplist(struct zone *preferred_zone,
				struct zone *zone, unsigned int order,
				int migratetype, unsigned int alloc_flags)
	{
		struct per_cpu_pages *pcp;
		struct list_head *list;
		struct page *page;
		unsigned long __maybe_unused UP_flags;

		/* spin_trylock may fail due to a parallel drain or IRQ reentrancy. */
		pcp_trylock_prepare(UP_flags);
		pcp = pcp_spin_trylock(zone->per_cpu_pageset);
		if (!pcp) {
			pcp_trylock_finish(UP_flags);
			return NULL;
		}

		/*
		 * On allocation, reduce the number of pages that are batch freed.
		 * See nr_pcp_free() where free_factor is increased for subsequent
		 * frees.
		 */
		pcp->free_factor >>= 1;
		//获取pcp下迁移的list链表
		list = &pcp->lists[order_to_pindex(migratetype, order)];
		//摘取list上的page结构
		page = __rmqueue_pcplist(zone, order, migratetype, alloc_flags, pcp, list);
		pcp_spin_unlock(pcp);
		pcp_trylock_finish(UP_flags);
		if (page) {
			__count_zid_vm_events(PGALLOC, page_zonenum(page), 1 << order);
			zone_statistics(preferred_zone, zone, 1);
		}
		return page;
	}

	//https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L686 
	/* Used for pages not on another list */
	static inline void add_to_free_list(struct page *page, struct zone *zone,
					    unsigned int order, int migratetype)
	{
		struct free_area *area = &zone->free_area[order];
		//把一组page的首个page加入对应的free_area中
		list_add(&page->buddy_list, &area->free_list[migratetype]);
		area->nr_free++;
	}

	// https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L719
	static inline void del_page_from_free_list(struct page *page, struct zone *zone,
					   unsigned int order)
	{
		/* clear reported state and update reported page count */
		if (page_reported(page))
			__ClearPageReported(page);
		//脱链
		list_del(&page->buddy_list);
		//清除page中伙伴系统的标志
		__ClearPageBuddy(page);
		set_page_private(page, 0);
		//减少free_area中页面计数
		zone->free_area[order].nr_free--;
	}

	static inline struct page *get_page_from_free_area(struct free_area *area,
					    int migratetype)
	{//返回free_list[migratetype]中的第一个page若没有就返回NULL
		return list_first_entry_or_null(&area->free_list[migratetype],
						struct page, buddy_list);
	}

	//https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L1409
	/*
	 * The order of subdivision here is critical for the IO subsystem.
	 * Please do not alter this order without good reasons and regression
	 * testing. Specifically, as large blocks of memory are subdivided,
	 * the order in which smaller blocks are delivered depends on the order
	 * they're subdivided in this function. This is the primary factor
	 * influencing the order in which pages are delivered to the IO
	 * subsystem according to empirical testing, and this is also justified
	 * by considering the behavior of a buddy system containing a single
	 * large block of memory acted on by a series of small allocations.
	 * This behavior is a critical factor in sglist merging's success.
	 *
	 * -- nyc
	 */
	//分割一组页
	static inline void expand(struct zone *zone, struct page *page,
		int low, int high, int migratetype)
	{//最高order下连续的page数 比如high = 3 size=8
		unsigned long size = 1 << high; 

		while (high > low) {
			high--;
			size >>= 1;//每次循环左移一位 4,2,1
			VM_BUG_ON_PAGE(bad_range(zone, &page[size]), &page[size]);

			/*
			 * Mark as guard pages (or page), that will allow to
			 * merge back to allocator when buddy will be freed.
			 * Corresponding page table entries will not be touched,
			 * pages will stay not present in virtual address space
			 */
			//标记为保护页，当其伙伴被释放时，允许合并
			if (set_page_guard(zone, &page[size], high, migratetype))
				continue;
			//把另一半pages加入对应的free_area中
			add_to_free_list(&page[size], zone, high, migratetype);
			//设置伙伴
			set_buddy_order(&page[size], high);
		}
	}

	//https://elixir.bootlin.com/linux/v6.5.1/source/mm/page_alloc.c#L1594
	/*
	 * Go through the free lists for the given migratetype and remove
	 * the smallest available page from the freelists
	 */
	static __always_inline
	struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
							int migratetype)
	{
		unsigned int current_order;
		struct free_area *area;
		struct page *page;

		/* Find a page of the appropriate size in the preferred list */
		for (current_order = order; current_order <= MAX_ORDER; ++current_order) {
			//获取current_order对应的free_area
			area = &(zone->free_area[current_order]);
			//获取free_area中对应migratetype为下标的free_list中的page
			page = get_page_from_free_area(area, migratetype);
			if (!page)
				continue;
			//脱链
			del_page_from_free_list(page, zone, current_order);
			//分割伙伴
			expand(zone, page, order, current_order, migratetype);
			set_pcppage_migratetype(page, migratetype);
			trace_mm_page_alloc_zone_locked(page, order, migratetype,
					pcp_allowed_order(order) &&
					migratetype < MIGRATE_PCPTYPES);
			return page;
		}

		return NULL;
	}
	```

	prepare_alloc_pages根据传递进入的参数，就能找出要分配内存区、候选内存区以及内存区中空闲链表的 migratetype 类型。它把这些全部分配参数收集到 ac 结构中，只要它返回 true，就说明分配内存页面的参数已经准备好了.

	为了优化内存页面的分配性能，在一定情况下可以进入快速分配路径，请注意快速分配路径不会处理内存页面合并和回收.

	当快速分配路径没有分配到页面的时候，就会进入慢速分配路径。跟快速路径相比，慢速路径最主要的不同是它会执行页面回收，回收页面之后会进行多次重复分配，直到最后分配到内存页面，或者分配失败.

	`__alloc_pages_slowpath`会唤醒所有用于内存交换回收的线程 get_page_from_freelist 函数分配失败了就会进行内存回收，内存回收主要是释放一些文件占用的内存页面。如果内存回收不行，就会就进入到内存压缩环节. 内存压缩不是指压缩内存中的数据，而是指移动内存页面，进行内存碎片整理，腾出更大的连续的内存空间。如果内存碎片整理了，还是不能成功分配内存，就要杀死进程以便释放更多内存页面.

	无论快速分配路径还是慢速分配路径，最终执行内存页面分配动作的始终是 get_page_from_freelist 函数，更准确地说，实际完成分配任务的是 rmqueue 函数.

	rmqueue_pcplist 和 `__rmqueue_smallest`，这是rmqueue分配内存页面的核心函数. 	rmqueue_pcplist主要是优化了请求分配单个内存页面的性能, 但是遇到多个内存页面的分配请求，就会调用 `__rmqueue_smallest` 函数, 从 free_area 数组中分配

	rmqueue_pcplist 函数，在请求分配一个页面的时候，就是用它从 pcplist 中分配页面的。所谓的 pcp 是指，每个 CPU 都有一个内存页面高速缓冲，由数据结构 per_cpu_pageset 描述，包含在内存区中.

	在 `__rmqueue_smallest` 函数中，首先要取得 current_order 对应的 free_area 区中 page，若没有，就继续增加 current_order，直到最大的 MAX_ORDER。要是得到一组连续 page 的首地址，就对其脱链，然后调用 expand 函数分割伙伴

	> 在 Linux 内核中，系统会经常请求和释放单个页面。如果针对每个 CPU，都建立出预先分配了单个内存页面的链表，用于满足本地 CPU 发出的单一内存请求，就能提升系统的性能.


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
参考:
- [linux内存源码分析 - SLUB分配器概述](https://www.cnblogs.com/tolimit/p/4654109.html)
- [linux内核之slob、slab、slub](https://blog.csdn.net/Rong_Toa/article/details/106440497)
- [如何诊断SLUB问题](http://linuxperf.com/?p=184)

Linux内核内存管理的一项重要工作就是如何在频繁申请释放内存的情况下，避免碎片的产生. Linux提供了两个层次的内存分配接口:
1. 伙伴系统管理物理内存页面, 解决外部碎片的问题

	最底层的内存管理机制, 提供页式内存管理

	调用alloc_pages(它以页为单位进行分配, 得到页面地址), 再调用page_address可得到内存地址. [__get_free_pages](https://elixir.bootlin.com/linux/v5.11/source/mm/page_alloc.c#L5034)封装了它俩.
1. slub负责分配比页更小的内存对象, 解决内部碎片的问题

	伙伴系统之上的内存管理, 基于对象. slab实际上是内核在buddy系统基础上构建的cache,它批量申请内存(多页),完成映射,用户需要使用内存的时候直接由它返回,不需要再经过buddy系统.

	slub针对多处理器、NUMA系统进行了优化.

	判断系统是否在用slub的方法: 看是否存在/sys/kernel/slab目录，存在即使用slub，没有就是slab.

	> 为了保证内核其它模块能够无缝迁移到SLUB分配器，SLUB还保留了原有SLAB分配器所有的接口API函数

	kmalloc使用的就是slub提供的对象.

	要从slub申请内存需要先使用[kmem_cache_create](https://elixir.bootlin.com/linux/v5.11/source/mm/slab_common.c#L407)创建一个slub对象. 再通过[kmem_cache_alloc](https://elixir.bootlin.com/linux/v5.11/source/mm/slab.c#L3484)和[kmem_cache_free](https://elixir.bootlin.com/linux/v5.11/source/mm/slab.c#L3686)来申请和释放内存.

	vmalloc: 把物理地址不连续的内存页拼凑成逻辑地址连续的内存区间.

	`sl[x]b`函数表(函数相同, 但实现有差异)
	- kmalloc/kzalloc: 申请内存
	- kfree: 释放申请的内存
	- kmem_cache_create: 创建一个cache
	- kmem_cache_alloc/kmem_cache_zalloc: 从cache申请内存
	- kmem_cache_free: 释放从cache申请的内存
	- kmem_cache_destroy: 销毁cache

	> 创建和销毁cache是有代价的,若非必要,使用内核提供的cache更加高效

申请内存因素表:
1. 小于4M, 连续: buddy, slub
1. 小于4M, 不连续: buddy, slub, vmalloc
1. 大于4M, 连续: grub, memblock
1. 大于4M, 不连续: vmalloc

使用grub、memblock和buddy得到的是物理内存,需要自行映射,slab和vmalloc得到的是虚拟内存.

### slub
在 SLUB 分配器中，它把一个内存页面或者一组连续的内存页面，划分成大小相同的块，其中这一个小的内存块就是 SLUB 对象，但是这一组连续的内存页面中不只是 SLUB 对象，还有 SLUB 管理头和着色区.

这个着色区也是一块动态的内存块，建立 SLUB 时才会设置它的大小，目的是为了错开不同 SLUB 中的对象地址，降低硬件 Cache 行中的地址争用，以免导致 Cache 抖动效应，整个系统性能下降.

```c
//https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/slub_def.h#L98
/*
 * Slab cache management.
 */
struct kmem_cache {
#ifndef CONFIG_SLUB_TINY
	struct kmem_cache_cpu __percpu *cpu_slab; //是每个CPU一个kmem_cache_cpu类型的变量，cpu_slab是用于管理空闲对象的
#endif
	/* Used for retrieving partial slabs, etc. */
	slab_flags_t flags;
	unsigned long min_partial;
	unsigned int size;	/* The size of an object including metadata */
	unsigned int object_size;/* The size of an object without metadata */
	struct reciprocal_value reciprocal_size;
	unsigned int offset;	/* Free pointer offset */
#ifdef CONFIG_SLUB_CPU_PARTIAL
	/* Number of per cpu partial objects to keep around */
	unsigned int cpu_partial;
	/* Number of per cpu partial slabs to keep around */
	unsigned int cpu_partial_slabs;
#endif
	struct kmem_cache_order_objects oo;

	/* Allocation and freeing of slabs */
	struct kmem_cache_order_objects min;
	gfp_t allocflags;	/* gfp flags to use on each alloc */
	int refcount;		/* Refcount for slab cache destroy */
	void (*ctor)(void *);
	unsigned int inuse;		/* Offset to metadata */
	unsigned int align;		/* Alignment */
	unsigned int red_left_pad;	/* Left redzone padding size */
	const char *name;	/* Name (only for display!) */
	struct list_head list;	/* List of slab caches */
#ifdef CONFIG_SYSFS
	struct kobject kobj;	/* For sysfs */
#endif
#ifdef CONFIG_SLAB_FREELIST_HARDENED
	unsigned long random;
#endif

#ifdef CONFIG_NUMA
	/*
	 * Defragmentation by allocating from a remote node.
	 */
	unsigned int remote_node_defrag_ratio;
#endif

#ifdef CONFIG_SLAB_FREELIST_RANDOM
	unsigned int *random_seq;
#endif

#ifdef CONFIG_KASAN_GENERIC
	struct kasan_cache kasan_info;
#endif

#ifdef CONFIG_HARDENED_USERCOPY
	unsigned int useroffset;	/* Usercopy region offset */
	unsigned int usersize;		/* Usercopy region size */
#endif

	struct kmem_cache_node *node[MAX_NUMNODES];
};
```

SLUB 头其实是一个数据结构，但是它不一定放在保存对象内存页面的开始。通常会有一个保存 SLUB 管理头的 SLUB，在 Linux 中，SLUB 管理头用 kmem_cache 结构来表示.

有多少个 CPU，就会有多少个 kmem_cache_cpu 类型的变量, 这种为每个 CPU 构造一个变量副本的同步机制，就是每 CPU 变量（per-cpu-variable）.

在`__init kmem_cache_init`建好第一个kmem_cache, 在 kmem_cache 结构中有个保存 kmem_cache_node 结构的指针数组.

kmem_cache_node 结构是每个内存节点对应一个，它就是用来管理 kmem_cache 结构的.

可从 Linux 内核中使用的 kmalloc 函数入手，了解了 SLUB 下整个内存对象的分配过程.

#### slabinfo工具
随内核源程序提供了一个slabinfo工具，但是需要自己手工编译. 源程序的位置是在源代码树下的`tools/vm/slabinfo.c`，编译方法是：
```bash
$ gcc -o slabinfo tools/vm/slabinfo.c
```

或者进入 tools/vm 目录下直接执行make：
```bash
$ make slabinfo
```

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

## Translation Lookaside Buffer(TLB)/大页（Hugepage）
TLB是特别的cpu缓存, 是MMU的核心部件, 通过缓存进程最近使用的分页映射来加速地址转换(虚拟地址 -> 物理地址), 因此被称为快表.

进程使用的内存越多, 分页表就越大. 同时进程在分页表上查询一个分页映射的开销也会很大. 因此tlb可很好地解决这个问题.

当上下文切换时, 为了将进程调度出来, 内核必须经常刷新tlb条目.

> tlb可通过`x86info -a`查看.
> linux通过hugepage来支持大号的分页. x86_64支持分页大小: 4K, 2M, 4M, 1G. 进程使用mmap 系统调用或shmat和shmget系统调用来请求大页.

> TTW(Translation Table walk, 转换表漫游): 当TLB未命中时, 需要通过内存中的页表来将虚拟地址映射为物理地址, TTW成功后, 将结果写入TLB中.

虚拟地址映射到物理地址的工作主要是TLB（Translation Lookaside Buffers）与MMU一起来完成的. 以4KB的页大小为例，虚拟地址寻址时，首先在TLB中查找，如果没有找到，则需要通过MMU加载的页表基地址进行多次寻表来找到对应的物理地址. 如果找不到，则产生缺页，这时会有相应的handler进行处理，来填充页表和更新TLB.

总的来说，通过页表查询而导致缺页带来的CPU开销是非常大的，TLB的出现能很好地解决性能问题. 但是经常性的缺页是不可避免的，为此可以采取大页的方式.

通过使用Hugepage分配大页可以提高性能. 因为页大小的增加，可以减少缺页异常. 例如，2MB大小的内容（假设是2MB对齐），如果是4KB大小的页粒度，则会产生512次缺页异常，但是使用2MB大小的页，只会产生一次缺页异常. 页粒度的改变，使得TLB同样的空间可以保存更多虚存空间到物理空间的映射. 尽可能地利用TLB，少用MMU，以减少寻址和缺页处理带来的开销，从而提高应用程序的整体性能.

大页还有一个优势是这些预先分配的内存基本上不会被换出. 当然，大页也有缺点，比如它需要额外配置，需要应用程序事先预估使用多少内存，大页需要在进程启动之前实现启用和分配好内存. 目前，在大部分场景下，系统配置的主内存越来越多，这个限制不会成为太大的障碍.

为了TLB一致性, 需要刷新, 具体场景:
1. 内存映射是与进程相关的,进程切换会导致一部分映射产生变化,需要刷新这部分映射对应的TLB
2. 物理内存或者虚拟内存重新映射,导致页表项产生变化,可能需要刷新产生变化的内存对应的TLB
3. 部分内存访问的权限等属性发生变化,可能需要刷新TLB

tlb刷新函数:
1. flush_tlb_mm: 刷新mm_struct相关的TLB, 针对场景1
1. flush_tlb_range/flush_tlb_page:  刷新一段或者一页内存相关的TLB, 针对场景2和场景3
1. flush_tlb_all: 刷新所有TLB

## 内存缓存
按照不同的缓存策略,x86上,一般可以将内存分为以下几类:
1. Strong Uncacheable(UC)内存,读和写操作都不会经过缓存。这种策略的内存访问效率较低,但写内存有副作用的情况下,它是正确的选择.
1. Uncacheable(UC-,又称为UC_MINUS)内存,与UC内存的唯一区别在于可以通过修改MTRR将它变成WC内存,只能通过PAT设置
1. Write Combining(WC)内存,与UC内存的唯一区别是WC的内存允许CPU缓冲多个写操作,在适当的时机一次写入,通俗点就是批量写操作
1. Write Back(WB)内存,也就是Cacheable内存,读写都会经过缓存,除非被迫刷新缓存,CPU可以自行决定何时将内容写回内存
1. Write Through(WT)内存,与WB内存类似,但写操作不仅写缓存,还会写入物理内存中
1. Write Protected(WP)内存,与WB内存类似,但每次写都会导致对应的缓存失效,只能通过MTRR设置

很显然,WB内存的访问效率最高,但并不是所有内存都可以设置成WB类型,至少需要满足两个基本条件:内存可读和写内存不能有副作用(side effect free)。所谓副作用就是写内存会触发其他操作,比如写设备的寄存器相关的MMIO会导致设备状态或行为产生变化。将这部分内存缓存起来会导致这些变化滞后,因为数据只是写到
了缓存中,并没有真正生效.

有两种方式修改内存缓存类型,就是MTRR和PAT:
1. MTRR的全称是Memory Type Range Register, 可以通过它设置一段物理内存的缓存策略.

	BIOS一般会为内存配置合理的缓存方式,开机后可以在/proc/mtrr文件中查看. 

	通过读写或者使用ioctl操作/proc/mtrr查看或者更改一段内存的缓存方式.

	MTRR有两个限制:一是只能按块配置缓存方式,二是有最大数量限制(硬件相关)
1. PAT(Page Attribute Table)可以对MTRR有效补充,它的粒度为页 ( Page ) , 而且没有数量限制

	编译内核的时候设置CONFIG_X86_PAT=y使能PAT

	PAT可以按照页控制内存的缓存属性,它的原理是修改页表项的属性位.

	内核定义了一系列PAT相关的函数满足不同的场景,它们多是平台相关的:
	1. mmio等

		- ioremap : UC-
		- ioremap_cache: WB
		- ioremap_uc: UC
		- ioremap_nocache: UC-
		- ioremap_wc: wc
		- ioremap_wt: wt

	1. ram

		- set_memory_wb: wb(默认)
		- set_memory_uc: UC-
		- set_memory_wc: WC
		- set_memory_wt: WT
	
除了直接调用函数外,还有一些特殊的文件可以控制缓存属性.

pci设备的resource文件,如果以_wc结尾,映射该文件得到的内存是WC类型,否则就是UC-类型.

通过/dev/mem文件, 可以根据物理内存的偏移量映射它获得虚拟内存. 如果文件置位了O_SYNC标志,映射后得到的内存是UC-类型,否则由当前指定的缓存类型和MTRR决定.

内核编译了debugfs的情况下,可以通过/sys/kernel/debug/x86/pat_memtype_list(/sys/kernel/debug是debugfs挂载点,不同的系统可能有差异)文件查看PAT列表.

除了CPU之外, 许多设备比如DMA也可访问内存, 也会存在一致性问题.

刷新缓存可以解决一致性的问题,可惜的是内核并没有统一的函数完成该任务,我们可以在x86上使用wbinvd(wb invalid),在arm上使用flush_cache类函数(flush_cache_range、flush_cache_all等).

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
swappiness默认60, 较高时倾向于在内存中保留页缓存, 较低时更倾向于清理也缓存而不是进行交换.

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

## `/proc/meminfo`
ref:
- [proc - process information pseudo-filesystem](https://man7.org/linux/man-pages/man5/proc.5.html)
- [**/PROC/MEMINFO之谜**](http://linuxperf.com/?cat=7)
- [解析meminfo](https://juejin.cn/post/7017002099254755335)

负责输出/proc/meminfo的源代码是：
fs/proc/meminfo.c : [`meminfo_proc_show()`](https://elixir.bootlin.com/linux/latest/source/fs/proc/meminfo.c#L32)

> page cache比较特殊，很难区分是属于kernel还是属于进程，其中被进程mmap的页面自然是属于进程的了，而另一些页面没有被mapped到任何进程，那就只能算是属于kernel了.

字段:
- MemTotal: 可供linux内核分配的内存总量

    比物理内存总量少一点，因为主板/固件会保留一部分内存、linux内核二进制和页描述符等数据在引导阶段就分配了, 也会占用一部分内存. 通过`dmesg |grep -i "Reserved"`可看到它们.

    这个值在系统运行期间一般是固定不变的. 可参阅[解读DMESG中的内存初始化信息](http://linuxperf.com/?p=139)
- MemFree: 空闲内存即系统尚未分配的内存
- MemAvailable: 当前可用内存

    有些应用程序会根据系统的可用内存大小自动调整内存申请的多少，所以需要一个记录当前可用内存数量的统计值，MemFree并不适用.

    因为MemFree只是尚未分配的内存，并不是所有可用的内存, 有些已经分配掉的内存是可以回收再分配的, 比如`cache/buffer、slab`都有一部分是可以回收的，这部分可回收的内存加上MemFree才是系统可用的内存，即MemAvailable. 同时要注意，MemAvailable是内核使用特定的算法估算出来的，并不精确.

    在kylin v10 arm版上遇到过`MemAvailable<MemFree`

    linux使用伙伴系统分配器管理空闲页的，因此MemFree等于伙伴系统空闲页.

- Buffers: 给文件的缓冲大小

    块设备(block device)所占用的缓存页，包括：直接读写块设备，以及文件系统元数据(metadata)比如superblock使用的缓存页. 它与“Cached”的区别在于，”Cached”表示普通文件所占用的缓存页.

    Buffers内存页同时也在LRU list中，被统计在Active(file)或Inactive(file)之中.

    > 通过阅读源代码可知，块设备的读写操作涉及的缓存被纳入了LRU，以读操作为例，do_generic_file_read()函数通过 mapping->a_ops->readpage() 调用块设备底层的函数，并调用 add_to_page_cache_lru() 把缓存页加入到LRU list中。参见：filemap.c: do_generic_file_read > add_to_page_cache_lru

- Cached: 高速缓冲存储器(http://baike.baidu.com/view/496990.htm)使用的大小

    所有file-backed pages.

    - Cached是`Mapped`的超集，就是说它不仅包括mapped，也包括unmapped的页面，当一个文件不再与进程关联之后，原来在page cache中的页面并不会立即回收，仍然被计入Cached，还留在LRU中，但是 Mapped 统计值会减小.ummaped = (Cached – Mapped)
    - Cached包含tmpfs中的文件，POSIX/SysV shared memory，以及shared anonymous mmap

    “Cached”和”SwapCached”两个统计值是互不重叠的. 所以，Shared memory和tmpfs在不发生swap-out的时候属于”Cached”，而在swap-out/swap-in的过程中会被加进swap cache中、属于”SwapCached”，一旦进了”SwapCached”，就不再属于”Cached”了。

    用户进程的内存页分为两种：file-backed pages（与文件对应的内存页）和anonymous pages（匿名页），比如进程的代码、映射的文件都是file-backed，而进程的堆、栈都是不与文件相对应的、就属于匿名页. file-backed pages在内存不足的时候可以直接写回对应的硬盘文件里，称为page-out，不需要用到交换区(swap)；而anonymous pages在内存不足时就只能写到硬盘上的交换区(swap)里，称为swap-out.


    Active(file)+Inactive(file)】不等于 Cached:
    1. 因为”Shmem”(即shared memory & tmpfs)包含在Cached中，而不在Active(file)和Inactive(file)中；
    2. Active(file)和Inactive(file)还包含Buffers

    如果不考虑mlock的话，一个更符合逻辑的等式是：【Active(file) + Inactive(file) + Shmem】== 【Cached + Buffers】
    如果有mlock的话，等式应该如下（mlock包括file和anon两部分，/proc/meminfo中并未分开统计，下面的mlock_file只是用来表意，实际并没有这个统计值）：
    【Active(file) + Inactive(file) + Shmem + mlock_file】== 【Cached + Buffers】
- SwapCached: 包含的是被确定要swap-out，但是尚未写入交换区的匿名内存页

    需要用到交换区的内存包括：”AnonPages”和”Shmem”, 姑且把它们统称为匿名页.

    交换区可以包括一个或多个交换区设备（裸盘、逻辑卷、文件都可以充当交换区设备），每一个交换区设备都对应自己的swap cache，可以把swap cache理解为交换区设备的”page cache”：page cache对应的是一个个文件，swap cache对应的是一个个交换区设备，kernel管理swap cache与管理page cache一样，用的都是radix-tree，唯一的区别是：page cache与文件的对应关系在打开文件时就确定了，而一个匿名页只有在即将被swap-out的时候才决定它会被放到哪一个交换区设备，即匿名页与swap cache的对应关系在即将被swap-out时才确立。

    并不是每一个匿名页都在swap cache中，只有以下情形之一的匿名页才在：

    - 匿名页即将被swap-out时会先被放进swap cache，但通常只存在很短暂的时间，因为紧接着在pageout完成之后它就会从swap cache中删除，毕竟swap-out的目的就是为了腾出空闲内存；
    
        【注：参见mm/vmscan.c: shrink_page_list()，它调用的add_to_swap()会把swap cache页面标记成dirty，然后它调用try_to_unmap()将页面对应的page table mapping都删除，再调用pageout()回写dirty page，最后try_to_free_swap()会把该页从swap cache中删除。】
    - 曾经被swap-out现在又被swap-in的匿名页会在swap cache中，直到页面中的内容发生变化、或者原来用过的交换区空间被回收为止。
    
        【注：当匿名页的内容发生变化时会删除对应的swap cache，代码参见mm/swapfile.c: reuse_swap_page()】
    
    /proc/meminfo中的SwapCached背后的含义是：系统中有多少匿名页曾经被swap-out、现在又被swap-in并且swap-in之后页面中的内容一直没发生变化。也就是说，如果这些匿名页需要被swap-out的话，是无需进行I/O write操作的。

    SwapCached内存页会同时被统计在`LRU`和`AnonPages/Shmem`中，它本身并不占用额外的内存
- Active: active包含active anon和active file. 活跃使用中的高速缓冲存储器页面文件大小

    LRU是一种内存页回收算法，Least Recently Used,最近最少使用。LRU认为，在最近时间段内被访问的数据在以后被再次访问的概率，要高于最近一直没被访问的页面。于是近期未被访问到的页面就成为了页面回收的第一选择。Linux kernel会记录每个页面的近期访问次数，然后设计了两种LRU list: active list 和 inactive list, 刚访问过的页面放进active list，长时间未访问过的页面放进inactive list，回收内存页时，直接找inactive list即可。另外，内核线程kswapd会周期性地把active list中符合条件的页面移到inactive list中。
- Inactive: 不经常使用的高速缓冲存储器页面文件大小
- Active(anon): 活跃匿名页，anonymous pages（匿名页）
- Inactive(anon): 非活跃匿名页
- Active(file): 活跃文件内存页
- Inactive(file): 非活跃文件内存页
- Unevictable: 因为种种原因无法回收(page-out)或者交换到swap(swap-out)的内存页

    Unevictable LRU list上是不能pageout/swapout的内存页，包括VM_LOCKED的内存页、SHM_LOCK的共享内存页（同时被统计在Mlocked中）、和ramfs。在unevictable list出现之前，这些内存页都在Active/Inactive lists上，vmscan每次都要扫过它们，但是又不能把它们pageout/swapout，这在大内存的系统上会严重影响性能，unevictable list的初衷就是避免这种情况的发生
- Mlocked: 被系统调用"mlock()"锁定到内存中的页面。Mlocked页面是不可收回的。

    被锁定的内存因为不能pageout/swapout，会从Active/Inactive LRU list移到Unevictable LRU list上。也就是说，当”Mlocked”增加时，”Unevictable”也同步增加，而”Active”或”Inactive”同时减小；当”Mlocked”减小的时候，”Unevictable”也同步减小，而”Active”或”Inactive”同时增加.

    Mlocked与以下统计项重叠：LRU Unevictable，AnonPages，Shmem，Mapped等。
- SwapTotal: swap空间总计
- SwapFree: 当前剩余swap
- Dirty: 需要写入磁盘的内存页的大小

    Dirty并不包括系统中全部的dirty pages，需要再加上另外两项：NFS_Unstable 和 Writeback，NFS_Unstable是发给NFS server但尚未写入硬盘的缓存页，Writeback是正准备回写硬盘的缓存页, 因此`系统中全部dirty pages = ( Dirty + NFS_Unstable + Writeback )`

    anonymous pages不属于dirty pages。参见mm/vmscan.c: page_check_dirty_writeback()的“Anonymous pages are not handled by flushers and must be written from reclaim context.”
- Writeback: 正在被写回的内存页的大小
- AnonPages: Anonymous pages(匿名页)数量 + AnonHugePages(透明大页)数量

    - 所有page cache里的页面(Cached)都是file-backed pages，不是Anonymous Pages。”Cached”与”AnoPages”之间没有重叠.
    - mmap private anonymous pages属于AnonPages(Anonymous Pages)，而mmap shared anonymous pages属于Cached(file-backed pages)，因为shared anonymous mmap也是基于tmpfs的
    - Anonymous Pages是与用户进程共存的，一旦进程退出，则Anonymous pages也释放，不像page cache即使文件与进程不关联了还可以缓存

    进程所占的内存页分为anonymous pages和file-backed pages，理论上，所有进程的PSS之和 = Mapped + AnonPages。PSS是Proportional Set Size，每个进程实际使用的物理内存（比例分配共享库占用的内存），可以在`/proc/[1-9]*/smaps`中查看


    Active(anon)+Inactive(anon)】不等于AnonPages:
    因为Shmem(即Shared memory & tmpfs) 被计入LRU Active/Inactive(anon)，但未计入 AnonPages。所以一个更合理的等式是：【Active(anon)+Inactive(anon)】 = 【AnonPages + Shmem】, 但是这个等式在某些情况下也不一定成立，因为：
    1. 如果shmem或anonymous pages被mlock的话，就不在Active(non)或Inactive(anon)里了，而是到了Unevictable里，以上等式就不平衡了；
    1. 当anonymous pages准备被swap-out时，分几个步骤：先被加进swap cache，再离开AnonPages，然后离开LRU Inactive(anon)，最后从swap cache中删除，这几个步骤之间会有间隔，而且有可能离开AnonPages就因某些情况而结束了，所以在某些时刻以上等式会不平衡。
    【注：参见mm/vmscan.c: shrink_page_list()：它调用的add_to_swap()会把swap cache页面标记成dirty，然后调用try_to_unmap()将页面对应的page table mapping都删除，再调用pageout()回写dirty page，最后try_to_free_swap()把该页从swap cache中删除。】
- Mapped: 正被用户进程关联的file-backed pages

    Cached包含了所有file-backed pages，其中有些文件当前不在使用，但Cached仍然可能保留着它们的file-backed pages；而另一些文件正被用户进程关联，比如shared libraries、可执行程序的文件、mmap的文件等，这些文件的缓存页就称为mapped.

    /proc/meminfo中的”Mapped”就统计了page cache(“Cached”)中所有的mapped页面。”Mapped”是”Cached”的子集。

    因为Linux系统上shared memory & tmpfs被计入page cache(“Cached”)，所以被attached的shared memory、以及tmpfs上被map的文件都算做”Mapped”。


    Active(file)+Inactive(file)】不等于Mapped:
    1. 因为LRU Active(file)和Inactive(file)中不仅包含mapped页面，还包含unmapped页面；
    2. Mapped中包含”Shmem”(即shared memory & tmpfs)，这部分内存被计入了LRU Active(anon)或Inactive(anon)、而不在Active(file)和Inactive(file)中
- Shmem

    Shmem统计的内容包括:
    1. shared memory

        - SysV shared memory [shmget etc.]
        - POSIX shared memory [shm_open etc.]
        - shared anonymous mmap [ mmap(…MAP_ANONYMOUS|MAP_SHARED…)]
    2. tmpfs和devtmpfs.

    所有tmpfs类型的文件系统占用的空间都计入共享内存，devtmpfs是/dev文件系统的类型，/dev/下所有的文件占用的空间也属于共享内存。可以用ls和du命令查看。如果文件在没有关闭的情况下被删除，空间仍然不会释放，shmem不会减小，可以用`lsof -a +L1 /<mount_point>`这样的文件

    因为[shared memory在内核中都是基于tmpfs实现的](https://www.kernel.org/doc/Documentation/filesystems/tmpfs.txt), 因此它们被视为基于tmpfs文件系统的内存页，既然基于文件系统，就不算匿名页，所以不被计入/proc/meminfo中的AnonPages，而是被统计进了：Cached或Mapped(当shmem被attached时候)。然而它们背后并不存在真正的硬盘文件，一旦内存不足的时候，它们是需要交换区才能swap-out的，所以在LRU lists里，它们被放在
    1. Inactive(anon) 或 Active(anon)

        虽然它们在LRU中被放进了anon list，但是不会被计入 AnonPages。这是shared memory & tmpfs比较拧巴的一个地方，需要特别注意
    2. unevictable （如果被locked的话）

    当shmget/shm_open/mmap创建共享内存时，物理内存尚未分配，要直到真正访问时才分配. /proc/meminfo中的 Shmem 统计的是已经分配的大小，而不是创建时申请的大小
- Slab: 通过slab分配的内存，Slab=SReclaimable+SUnreclaim. 内核数据结构缓存的大小，可减少申请和释放内存带来的消耗

    slab是linux内核的一种内存分配器.
- SReclaimable: slab中可回收的部分

    调用kmem_getpages()时加上SLAB_RECLAIM_ACCOUNT标记，表明是可回收的，计入SReclaimable，否则计入SUnreclaim.
- SUnreclaim: slab中不可回收的部分
- KernelStack: 给用户线程分配的内核栈消耗的内存页

    每一个用户线程都会分配一个kernel stack（内核栈），内核栈虽然属于线程，但用户态的代码不能访问，只有通过系统调用(syscall)、自陷(trap)或异常(exception)进入内核态的时候才会用到，也就是说内核栈是给kernel code使用的。在x86系统上Linux的内核栈大小是固定的8K或16K。Kernel stack（内核栈）是常驻内存的，既不包括在LRU lists里，也不包括在进程的RSS/PSS内存里。RSS是Resident Set Size 实际使用物理内存（包含共享库占用的内存），可以在`/proc/[1-9]*/smaps`中查看
- PageTables: Page Table的消耗的内存页. 管理内存分页的索引表的大小

    Page Table的用途是翻译虚拟地址和物理地址，随着内存地址分配得越来越多，Page Table会增大，/proc/meminfo中的PageTables统计了Page Table所占用的内存大小.

    需把Page Table与Page Frame（页帧）区分开，物理内存的最小单位是page frame，每个物理页对应一个描述符([struct page](https://elixir.bootlin.com/linux/v6.5.1/source/include/linux/mm_types.h#L74))，在内核的引导阶段就会分配好、保存在mem_map[]数组中，mem_map[]所占用的内存被统计在dmesg显示的reserved中，/proc/meminfo的MemTotal是不包含它们的。（在NUMA系统上可能会有多个mem_map数组，在node_data中或mem_section中）.

    > page 结构正是通过 flags 表示它处于哪种状态，根据不同的状态来使用 union 联合体的变量表示的数据信息。如果 page 处于空闲状态，它就会使用 union 联合体中的 lru 字段，挂载到对应空闲链表中
- NFS_Unstable: 发给NFS server但尚未写入硬盘的缓存页

    NFS_Unstable的内存被包含在Slab中，因为nfs request内存是调用kmem_cache_zalloc()申请的.
- Bounce: bounce buffering消耗的内存页

    有些老设备只能访问低端内存，比如16M以下的内存，当应用程序发出一个I/O 请求，DMA的目的地址却是高端内存时（比如在16M以上），内核将在低端内存中分配一个临时buffer作为跳转，把位于高端内存的缓存数据复制到此处

    这种额外的数据拷贝被称为“bounce buffering”，会降低I/O 性能。大量分配的bounce buffers 也会占用额外的内存.
- WritebackTmp: 正准备回写硬盘的缓存页
- CommitLimit: overcommit阈值，CommitLimit = (Physical RAM * vm.overcommit_ratio / 100) + Swap

    Linux是允许memory overcommit的，即承诺给进程的内存大小超过了实际可用的内存。commit(或overcommit)针对的是内存申请，内存申请不等于内存分配，内存只在实际用到的时候才分配。但可以申请的内存有个上限阈值，即CommitLimit，超出以后就不能再申请了。
- Committed_AS: 所有进程已经申请的内存总大小
- VmallocTotal: 虚拟内存大小. 可分配的虚拟内存总计
- VmallocUsed: 已通过vmalloc分配的内存，不止包括了分配的物理内存，还统计了VM_IOREMAP、VM_MAP等操作的值

    VM_IOREMAP是把IO地址映射到内核空间、并未消耗物理内存, 需要排除. 通过`grep vmalloc /proc/vmallocinfo`能看到vmalloc来自哪个调用者(caller), 那是vmalloc()记录下来的，相应的源代码可见：`mm/vmalloc.c: vmalloc > __vmalloc_node_flags > __vmalloc_node > __vmalloc_node_range > __get_vm_area_node > setup_vmalloc_vm`

    通过vmalloc分配了多少内存，可以统计/proc/vmallocinfo中的vmalloc记录: `grep vmalloc /proc/vmallocinfo | awk '{total+=$2}; END {print total}'`

    一些driver以及网络模块和文件系统模块可能会调用vmalloc，加载内核模块(kernel module)时也会用到，可参见`kernel/module.c`

- VmallocChunk: 通过vmalloc可分配的虚拟地址连续的最大内存
- Percpu: 
- HardwareCorrupted: 因为内存的硬件故障而删除的内存页

    相应的代码参见`mm/memory-failure.c: memory_failure()`
- AnonHugePages:   AnonHugePages统计的是Transparent HugePages (THP)

    不要把 Transparent HugePages (THP)跟 Hugepages 搞混了.

    AnonHugePages与/proc/meminfo的其他统计项有重叠，首先它被包含在AnonPages之中，而且在/proc/<pid>/smaps中也有单个进程的统计，与进程的RSS/PSS是有重叠的，如果用户进程用到了THP，进程的RSS/PSS也会相应增加，这与Hugepages是不同的.

    THP也可以用于shared memory和tmpfs，缺省是禁止的，[打开的方法如下](https://www.kernel.org/doc/Documentation/vm/transhuge.txt):
    - mount时加上”huge=always”等选项
    - 通过/sys/kernel/mm/transparent_hugepage/shmem_enabled来控制

    因为缺省情况下shared memory和tmpfs不使用THP，所以进程之间不会共享AnonHugePages，于是就有`/proc/meminfo的AnonHugePages】== 所有进程的/proc/<pid>/smaps中AnonHugePages之和`, 即`grep AnonHugePages /proc/[1-9]*/smaps | awk '{total+=$2}; END {print total}'`=`grep AnonHugePages /proc/meminfo`

    Transparent Huge Pages 缩写 THP ，这个是 RHEL 6 开始引入的一个功能，在 Linux6 上透明大页是默认启用的。由于 Huge pages 很难手动管理，而且通常需要对代码进行重大的更改才能有效的使用，因此 RHEL 6 开始引入了 Transparent Huge Pages （ THP ）， THP 是一个抽象层，能够自动创建、管理和使用传统大页。THP 为系统管理员和开发人员减少了很多使用传统大页的复杂性 , 因为 THP 的目标是改进性能 , 因此其它开发人员 ( 来自社区和红帽 ) 已在各种系统、配置、应用程序和负载中对 THP 进行了测试和优化。这样可让 THP 的默认设置改进大多数系统配置性能。但是 , 不建议对数据库工作负载使用 THP 。这两者最大的区别在于 : 标准大页管理是预分配的方式，而透明大页管理则是动态分配的方式
- ShmemHugePages:        Memory used by shared memory (shmem) and tmpfs allocated with huge page
- ShmemPmdMapped:        Shared memory mapped into user space with huge pages
- FileHugePages:         0 kB
- FilePmdMapped:         0 kB
- HugePages_Total:    预分配的可使用的标准大页池的大小。HugePages在内核中独立管理，只要一经定义，无论是否被使用，都不再属于free memory

    Huge pages(标准大页) 是从 Linux Kernel 2.6 后被引入的，目的是通过使用大页内存来取代传统的 4kb 内存页面， 以适应越来越大的系统内存，让操作系统可以支持现代硬件架构的大页面容量功能.

    HugePages_Total 对应内核参数 vm.nr_hugepages，也可以在运行中的系统上直接修改 /proc/sys/vm/nr_hugepages，修改的结果会立即影响空闲内存 MemFree的大小，因为HugePages在内核中独立管理，只要一经定义，无论是否被使用，都不再属于free memory.

    Hugepages在/proc/meminfo中是被独立统计的，与其它统计项不重叠，既不计入进程的RSS/PSS中，又不计入LRU Active/Inactive，也不会计入cache/buffer。如果进程使用了Hugepages，它的RSS/PSS不会增加, 举例: 一个进程通过mmap()申请并使用了Hugepages，在/proc/<pid>/smaps中可以看到如下内存段，VmFlags包含的”ht”表示Hugepages, kernelPageSize是2048kB，但RSS/PSS都是0.

    [使用Hugepages有三种方式](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt):
    - mount一个特殊的 hugetlbfs 文件系统，在上面创建文件，然后用mmap() 进行访问，如果要用 read() 访问也是可以的，但是 write() 不行
    - 通过shmget/shmat也可以使用Hugepages，调用shmget申请共享内存时要加上 SHM_HUGETLB 标志
    - 通过 mmap()，调用时指定MAP_HUGETLB 标志也可以使用Huagepages

    用户程序在申请Hugepages的时候，其实是reserve了一块内存，并未真正使用，此时/proc/meminfo中的 HugePages_Rsvd 会增加，而 HugePages_Free 不会减少.
    等到用户程序真正读写Hugepages的时候，它才被消耗掉了，此时HugePages_Free会减少，HugePages_Rsvd也会减少.

- HugePages_Free:     标准大页池中尚未分配的标准大页
- HugePages_Rsvd:     用户程序预申请的标准大页，尚未真的分配走
- HugePages_Surp:     标准大页池的盈余
- Hugepagesize:       标准大页大小，这里是2M
- Hugetlb:            记录在TLB 中条目并指向到Hugepage

    TLB(A Translation Lookaside Buffer)是在cpu中分配的一个固定大小的buffer(or cache)，用于保存“page table”的部分内容，使CPU更快的访问并进行地址转换.
- DirectMap4k: 映射为4kB的内存数量

    DirectMap所统计的不是关于内存的使用，而是一个反映TLB效率的指标。TLB(Translation Lookaside Buffer)是位于CPU上的缓存，用于将内存的虚拟地址翻译成物理地址，由于TLB的大小有限，不能缓存的地址就需要访问内存里的page table来进行翻译，速度慢很多。为了尽可能地将地址放进TLB缓存，新的CPU硬件支持比4k更大的页面从而达到减少地址数量的目的， 比如2MB，4MB，甚至1GB的内存页，视不同的硬件而定。”DirectMap4k”表示映射为4kB的内存数量， “DirectMap2M”表示映射为2MB的内存数量，以此类推, 所以DirectMap其实是一个反映TLB效率的指标。
- DirectMap2M:     映射为2MB的内存数量

公式:
- Mem.total = Mem.used + Mem.free + Mem.buffer/cache

    - MemTotal-MemFree=已被用掉的内存
- 用户进程部分：

    - 围绕LRU进行统计:(Active + Inactive + Unevictable) + (HugePages_Total * Hugepagesize) 
    - 围绕Page Cache进行统计: Cached + AnonPages + Buffers + (HugePages_Total * Hugepagesize) 

        1. 当SwapCached为0的时候，用户进程的内存总计如下：【(Cached + AnonPages + Buffers) + (HugePages_Total * Hugepagesize)】
        1. 当SwapCached不为0的时候，以上公式不成立，因为SwapCached可能会含有Shmem，而Shmem本来被含在Cached中，一旦swap-out就从Cached转移到了SwapCached，可是我们又不能把SwapCached加进上述公式中，因为SwapCached虽然不与Cached重叠却与AnonPages有重叠，它既可能含有Shared memory又可能含有Anonymous Pages.
    - 围绕RSS/PSS进行统计

        把/proc/[1-9]*/smaps 中的 Pss 累加起来就是所有用户进程占用的内存，但是还没有包括Page Cache中unmapped部分、以及HugePages，所以公式如下：
        `ΣPss + (Cached - mapped) + Buffers + (HugePages_Total * Hugepagesize)`


    以上结果约等于`smem --pie name -c pss`的用户部分
- 内核部分: Slab+ VmallocUsed + PageTables + KernelStack + HardwareCorrupted + Bounce + X

    > 直接通过`alloc_pages/__get_free_page`分配的内存，没有在/proc/meminfo中统计

    > VmallocUsed其实不是我们感兴趣的，因为它还包括了VM_IOREMAP等并未消耗物理内存的IO地址映射空间，我们只关心VM_ALLOC操作，（参见1.2节），所以实际上应该统计/proc/vmallocinfo中的vmalloc记录

Kernel的动态内存分配通过以下几种接口:
- `alloc_pages/__get_free_page`: 以页为单位分配
- `vmalloc`: 以字节为单位分配虚拟地址连续的内存块
- `slab allocator`: 对小对象进行分配，不用为每个小对象分配一个页，节省了空间

        内核中一些小对象创建析构很频繁，Slab对这些小对象做缓存，可以重复利用一些相同的对象，减少内存分配次数
- kmalloc: 以slab为基础，以字节为单位分配物理地址连续的内存块. 它使用slab层的general caches — 大小为2^n，名称是kmalloc-32、kmalloc-64等（在老kernel上的名称是size-32、size-64等）


通过slab层分配的内存会被精确统计，可以参见/proc/meminfo中的slab/SReclaimable/SUnreclaim.

通过vmalloc分配的内存也有统计，参见/proc/meminfo中的VmallocUsed

而通过alloc_pages分配的内存不会自动统计，除非调用alloc_pages的内核模块或驱动程序主动进行统计，否则我们只能看到free memory减少了，但从/proc/meminfo中看不出它们具体用到哪里去了. 比如在VMware guest上有一个常见问题，就是VMWare ESX宿主机会通过guest上的Balloon driver(vmware_balloon module)占用guest的内存，有时占用得太多会导致guest无内存可用，这时去检查guest的/proc/meminfo只看见MemFree很少、但看不出内存的去向，原因就是Balloon driver通过alloc_pages分配内存，没有在/proc/meminfo中留下统计值，所以很难追踪.

## FAQ
### memory dump
`head /dev/mem |hexdump -C`

### LRU
LRU是Kernel的页面回收算法(Page Frame Reclaiming)使用的数据结构，在[解读vmstat中的Active/Inactive memory](http://linuxperf.com/?p=97)一文中有介绍。Page cache和所有用户进程的内存（kernel stack和huge pages除外）都在LRU lists上.

LRU lists包括如下几种，在/proc/meminfo中都有对应的统计值：

- LRU_INACTIVE_ANON  –  对应 Inactive(anon)
- LRU_ACTIVE_ANON  –  对应 Active(anon)
- LRU_INACTIVE_FILE  –  对应 Inactive(file)
- LRU_ACTIVE_FILE  –  对应 Active(file)
- LRU_UNEVICTABLE  –  对应 Unevictable

注：
- Inactive list里的是长时间未被访问过的内存页，Active list里的是最近被访问过的内存页，LRU算法利用Inactive list和Active list可以判断哪些内存页可以被优先回收。
- 括号中的 anon 表示匿名页(anonymous pages)
- 括号中的 file 表示 file-backed pages（与文件对应的内存页）。
- Unevictable LRU list上是不能pageout/swapout的内存页，包括VM_LOCKED的内存页、SHM_LOCK的共享内存页（又被统计在”Mlocked”中）和ramfs. 在unevictable list出现之前，这些内存页都在Active/Inactive lists上，vmscan每次都要扫过它们，但是又不能把它们pageout/swapout，这在大内存的系统上会严重影响性能，[设计unevictable list的初衷就是避免这种情况](https://www.kernel.org/doc/Documentation/vm/unevictable-lru.txt).

LRU与/proc/meminfo中其他统计值的关系：
- LRU中不包含`HugePages_*`
- LRU包含了 Cached 和 AnonPages

### kernel modules所用内存
lsmod列出的是[init_size+core_size]，这不是实际内存占用, 原因见下文.

> 可以在`/sys/module/<module-name>/`目录下分别看到coresize和initsize的值

lsmod的信息来自/proc/modules，它显示的size包括init_size和core_size，相应的源代码参见: `kernel/module.c#static int m_show(struct seq_file *m, void *p)`

kernel module的内存是通过vmalloc()分配的，所以它被包含在VmallocUsed中. 也就是说可以不必通过`lsmod`来统计kernel module所占的内存大小，通过/proc/vmallocinfo就行了，而且还比lsmod更准确. 因为给kernel module分配内存是以page为单位的，不足 1 page的部分也会得到整个page，此外，每个module还会分到一页额外的guard page。
详见：`mm/vmalloc.c: __get_vm_area_node()`

## 页表查询
ref:
- [arm PTE的页表查询](https://elixir.bootlin.com/linux/v6.6.22/source/arch/arm/lib/uaccess_with_memcpy.c#L23)

	pgd_offset, pud_offset, pmdd_offset分别是一/二/三级页表的入口, 最后通过pte_offset_map_lock得到pte. 通过pmd_thp_or_huge()判断是否有巨页, 如果是巨页, 直接访问pmd

	> linux支持不带MMU的处理器

	> 32位内存映射见<<Linux设备驱动开发详解>>的 11.2 Linux内存管理

## 内存存取
### 用户空间
用户空间使用malloc()和free()来申请和释放内存. malloc()是通过brk()和mmap()这两个系统调用从内核申请内存.

c库的malloc()通常具备二次管理能力, free()时并不是直接还给系统, 而是还给了c库的分配算法, 进程结束或c库归还内存时才真正释放了内存.

### kernel
内核申请内存涉及的函数主要包括kmalloc(), `__get_free_pages()`, vmalloc()等. kmalloc()和`__get_free_pages()`(及其类似函数)申请的内存, 物理地址是连续的, 只是与真实物理地址有一个固定的偏移, 映射关系简单. vmalloc()在虚拟内存空间分配一块连续的内存, 实质上, 物理内存不一定连续, 没法简单映射.

> kmalloc底层依赖`__get_free_pages()`, 使用GFP_KERNEL, 暂时不能满足时会睡眠进程等待页; 在中断处理函数, tasklet和内核定时器等非进程上下文中不能阻塞, 此时应使用GFP_ATOMIC, 如果不存在空闲页, 不等待, 直接返回. 还有很多申请的flag, 这里不举例了.

> kmalloc申请的, 使用kfree释放.

`__get_free_pages()`系列函数/宏的本质是内核最底层用于获取空闲内存的方法. 因为底层使用buddy算法以2^n页为单位管理内存, 所有它申请的内存总是以2^n页为单位.

> `__get_free_pages()`系列函数/宏包括get_zeroed_page(), `__get_free_page()`和`__get_free_pages()`. 它们的申请内存flag与kmalloc一致.

vmalloc()远大于__get_free_pages()的开销, 它适合分配较大的内存, 用vfree()释放.

vmalloc不能用在原子上下文中, 因为它内部实现使用了GFP_KERNEL的kmalloc().

vmalloc在申请内存时, 会进行内存映射, 改变页表项.

### slub和内存池
以页为单位分配和释放内存容易导致浪费, 特别是涉及大量小对象的时候, 比如inode, task_struct等.

slub建立在buddy上, 它从buddy拿到2^n页面后进行二次管理. 通过`cat /proc/slabinfo`可获得slub的分配和使用情况.

linux使用mempool_create创建内存池.

### 内存映射和VMA
一般情况下, 用户空间是不能也不应该直接访问设备. 但驱动可用mmap()使得用户空间能直接访问设备的物理地址. 这种能力对显示器很有用.

> mmap()必须以PAGE_SIZE为单位进行映射.

驱动中的mmap()的实现机制是建立页表, 并填充VMA结构体(vm_area_struct)中vm_operations_struct指针.

在驱动中, 可使用remap_pfn_range()映射内存中的保留页, 设备I/O, framebuffer, camera等内存. 在remap_pfn_range基础上, 进一步封装出io_remap_pfn_range(), vm_iomap_memory()等api.

> LCD驱动映射framebuffer物理地址到用户空间的范例, 见[fb_mmap](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/video/fbdev/core/fb_chrdev.c#L314)

通常, I/O内存被映射时需要时nocache, 即将vma->vm_page_prot设为nocache后再映射.

除了remap_pfn_range(), 实现VMA的fault()可为设备提供更加灵活的内存映射途径.

### IOMMU
针对外设总线和内存地址间的转化

## 缺页异常
缺页异常不止没有对应的物理内存一种,访问权限不足等也会导致异常. 。为了帮助区分、处理缺页异常, CPU会额外提供两项信息:错误码和异常地址.

错误码error_code存储在栈中,包含以下信息:
1. 异常的原因是物理页不存在,还是访问权限不足,由X86_PF_PROT标志区分,当error_code&X86_PF_PROT等于0时,表示前者
1. 导致异常时,处于用户态还是内核态,由X86_PF_USER标志区分
1. 导致异常的操作是读还是写,由X86_PF_WRITE标志区分
1. 物理页存在,但页目录项或者页表项使用了保留的位, X86_PF_RSVD标志会被置位
1. 读取指令的时候导致异常,X86_PF_INSTR标志会被置位
1. 访问违反了protection key导致异常,X86_PF_PK标志会被置位. 所谓protection key,简单理解就是有些寄存器可以控制部分内存的读写权限,越权访问会导致异常

异常地址,就是导致缺页异常的虚拟地址,存储在CPU的cr2寄存器中.

常见的导致缺页异常的场景和合理的处理方式:
1. 程序逻辑错误

	分为以下三类:
	1. 访问不存在的地址,最简单的是访问空指针
	1. 访问越界,比如用户空间的程序访问了内核的地址
	1. 违反权限,比如以只读形式映射内存的情况下写内存

	缺页异常对此无能为力,程序的执行没有按照程序员的预期执行,应该是程序员来解决。缺页异常并不是用来解决程序错误的,对此只能是oops、kernel panic等.
1. 访问的地址未映射物理内存

	这是正常的,也是对系统有益的. 物理内存有限, 按需映射物理内存可以避免浪费
1. TLB过时,页表更新后,TLB未更新,这种情况下,绕过TLB访问内存中的页表即可
1. COW(Copy On Write,写时复制)等,内存没有写权限,写操作导致缺页异常,但它与第一种场景的第三类错误是不同的, 产生异常的内存是可以有写权限的,是`可以有`和`真没有`的区别

### 处理缺页异常
缺页异常的处理函数是page_fault,它是由汇编语言完成的,除了保存现场外,它还会从栈中获得error_code,然后调用do_page_fault函数. 后者读取cr2寄存器得到导致异常的虚拟地址,然后调用`__do_page_fault`, `__do_page_fault`根据不同的场景处理异常.

X86_PF_RSVD类型的异常被当作一种错误:使用保留位有很大的风险, 它们都是X86预留未来使用的,使用它们有可能会造成冲突。所以无论产生异常的地址属于内核空间(第1步)还是用户空间(第3
步),都不会尝试处理这种情况,而是产生错误.

如果导致异常的虚拟地址address属于内核空间, X86_PF_USER意味着程序在用户态访问了内核地址, 同样是错误,能够得到处理的只有vmalloc和spurious等. 使用vmalloc申请内存的时候更新的是主内核
页表, 并没有更新进程的页表(进程拥有独立页表的情况下), 进程在访问这部分地址时就会产生异常,此时只需要复制主内核页表的内容给进程的页表即可, 这也是第1步的vmalloc_fault函数的逻辑.

内核中,页表的访问权限更新了,出于效率的考虑,可能不会立即刷新TLB. 比如某段内存由只读变成读写,TLB没有刷新,写这段内存可能导致X86_PF_PROT异常,spurious_fault就是处理这类异常
的,也就是导致缺页异常的第三种场景.

如果异常得不到有效处理,就属于bad_area,会调用bad_area、bad_area_nosemaphore、bad_area_access_error等函数,它们的逻辑类似 , 如果产生异常时进程处于用户态(X86_PF_USER),发送
SIGSEGV(SEGV的全称为Segmentation Violation)信号给进程;如果进程处于内核态,尝试修复错误,失败的情况下使进程退出。

从第3步开始,导致异常的虚拟地址都属于用户空间,所以问题就变成找到address所属的vma,映射内存。第4步,find_vma找到第一个
结尾地址(vma->vm_end)大于address的vma(进程的vma是有顺序的),找不到则出错。如果该vma包含了address,那么它就是address
所属的vma,否则尝试扩大vma来包含address,失败则出错.

第5步,找到了vma之后,过滤几种因为权限不足导致异常的场景:内存映射为只读,尝试写内存;读取不可读的内存;内存映射为不可读、不可写、不可执行。这几种场景都是程序逻辑错误,不予处
理.

到了第6步,才真正进入处理缺页异常的核心部分,前5步讨论了address属于内核空间的情况和哪些情况算作错误两个话题,算是“前
菜”。至此, 可以把导致缺页异常的场景总结为错误和异常两类,除了vmalloc_fault和spurious_fault外,前面讨论的场景均为错误,都不会得到处理,理解缺页异常的第一个关键就是清楚处理异常并不是为
了纠正错误.

handle_mm_fault调用__handle_mm_fault继续处理异常. 到此已经找到了address所属的vma,目标是完成address所需的映射。面临
的问题可以分为以下三种类型:
1. 没有完整的内存映射,也就是没有映射物理内存,申请物理内存完成映射即可
1. 映射完整,但物理页已经被交换出去,需要将原来的内容读入内存,完成映射
1. 映射完整,内存映射为可写,页表为只读,写内存导致异常,常见的情况就是COW

`__do_page_fault`的第5步和类型3都提到了内存映射的权限和页表的权限,在此总结.

用户空间虚拟内存访问权限分为两部分,一部分存储在 vma->vm_flags中(VM_READ、VM_EXEC和VM_WRITE等),另一部分存储在页表中. 前者表示内存映射时设置的访问权限(内存映射的权
限),表示被允许的访问权限,是一个全集,允许范围外的访问是错误,比如尝试写以PROT_READ方式映射的内存。后者表示实际的访问权限(页表的权限),内存映射后,物理页的访问权限可能会发生
变化,比如以可读可写方式映射的内存,在某些情况下页表被改变,变成了只读,写内存就会导致异常,这种情况不是错误,因为访问是权限允许范围内的,COW就是如此。这是理解缺页异常的第二个关
键,区分内存访问权限的两部分。

明确了问题之后,`__handle_mm_fault`的逻辑就清晰了,它访问address对应的页目录,如果某一级的项为空,表示是第一种问题,申请一页内存填充该项即可。它访问到pmd(页中级目录),接下来这
三种类型的问题的“分水岭”出现了:pte内容的区别导致截然不同的处理逻辑,由handle_pte_fault函数完成.

vm_fault结构体(以下简称vmf)是用来辅助处理异常的,保存了处理异常需要的信息.

```
// https://elixir.bootlin.com/linux/v6.6.28/source/include/linux/mm.h#L508
/*
 * vm_fault is filled by the pagefault handler and passed to the vma's
 * ->fault function. The vma's ->fault is responsible for returning a bitmask
 * of VM_FAULT_xxx flags that give details about how the fault was handled.
 *
 * MM layer fills up gfp_mask for page allocations but fault handler might
 * alter it if its implementation requires a different allocation context.
 *
 * pgoff should be used in favour of virtual_address, if possible.
 */
struct vm_fault {
	const struct {
		struct vm_area_struct *vma;	/* Target VMA */ // address对应的vma
		gfp_t gfp_mask;			/* gfp mask to be used for allocations */
		pgoff_t pgoff;			/* Logical page offset based on vma */ // address相对于映射文件的偏移量, 以页为单位
		unsigned long address;		/* Faulting virtual address - masked */
		unsigned long real_address;	/* Faulting virtual address - unmasked */
	};
	enum fault_flag flags;		/* FAULT_FLAG_xxx flags
					 * XXX: should really be 'const' */ FAULT_FLAG_xxx标志
	pmd_t *pmd;			/* Pointer to pmd entry matching
					 * the 'address' */ // 页中级目录项的指针
	pud_t *pud;			/* Pointer to pud entry matching
					 * the 'address'
					 */ // 页上级目录项 的指针
	union {
		pte_t orig_pte;		/* Value of PTE at the time of fault */ // 导致异常时页表项的内存
		pmd_t orig_pmd;		/* Value of PMD at the time of fault,
					 * used by PMD fault only.
					 */
	};

	struct page *cow_page;		/* Page handler may use for COW fault */ // cow使用的内存页
	struct page *page;		/* ->fault handlers should return a
					 * page here, unless VM_FAULT_NOPAGE
					 * is set (which is also implied by
					 * VM_FAULT_ERROR).
					 */ // 处理异常函数返回的内存页
	/* These three entries are valid only while holding ptl lock */
	pte_t *pte;			/* Pointer to pte entry matching
					 * the 'address'. NULL if the page
					 * table hasn't been allocated.
					 */ // 页表项的指针
	spinlock_t *ptl;		/* Page table lock.
					 * Protects pte page table if 'pte'
					 * is not NULL, otherwise pmd.
					 */
	pgtable_t prealloc_pte;		/* Pre-allocated pte page table.
					 * vm_ops->map_pages() sets up a page
					 * table from atomic context.
					 * do_fault_around() pre-allocates
					 * page table to avoid allocation from
					 * atomic context.
					 */
};
```

,进入handle_pte_fault函数前,除了与pte和page相关的字段外,其他多数字段都已经被__handle_mm_fault函数赋值了,它处理到
pmd 为 止 。 pgoff 由 计 算 得 来 , 等 于 (address-vma->vm_start)>>PAGE_SHIFT加上vma->vm_pgoff.

第1步,判断pmd项是否有效(指向一个页表),无效则属于第一种问题;有效则判断pte是否有效,无效也属于第一种问题,有效则属
于后两种.

请注意区分pte_none和!vmf->pte,前者判断页表项的内容是否有效,后者判断pte是否存在。pmd项没有指向页表的情况下后者为真, 页表项没有期望的内容时前者为真。

第2步针对第一种问题,非匿名映射由do_fault函数处理。do_fault根据不同的情况调用相应的函数.

如果是读操作导致异常,调用do_read_fault;如果是写操作导致异常,以MAP_PRIVATE映射的内存,调用do_cow_fault;如果是写操作导致异常,以MAP_SHARED映射的内存,调用do_shared_fault.

以上三个do_xxx_fault都会调用__do_fault,后者回调vma->vm_ops->fault函数得到一页内存(vmf->page);得到内存后,再调用finish_fault函数更新页表完成映射。do_read_fault和do_shared_fault
的区别在于内存的读写权限,do_cow_fault与它们的区别在于最终使用的物理页并不是得到的vmf->page,它会重新申请一页内存(vmf->cow_page) , 将 vmf->page 复 制 到 vmf->cow_page , 然 后 使 用 vmf->cow_page更新页表.

此处需要强调两点,首先,vma->vm_ops->fault是由映射时的文件定义的(vma->vm_file),文件需要根据vmf的信息返回一页内存,赋值给vmf->page。其次,do_cow_fault最终使用的物理页是新申请的
vmf->cow_page,与文件返回的物理页只是此刻内容相同,此后便没有任何关系,之后写内存并不会更新文件.

第3步,页表项内容有效,但物理页不存在,也就是第二种问题, 由do_swap_page函数处理.

第4步,写操作导致异常,但物理页没有写权限,也就是第三种问题,由do_wp_page函数处理。需要注意的是,写映射为只读的内存导致异常的情况已经被__do_page_fault函数的第5步过滤掉了,所以此处
的情况是,内存之前被映射为可写,但实际不可写,具体场景之后分析.

do_wp_page主要处理三种情况,前两种情况见下面的分析,第三种在下节分析.
1. 以PROT_WRITE(可写)和MAP_SHARED(共享)标志映射的内存,调用wp_pfn_shared函数尝试将其变为可写即可.
2. 以PROT_WRITE(可写)和MAP_PRIVATE(不共享)标志映射的内存,就是所谓的COW,调用wp_page_copy函数处理。wp_page_copy新申请一页内存,复制原来页中的内容,更新页表,后续写操作只会更新新的内存,与原内存无关

举例理解:
1. mmap映射内存,得到虚拟地址,如果实际上并没有物理地址与之对应,访问内存会导致缺页异常,由handle_pte_fault的第2步处理。接下来不同的情况由不同的函数处理,结果也不一样

	场景, 处理函数, 结果:
	- 读访问, do_read_fault, 完成映射, 不尝试将内存变为可写
	- 写访问, MAP_SHARED映射内存, do_shared_fault, 完成映射, 尝试将内存变为可写
	- 写访问, MAP_PRIVATE映射内存, do_cow_fault, 申请一页新内存, 复制内容, 完成映射, 新内存可写
2.  接例1,读访问导致异常处理后,内存依然不可写,但此时内存映射是完整的,写内存会导致异常,由handle_pte_fault的第4步处理,至于是do_wp_page函数处理的哪种情况由映射的方式决定
3. mmap映射内存,得到虚拟地址,并没有物理地址与之对应的情况下,内核调用get_user_pages尝试访问该内存。由于此时内存映射不完整,内核会调用handle_mm_fault完成映射,这种情况下物理页
的访问权限由内核的需要决定。如果访问权限为只读,用户空间下次写该内存就会导致异常,处理过程与例2相同
4. 以MAP_PRIVATE方式映射内存,得到的内存不可写,写内存导致异常,处理过程与例1的情况3类似

### cow
COW的全称是Copy On Write,也就是写时复制,handle_pte_fault的第2步和第4步都有它的身影。从缺页异常的处理过程来看,缺页异常认定为COW的条件是以MAP_PRIVATE方式映射的内存,映射不完
整,或者物理内存权限为只读;写内存(FAULT_FLAG_WRITE)。

由此, 可以总结出COW的认定条件.

首先,必须是以MAP_PRIVATE方式映射的内存,它的意图是对内存的修改,对其他进程不可见,而以MAP_SHARED方式映射的内
存,本意就是与其他进程共享内存,并不存在复制一说,也就不存在COW中的Copy了.

其次,写内存导致异常。有两层含义,首先必须是写,也就是COW中的Write;其次是导致异常,也就是映射不完整或者访问权限
不足。映射不完整容易理解,就是得到虚拟内存后,没有实际的物理内存与之对应。至于访问权限不足,上节中的例2到例4都属于这种情
况,访问权限不足的场景,与子进程的创建有关。

子进程被创建时,会负责父进程的很多信息,包括部分内存信息,其中复制内存映射信息的任务由dup_mmap函数完成。
dup_mmap函数复制父进程的没有置位VM_DONTCOPY标志的vma,调用copy_page_range函数尝试复制vma的页表项信息。后者访问
子进程的四级页目录,如果不存在则申请一页内存并使用它更新目录项,然后复制父进程的页表项,完成映射。这就是复制vma信息的一
般过程,不过实际上存在以下几种情况需要特殊处理。

copy_page_range不会复制父进程以MAP_SHARED方式映射普通内存对应的vma。所谓普通内存是相对于MMIO等内存来说的,MMIO的映射信息是需要复制的。这种情况下,子进程得到执行后,访问这
段内存时会导致缺页异常,由缺页异常程序来处理.

不复制是基于一个事实:子进程被创建后,多数情况下会执行新的程序,拥有自己的内存空间,父进程的内存它多半不会全部使用,
所以创建进程时尽量较少复制。

针对MAP_PRIVATE和PROT_WRITE方式映射的内存,首先,仅复制页表项信息是不够的,因为复制了页表项,子进程和父进程随后
可以访问相同的物理内存,这违反了MAP_PRIVATE的要求。其次,如果不复制,由缺页异常来处理,缺页异常会申请新的一页,使用新页完成映射。这看似没有问题,但实际上退化成了映射不完整的场景
中,丢失了内存中的数据。

copy_page_range函数针对这种情况的策略就是将页表项的权限降级为只读,复制页表项完成映射。子进程和父进程写这段内存时,就会导致缺页异常,由handle_pte_fault的第4步处理.

子进程创建后,子进程和父进程写COW的内存都会导致异常,如果父进程先写,就会触发复制,即使子进程不需要写。COW在这种情况下失效,所以子进程先执行对COW更加有利,如果子进程直接执行
新的程序,就不需要复制了。内核定义了一个变量sysctl_sched_child_runs_first,它可以控制子进程是否抢占父进程, 可以通过写/proc/sys/kernel/sched_child_runs_first文件设置它的值。

另外,如果子进程执行新的程序,或者取消了它与COW有关的映射,父进程再次写COW的内存导致缺页异常后还需要复制吗?这取决于拥有COW内存的进程的数量,如果仅剩一个进程映射这段内存,只需要修改页表项将内存标记为可写即可;如果还有其他进程需要访问这段内存,父进程也需要复制内存,这就是do_wp_page处理的第3种情况。当然了,如果父进程放弃了映射,子进程写内存可能同样不需
要复制内存,二者在程序上并没有地位上的差别.