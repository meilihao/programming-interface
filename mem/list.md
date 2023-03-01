# list
## malloc
- [18张图解密新时代内存分配器TCMalloc](http://tigerb.cn/2021/01/31/go-base/tcmalloc/)
- [分配器，比如jemalloc, tcmalloc, ptmalloc，有一个论文 做了比较](https://adms-conf.org/2019-camera-ready/durner_adms19.pdf)

	结论: jemalloc最优

## next
- [观察进程的内存占用情况](https://www.cnblogs.com/bravery/archive/2012/06/27/2560611.html)
- [Linux 内核将弃用并删除 SLOB 内存分配器](https://www.oschina.net/news/217107/linux-wants-to-drop-slob)

	SLOB （simple list of blocks）分配器是 Linux 内核中三个可用的内存分配器之一. 另外两个是 SLAB (slab allocator) 和 SLUB（the unqueued slab allocator）.

	Vlastimil 在邮件中提到的是放弃 SLOB 和 SLAB 两个内存分配器，只留下 SLUB. 到目前为止，其他上游开发人员都赞成弃用和移除 SLOB，而移除 SLAB 可能需要更多时间.