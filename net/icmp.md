# icmp
icmp主要用作发送有关网络层(L3)错误和控制消息的机制, 通过icmp消息可获取有关通信环境中问题的反馈, 以便进行错误处理和诊断功能.

RFC 792对icmpv4做了基本定义, 包括icpmv4的目标以及各种icpmv4消息的格式. RFC 1122/4443/1812分别定义了对多种icmp消息的要求, icmpv6协议以及对路由器的要求.

> ping, traceroute是最典型的icmp应用.

## icmpv4
icmpv4消息分两类:
1. 错误消息
1. 查询消息

ping命令是在用户空间(by iputils)打开一个原始套接字并发送一条ICMP_ECHO消息, 响应是ICMP_REPLY消息.
traceroute是用于确定主机与给定目标ip间路径的工具(默认使用udp), 它基于ip报头中的ttl, 通过将ttl设为不同的值(从1开始, 收到ICMP_DEST_UNREACH消息就递增， 直至到达或超限制), 当ttl=0时转发设备会返回一条ICMP_TIME_EXCEED消息. 它利用返回的icmp 超时消息来创建数据包经过的路由列表, 直到到达目标ip并返回icmp echo reply消息.

icmpv4模块在[net/ipv4/icmp.c](https://elixir.bootlin.com/linux/v5.10.52/source/net/ipv4/icmp.c).

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

    - ICMP_ECHO由icmp_echo()处理
    - 除ping外, 其他消息只有ICMP_TIMESTAMP需要响应, 由icmp_timestamp()处理.
    - icmp应答有icmp_reply()处理.