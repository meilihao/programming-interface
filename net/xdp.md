# xdp
ref:
- [XDP技术——Linux网络处理的高速公路](https://dockone.io/article/2434730)
- [Facebook 流量路由最佳实践：从公网入口到内网业务的全路径 XDP/BPF 基础设施](http://arthurchiao.art/blog/facebook-from-xdp-to-socket-zh/)
- [Cilium eBPF实现机制源码分析](https://www.cnxct.com/how-does-cilium-use-ebpf-with-go-and-c/)
- [Express UDP 是基于 XDP Socket 实现的 UDP 通信软件库](https://gitee.com/anolis/ExpressUDP)
- [Get started with XDP](https://developers.redhat.com/blog/2021/04/01/get-started-with-xdp)
- [Using the eXpress Data Path (XDP) in Red Hat Enterprise Linux 8](https://www.redhat.com/en/blog/using-express-data-path-xdp-red-hat-enterprise-linux-8)
- [XDP driver support status](https://github.com/xdp-project/xdp-project/blob/master/areas/drivers/README.org)

XDP需要驱动程序支持, 或通过[`git grep -l XDP_SETUP_PROG drivers/`](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)

## AF_XDP Socket
前提: `CONFIG_XDP_SOCKETS=y`, 推荐使用4.19及以上的kernel.

基于Kernel提供的BPF能力, XDP可以提供高性能的数据面处理能力.

AF_XDP, 和AF_INET一样, 也是address family的一种, 用于规定socket通讯的类型, 相当于socket底层通讯方式的不同实现（多态）, 即AF_XDP就是一套基于XDP的通讯的实现.

XDP程序在Kernel提供的网卡驱动中直接取得网卡收到的数据帧, 然后直接送到用户态应用程序. 应用程序利用AF_XDP类型的socket接收数据. 用虚拟化领域的完全虚拟化和半虚拟化概念类比, 如果DPDK是"完全Kernel bypass", 那么AF_XDP就是"半Kernel bypass"(bpf部分还是在kernel中).

XDP程序会把数据帧送到一个在用户态可以读写的队列Memory Buffer中, 叫做UMEM. 用户态应用在UMEM中完成数据帧的读取和写入. 当然了, 整个过程都是**零拷贝（Zero Copy）**的.

![](/misc/img/net/xdp/AUoFzQ.png)

AF_XDPSocket的创建过程可以使用在网络编程中常见的socket()系统调用, 就是参数需要特别配置一下. 在创建之后, 每个socket都各自分配了一个RX ring和TX ring. 这两个ring保存的都是descriptor, 里面有指向UMEM中真正保存帧的地址.

UMEM也有两个队列, 一个叫FILL ring, 一个叫COMPLETION ring. 其实就和传统网络IO过程中给DMA填写的接收和发送环形队列很类似. 在RX过程中, 用户应用程序在FILL ring中填入接收数据包的地址, XDP程序会将接收到的数据包放入该地址中, 并在socket的RX ring中填入对应的descriptor.

但COMPLETION ring中保存的并非用户应用程序"将要"发送的帧的地址, 而是已经完成发送的帧的地址. 这些帧可以用来被再次发送或者接收. "将要"发送的帧的地址是在socket的TX ring中, 同样由用户应用填入.

RX/TX ring和FILL/COMPLETION ring之间是多对一（n:1）的关系. 也就是说可以有多个socket和它们的RX/TX ring共享一个UMEN和它的FILL/COMPLETION ring.

## Smart NIC
和DPDK把数据包拉到用户态的方向相反, 与其把数据上拉被CPU处理, 把处理数据包的代码向下注入网卡是殊途同归的, 为了让网卡可以执行注下去的代码, 给网卡内置几个CPU即可, 这就是智能网卡的思路.

> 某种意义上DPDK其实就是把x86 CPU+Ringbuffer和Intel网卡一起, 当成了一块"智能网卡".