# icmp
icmp主要用作发送有关网络层(L3)错误和控制消息的机制, 通过icmp消息可获取有关通信环境中问题的反馈, 以便进行错误处理和诊断功能.

RFC 792对icmpv4做了基本定义, 包括icpmv4的目标以及各种icpmv4消息的格式. RFC 1122/4443/1812分别定义了对多种icmp消息的要求, icmpv6协议以及对路由器的要求.

> ping, traceroute是最典型的icmp应用.

> ICMPv4和ICMPv6是kernel必须的, 因此不能构建成module.

## icmpv4
icmpv4消息分两类:
1. 错误消息
1. 查询消息

ping命令是在用户空间(by iputils)打开一个原始套接字并发送一条ICMP_ECHO消息, 响应是ICMP_REPLY消息.
traceroute是用于确定主机与给定目标ip间路径的工具(默认使用udp), 它基于ip报头中的ttl, 通过将ttl设为不同的值(从1开始, 收到ICMP_DEST_UNREACH消息就递增， 直至到达或超限制), 当ttl=0时转发设备会返回一条ICMP_TIME_EXCEED消息. 它利用返回的icmp 超时消息来创建数据包经过的路由列表, 直到到达目标ip并返回icmp echo reply消息.

icmpv4模块在[net/ipv4/icmp.c](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/icmp.c).

> 组播组成员关系在ipv4中由IGMP(internet组管理协议)完成的.

### icmpv4的初始化
icmpv4的初始化是在引导阶段的[`inet_init()`](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/af_inet.c#L1938)中的[`icmp_init()`](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/af_inet.c#L2023)里完成.

icmp_init()会调用[`icmp_sk_init`](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/icmp.c#L1318), 以创建用于发送icmp消息的内核icmp套接字, 并将一些icmp procfs变量初始化为默认值.

icmp_sk_init()中会为每个cpu创建一个icmpv4套接字, 并将其存放在一个array中, 要访问当前的套接字可使用icmp_sk(struct net *net)

与其他ipv4协议一样, [icmp_protocol](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/af_inet.c#L1751)(icmpv4)的[注册也在inet_init()中完成](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/af_inet.c#L1976):
- icmp_rcv : handler回调函数. 对于收到的数据包, 如果其ip报头中的协议字段是IPPROTO_ICMP(0x1)将由它处理.
- no_policy : 1, 表示无需执行IPsec策略检查
- netns_ok : 1, 表示支持网络命名空间

### icmpv4报头
icmpv4报头[icmphdr](https://elixir.bootlin.com/linux/v5.10.52/source/include/uapi/linux/icmp.h#L70)的组成:
1. type(8b)
1. code(8b)
1. checksum(16b)
1. variable part(可变部分, 内容取决于type和code, 32b)
1. payload

    包含原始数据包的ipv4报头和部分有效载荷.

    RFC 1812支出在保证icmpv4数据包不超过576B(该长度由RFC 791确定)的前提下, payload应尽可能多地包含原始数据报的内容.

icmpv4模块定义了一个[icmp_control](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/icmp.c#L188)对象数组[icmp_pointers](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/icmp.c#L1237), 它以icmpv4消息类型为索引, 这个数组中的icmp_control都是错误消息, 因为它们的error字段为1, 比如ICMP_DEST_UNREACH(目的不可达).

> icmp_control->error为0时表示查询消息, 比如ICMP_ECHO.

> 消息类型ICMP_DEST_UNREACH, ICMP_TIME_EXCEED, ICMP_PARAMETERPROB和ICMP_QUENCH由icmp_unreach()处理.

> ICMP_QUENCH已废弃.

[ping_rcv()](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/ping.c#L950)负责处理接收到的ping应答(ICMP_ECHOREPLY).

**在kernel 3.0前**, 要发送ping必须在用户空间创建一个原始socket, 有ICMP_ECHOREPLY到来时, 由发送ping的套接字进行处理, 逻辑在[ip_local_deliver_finish()](https://elixir.bootlin.com/linux/v2.6.39.4/source/net/ipv4/ip_input.c#L188).

ip_local_deliver_finish()收到ICMP_ECHOREPLY数据包后, 首先调用raw_local_deliver(), 通过一个原始套接字对其进行处理(用户空间的应用处理), 再调用ipprot->handler(skb), 这里就icmpv4数据包而言handler=icmp_rcv(), 最终被[`icmp_pointers[icmph->type].handler(skb)`即`icmp_pointers[ICMP_ECHOREPLY].handler=icmp_discard`](https://elixir.bootlin.com/linux/v2.6.39.4/source/net/ipv4/icmp.c#L1032)处理.

**从kernel 3.0开始**, kernel集成了icmp套接字(也叫ping套接字). 引入它后, ping的发送方可以不是原始套接字, 比如socket(PF_INET, SOCK_DGRAM, PROT_ICMP)创建的不是原始套接字， 但可以发送ping数据包, 此时响应不会被交给原始套接字, 因为没有监听它的原始套接字. 为了避免这种问题, icmpv4模块使用回调函数ping_rcv()处理ICMP_ECHOREPLY. ping模块位于ipv4层(net/ipv4/ping.c), 但net/ipv4/ping.c中的大多数代码都是双栈代码(适用于ipv4/6), 因此ping_rcv()也可处理ICMPV6_ECHO_REPLY消息.

icmp_discard()是一个noop, 用于未声明的消息类型和不需要做任何处理的消息, 比如ICMP_TIMESTAMPREPLY.

有两种情况会发送ICMP_TIME_EXCEEDED:
1. 在[`ip_forward()`](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/ip_forward.c#L86)中, ttl都会递减, 当ttl=0时表示应丢弃数据包, 因为可能存在环路, 此时会执行[`icmp_send(skb, ICMP_TIME_EXCEEDED, ICMP_EXC_TTL, 0)`](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/ip_forward.c#L171).

    > RFC 1700推荐将ipv4的ttl设为64.
1. 在[`ip_expire`](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/ip_fragment.c#L133)中, 如果分段超时会执行[`icmp_send(head, ICMP_TIME_EXCEEDED, ICMP_EXC_FRAGTIME, 0)`](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/ip_fragment.c#L189).

在[ip_optins_compile()或ip_options_rcv_srr()](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/ip_options)中, 未成功解析ipv4报头选项(变长字段, 最多40B)时会发送ICMP_PARAMETERPROB.

ICMP_REDIRECT消息由icm_redirect()处理, 且该消息仅供网关发送. icm_redirect()首先执行完整性检查, 再调用icmp_socket_deliver(). icmp_socket_deliver()会将数据包交给原始套接字并调用协议错误处理程序(如果存在的话).

ICMP_ECHO有[icmp_echo()](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/icmp.c#L992)处理. 如果设置了`ipv4.sysctl_icmp_echo_ignore_all`将不响应ping.

icmp_reply()和icmp_send()都会使用到[icmp_bxm(icmp生成的xmit消息)](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/icmp.c#L100):
- skb : 就icmp_reply()而言, skb是请求数据包; 就icmp_send()而言, skb是被发送的icmpv4数据包
- offset : skb_network_header(skb)和skb->data间的偏移量
- data_len : icmpv4数据包的有效载荷的长度
- icmph : icmpv4报头
- times[3] : 包含3个时间戳, 由icmp_timestamp()填充
- head_len : icmpv4报头的长度(对icmp_timestamp()而言, 还包含times[3]的长度)
- replyopts : 一个ip_options_data对象.

### 接收icmpv4
ip_local_deliver_finish()处理的是目的地为当前机器的数据包:
1. 它首先将InMsgs SNMP计数器(ICMP_MIB_INMSGS)加1, 在检查checksum

    如果不正确就将SNMP计数器InCsumErrors和InErrors(ICMP_MIB_CSUMERRORS和ICMP_MIB_INERRORS)加1， 再释放skb， 并返回[NET_RX_DROP](https://elixir.bootlin.com/linux/v5.10.52/source/include/linux/netdevice.h#L79)
1. 检查icmp报头, 以确定icmp消息的类型, 并将相应的procfs消息类型计数器(每种icmp消息类型都有一个procfs计数器)加1, 再检查type是否超过NR_ICMP_TYPES

    如果超过ICMP_MIB_INERRORS加1并释放skb
1. 如果数据包是RTCF_BROADCAST(广播)或RTCF_MULTICAST(组播)

    1. 是ICMP_ECHO或ICMP_TIMESTAMP)根据ipv4.sysctl_icmp_echo_ignore_broadcasts(`/proc/sys/net/ipv4/icmp_echo_ignore_broadcasts`)确定是否允许响应(by RFC 1122的3.2.2.6和3.2.2.8). 如果为1表示丢弃该包.
    1. 不是ICMP_ECHO, ICMP_TIMESTAMP, ICMP_ADDRESS, ICMP_ADDRESSREPLY中的一个时直接丢包
1. 根据消息类型从icmp_pointers获取对应的hanler进行处理

    - ICMP_ECHO由icmp_echo()处理; ICMP_TIMESTAMP由icmp_timestamp()处理
    - 除上述两种消息外的其他消息不需要响应

### 发送icmpv4消息
发送icmpv4消息由两种方法:
1. icmp_reply() : 用于发送ICMP_ECHO和ICMP_TIMESTAMP的响应
1. icmp_send() : 发送当前机器在特定条件下主动发送的icmpv4消息

    在ipv4网络栈的很多地方都用了它, 比如netfilter, ip_forward.c, ipip和ip_gre之类的隧道中等等

上面两种方法最终都调用[icmp_push_reply()](https://elixir.bootlin.com/linux/v5.10.51/source/net/ipv4/icmp.c#L366)来实际发送数据包.

ip报头协议字段中的协议不存在时需要发送ICMP_DEST_UNREACH/ICMP_PROT_UNREACH消息(没有对应的处理程序), 该情况由两种:
1. ip报头中的协议是错误的, 没在[协议列表](https://elixir.bootlin.com/linux/v5.10.51/source/include/uapi/linux/in.h#L28)中.
1. kernel不支持该协议

[ip_local_deliver_finish()](https://elixir.bootlin.com/linux/v5.10.51/source/net/ipv4/ip_input.c#L226)->[ip_protocol_deliver_rcu()](https://elixir.bootlin.com/linux/v5.10.51/source/net/ipv4/ip_input.c#L187)存在没找到协议时的处理逻辑.

[__udp4_lib_rcv](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv4/udp.c#L2339)接收udpv4数据包时, 将由__udp4_lib_lookup_skb()查找匹配的udp套接字, 如果没有, 将检查checksum是否正确. 如果不正确, 直接丢弃; 如果正确则更新统计信息, 并返回code是ICMP_PORT_UNREACH的ICMP_DEST_UNREACH消息.

[ip_forward()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv4/ip_forward.c#L86)转发数据包时, 如果其长度超过了出站链路的MTU, 且在ipv4报头没有设置分段(DF)标志, 则将数据包丢弃, 并返回一条`ICMP_FRAG_NEEDED`. 同时在ip_forward()中其严格路由选择和网关选项(`if (opt->is_strictroute && rt->rt_uses_gateway)`)被设置, 将数据包丢弃并返回code是ICMP_SR_FAILED的ICMP_DEST_UNREACH.

icmp_reply()和icmp_send()还支持限速, 通过调用[icmpv4_xrlim_allow()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv4/icmp.c#L313)实现, 它返回true表示允许发送, 但它也存在部分不需要限速的流量类型:
- 消息的类型未知
- 数据包为PMTU发现数据包
- 设备是环回设备
- ICMP类型在限速掩码中未指定

仅当不满足上述任何条件时, 它才调用inet_peer_xrlim_allow()限速.

icmp_send()的参数:
- skb_in : 要发送的skb
- type : icmpv4消息类型
- code : icmpv4消息code
- info

    - 对于ICMP_PARAMETERPROB, 表示ipv4报头中发生分析问题的位置的偏移量
    - 对于ICMP_FRAG_NEEDED的ICMP_DEST_UNREACH是表示MTU
    - 对于ICMP_REDIR_HOST的ICMP_REDIRECT, 表示skb的ipv4报头中的目标ip地址

[icmp_send()](https://elixir.bootlin.com/linux/v5.10.53/source/include/net/icmp.h#L41)发送时会先进行完整性检查, 接着, 组播和广播包会被拒绝. 为检查数据包是否经过分段, 需要检查ipv4报头的frag_off字段. 如果已分段, 将发送一条ICMPv4消息, 但这仅针对第一个分段才这样做. 因此首先需要检查发送的icmpv4消息是不是错误消息, 如果是, 在检查skb包含的是不是icmpv4错误消息, 如果仍是则直接返回, 而不发送icmpv4消息. 另外, 如果类型是icmpv4未知类型(`>NR_ICMP_TYPES`), 也不发送, 直接返回. 之后由net->ipv4.sysctl_icmp_errors_use_inbound_ifaddr的值确定目标地址, 然后调用ip_options_echo()来复制skb的ipv4报头中的ip选项, 分配并初始化一个icmp_bxm对象, 并调用icmp_route_lookup()在路由选择子系统中执行查找操作, 最终调用icmp_push_reply().

[icmp_push_reply()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv4/icmp.c#L366)首先需要确定通过哪个套接字发送数据包`icmp_sk(dev_net((*rt)->dst.dev))`. dev_net()返回出站网络设备的网络命名空间. 接下来由icmp_sk()获取该套接字(在SMP中, 每个cpu都由一个套接字). 然后, 调用[ip_append_data()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv4/ip_output.c#L1306)将数据包交给ip层. 如果ip_append_data()失败将更新统计信息即ICMP_MIB_OUTERRORS加1, 并通过ip_flush_pending_frames()释放skb.

## ICMPv6
在L3报告错误方面, icmpv6与icmpv4很类似, 但icmpv6也支持更多的任务:
- 用于邻居发现(Neighbour Discovery, ND)协议, ND协议取代并改进了ipv4的ARP协议.
- 组播侦听者发现(Multicast Listener Discovery, MLD)协议, MLD协议相当于ipv4中的IGMP协议.

icmpv6在rfc 4443中定义, 它的实现基于ipv4, 但更复杂.

pingv6和traceroute6是基于ICMPv6. ICMPv6是在net/ipv6/icmp.c和net/ipv6/ip6_icmp.c中实现的.

### ICMPv6初始化
icmpv6的初始化是由[`inet6_init()`](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/af_inet6.c#L1038)中的[`icmpv6_init()`](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/icmp.c#L1060)和[icmpv6_sk_ops->init=icmpv6_sk_init()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/icmp.c#L1023)完成.

[icmpv6_protocol](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/icmp.c#L105)的成员有:
1. 回调函数handler : icmpv6_rcv(), 这意味着协议字段是IPPROTO_ICMPV6(58)的数据包都由icmpv6_rcv()处理

    设置了标志INET6_PROTO_NOPOLICY就不检查IPsec策略, 此时ip6_input_finish()将不会调用xfrm6_policy_check().

与ICMPv4一样, kernel为每个cpu都创建了一个原始ICMPv6套接字, 并将它们存储在一个array中. 要访问当前的sk, 需调用icmpv6_sk().

### ICMPv6报头
icmpv6报头[icmp6hdr](https://elixir.bootlin.com/linux/v5.10.53/source/include/uapi/linux/icmpv6.h#L8)的组成:
1. [type(8b)](https://elixir.bootlin.com/linux/v5.10.53/source/include/uapi/linux/icmpv6.h#L89)

    type的最高位是0时表示错误消息(即取值范围0~127); 最高位是1时(取值:128~255)表示信息消息. type具体列表可见[icmpv6-parameters.xml](https://www.iana.org/assignments/icmpv6-parameters/icmpv6-parameters.xml).
1. code(8b)
1. checksum(16b)
1. payload

### 接收ICMPv6消息
参考:
- [Linux内核中的IPSEC实现(3)](https://blog.csdn.net/dolphin98629/article/details/17713455)

收到的ICMPv6数据包交由[icmpv6_rcv()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/icmp.c#L859), 它的参数只有一个skb.

在icmpv6_rcv(), 在执行`xfrm6_policy_check()`(普通包返回true, 因此跳过IPsec策略检查)后, 将InMsgs SNMP计数器(ICMP6_MIB_INMSGS)加1. 接下来检查checksum: 如果不对, 将InErrors SNMP计数器(ICMP6_MIB_INERRORS)加1， 然后释放skb, 最终返回0即不返回错误. 然后, 读取ICMPv6报头的消息类型, 并调用ICMP6MSGIN_INC_STATS()宏将相应的procfs消息类型计数器(每种ICMPv6消息类型都有一个procfs计数器)加1, 例如: ICMPv6响应ping时, /proc/net/snmp6.Icmp6InEchos加1; 收到ICMPv6邻居请求是, 将/proc/net/snmp6.Icmp6InNeighborSolicits加1.

在ICMPv6中, 没有ICMPv4 icmp_pointers那样的分派表, 而是使用了一个较长的switch(type)来处理:
1. ICMP6_ECHO_REQUEST : 回应请求, 由icmpv6_echo_reply()处理
1. ICMP6_ECHO_REPLY : 回应应答, 由ping_rcv()处理, 它能处理ipv4/6双栈.
1. ICMP6_PKT_TOOGIG : 数据包太长

    首先检查数据块区域(skb->data指向的区域)包含的数据块长度是否不短于ICMP报头的长度, 该逻辑由pskb_may_pull()完成, 如果不满足该条件则丢包. 然后调用icmp6_notify()->raw6_icmp_error(), 让注册的套接字对ICMP消息进行处理.
1. ICMP6_ECHO_REQUEST, ICMP6_TIME_EXCEED和ICMP6_PARAMPROB也由icmp6_notify()处理.
1. 邻居消息

    所有邻居发现消息都由邻居发现方法[ndisc_rcv()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/ndisc.c#L1730)处理, 全部消息有:

    1. NDISC_ROUTER_SOLICITATION : 这些消息通常发送到表示所有路由器的组播地址FF02::2, 并使用路由器通知消息进行应答.
    1. NDISC_ROUTER_ADVERTISEMENT : 这些消息由路由器定期发送或为响应路由器请求消息而发送. 路由器通告包含用于确定链路和(或)地址配置, 建议ttl等信息的前缀.
    1. NDISC_NEIGHBOUR_SOLICITATION : 相当于ipv4中的arp请求
    1. NDISC_NEIGHBOUR_ADVERTISEMENT ： 相当于ipv4中的arp应答
    1. NDISC_REDIRECT : 路由器使用它将前往目的地的最佳第一跳告诉主机
1. ICMP6_MGM_QUERY : 组播侦听者查询, 由igmp6_event_query()处理
1. ICMP6_MGM_REPORT : 组播侦听者报告, 由igmp6_event_report()处理
1. 类型未知消息及下述消息由icmpv6_notify()处理

    1. ICMP6_MGM_REDUCTION : 退出组播组时, 主机会发送一条MLDv2 ICMP6_MGM_REDUCTION消息, 可见`net/ipv6/mcast.c`中的igmp6_leave_group().
    1. ICMP6_MLD2_REPORT : MLDv2组播侦听者报告数据包, 其目标地址通常为组播组地址FF02:16-表示所有支持MLDv2的路由器
    1. ICMP6_NI_QUERY : 结点信息查询
	1. ICMPV6_NI_REPLY : 结点信息响应
	1. ICMPV6_DHAAD_REQUEST : ICMP归属代理地址发现请求消息(Home Agent Address Discovery Request Message), 见RFC 6275.
	1. ICMPV6_DHAAD_REPLY : ICMP归属代理地址发现应答消息(Home Agent Address Discovery Reply Message), 见RFC 6275.
	1. ICMPV6_MOBILE_PREFIX_SOL : ICMP移动前缀请求消息格式(Mobile Prefix Solicitation Message Format)
	1. ICMPV6_MOBILE_PREFIX_ADV : ICMP移动前缀通告消息格式(Mobile Prefix Advertisement Message Format)

switch(type)的default 分支中, 如果消息满足`type & ICMPV6_INFOMSG_MASK`则丢包; 而不满足该条件的其他消息(即错误消息)将交给上层处理, 者符合RFC 4443的"Message Processing Rules"中的规定.

### 发送ICMPv6消息
发送ICMPv6消息主要方法是icmpv6_send(), 在ipv6栈中很多地方调用了该方法, 但仅在响应ICMPV6_ECHO_REQUEST(ping)时是使用icmpv6_echo_reply().

1. 发送"跳数限制超时"消息
[ip6_forward()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/ip6_output.c#L460)在`hop_limit <= 1`时调用`icmpv6_send(skb, ICMPV6_TIME_EXCEED, ICMPV6_EXC_HOPLIMIT, 0)`

1. 发送"分段重组超时"消息
[ip6frag_expire_frag_queue()](https://elixir.bootlin.com/linux/v5.10.53/source/include/net/ipv6_frag.h#L64)在分段超时发送`icmpv6_send(head, ICMPV6_TIME_EXCEED, ICMPV6_EXC_FRAGTIME, 0)`.

1. 发送"目的地不可达/端口不可达"消息
[__udp6_lib_rcv()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/udp.c#L894)收到udpv6数据包后, 将查找相应的UDPv6套接字. 如果没找到, 将检查checksum是否正确, 如果不正确, 将直接丢弃; 如果正确将更新统计信息(MIB计数器: /proc/net/snmp6.Udp6NoPorts), 并调用icmpv6_send()发送该消息`icmpv6_send(skb, ICMPV6_DEST_UNREACH, ICMPV6_PORT_UNREACH, 0)`.

1. 发送"需要分段"消息
ip6_forward()转发数据包时, 如果其长度大于出站MTU且skb的local_df位未设置(`if (ip6_pkt_too_big(skb, mtu))`), 将数据包丢弃, 并发送一条`icmpv6_send(skb, ICMPV6_PKT_TOOBIG, 0, mtu)`. 路径MTU(PMTU)发现过程使用了这种消息包含的信息.

> 在ipv4中, 这种情形下是发送ICMP_FRAG_NEEDED的ICMP_DEST_UNREACH消息.

1. 发送"参数问题"消息
[ip6_tlvopt_unknown()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/exthdrs.c#L74)在分析扩展报头时出错将发送`icmpv6_param_prob(skb, ICMPV6_UNK_OPTION, optoff)`.

icmpv6_send()会调用icmpv6_xrlim_allow()来实现限速, 和ICMPv4一样, 也不会对所有类型的流量执行限速, 例外有:
1. 信息消息
1. PMTU消息
1. 环回设备

如果上述条件不满足, 将调用与ipv4共享的inet_peer_xrlim_allow()来限速, 但不同的是在ipv6中不能设置速率掩码. ICMPv6规范RFC 4443并未禁止这样做, 但这种操作从未实现过.

[icmp6_send()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/icmp.c#L447)与icmp_send()类似, 执行一系列完整性检查后， 调用is_ineligible()来检查触发消息是否为ICMPv6错误消息, 如果是就退出了. 这种消息的长度不应超过1280, 即ipv6最小MTU: IPV6_MIN_MTU, 这是RFC 4443的2.4(c)节规定的: 所有ICMPv6错误消息多必须在长度不超过IPv6最小MTU的情况下, 尽可能多地包含IPv6触发数据包(导致错误的数据包)的内容. 接下来, 调用ip6_append_data()将消息交给ipv6层， 并调用icmpv6_push_pending_frames()释放skb.

[icmpv6_echo_reply()](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/icmp.c#L713), 是响应ICMPV6_ECHO消息时被调用. 它创建一个icmpv6_msg对象, 并将其类型设置为ICMPV6_ECHO_REPLY, 在调用ip6_append_data()和icmpv6_push_pending_frames()将这条消息交给ipv6层. 如果ip6_append_data()失败, 会把SNMP计数器(ICMP6_MIB_OUTERRORS)加1, 并调用ip6_flush_pending_frames()释放skb.

## icmp套接字(也叫ping套接字)
创建方法:
- ipv4 : `socket(PF_INET, SOCK_DGRAM, IPPROTO_ICMP)`
- ipv6 : `socket(PF_INET6, SOCK_DGRAM, IPPROTO_ICMP)`

icmp套接字的大部分实现代码在`net/ipv4/ping.c`中, 实际上`net/ipv4/ping.c`的大部分代码是双栈代码. 启用icmp套接字需设置`/proc/sys/net/ipv4/ping_group_range`, ipv4/6共用该参数, 默认情况下是`1 0`即禁止使用icmp套接字(包括root). `echo 0 2147483647 > /proc/sys/net/ipv4/ping_group_range`(其中 2147483647是[GID_T_MAX](https://elixir.bootlin.com/linux/v5.10.53/source/include/net/ping.h#L26)) 即可启用它. ICMP套接字只支持ICMP_ECHO和ICMPV6_ECHO_REQUEST, 且它们的code都必须是0.

[`ping_supported()`](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv4/ping.c#L453)可用于检查创建icmpv4/6消息的参数是否有效.

icmp套接字导出了procfs: `/proc/net/icmp`和`/proc/net/icmp6`, 更多见`<<精通Linux内核网络>> 3.5.3 procfs条目`

## 实践
### 通过iptables创建"目的地不可达"消息
iptables rule由netfilter子系统处理.

`iptables -A INPUT -j REJECT --reject-with icmp-host-prohibited`, `--reject-with`指定了应答icmpv4消息的类型, 其他可指定类型见[net/ipv4/netfilter/ipt_REJECT.c](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv4/netfilter/ipt_REJECT.c).

`iptables -A INPUT -s 2001::/64 -p ICMPv6 -j REJECT --reject-with icmp-adm-prohibited`, `--reject-with`指定了应答icmpv6消息的类型, 其他可指定类型见[net/ipv6/netfilter/ip6t_REJECT.c](https://elixir.bootlin.com/linux/v5.10.53/source/net/ipv6/netfilter/ip6t_REJECT.c)

## icmp重要方法
见`<<精通Linux内核网络>> 3.5.1 方法`