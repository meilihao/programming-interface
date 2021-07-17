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
netlink协议实现大都位于[`net/netlink`](https://elixir.bootlin.com/linux/v5.10.50/source/net/netlink)下. 最常用的是af_netlink.c, 它提供了netlink内核套接字api; 而genetlink.c提供了新的通用netlink api, 使用它创建netlink消息更容易. diag.c提供的api用于读写有关netlink套接字的信息.

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
Netlink 消息遵循非常特殊的格式: **所有消息必须与 4 字节边界对齐**. 例如，16 字节的消息必须按原样发送，但是 17 字节的消息必须被填充到 20 个字节.

与典型的网络通信不同，netlink 使用主机字节顺序来编码和解码整数，而不是普通的网络字节顺序（大端）.

Netlink 消息头使用以下格式：（来自 [RFC 3549](https://tools.ietf.org/html/rfc3549#section-2.3.2)）
1. 长度（32位）：整个消息的长度，包括报头和有效载荷（消息体）
1. 类型（16位）：消息包含什么样的信息，如错误，multi-part 消息的结束等
1. 标志（16位）：指示消息是请求的位标志，如 multi-part ，请求确认等
1. 序列号（32位）：用于关联请求和响应的数字；每个请求递增
1. PortID（PID）（32位）：有时称为端口号；用于唯一标识特定 netlink 套接字的数字, 类似于TCP/UDP的port, 与进程的pid是两码事

最后，有效载荷可能会立即跟随 netlink 消息头. 再次注意, 有效载荷必须填充到 4 字节的边界(header已对齐).

## struct
### [sockaddr_nl](https://elixir.bootlin.com/linux/v5.10.50/source/include/uapi/linux/netlink.h#L37)
它表示netlink套接字的地址:
- nl_family : 始终为AF_NETLINK
- nl_pad : 总是0
- nl_pid : netlink套接字的单播地址. 对于内核netlink套接字应为0; 用户态应用会将其设为pid, 但开发者显式设为0或不设置, 调用bind()后, 内核netlink_autobind()会尝试将其赋值为当前线程的进程id. 用户态创建的多个netlink套接字需保证nl_pid唯一.
- nl_groups : 组播组