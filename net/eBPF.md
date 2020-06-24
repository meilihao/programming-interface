# eBPF
参考:
- [eBPF技术实践：高性能ACL](https://www.tuicool.com/articles/NZJjUbi)

随着 eBPF 技术的快速发展，bpfilter 有望取代 iptables/nftables，成为下一代网络 ACL 的解决方案.

XDP（eXpress Data Path）是基于 eBPF 实现的高性能、可编程的数据平面技术.

![xdp架构](/misc/img/net/2M36Vbb.webp)

XDP 位于网卡驱动层, 当数据包经过 DMA 存放到 ring buffer 之后, 分配 skb 之前即可被 XDP 处理. 数据包经过 XDP 之后, 会有 4 种去向：
- XDP_DROP：丢包
	
	网上有比较数据，利用 XDP 技术的丢包速率要比 iptables 高 4 倍左右.
- XDP_PASS：上送协议栈
- XDP_TX：从当前网卡发送出去
- XDP_REDIRECT：从其他网卡发送出去

由于 XDP 位于整个 Linux 内核网络软件栈的底部, 能够非常早地识别并丢弃攻击报文, 具有很高的性能. 这为改善 iptables/nftables 协议栈丢包的性能瓶颈，提供了非常棒的解决方案.

BPF（Berkeley Packet Filter）是 Linux 内核提供的基于 BPF 字节码的动态注入技术（常应用于 tcpdump、raw socket 过滤等）. eBPF(extended Berkeley Packet Filter)是针对于 BPF 的扩展增强，丰富了 BPF 指令集，提供了 Map 的 KV 存储结构. 开发者可以利用 bpf() 系统调用，初始化 eBPF 的 Program 和 Map，利用 netlink 消息或者 setsockopt() 系统调用，将 eBPF 字节码注入到特定的内核处理流程中（如 XDP、socket filter 等）.

![eBPF注入位置](/misc/img/net/eeuMjee.webp)

ACL 控制平面负责创建 eBPF 的 Program、Map，注入 XDP 处理流程中: 其中 eBPF 的 Program 存放报文匹配、丢包等处理逻辑，eBPF 的 Map 存放 ACL 规则.
![ACL 控制平面架构](/misc/img/net/mmqQ7fA.webp)