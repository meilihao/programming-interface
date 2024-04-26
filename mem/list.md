# list
## layout
- [riscv/vm-layout.rst](https://elixir.bootlin.com/linux/v6.4-rc7/source/Documentation/riscv/vm-layout.rst)
- [RISCV MMU 概述](https://tinylab.org/riscv-mmu/)
- [**Linux系统启动之后，物理内存的布局是怎么样的？**](https://www.zhihu.com/question/274054284)

	内存模型: 平坦模型 -> Discontiguous Memory -> sparse memory

- [x86_64/mm.txt](https://elixir.bootlin.com/linux/v4.20.17/source/Documentation/x86/x86_64/mm.txt)

[仅找到risv64支持4k内存页(Sv39/Sv48/Sv57的page offset都是4k), 见`Page-Based <39/48/57>-bit Virtual-Memory System`](https://five-embeddev.com/riscv-isa-manual/latest/supervisor.html), 没找到其他大小的支持文档.

## malloc
- [18张图解密新时代内存分配器TCMalloc](http://tigerb.cn/2021/01/31/go-base/tcmalloc/)
- [分配器，比如jemalloc, tcmalloc, ptmalloc，有一个论文 做了比较](https://adms-conf.org/2019-camera-ready/durner_adms19.pdf)

	结论: jemalloc最优
- ION
	ION是Android为了解决碎片化而引入的内存管理器,可以支持不同的内存分配方式. 在内核和用户空间中都可以使用它管理内存.

## next
- [观察进程的内存占用情况](https://www.cnblogs.com/bravery/archive/2012/06/27/2560611.html)
- [Linux 内核将弃用并删除 SLOB 内存分配器](https://www.oschina.net/news/217107/linux-wants-to-drop-slob)

	SLOB （simple list of blocks）分配器是 Linux 内核中三个可用的内存分配器之一. 另外两个是 SLAB (slab allocator) 和 SLUB（the unqueued slab allocator）.

	Vlastimil 在邮件中提到的是放弃 SLOB 和 SLAB 两个内存分配器，只留下 SLUB. 到目前为止，其他上游开发人员都赞成弃用和移除 SLOB，而移除 SLAB 可能需要更多时间.

	> [SLOB在6.4移除](https://www.solidot.org/story?sid=75338)
	> [开始废除 SLAB 分配器](https://www.oschina.net/news/248695/linux-6-5-rc1-released), [CONFIG_SLAB_DEPRECATED](https://www.phoronix.com/news/SLAB-Officially-Deprecated)
- page大小

	[在 OS X 和早期的iOS里，页大小均为4K, macos也一直沿用4k; 但之后基于A7和A8的iOS里，采用虚拟内存每页16K，物理内存每页4K；基于A9或更新CPU的iOS里，页大小均为16K.](https://www.jianshu.com/p/961d819096a7)

	在x86通常为 4k（4096）, 在 ARM64 中通常为 16K.