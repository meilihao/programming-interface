# 内核协议栈

![内核协议栈](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nj6QFbU1dhIQWqgwbG7hdO6LiaDTLXrYBBrLKicTMBU9Gc9ibNo3mXMEdwIia1XaYFuQFP83om89s9zg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

随着互联网流量越来愈大, 网卡性能越来越强, Linux内核协议栈在10Mbps/100Mbps网卡的慢速时代是没有任何问题的, 那个时候应用程序大部分时间在等网卡送上来数据. 现在到了1000Mbps/10Gbps/40Gbps网卡的时代, 数据被很快地收入, 协议栈复杂处理逻辑, 效率捉襟见肘, 把大量报文堵在内核里.

问题有:
- 各类链表在多CPU环境下的同步开销
- 不可睡眠的软中断路径过长
- sk_buff的分配和释放
- 内存拷贝的开销
- 上下文切换造成的cache miss
- …

于是, 内核协议栈各种优化措施应着需求而来：
- 网卡RSS, 多队列
- 中断线程化
- 分割锁粒度
- Busypoll

但却都是见招拆招, 治标不治本. 问题的根源不是这些机制需要优化, 而是这些机制需要推倒重构. 重构的思路很显然有两个：
- upload方法：别让应用程序等内核了, 让应用程序自己去网卡直接拉数据
- offload方法：别让内核处理网络逻辑了, 让网卡自己处理

总之, 绕过内核就对了, 内核协议栈背负太多历史包袱.

DPDK让用户态程序直接处理网络流, bypass掉内核, 使用独立的CPU专门干这个事.
XDP让灌入网卡的eBPF程序直接处理网络流, bypass掉内核, 使用网卡NPU专门干这个事.

如此一来, 内核协议栈就不再参与数据平面的事了, 留下来专门处理诸如路由协议, 远程登录等控制平面和管理平面的数据流.

改善iptables/netfilter的规模瓶颈, 提高Linux内核协议栈IO性能, 内核需要提供新解决方案, 那就是eBPF/XDP框架.

# 网络编程
ref:
- [如何用Go实现一个异步网络库？](https://zhuanlan.zhihu.com/p/544038899)
- [百万级WebSockets和Go语言](https://colobu.com/2017/12/13/A-Million-WebSockets-and-Go/)
- [gaio小记](https://zhuanlan.zhihu.com/p/102890337)

服务端网络编程主要解决两个问题:
1. 服务端如何管理连接，特别是海量连接、高并发连接（经典的c10k/c100k问题）
1. 服务端如何处理请求（高并发时正常响应）

针对这两个问题，有三种解决方案，分别对应三种模型：
- 传统IO阻塞模型

	每条连接都是由单独的线/进程管理，业务逻辑（crud）跟数据处理（网络连接上的read和write）都在该线/进程完成。缺点很明显，并发大时，需要创建大量的线/进程，系统资源开销大；连接建立后，如果当前线/进程暂时还没数据可读，会阻塞在Read调用上，浪费系统资源.
- Reactor模型

	Reactor模型就是传统IO阻塞模型的改进，Reactor会起单独的线/进程去监听和分发事件，分发给其他EventHandlers处理数据读写和业务逻辑。这样，与传统IO阻塞模型不同的是，Reactor的连接都先到一个EventDispatcher上，一个核心的事件分发器，同时Reactor会使用IO多路复用在事件分发器上非阻塞地处理多个连接.

	这个EventDispatcher跟后面的EventHandlers可以都在一个线/进程，也可以分开. 整体来看，Reactor就是一种事件分发机制，所以Reactor也被称为事件驱动模型。简而言之，Reactor=IO多路复用（I/O multiplexing）+非阻塞IO（non-blocking I/O）.
- Proactor模型

	Proactor模型跟Reactor模型的本质区别是异步I/O和同步I/O的区别，即底层I/O实现.

	Reactor模型依赖的同步I/O需要不断检查事件发生，然后拷贝数据处理，而Proactor模型使用的异步I/O只需等待系统通知，直接处理内核拷贝过来的数据. 因此明显Proactor更优. 没有流行的原因是在Linux下的AIO API--io_uring还没有像同步I/O那样能够覆盖和支持很多场景，即还没成熟到被广泛使用.

## Reactor模型
根据Reactor的数量和业务线程的工作安排有3种典型实现：
1. 单Reactor多线程
1. 单Reactor多线程带线程池

	单Reator1和2的区别是2带了个线程池，一定程度上解放Event Handler线程，让Handler专注数据读写处理，特别是在遇到一些笨重、高耗时的业务逻辑时
1. 主从Reactor多线程（带线程池）

	多Reactor就是主从多Reactor，它的特点是多个Reactor在多个单独的线/进程中运行，MainReactor负责处理建立连接事件，交给它的Acceptor处理，处理完了，它再分配连接给SubReactor；SubReactor则处理这个连接后续的读写事件，SubReactor自己调用EventHandlers做事情。

	这种实现看起来职责就很明确，可以方便通过增加SubReactor数量来充分利用CPU资源，也是当前主流的服务端网络编程模型.

Go原生网络模型就是个单Reactor多协程模型. Go处理连接的方式是一个连接给分配一个协程处理，即goroutine-per-conn模式. 因此随着连接数上升，Go的协程数也随之线性上升，内存开销增大，GC时间占比增加。当连接数到达一定数值时，Go的强制GC还会把进程搞挂，服务不可用. 通常会用[epoll的方式代替goroutine-per-conn模式](https://link.zhihu.com/?target=https%3A//www.freecodecamp.org/news/million-websockets-and-go-cc58418460bb/), 百万连接场景下用少量的goroutine去代替一百万的goroutine.

相比Reactor网络库而言，Go原生网络库可以看作是以空间（内存、runtime）来换取时间（高吞吐量和低延时）。当空间紧张时，也就是连接数上来后，巨大的内存开销和相应的GC会导致服务不可用，而这种海量连接场景才是Reactor网络库的优势所在.