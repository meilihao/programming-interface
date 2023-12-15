# xdp
ref:
- [Cilium：BPF 和 XDP 参考指南（2021）](https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/)
- [eBPF XDP 子系统](https://houmin.cc/posts/b7703758/)
- [XDP技术——Linux网络处理的高速公路](https://dockone.io/article/2434730)
- [Facebook 流量路由最佳实践：从公网入口到内网业务的全路径 XDP/BPF 基础设施](http://arthurchiao.art/blog/facebook-from-xdp-to-socket-zh/)
- [Cilium eBPF实现机制源码分析](https://www.cnxct.com/how-does-cilium-use-ebpf-with-go-and-c/)
- [Express UDP 是基于 XDP Socket 实现的 UDP 通信软件库](https://gitee.com/anolis/ExpressUDP)
- [Get started with XDP](https://developers.redhat.com/blog/2021/04/01/get-started-with-xdp)
- [Using the eXpress Data Path (XDP) in Red Hat Enterprise Linux 8](https://www.redhat.com/en/blog/using-express-data-path-xdp-red-hat-enterprise-linux-8)
- [XDP driver support status](https://github.com/xdp-project/xdp-project/blob/master/areas/drivers/README.org)
- [XDP](https://tonydeng.github.io/sdn-handbook/linux/XDP/)

相对于DPDK, XDP具有以下优点:
- 无需第三方代码库和许可
- 同时支持轮询式和中断式网络
- 无需分配大页
- 无需专用的CPU
- 无需定义新的安全网络模型

XDP的性能提升是有代价的, 它牺牲了通用型和公平性. 缺点即:

- XDP不提供缓存队列（qdisc）, TX设备太慢时直接丢包, 因而不要在RX比TX快的设备上使用XDP
- XDP程序是专用的, 不具备网络协议栈的通用性

## 场景
- **DDoS 防御、防火墙**

	XDP BPF 的一个基本特性就是用 XDP_DROP 命令驱动将包丢弃，由于这个丢弃的位置非常早，因此这种方式可以实现高效的网络策略，平均到每个包的开销非常小。这对于那些需要处理任何形式的 DDoS 攻击的场景来说是非常理想的，而且由于其通用性，使得它能够在 BPF 内实现任何形式的防火墙策略，开销几乎为零， 例如，

	- 作为 standalone 设备（例如通过 XDP_TX 清洗流量）
	- 广泛部署在节点上，保护节点的安全（通过 XDP_PASS 或 cpumap XDP_REDIRECT 允许「好流量」经 过）
	- Offloaded XDP 更进一步，将本来就已经很小的 per-packet cost 全部下放到网卡以线速（line-rate）进行处理

- **包转发和负载均衡**：通过 XDP_TX 或 XDP_REDIRECT 动作实现

	- XDP 层运行的 BPF 程序能够任意修改（mangle）数据包，即使是 BPF 辅助函数都能增加或减少包的 headroom，这样就可以在将包再次发送出去之前，对包进行任何的封装/解封装。
	- 利用 XDP_TX 能够实现 hairpinned（发卡）模式的负载均衡器，这种均衡器能够在接收到包的网卡再次将包发送出去，而 XDP_REDIRECT 动作能够将包转发到另一个 网卡然后发送出去。
	- XDP_REDIRECT 返回码还可以和 BPF cpumap 一起使用，对那些目标是本机协议栈、 将由 non-XDP 的远端（remote）CPU 处理的包进行负载均衡。
- **栈前过滤与处理**

	除了策略执行，XDP 还可以用于加固内核的网络栈，这是通过 XDP_DROP 实现的。 这意味着，XDP 能够在可能的最早位置丢弃那些与本节点不相关的包，这个过程发生在内核网络栈看到这些包之前。例如假如我们已经知道某台节点只接受 TCP 流量，那任 何 UDP、SCTP 或其他四层流量都可以在发现后立即丢弃。

	这种方式的好处是包不需要再经过各种实体（例如 GRO 引擎、内核的 flow dissector 以及其他的模块），就可以判断出是否应该丢弃，因此减少了内核的受攻击面。正是由于 XDP 的早期处理阶段，这有效地对内核网络栈「假装」这些包根本 就没被网络设备看到。

	另外，如果内核接收路径上某个潜在 bug 导致 ping of death 之类的场景，那我们能够利用 XDP 立即丢弃这些包，而不用重启内核或任何服务。而且由于能够原子地替换程序，这种方式甚至都不会导致宿主机的任何流量中断。

	栈前处理的另一个场景是：在内核分配 skb 之前，XDP BPF 程序可以对包进行任意 修改，而且对内核「假装」这个包从网络设备收上来之后就是这样的。对于某些自定义包修改（mangling）和封装协议的场景来说比较有用，在这些场景下，包在进入 GRO 聚合之前会被修改和解封装，否则 GRO 将无法识别自定义的协议，进而无法执行任何形 式的聚合。

	XDP 还能够在包的前面 push 元数据（非包内容的数据）。这些元数据对常规的内核栈是不可见的（invisible），但能被 GRO 聚合（匹配元数据），稍后可以和 tc ingress BPF 程序一起处理，tc BPF 中携带了 skb 的某些上下文，例如，设置了某些 skb 字段。

- **流抽样和监控**

	XDP 还可以用于包监控、抽样或其他的一些网络分析，例如作为流量路径中间节点的一部分；或运行在终端节点上，和前面提到的场景相结合。对于复杂的包分析，XDP 提供了设施来高效地将网络包（截断的或者是完整的 payload）或自定义元数据 push 到 perf 提供的一个快速、无锁、per-CPU 内存映射缓冲区，或者是一 个用户空间应用。

	这还可以用于流分析和监控，对每个流的初始数据进行分析，一旦确定是正常流量，这个流随后的流量就会跳过这个监控。感谢 BPF 带来的灵活性，这使得我们可以实现任何形式 的自定义监控或采用

## XDP 工作模式
XDP需要驱动程序支持, 或通过[`git grep -l XDP_SETUP_PROG drivers/`](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)查看哪些驱动支持.

并非所有驱动支持xdp, 因此也可切换到[generic模式](https://www.mail-archive.com/netdev@vger.kernel.org/msg165397.html)即在网络栈使用该eBPF, 此时性能有衰减.

XDP 有三种工作模式, 分别是：
- xdpdrv：默认模式, 当讨论 XDP 时通常隐含的都是指这种模式

	在这种模式中，BPF 程序直接在驱动的接收路径上运行，理论上这是软件层最早可以处理包的位置。这是常规的 XDP 模式，需要**驱动实现对 XDP 的支持**，目前 Linux 内核中主流的 10G/40G 网卡都已经支持.
- xdpoffload：在这种模式中, XDP BPF 程序直接 offload 到网卡, 而不是在主机的 CPU 上执行

	本来就已经很低的 per-packet 开销完全从主机下放到网卡，能够比运行在 native XDP 模式取得更高的性能. 这种 offload 通常由智能网卡（例如支持 Netronome’s nfp 驱动的网卡）实现, 这些网卡有多线程、多核流处理器（flow processors），一个位于内核中的 JIT 编译器（ in-kernel JIT compiler）将 BPF 翻译成网卡的原生指令.

	虽然在这种模式中某些 BPF map 类型 和 BPF 辅助函数是不能用的。BPF 校验器检测到这种情况时会直接报错，告诉用户哪些东西是不支持的。除了这些不支持的 BPF 特性之外，其他方面与 native XDP 都是一样的.
- xdpgeneric：用于给那些还没有原生支持 XDP 的驱动进行试验性测试

	对于还没有实现 native 或 offloaded XDP 的驱动，内核提供了一个 generic XDP 选项，这种模式不需要任何驱动改动。

	generic XDP hook 位于内核协议栈的主接收路径上，接受的是 skb 格式的包，但由于 这些 hook 位于 ingress 路径的很后面，因此与 native XDP 相比性能有明显下降。因此，xdpgeneric 大部分情况下只能用于试验目的，很少用于生产环境.

ip link命令查看已经安装的XDP模式: generic/SKB (**xdpgeneric**), native/driver (**xdp**), hardware offload (**xdpoffload**).

> 虚拟机上的设备可能无法支持native模式, 比如阿里云的ecs.

> 使用`ethtool -S eth0`可查看xdp处理的报文统计

> 执行 `ip link set dev em1 xdp obj [...]`时, 内核会先尝试以 native XDP 模 式加载程序, 如果驱动不支持再自动回退到 generic XDP 模式. 当显示指定模式时, 如果不支持该模式就直接失败.

## Smart NIC
和DPDK把数据包拉到用户态的方向相反, 与其把数据上拉被CPU处理, 把处理数据包的代码向下注入网卡是殊途同归的, 为了让网卡可以执行注下去的代码, 给网卡内置几个CPU即可, 这就是智能网卡的思路.

> 某种意义上DPDK其实就是把x86 CPU+Ringbuffer和Intel网卡一起, 当成了一块"智能网卡".

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