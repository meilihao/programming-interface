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
- 便于实现内存保护机制

	可对不同进程的页表条目(共享ram时)进行标记,以表示相关页面内容是可读、可写、可执行亦或是这些保护措施的组合

- 程序员和编译器, 链接器之类的工具不用关心程序的内存布局

## 概念
内存的通道数，实际上是一种内存的带宽加速技术, 使得带宽倍增. 通常pc是双通道, 服务器可能是3/4/8通道.
![](/misc/img/os/FNdSBT.png)

双通道，就是在cpu里设计两组内存控制器，这两个内存控制器可相互独立工作，每个控制器控制一个内存通道.


内存Interleave: 把内存打散了平均分配在多根DIMM上，进行交错，从根本上让多通道利用起来，也叫做Channel Interleaving（和Rank interleaving不同）, 需要设置biso开启.

> 相同的, ssd,pcie也有类似的通道概念.

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

	伙伴系统之上的内存管理, 基于对象

	slub针对多处理器、NUMA系统进行了优化.

	判断系统是否在用slub的方法: 看是否存在/sys/kernel/slab目录，存在即使用slub，没有就是slab.

	> 为了保证内核其它模块能够无缝迁移到SLUB分配器，SLUB还保留了原有SLAB分配器所有的接口API函数

	kmalloc使用的就是slub提供的对象.

	要从slub申请内存需要先使用[kmem_cache_create](https://elixir.bootlin.com/linux/v5.11/source/mm/slab_common.c#L407)创建一个slub对象. 再通过[kmem_cache_alloc](https://elixir.bootlin.com/linux/v5.11/source/mm/slab.c#L3484)和[kmem_cache_free](https://elixir.bootlin.com/linux/v5.11/source/mm/slab.c#L3686)来申请和释放内存.

	vmalloc: 把物理地址不连续的内存页拼凑成逻辑地址连续的内存区间.

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
TLB是特别的cpu缓存, 通过缓存进程最近使用的分页映射来加速地址转换(虚拟地址 -> 物理地址).

进程使用的内存越多, 分页表就越大. 同时进程在分页表上查询一个分页映射的开销也会很大. 因此tlb可很好地解决这个问题.

当上下文切换时, 为了将进程调度出来, 内核必须经常刷新tlb条目.

> tlb可通过`x86info -a`查看.
> linux通过hugepage来支持大号的分页. x86_64支持分页大小: 4K, 2M, 4M, 1G. 进程使用mmap 系统调用或shmat和shmget系统调用来请求大页.

虚拟地址映射到物理地址的工作主要是TLB（Translation Lookaside Buffers）与MMU一起来完成的. 以4KB的页大小为例，虚拟地址寻址时，首先在TLB中查找，如果没有找到，则需要通过MMU加载的页表基地址进行多次寻表来找到对应的物理地址. 如果找不到，则产生缺页，这时会有相应的handler进行处理，来填充页表和更新TLB.

总的来说，通过页表查询而导致缺页带来的CPU开销是非常大的，TLB的出现能很好地解决性能问题. 但是经常性的缺页是不可避免的，为此可以采取大页的方式.

通过使用Hugepage分配大页可以提高性能. 因为页大小的增加，可以减少缺页异常. 例如，2MB大小的内容（假设是2MB对齐），如果是4KB大小的页粒度，则会产生512次缺页异常，但是使用2MB大小的页，只会产生一次缺页异常. 页粒度的改变，使得TLB同样的空间可以保存更多虚存空间到物理空间的映射. 尽可能地利用TLB，少用MMU，以减少寻址和缺页处理带来的开销，从而提高应用程序的整体性能.

大页还有一个优势是这些预先分配的内存基本上不会被换出. 当然，大页也有缺点，比如它需要额外配置，需要应用程序事先预估使用多少内存，大页需要在进程启动之前实现启用和分配好内存. 目前，在大部分场景下，系统配置的主内存越来越多，这个限制不会成为太大的障碍.

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