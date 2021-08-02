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
ipv4数据包的主要接收方法是[ip_rcv()](https://elixir.bootlin.com/linux/v5.10.54/source/net/ipv4/ip_input.c#L530), 它是所有ipv4数据包(包括组播和广播)的处理程序. 它通过ip_rcv_core()完成完整性检查, 但实际工作由ip_rcv_finish()完成, 在这俩方法中间是NF_HOOK(netfilter钩子) NF_INET_PRE_ROUTING. 数据包在网络栈传输中, netfilter允许在6个挂接点注册回调函数, 添加netfilter钩子旨在支持在kernel运行阶段加载netfilter内核模块. NF_HOOK_COND是NF_HOOK宏的变种, 支持接收一个bool参数, 在其true时才执行该钩子. netfilter钩子也可丢弃数据包.

> netfilter钩子共计6中, 见[enum nf_inet_hooks](https://elixir.bootlin.com/linux/v5.10.55/source/include/uapi/linux/netfilter.h#L42).

在ip_rcv_finish()后, 会在路由子系统中进行查找(入口是`ip_rcv_finishi_core()`), 查找结果决定了要将数据包交给本机还是进行转发. 如果数据包的目的地是当前主机, 会依次调用ip_local_deliver()和ip_local_deliver_finish(). 如果数据包要转发, 将由方法ip_forward()进行处理.

> 经过路由子系统查找后, 组播流量由ip_mr_input()处理.

[ip_rcv_core()](https://elixir.bootlin.com/linux/v5.10.54/source/net/ipv4/ip_input.c#L435)具体逻辑:
1. `if (iph->ihl < 5 || iph->version != 4)`

    1. 检查报头长度, 其最短为20B, 因此min(ihl)=20B/4=5
    1. version必须是4

    不满足更新统计信息IPSTATS_MIB_INHDRERRORS后丢包.

1. ip_fast_csum()检查checksum

    它成功时返回0

    根据RFC 1122的3.2.1.2节的规定, 对于收到的每个数据包, 主机都必须检查ipv4报头中的校验和, 不正确则丢弃

1. `NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, ..., ip_rcv_finish)`

    执行NF_HOOK(), 返回:
    - NF_DR0P : 数据包丢弃, 数据包处理流程结束
    - NF_STOLEN : 数据包由netfilter子系统接管, 数据包处理流程结束
    - NF_ACCEPT : 数据继续处理
    - 其他值(统称verdicts) : NF_QUEUE, NFREPEAT, NF_STOP

    如果没有在NF_INET_PRE_ROUTING注册netfilter回调函数, 它会立即执行ip_rcv_finish()

[ip_rcv_finish()](https://elixir.bootlin.com/linux/v5.10.55/source/net/ipv4/ip_input.c#L414)具体逻辑:
1. [ip_rcv_finish_core](https://elixir.bootlin.com/linux/v5.10.55/source/net/ipv4/ip_input.c#L314):
    1. skb_dist()用于检查是否有与skb相关联的dst对象

        dst是一个[dst_entry](https://elixir.bootlin.com/linux/v5.10.55/source/include/net/dst.h#L24)实例, 表示路由选择子系统的查找结果. 查找是根据路由选择表和数据包报头进行的. 在路由选择子系统中查找时, 也可设置dst的input/output回调函数. 如果需要转发, 查找时可将input设为ip_forward(); 如果数据包目的地是当前机器, 查找时可将input设为ip_local_deliver(); 对于组播包, 可将input设为ip_mr_input.

        dst对象的内容决定了数据包的后续处理流程. 转发数据包时, 将根据dist来决定调用dst_input()时会调用的哪个input回调函数, 或应将数据包从哪个接口发送出去.

        如果没有与skb关联的dst, 将由ip_route_input_noref()在路由选择子系统中执行查找. 如果查找失败则丢包.
    1. `if (iph->ihl > 5 && ip_rcv_options(skb, dev))`

        接下来, 检查ipv4报头的选项, 如果有则由ip_rcv_options()处理
    1. `rt = skb_rtable(skb); ...`

        针对组播和广播, 分别更新PSTATS_MIB_INMCAST, IPSTATS_MIB_INBCAST
1. `dst_iput(skb)`

    通过调用`skb_dst(skb)->input(skb)`来调用input回调函数

## 接收ipv4组播数据包

ip_rcv()也是组播数据包处理程序. 它执行ip_rcv_finish()->ip_route_input_noref()->[ip_route_input_rcu()](https://elixir.bootlin.com/linux/v5.10.55/source/net/ipv4/route.c#L2332). ip_route_input_rcu()首先调用ip_check_mc_rcu()来检查当前机器是否属于目标组播地址指定的组播组, 如果是这样的组播组或当前机器是组播路由器(设置了CONFIG_IP_MROUTE)就调用ip_route_input_mc().

    ip_check_mc_rcu()返回值是our, 当our=1(当前机器属于目标组播地址指定的组播组)时, [ip_route_input_mc()](https://elixir.bootlin.com/linux/v5.10.55/source/net/ipv4/route.c#L1735)在[rt_dst_alloc()](https://elixir.bootlin.com/linux/v5.10.55/source/net/ipv4/route.c#L1639)中会将dst的回调函数设为ip_local_deliver() ;如果当前主机是组播路由器, 且设置了IN_DEV_MFORWARD(in_dev), 则将dst的input回调函数设为ip_mr_input.

    > IN_DEV_MFORWARD宏会检查procfs组播转发条目`/proc/sys/net/ipv4/conf/all/mc_forwarding`(只读). [pimd](https://github.com/troglobit/pimd)是一个PIM-SM v2组播路由选择守护程序, 它运行时会将它设为1, 停止时设为0.

在[ip_mr_input()](https://elixir.bootlin.com/linux/v5.10.55/source/net/ipv4/ipmr.c#L2071)中发现, 组播层存储了一种叫组播转发缓存(Multicast Forwarding Cache, MFC)的数据结构, 如果在MFC中找到了有效的条目, 将调用[ip_mr_forward()](https://elixir.bootlin.com/linux/v5.10.55/source/net/ipv4/ipmr.c#L1925). ip_mr_forward()执行一些检查, 并最终调用[ipmr_queue_xmit()](https://elixir.bootlin.com/linux/v5.10.55/source/net/ipv4/ipmr.c#L1813). 在ipmr_queue_xmit()中, 调用ip_decrease_ttl()将ttl减1, 并更新checksum. 接下来调用NFHOOK NF_INET_FORWARD来调用ipmr_forward_finish().

[ipmr_forward_finish()](https://elixir.bootlin.com/linux/v5.10.55/source/net/ipv4/ipmr.c#L1775), 它先更新统计信息, 在存在ipv4报头选项时调用ip_forward_options(), 在调用dst_output().