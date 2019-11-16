# 内存
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

在linux里面，swappiness的值的大小对如何使用swap分区是有着很大的联系的.两个极端: swappiness=0的时候表示最大限度使用物理内存，然后才是 swap空间, 会增加文件系统开销 ;swappiness＝100的时候表示积极的使用swap分区，并且把内存上的数据及时的搬运到swap空间里面. 对于ubuntu的默认设置，这个值等于60，建议修改为10. 具体这样做：

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