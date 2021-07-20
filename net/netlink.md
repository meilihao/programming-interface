# netlink
参考:
- [linux netlink通信机制](https://www.cnblogs.com/wenqiang/p/6306727.html)
- [Linux，Netlink 和 Go](http://blog.studygolang.com/2017/07/linux-netlink-and-go-part-1-netlink/)

netlink是一种在RFC 3549中定义的 Linux 内核进程间通信机制(IPC), 是用以实现**用户进程与内核进程通信或多个用户空间进程间的双向通讯**, 是对标准socket实现的扩展,  也是网络应用程序与内核通信的最常用的接口.

> netlink可用于在用户态进程间通信, 但不推荐, 也不是开发netlink的初衷, 且unix域套接字更适合此场景.

> netlink不仅用于网络子系统, 还可用于selinux, audit, uevent等其他子系统.

> rtnetlink是专用于联网的netlink套接字, 用于路由消息, 邻接消息, 链路消息和其他网络子系统消息.

Netlink 是一种特殊的 socket(使用标准 BSD 套接字 API)名为AF_NETLINK, 旨在提供一种更灵活的用户空间进程和内核间通信方式, 用于替代笨拙的ioctl. 它是 Linux 所特有的，类似于 BSD 中的AF_ROUTE 但又远比它的功能强大. 目前在Linux 内核中使用netlink 进行应用与内核通信的应用很多; 包括：路由 daemon（NETLINK_ROUTE），用户态 socket 协议（NETLINK_USERSOCK），防火墙（NETLINK_FIREWALL），netfilter 子系统（NETLINK_NETFILTER），内核事件向用户态通知（NETLINK_KOBJECT_UEVENT），通用 netlink（NETLINK_GENERIC）等.
    
Netlink 是一种在内核与用户应用间进行双向数据传输的非常好的方式，用户态应用使用标准的 socket API 就可以使用 netlink 提供的强大功能，内核态需要使用专门的内核 API 来使用 netlink.
    
Netlink 相对于系统调用，ioctl 以及 /proc文件系统而言具有以下优点：
1. 使用ioctl必须定义ioctl号. netlink使用简单，只需要在include/linux/netlink.h中增加一个新类型的 netlink 协议定义即可,(如 #define NETLINK_TEST 20 然后，内核和用户态应用就可以立即通过 socket API 使用该 netlink 协议类型进行数据交换)
2. ioctl不能从内核向用户空间发送异步消息, 而netlink是一种**异步**通信机制，在内核与用户态应用之间传递的消息保存在socket缓存队列中，发送消息只是把消息保存在接收者的socket的接收队列，而不需要等待接收者收到消息
3. 使用 netlink 的内核部分可以采用模块的方式实现，使用 netlink 的应用部分和内核部分没有编译时依赖
4. netlink **支持多播(即组播)**，内核模块或应用可以把消息多播给一个netlink组，属于该neilink 组的任何内核模块或应用都能接收到该消息，内核事件向用户态的通知机制就使用了这一特性
5. 内核可以使用 netlink 首先发起会话

> [Netlink 消息格式](https://tools.ietf.org/html/rfc3549#section-2.3.2)

## netlink常用数据结构及函数
netlink协议实现大都位于[`net/netlink`](https://elixir.bootlin.com/linux/v5.10.51/source/net/netlink)下. 最常用的是af_netlink.c, 它提供了netlink内核套接字api; 而genetlink.c提供了新的通用netlink api, 使用它创建netlink消息更容易. diag.c提供的api用于读写有关netlink套接字的信息.

用户态应用使用标准的 socket API有`sendto()，recvfrom()； sendmsg(), recvmsg()`

要与 netlink 进行通信，必须打开 netlink 套接字:
- 用户态使用 socket() 系统调用, 可使用SOCK_RAW或SOCK_DGRAM -> netlink_create()
- kernel使用netlink_kernel_create() -> __netlink_kernel_create(), 会设置标志NETLINK_KERNEL_SOCKET.

最终这两种方法都会调用__netlink_create(), 它通过sk_alloc()分配套接字, 并对其进行初始化.  

一旦创建套接字，必须调用 bind() 来准备发送和接收消息.

## netlink lib
推荐使用libnl api, iproute2就使用了它.

> iproute2仍少量使用ioctl, 比如ip tuntap使用ioctl添加/删除tun/tap设备.

> 已废弃的net-tools基于ioctl.

## Netlink 消息格式
Netlink 消息遵循非常特殊的格式(定义在RFC 3549的2.2 `Message Format`): **所有消息必须与 4 字节边界对齐**. 例如，16 字节的消息必须按原样发送，但是 17 字节的消息必须被填充到 20 个字节.

与典型的网络通信不同，netlink 使用主机字节顺序来编码和解码整数，而不是普通的网络字节顺序（大端）.

Netlink [nlmsghdr](https://elixir.bootlin.com/linux/v5.10.51/source/include/uapi/linux/netlink.h#L44) 消息头(来自 [RFC 3549](https://tools.ietf.org/html/rfc3549#section-2.3.2))使用以下格式, 长度固定, 共16B:
1. nlmsg_len,长度（32位）：整个消息的长度，包括报头和有效载荷（消息体）
1. nlmsg_type,类型（16位）：消息包含什么样的信息，如错误，multi-part 消息的结束等

    - NLMSG_NOOP : 不执行任何操作， 必须将消息丢弃
    - NLMSG_ERROR : 发生了错误
    - NLMSG_DONE : 标识由多部分组成的消息的末尾
    - NLMSG_OVERRUN : 缓冲区溢出通知, 表示发生了错误, 数据已丢失

    协议簇可添加自己的netlink消息类型, 比如rtnetlink添加的RTM_NEWLINK等. 但小于NLMSG_MIN_TYPE(0x10)的消息类型值需要保留, 用于控制消息.
1. [nlmsg_flags](https://elixir.bootlin.com/linux/v5.10.51/source/include/uapi/linux/netlink.h#L54), 标志（16位）：指示消息是请求的位标志，如 multi-part ，请求确认等

    - NLM_F_REQUEST : 消息为请求消息
    - NLM_F_MULTI :
    - NLM_F_ACK : 希望接收方收到ack对消息进行应答. netlink ack消息由[`netlink_ack()`](https://elixir.bootlin.com/linux/v5.10.51/source/net/netlink/af_netlink.c#L2398)发送.
    - NLM_F_DUMP : 检索有关表/条目的消息
    - NLM_F_ROOT : 指定root
    - NLM_F_MATCH : 返回所有匹配的条目
    - NLM_F_ATOMIC : 该标志已废弃
    - NLM_F_REPLACE : 覆盖已有条目 # 当前开始的标志都是针对条目创建的说明符
    - NLM_F_EXCL : 保留已有条目不动
    - NLM_F_CREATE : 创建条目(如果它不存在)
    - NLM_F_APPEND : 在列表末尾添加条目
    - NLM_F_ECHO : 回应当前请求
1. nlmsg_seq, 序列号（32位）：用于关联请求和响应的数字；每个请求递增. netlink并未要求必须使用序列号.
1. nlmsg_pid, PortID（PID）（32位）：有时称为端口号；用于唯一标识特定 netlink 套接字的数字, 类似于TCP/UDP的port. 对于从内核发出的消息总为0; 对于从用户空间发往kernel时, 可将其设置为发送消息的用户空间进程的pid.

最后，有效载荷可能会立即跟随 netlink 消息头. 再次注意, 有效载荷必须填充到 4 字节的边界(header已对齐). 它用TLV(类型-长度-值)表示, 其中类型和长度字段的长度是固定值, 通常是1~4B.

netlink 属性允许往 neltink 消息中添加任意数量的任意长度数据段, 它在playload和pad(如果有的话)后, 所有的netlink属性必须与4B边界(NLA_ALIGNTO)对其.
![](/misc/img/net/netlink/attribute_hdr.png)

netlink属性头用[nlattr](https://elixir.bootlin.com/linux/v5.10.51/source/include/uapi/linux/netlink.h#L213)表示:
- nla_len : 属性的长度
- [nla_type](https://elixir.bootlin.com/linux/v5.10.51/source/include/net/netlink.h#L165) : 属性的类型, 常用的有:

    - NLA_U32 : 32位无符号整数
    - NLA_STRING : 变长字符串
    - NLA_NESTED : 嵌套属性
    - NLA_UNSPEC : 类型和长度未知

每个协议簇都可定义属性有效性策略, 即对收到的属性的期望, 它用[nla_policy](https://elixir.bootlin.com/linux/v5.10.51/source/include/net/netlink.h#L315)表示.
属性有效性策略是nla_policy的数组, 它用属性号作为索引, 对每个属性(定长属性除外), 如果nla_policy的len为0则不执行有效性检查. 如果属性类型是NLA_STRING, 则len值应为字符串的最大长度(不包括末尾的NULL); 如果属性类型是NLA_UNSPEC, 应将len设置为属性有效载荷的长度; 如果是NLA_FLAG, 将不使用len(因为属性存在表示true, 不存在表示false)

在kernel中, 接收通用netlink消息由[genl_rcv_msg()](https://elixir.bootlin.com/linux/v5.10.51/source/net/netlink/genetlink.c#L787)负责, 如果消息flag是NLM_F_DUMP则调用netlink_dump_start()来转储表; 否则调用nlmsg_parse()对playload进行分析. nlmsg_parse()调用validate_nla()来验证属性的有效性. 如果属性的类型值大于maxtype, 出于向后兼容考虑会丢弃; 没有通过验证将不执行genl_rcv_msg()的后续操作. 

## kernel
在kernel网络栈中, 可创建多种netlink套接字, 它们分别处理不同类型的消息. 比如处理NETLINK_ROUTE消息的netlink套接字是在[rtnetlink_net_init()](https://elixir.bootlin.com/linux/v5.10.51/source/net/core/rtnetlink.c#L5631)中创建到来.

rtnetlink套接字支持网络命名空间. 网络命名空间对象([`struct net`](https://elixir.bootlin.com/linux/v5.10.51/source/include/net/net_namespace.h#L56))包含一个名为rtnl的成员(rtnetlink套接字). 因此rtnetlink_net_init()调用netlink_kernel_create()创建rtnetlink套接字后会将其赋值给rtnl.

netlink_kernel_create()参数:
1. net : 网络命名空间
1. netlink协议 : 比如NETLINK_ROUTE表示rtnetlink消息, NETLINK_XFRM表示IPsec消息, NETLINK_AUDIT表示审计子系统消息等. 有20多种netlink协议, 但不多于32(MAX_LINKS), 它们定义在[include/linux/netlink.h](https://elixir.bootlin.com/linux/v5.10.51/source/include/uapi/linux/netlink.h#L9)
1. [netlink_kernel_cfg](https://elixir.bootlin.com/linux/v5.10.51/source/include/linux/netlink.h#L44)的指针 : 用于创建netlink套接字的可选参数
    
    - groups : 用于指定组播组(或组播组掩码). 要加入组播组, 可通过设置sockaddr_nl->nl_groups(也可调用libnl中的`nl_join_groups()`)来实现, 但使用这种方式最多只能加入32个组播组. 从2.6.14起, libnl的nl_socket_add_memberships()/nl_socket_drop_memberships()可利用套接字选项NETLINK_ADD_MEMBERSHIP/NETLINK_DROP_MEMBERSHIP来加入/退出组播组, 并可加入更多的组播组.
    - flags : NL_CFG_F_NONROOT_RECV, 非root用户可绑定到组播组, netlink_bind()绑定到组播组时会检查它; NL_CFG_F_NONROOT_SEND, 非root用户可发送组播.
    - input ： 指定回调函数. NULL时表示内核套接字将无法接收来自用户空间的数据(但能从kernel向userspace发送), 对于uevent内核事件就只需该种单项发送, 因此lib/kobject_uevent.c#uevnet_net_init()中就指定成了NULL. rtnetlink套接字使用rtnetlink_rcv处理来自用户空间的数据.
    - cb_mutex : 互斥锁. 没有指定时默认使用[cb_def_mutex](https://elixir.bootlin.com/linux/v5.10.51/source/net/netlink/af_netlink.c#L642), 只有rtnetlink套接字使用了rtnl_mutex, 其他均未指定互斥锁.

netlink_kernel_create()->[__netlink_kernel_create()](https://elixir.bootlin.com/linux/v5.10.51/source/net/netlink/af_netlink.c#L2029)->[netlink_insert()](https://elixir.bootlin.com/linux/v5.10.51/source/net/netlink/af_netlink.c#L560)

netlink_insert()在nl_table表中创建一个条目. 对nl_table表的访问由读写锁nl_table_lock保护; netlink_lookup()可根据指定的协议和端口号对它进行查找; 要为特定消息类型注册回调函数可用rtnl_registerr(). 网络栈很多地方都注册了这样的回调函数, 比如rtnetlink_init()中为RTM_NEWLINK(新建链路), RTM_DELLINK(删除链路), RTM_GETROUTE(转储路由表)等; 在net/core/neighbour.c中, 为RTM_NEWNEIGH(创建新邻居), RTM_DELHEIGH(删除邻居), RTM_GETNEIGHTBL(转储邻居表)等消息注册了回调函数.

要使用netlink内核套接字, 需要使用[`rtnl_register()`](https://elixir.bootlin.com/linux/v5.10.51/source/net/core/rtnetlink.c#L267)注册, 参数是:
1. protocol, 协议簇(如果不针对任何协议, 可以设为PF_UNSPEC), 其他见[`include/linux/socket.h`](https://elixir.bootlin.com/linux/v5.10.51/source/include/linux/socket.h#L230).
1. netlink消息类型, 如RTM_NEWLINK, RTM_NEWNEIGH. rtnetlink有一些专用的消息类型, 见[include/uapi/linux/rtnetlink.h](https://elixir.bootlin.com/linux/v5.10.51/source/include/uapi/linux/rtnetlink.h#L25).
1. 回调函数:doit, dumpit和calcit, 指定了为处理消息而执行的操作, 通常只指定一个即可.

    - doit用于指定添加, 删除, 修改等操作
    - dumpit用于检索操作
    - calcit用于计算缓冲区大小

rtnl_register()会将组织好的结构放入rtnl_msg_handlers. rtnl_msg_handlers的第一级是以protocol为索引的数组, 每个成员本身也是一个将消息类型作为其索引的表.

rtnetlink消息是使用rtmsg_ifinfo()发送的. 比如[`dev_open()`](https://elixir.bootlin.com/linux/v5.10.51/source/net/core/dev.c#L1564)使用了`rtmsg_ifinfo(RTM_NEWLINK, dev, IFF_UP|IFF_RUNNING, GFP_KERNEL)`来创建一条新链路. [`rtmsg_ifinfo()`](https://elixir.bootlin.com/linux/v5.10.51/source/net/core/rtnetlink.c#L3845)首先调用`nlmsg_new()`分配一个大小合适的sk_buff; 然后创建两个对象: netlink消息报头nlmsghdr和ifinfomsg对象, 后者紧跟在nlmsghdr后面, 并由rtnl_fill_ifinfo()初始化这两对象. 再调用[`rtnl_notify()`](https://elixir.bootlin.com/linux/v5.10.51/source/net/core/rtnetlink.c#L728)发送数据包, 其实实际发送由[`nlmsg_notify()`](https://elixir.bootlin.com/linux/v5.10.51/source/net/netlink/af_netlink.c#L2524)完成.

## struct
### [sockaddr_nl](https://elixir.bootlin.com/linux/v5.10.51/source/include/uapi/linux/netlink.h#L37)
它表示netlink套接字的地址:
- nl_family : 始终为AF_NETLINK
- nl_pad : 总是0
- nl_pid : netlink套接字的单播地址. 对于内核netlink套接字应为0; 用户态应用会将其设为pid, 但开发者显式设为0或不设置, 调用bind()后, 内核netlink_autobind()会尝试将其赋值为当前线程的进程id. 用户态创建的多个netlink套接字需保证nl_pid唯一.
- nl_groups : 组播组