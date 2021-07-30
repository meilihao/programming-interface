# ipv4
ipv4(RFC 791)是当今基于标准的internet中的核心协议之一, 由于ip地址耗尽的问题, 正在向ipv6过渡, 但需要熟悉它.

ipv4报头用[iphdr](https://elixir.bootlin.com/linux/v5.10.54/source/include/uapi/linux/ip.h#L86)表示, 它的内容决定了ipv4栈处理数据包的方式, 比如版本不是4(ipv4)或checksum不正确则数据包会被丢弃.

iphdr成员:
- ihl : internet报头长度. ip报头的长度是以4B为单位, 因此报头的实际长度是`ihl * 4`.
- version : 固定值4
- tos : tos表示服务类型. RFc 2474为ipv4/6定义了区分服务(DS)字段, 它是tos的前6位, 也叫区分服务码点(Differentiated Services Code Point, DSCP). 2001年的RFC 3168定义了显式拥塞通知(Explicit Congestion Notification, ECN), 它是tos的第7~8位.
- tot_len : 包括报头在内的数据包总长度, 单位是B. 因为它是uint16, 最长表示64kb. RFC 791规定, 数据包最短不得少于576B.
- id : ipv4报头标识. 对于分段而言, 它很重要. 对skb分段时, 所有分段的id必须相同, 以便接收方重组数据包.
- frag_off: 分段偏移量, 16bit. 后13bit是具体分段的偏移量, 以8字节为单位. 前3bit是分段标志(具体可见`include/net/ip.h`的IP_MF, IP_DF, IP_CE):

    - 001 : 表示后面还有其他分段(More Fragments, MF). 除最后一个分段外, 其他分段都必须设置该标志.
    - 010 : 不分段(Don't Fragment, DF)
    - 100 : 表示拥塞(Congestion, CE)
- ttl : 存活时间. 每进过一个转发节点都会减1, 当ttl=0时, 丢弃数据包并发送一条icmp 超时消息, 避免数据包因某些原因导致无限转发.
- protocol : 数据包的第4层协议, 支持列表在`include/linux/in.h`, 常见的IPPROTO_TCP是tcp, IPPROTO_UDP是udp.
- check : checksum, 16bit. 该校验和是仅根据ipv4报头计算的
- saddr : 源ipv4地址
- daddr : 目的地ipv4地址

## ipv4的初始化
ipv4数据包的以太类型是0x0800([ETH_P_IP](https://elixir.bootlin.com/linux/v5.10.54/source/include/uapi/linux/if_ether.h#L52), 它在14B的以太网报头的前2B中). 每种协议都必须指定一个协议处理程序并初始化, 以便让网络栈能处理该协议的数据包.

在引导阶段, `inet_init()`的[`dev_add_pack(&ip_packet_type)`](https://elixir.bootlin.com/linux/v5.10.54/source/net/ipv4/af_inet.c#L2047)将ip_rcv()指定为ipv4数据包的协议处理程序.

ipv4协议的主要功能体现在接收路径和传输路径两部分.

## 接收ipv4数据包
ipv4数据包的主要接收方法是[ip_rcv()](https://elixir.bootlin.com/linux/v5.10.54/source/net/ipv4/ip_input.c#L530), 它是所有ipv4数据包(包括组播和广播)的处理程序. 它通过ip_rcv_core()完成完整性检查, 实际工作由ip_rcv_finish()完成, 在这俩方法中间是NF_HOOK(netfilter钩子) NF_INET_PRE_ROUTING. 数据包在网络栈传输中, netfilter允许在5个挂接点注册回调函数, 添加netfilter钩子旨在支持在kernel运行阶段加载netfilter内核模块. NF_HOOK_COND是NF_HOOK宏的变种, 支持接收一个bool参数, 在其true时才执行该钩子. netfilter钩子也可丢弃数据包.

在ip_rcv_finish()后, 会在路由子系统中进行查找, 查找结果决定了要将数据包交给本机还是进行转发.