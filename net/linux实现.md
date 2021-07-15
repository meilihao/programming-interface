# net的linux实现
kernel不涉及L4之上的各层, 这些层的任务由用户空间应用来处理. 同时kernel不涉及物理层(L1).

kernel涉及的L2,L3,L4分别对应OSI 7层model的数据链路层,网络层,传输层. 从本质上讲, linux网络栈的任务就是将接收到的数据包从L2(网络设备驱动程序)传递给L3(网络层, 通常是ipv4/ipv6). 接下来, 如果数据包的目的地是当前设备, 它就将数据包传递到L4(传输层);如果需要转发, 就将数据包还给L2进行传输. 对于本地生成的出站数据包将从L4依次传递给L3, L2, 再由网络设备驱动程序进行传输.

linux网络栈处理期间可能有以下行为:
1. 根据协议规则(IPsec, NAT等), 可能需要对数据包进行修改
1. 数据包可能被丢弃
1. 数据包可能导致设备发送错误消息
1. 可能对数据包进行分段
1. 可能需要重组数据包
1. 需要计算数据包的checksum

应用层和内核互通的机制，就是通过 Socket 系统调用. socket属于操作系统的概念，而非网络协议分层的概念. 只不过操作系统选择对于网络协议的实现模式是，二到四层的处理代码在内核里面，七层的处理代码让应用自己去做，两者需要跨内核态和用户态通信，就需要一个系统调用完成这个衔接，这就是 Socket.

> rsockets is a protocol over RDMA that supports a socket-level API for applications.

![](/misc/img/net/98a4496fff94eb02d1b1b8ae88f8dc28.jpeg)

从本质上来讲，所谓的建立连接，其实是为了在客户端和服务端维护连接，而建立一定的数据结构来维护双方交互的状态，并用这样的数据结构来保证面向连接的特性. TCP 无法左右中间的任何通路，也没有什么虚拟的连接，中间的通路根本意识不到两端使用了 TCP 还是 UDP.

所谓的连接，就是两端数据结构状态的协同，两边的状态能够对得上。符合 TCP 协议的规则，就认为连接存在；两面状态对不上，连接就算断了.

流量控制和拥塞控制其实就是根据收到的对端的网络包，调整两端数据结构的状态. TCP 协议的设计理论上认为，这样调整了数据结构的状态，就能进行流量控制和拥塞控制了，其实在通路上是不是真的做到了，谁也管不着.

所谓的可靠，也是两端的数据结构做的事情. 不丢失其实是数据结构在“点名”，顺序到达其实是数据结构在“排序”，面向数据流其实是数据结构将零散的包，按照顺序捏成一个流发给应用层.

总而言之，“连接”两个字让人误以为功夫在通路，其实功夫在两端.

无论是用 socket 操作 TCP，还是 UDP，我们首先都要调用`nt socket(int domain, int type, int protocol)`.

socket 函数用于创建一个 socket 的文件描述符，唯一标识一个 socket. 把它叫作文件描述符，是因为在内核会创建类似文件系统的数据结构，并且后续的操作都有用到它.

socket 函数有三个参数:
- domain：表示使用什么 IP 层协议

    AF_INET 表示 IPv4，AF_INET6 表示 IPv6

- type：表示 socket 类型

    SOCK_STREAM，顾名思义就是 TCP 面向流的，SOCK_DGRAM 就是 UDP 面向数据报的，SOCK_RAW 可以直接操作 IP 层，或者非 TCP 和 UDP 的协议. 例如 ICMP

- protocol 表示的协议，包括 IPPROTO_TCP、IPPTOTO_UDP

通信结束后，还要像关闭文件一样，关闭 socket.

## tcp编程
![tcp编程模型](/misc/img/net/997e39e5574252ada22220e4b3646dda.png)

TCP 的服务端要先监听一个端口，一般是先调用 bind 函数，给这个 socket 赋予一个端口和 IP 地址.
```c

int bind(int sockfd, const struct sockaddr *addr,socklen_t addrlen);

struct sockaddr_in {
  __kernel_sa_family_t  sin_family;  /* Address family    */
  __be16    sin_port;  /* Port number      */
  struct in_addr  sin_addr;  /* Internet address    */

  /* Pad to size of `struct sockaddr'. */
  unsigned char    __pad[__SOCK_SIZE__ - sizeof(short int) -
      sizeof(unsigned short int) - sizeof(struct in_addr)];
};

struct in_addr {
  __be32  s_addr;
};
```

其中，sockfd 是创建的 socket 文件描述符. 在 sockaddr_in 结构中，sin_family 设置为 AF_INET，表示 IPv4；sin_port 是端口号；sin_addr 是 IP 地址.

服务端所在的服务器可能有多个网卡、多个地址，可以选择监听在一个地址，也可以监听 0.0.0.0 表示所有的地址都监听. 服务端一般要监听在一个众所周知的端口上，例如，Nginx 一般是 80/443.

客户端要访问服务端，肯定事先要知道服务端的地址. 客户端不需要 bind，是因为随机分配一个端口就可以了，只有你主动去连接别人，别人不会主动连接你，没有人关心客户端监听到了哪里.

如果上面代码中的数据结构，里面的变量名称都有“be”两个字母，代表的意思是“big-endian”. 如果在网络上传输超过 1 Byte 的类型，就要区分大端（Big Endian）和小端（Little Endian）.

TCP/IP 栈是按照大端来设计的，而 x86 机器多按照小端来设计，因而发出去时需要做一个转换.

接下来，服务端要调用 `int listen(int sockfd, int backlog)` 进入 LISTEN 状态，等待客户端进行连接.

连接的建立过程，也即三次握手，是 TCP 层的动作，是在内核完成的，应用层不需要参与.

接着，服务端只需要调用 `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`，等待内核完成了至少一个连接的建立，才返回. 如果没有一个连接完成了三次握手，accept 就一直等待；如果有多个客户端发起连接，并且在内核里面完成了多个三次握手，建立了多个连接，这些连接会被放在一个队列里面. accept 会从队列里面取出一个来进行处理. 如果想进一步处理其他连接，需要调用多次 accept，所以 accept 往往在一个循环里面.

接下来，客户端可以通过 `int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen)` 函数发起连接.

connect()先在参数中指明要连接的 IP 地址和端口号，然后发起三次握手. 内核会给客户端分配一个临时的端口. 一旦握手成功，服务端的 accept 就会返回另一个 socket. 这里需要注意的是，监听的 socket 和真正用来传送数据的 socket，是两个 socket，一个叫作监听 socket，一个叫作已连接 socket. 成功连接建立之后，双方开始通过 read 和 write 函数来读写数据，就像往一个文件流里面写东西一样.

## udp编程
![udp编程模型](/misc/img/net/283b0e1c21f0277ba5b4b5cbcaca03b2.png)

UDP 是没有连接的，所以不需要三次握手，也就不需要调用 listen 和 connect，但是 UDP 的交互仍然需要 IP 地址和端口号，因而也需要 bind.

对于 UDP 来讲，没有所谓的连接维护，也没有所谓的连接的发起方和接收方，甚至都不存在客户端和服务端的概念，大家就都是客户端，也同时都是服务端. 只要有一个 socket，多台机器就可以任意通信，不存在哪两台机器是属于一个连接的概念. 因此，每一个 UDP 的 socket 都需要 bind. 每次通信时，调用 sendto 和 recvfrom，都要传入 IP 地址和端口.

```c
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
```

## 解析 socket 函数
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501
int __sys_socket(int family, int type, int protocol)
{
	int retval;
	struct socket *sock;
	int flags;

	/* Check the SOCK_* constants for consistency.  */
	BUILD_BUG_ON(SOCK_CLOEXEC != O_CLOEXEC);
	BUILD_BUG_ON((SOCK_MAX | SOCK_TYPE_MASK) != SOCK_TYPE_MASK);
	BUILD_BUG_ON(SOCK_CLOEXEC & SOCK_TYPE_MASK);
	BUILD_BUG_ON(SOCK_NONBLOCK & SOCK_TYPE_MASK);

	flags = type & ~SOCK_TYPE_MASK;
	if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
		return -EINVAL;
	type &= SOCK_TYPE_MASK;

	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

	retval = sock_create(family, type, protocol, &sock);
	if (retval < 0)
		return retval;

	return sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
}

SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
	return __sys_socket(family, type, protocol);
}
```

Socket 系统调用会调用 sock_create 创建一个 struct socket 结构，然后通过 sock_map_fd 和文件描述符对应起来.

在创建 Socket 的时候，有三个参数:
1. family，表示地址族. 不是所有的 Socket 都要通过 IP 进行通信，还有其他的通信方式. 例如，domain sockets 就是通过本地文件进行通信的，不需要 IP 地址. 只不过，通过 IP 地址只是最常用的模式，所以这里着重分析这种模式.

  ```c
  // https://elixir.bootlin.com/glibc/latest/source/bits/socket.h#L109
  /* Protocol families.  */
  #define	PF_UNSPEC	0	/* Unspecified.  */
  #define	PF_LOCAL	1	/* Local to host (pipes and file-domain).  */
  #define	PF_UNIX		PF_LOCAL /* Old BSD name for PF_LOCAL.  */
  #define	PF_FILE		PF_LOCAL /* POSIX name for PF_LOCAL.  */
  #define	PF_INET		2	/* IP protocol family.  */
  ...

  /* Address families.  */
  #define	AF_UNSPEC	PF_UNSPEC
  #define	AF_LOCAL	PF_LOCAL
  #define	AF_UNIX		PF_UNIX
  #define	AF_FILE		PF_FILE
  #define	AF_INET		PF_INET
  ...

  // 或者from kernel, **推荐**
  // https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/socket.h#L175
  /* Supported address families. */
  #define AF_UNSPEC	0
  #define AF_UNIX		1	/* Unix domain sockets 		*/
  #define AF_LOCAL	1	/* POSIX name for AF_UNIX	*/
  #define AF_INET		2	/* Internet IP Protocol 	*/
  #define AF_AX25		3	/* Amateur Radio AX.25 		*/
  #define AF_IPX		4	/* Novell IPX 			*/
  #define AF_APPLETALK	5	/* AppleTalk DDP 		*/
  #define AF_NETROM	6	/* Amateur Radio NET/ROM 	*/
  #define AF_BRIDGE	7	/* Multiprotocol bridge 	*/
  #define AF_ATMPVC	8	/* ATM PVCs			*/
  #define AF_X25		9	/* Reserved for X.25 project 	*/
  #define AF_INET6	10	/* IP version 6			*/
  #define AF_ROSE		11	/* Amateur Radio X.25 PLP	*/
  #define AF_DECnet	12	/* Reserved for DECnet project	*/
  #define AF_NETBEUI	13	/* Reserved for 802.2LLC project*/
  #define AF_SECURITY	14	/* Security callback pseudo AF */
  #define AF_KEY		15      /* PF_KEY key management API */
  #define AF_NETLINK	16
  #define AF_ROUTE	AF_NETLINK /* Alias to emulate 4.4BSD */
  #define AF_PACKET	17	/* Packet family		*/
  #define AF_ASH		18	/* Ash				*/
  #define AF_ECONET	19	/* Acorn Econet			*/
  #define AF_ATMSVC	20	/* ATM SVCs			*/
  #define AF_RDS		21	/* RDS sockets 			*/
  #define AF_SNA		22	/* Linux SNA Project (nutters!) */
  #define AF_IRDA		23	/* IRDA sockets			*/
  #define AF_PPPOX	24	/* PPPoX sockets		*/
  #define AF_WANPIPE	25	/* Wanpipe API Sockets */
  #define AF_LLC		26	/* Linux LLC			*/
  #define AF_IB		27	/* Native InfiniBand address	*/
  #define AF_MPLS		28	/* MPLS */
  #define AF_CAN		29	/* Controller Area Network      */
  #define AF_TIPC		30	/* TIPC sockets			*/
  #define AF_BLUETOOTH	31	/* Bluetooth sockets 		*/
  #define AF_IUCV		32	/* IUCV sockets			*/
  #define AF_RXRPC	33	/* RxRPC sockets 		*/
  #define AF_ISDN		34	/* mISDN sockets 		*/
  #define AF_PHONET	35	/* Phonet sockets		*/
  #define AF_IEEE802154	36	/* IEEE802154 sockets		*/
  #define AF_CAIF		37	/* CAIF sockets			*/
  #define AF_ALG		38	/* Algorithm sockets		*/
  #define AF_NFC		39	/* NFC sockets			*/
  #define AF_VSOCK	40	/* vSockets			*/
  #define AF_KCM		41	/* Kernel Connection Multiplexor*/
  #define AF_QIPCRTR	42	/* Qualcomm IPC Router          */
  #define AF_SMC		43	/* smc sockets: reserve number for
          * PF_SMC protocol family that
          * reuses AF_INET address family
          */
  #define AF_XDP		44	/* XDP sockets			*/

  #define AF_MAX		45	/* For now.. */

  /* Protocol families, same as address families. */
  #define PF_UNSPEC	AF_UNSPEC
  #define PF_UNIX		AF_UNIX
  #define PF_LOCAL	AF_LOCAL
  #define PF_INET		AF_INET
  #define PF_AX25		AF_AX25
  #define PF_IPX		AF_IPX
  #define PF_APPLETALK	AF_APPLETALK
  #define	PF_NETROM	AF_NETROM
  #define PF_BRIDGE	AF_BRIDGE
  #define PF_ATMPVC	AF_ATMPVC
  #define PF_X25		AF_X25
  #define PF_INET6	AF_INET6
  #define PF_ROSE		AF_ROSE
  #define PF_DECnet	AF_DECnet
  #define PF_NETBEUI	AF_NETBEUI
  #define PF_SECURITY	AF_SECURITY
  #define PF_KEY		AF_KEY
  #define PF_NETLINK	AF_NETLINK
  #define PF_ROUTE	AF_ROUTE
  #define PF_PACKET	AF_PACKET
  #define PF_ASH		AF_ASH
  #define PF_ECONET	AF_ECONET
  #define PF_ATMSVC	AF_ATMSVC
  #define PF_RDS		AF_RDS
  #define PF_SNA		AF_SNA
  #define PF_IRDA		AF_IRDA
  #define PF_PPPOX	AF_PPPOX
  #define PF_WANPIPE	AF_WANPIPE
  #define PF_LLC		AF_LLC
  #define PF_IB		AF_IB
  #define PF_MPLS		AF_MPLS
  #define PF_CAN		AF_CAN
  #define PF_TIPC		AF_TIPC
  #define PF_BLUETOOTH	AF_BLUETOOTH
  #define PF_IUCV		AF_IUCV
  #define PF_RXRPC	AF_RXRPC
  #define PF_ISDN		AF_ISDN
  #define PF_PHONET	AF_PHONET
  #define PF_IEEE802154	AF_IEEE802154
  #define PF_CAIF		AF_CAIF
  #define PF_ALG		AF_ALG
  #define PF_NFC		AF_NFC
  #define PF_VSOCK	AF_VSOCK
  #define PF_KCM		AF_KCM
  #define PF_QIPCRTR	AF_QIPCRTR
  #define PF_SMC		AF_SMC
  #define PF_XDP		AF_XDP
  #define PF_MAX		AF_MAX
  ```

> netlink相关的协议在[这里](https://elixir.bootlin.com/linux/v5.8.1/source/include/uapi/linux/netlink.h#L9). netlink[支持实现自定义协议](https://www.cnblogs.com/wenqiang/p/6306727.html), 有公司就基于netlink自定义协议实现了cdp功能.
  
1. type，也即 Socket 的类型. 类型是比较少的
1. protocol，是协议. 协议数目是比较多的，也就是说，多个协议会属于同一种类型. 常用的 Socket 类型有三种，分别是 SOCK_STREAM、SOCK_DGRAM 和 SOCK_RAW

  ```c
  // https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/net.h#L59
  /**
  * enum sock_type - Socket types
  * @SOCK_STREAM: stream (connection) socket
  * @SOCK_DGRAM: datagram (conn.less) socket
  * @SOCK_RAW: raw socket
  * @SOCK_RDM: reliably-delivered message
  * @SOCK_SEQPACKET: sequential packet socket
  * @SOCK_DCCP: Datagram Congestion Control Protocol socket
  * @SOCK_PACKET: linux specific way of getting packets at the dev level.
  *		  For writing rarp and other similar things on the user level.
  *
  * When adding some new socket type please
  * grep ARCH_HAS_SOCKET_TYPE include/asm-* /socket.h, at least MIPS
  * overrides this enum for binary compat reasons.
  */
  enum sock_type {
    SOCK_STREAM	= 1,
    SOCK_DGRAM	= 2,
    SOCK_RAW	= 3,
    SOCK_RDM	= 4,
    SOCK_SEQPACKET	= 5,
    SOCK_DCCP	= 6,
    SOCK_PACKET	= 10,
  };
  ```

  SOCK_STREAM 是面向数据流的，协议 IPPROTO_TCP 属于这种类型. SOCK_DGRAM 是面向数据报的，协议 IPPROTO_UDP 属于这种类型. 如果在内核里面看的话，IPPROTO_ICMP 也属于这种类型. SOCK_RAW 是原始的 IP 包，IPPROTO_IP 属于这种类型.

这里重点看 SOCK_STREAM 类型和 IPPROTO_TCP 协议. 为了管理 family、type、protocol 这三个分类层次，内核会创建对应的数据结构. 接下来，打开 [sock_create](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501) 函数看一下, 它会调用 [__sock_create](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501).

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501
/**
 *	__sock_create - creates a socket
 *	@net: net namespace
 *	@family: protocol family (AF_INET, ...)
 *	@type: communication type (SOCK_STREAM, ...)
 *	@protocol: protocol (0, ...)
 *	@res: new socket
 *	@kern: boolean for kernel space sockets
 *
 *	Creates a new socket and assigns it to @res, passing through LSM.
 *	Returns 0 or an error. On failure @res is set to %NULL. @kern must
 *	be set to true if the socket resides in kernel space.
 *	This function internally uses GFP_KERNEL.
 */

int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
{
	int err;
	struct socket *sock;
	const struct net_proto_family *pf;

	/*
	 *      Check protocol is in range
	 */
	if (family < 0 || family >= NPROTO)
		return -EAFNOSUPPORT;
	if (type < 0 || type >= SOCK_MAX)
		return -EINVAL;

	/* Compatibility.

	   This uglymoron is moved from INET layer to here to avoid
	   deadlock in module load.
	 */
	if (family == PF_INET && type == SOCK_PACKET) {
		pr_info_once("%s uses obsolete (PF_INET,SOCK_PACKET)\n",
			     current->comm);
		family = PF_PACKET;
	}

	err = security_socket_create(family, type, protocol, kern);
	if (err)
		return err;

	/*
	 *	Allocate the socket and allow the family to set things up. if
	 *	the protocol is 0, the family is instructed to select an appropriate
	 *	default.
	 */
	sock = sock_alloc();
	if (!sock) {
		net_warn_ratelimited("socket: no more sockets\n");
		return -ENFILE;	/* Not exactly a match, but its the
				   closest posix thing */
	}

	sock->type = type;

#ifdef CONFIG_MODULES
	/* Attempt to load a protocol module if the find failed.
	 *
	 * 12/09/1996 Marcin: But! this makes REALLY only sense, if the user
	 * requested real, full-featured networking support upon configuration.
	 * Otherwise module support will break!
	 */
	if (rcu_access_pointer(net_families[family]) == NULL)
		request_module("net-pf-%d", family);
#endif

	rcu_read_lock();
	pf = rcu_dereference(net_families[family]);
	err = -EAFNOSUPPORT;
	if (!pf)
		goto out_release;

	/*
	 * We will call the ->create function, that possibly is in a loadable
	 * module, so we have to bump that loadable module refcnt first.
	 */
	if (!try_module_get(pf->owner))
		goto out_release;

	/* Now protected by module ref count */
	rcu_read_unlock();

	err = pf->create(net, sock, protocol, kern);
	if (err < 0)
		goto out_module_put;

	/*
	 * Now to bump the refcnt of the [loadable] module that owns this
	 * socket at sock_release time we decrement its refcnt.
	 */
	if (!try_module_get(sock->ops->owner))
		goto out_module_busy;

	/*
	 * Now that we're done with the ->create function, the [loadable]
	 * module can have its refcnt decremented
	 */
	module_put(pf->owner);
	err = security_socket_post_create(sock, family, type, protocol, kern);
	if (err)
		goto out_sock_release;
	*res = sock;

	return 0;

out_module_busy:
	err = -EAFNOSUPPORT;
out_module_put:
	sock->ops = NULL;
	module_put(pf->owner);
out_sock_release:
	sock_release(sock);
	return err;

out_release:
	rcu_read_unlock();
	goto out_sock_release;
}
EXPORT_SYMBOL(__sock_create);
```

__sock_create先是分配了一个 [struct socket](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/net.h#L112) 结构. 接下来要用到 family 参数. 这里有一个 [net_families](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L173) 数组，可以以 family 参数为下标，找到对应的 [struct net_proto_family](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/net.h#L211).

> net_families的内容由[`sock_register`](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501)而来, 比如ipv4的`[(void)sock_register(&inet_family_ops)](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1974)`. family的值可参考[这里](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/socket.h#L175).

每一个地址族在net_families数组里面都有一项，里面的内容是 net_proto_family. 即每一种地址族都有自己的 net_proto_family，IP 地址族的 net_proto_family 定义如下，里面最重要的就是，create 函数指向 inet_create.

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1111
static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,
	.create = inet_create,
	.owner	= THIS_MODULE,
};
```

回到函数 __sock_create. 接下来, [inet_create](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L248) 会被调用.

在 inet_create 中，先会看到一个循环 list_for_each_entry_rcu. 在这里，第二个参数 type 开始起作用. 因为循环查看的是 inetsw[sock->type]. 这里的 inetsw 也是一个数组，type 作为下标，里面的内容是 struct inet_protosw，是协议，也即 inetsw 数组对于每个类型有一项，这一项里面是属于这个类型的协议.

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L126
/* The inetsw table contains everything that inet_create needs to
 * build a new socket.
 */
static struct list_head inetsw[SOCK_MAX];

// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1946
static int __init inet_init(void)
{
......
/* Register the socket-side information for inet_create. */
	for (r = &inetsw[0]; r < &inetsw[SOCK_MAX]; ++r)
		INIT_LIST_HEAD(r);

	for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
		inet_register_protosw(q);
......
}
```

inetsw 数组是在系统初始化的时候初始化的，就像上面代码里面实现的一样. 首先，一个循环会将 inetsw 数组的每一项，都初始化为一个链表. 一个 type 类型会包含多个 protocol，因而需要一个链表. 接下来一个循环，是将 [inetsw_array](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1120) 注册到 inetsw 数组里面去. inetsw_array 的定义如下，这个数组里面的内容很重要，后面会用到它们.

回到 inet_create 的 list_for_each_entry_rcu 循环中. 到这里就好理解了，这是在 inetsw 数组中，根据 type 找到属于这个类型的列表，然后依次比较列表中的 struct inet_protosw 的 protocol 是不是用户指定的 protocol；如果是，就得到了符合用户指定的 family->type->protocol 的 struct inet_protosw *answer 对象.

接下来，struct socket *sock 的 ops 成员变量，被赋值为 answer 的 ops. 对于 TCP 来讲，就是 inet_stream_ops. 后面任何用户对于这个 socket 的操作，都是通过 inet_stream_ops 进行的.

接下来，创建一个 struct sock *sk 对象. 这里比较让人困惑. socket 和 sock 看起来几乎一样，容易让人混淆，这里需要说明一下，**socket 是用于负责对上给用户提供接口，并且和文件系统关联, 而 sock，负责向下对接内核网络协议栈**.

在 sk_alloc 函数中，struct inet_protosw *answer 结构的 tcp_prot 赋值给了 struct sock *sk 的 sk_prot 成员. [tcp_prot](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L2638) 的定义如下，里面定义了很多的函数，都是 sock 之下内核协议栈的动作.

在 inet_create 函数中，接下来创建一个 struct inet_sock 结构，这个结构一开始就是 struct sock，然后扩展了一些其他的信息，剩下的代码就填充这些信息. 这一幕会经常看到，将一个结构放在另一个结构的开始位置，然后扩展一些成员，通过对于指针的强制类型转换，来访问这些成员.

socket 的创建至此结束.

## 解析 bind 函数
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501
/*
 *	Bind a name to a socket. Nothing much to do here since it's
 *	the protocol's responsibility to handle the local address.
 *
 *	We move the socket address to kernel space before we call
 *	the protocol layer (having also checked the address is ok).
 */

int __sys_bind(int fd, struct sockaddr __user *umyaddr, int addrlen)
{
	struct socket *sock;
	struct sockaddr_storage address;
	int err, fput_needed;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
		err = move_addr_to_kernel(umyaddr, addrlen, &address);
		if (!err) {
			err = security_socket_bind(sock,
						   (struct sockaddr *)&address,
						   addrlen);
			if (!err)
				err = sock->ops->bind(sock,
						      (struct sockaddr *)
						      &address, addrlen);
		}
		fput_light(sock->file, fput_needed);
	}
	return err;
}

SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
{
	return __sys_bind(fd, umyaddr, addrlen);
}
```

在 bind 中，sockfd_lookup_light 会根据 fd 文件描述符，找到 struct socket 结构. 然后将 sockaddr 从用户态拷贝到内核态，然后调用 struct socket 结构里面 ops 的 bind 函数. 根据前面创建 socket 的时候的设定，调用的是 [inet_stream_ops](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1015) 的 bind 函数，也即调用 [inet_bind](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L435).

bind 里面会[调用 sk_prot 的 get_port 函数](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L525)，也即 inet_csk_get_port 来检查端口是否冲突，是否可以绑定. 如果允许，则会设置 struct inet_sock 的本方的地址 inet_saddr 和本方的端口 inet_sport，对方的地址 inet_daddr 和对方的端口 inet_dport 都初始化为 0.

bind 的逻辑相对比较简单，就到这里了.

## 解析 listen 函数
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501
/*
 *	Perform a listen. Basically, we allow the protocol to do anything
 *	necessary for a listen, and if that works, we mark the socket as
 *	ready for listening.
 */

int __sys_listen(int fd, int backlog)
{
	struct socket *sock;
	int err, fput_needed;
	int somaxconn;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
		somaxconn = sock_net(sock->sk)->core.sysctl_somaxconn;
		if ((unsigned int)backlog > somaxconn)
			backlog = somaxconn;

		err = security_socket_listen(sock, backlog);
		if (!err)
			err = sock->ops->listen(sock, backlog);

		fput_light(sock->file, fput_needed);
	}
	return err;
}

SYSCALL_DEFINE2(listen, int, fd, int, backlog)
{
	return __sys_listen(fd, backlog);
}
```

在 listen 中，还是通过 sockfd_lookup_light，根据 fd 文件描述符，找到 struct socket 结构. 接着，我们调用 struct socket 结构里面 ops 的 listen 函数. 根据上面创建 socket 的时候的设定，调用的是 inet_stream_ops 的 listen 函数，也即调用 [inet_listen](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L196).

如果这个 socket 还不在 TCP_LISTEN 状态，会调用 [inet_csk_listen_start](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/inet_connection_sock.c#L909) 进入监听状态.

inet_csk_listen_start里面建立了一个新的结构 [inet_connection_sock](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/inet_connection_sock.h#L87)，这个结构一开始是 struct inet_sock，inet_csk 其实做了一次强制类型转换，扩大了结构.

struct inet_connection_sock 结构比较复杂. 如果打开它，就能看到处于各种状态的队列，各种超时时间、拥塞控制等字眼.

说TCP 是面向连接的，就是客户端和服务端都是有一个结构维护连接的状态，就是指这个结构.

首先，inet_connection_sock中的 icsk_accept_queue 是干什么的呢？

在 TCP 的状态里面，有一个 listen 状态，当调用 listen 函数之后，就会进入这个状态，虽然写程序的时候，一般要等待服务端调用 accept 后，等待在哪里的时候，让客户端就发起连接. 其实服务端一旦处于 listen 状态，不用 accept，客户端也能发起连接. 其实 TCP 的状态中，没有一个是否被 accept 的状态，那 accept 函数的作用是什么呢？

在内核中，为每个 Socket 维护两个队列. 一个是已经建立了连接的队列，这时候连接三次握手已经完毕，处于 established 状态；一个是还没有完全建立连接的队列，这个时候三次握手还没完成，处于 syn_rcvd 的状态. 服务端调用 accept 函数，其实是在第一个队列中拿出一个已经完成的连接进行处理. 如果还没有完成就阻塞等待. 这里的 icsk_accept_queue 就是第一个队列.

初始化完之后，将 TCP 的状态设置为 TCP_LISTEN，再次调用 get_port 判断端口是否冲突.

至此，listen 的逻辑就结束了.

## 解析 accept 函数
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501
/*
 *	For accept, we attempt to create a new socket, set up the link
 *	with the client, wake up the client, then return the new
 *	connected fd. We collect the address of the connector in kernel
 *	space and move it to user at the very end. This is unclean because
 *	we open the socket then return an error.
 *
 *	1003.1g adds the ability to recvmsg() to query connection pending
 *	status to recvmsg. We need to add that support in a way thats
 *	clean when we restructure accept also.
 */

int __sys_accept4(int fd, struct sockaddr __user *upeer_sockaddr,
		  int __user *upeer_addrlen, int flags)
{
	int ret = -EBADF;
	struct fd f;

	f = fdget(fd);
	if (f.file) {
		ret = __sys_accept4_file(f.file, 0, upeer_sockaddr,
						upeer_addrlen, flags,
						rlimit(RLIMIT_NOFILE));
		if (f.flags)
			fput(f.file);
	}

	return ret;
}

SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr,
		int __user *, upeer_addrlen, int, flags)
{
	return __sys_accept4(fd, upeer_sockaddr, upeer_addrlen, flags);
}

SYSCALL_DEFINE3(accept, int, fd, struct sockaddr __user *, upeer_sockaddr,
		int __user *, upeer_addrlen)
{
	return __sys_accept4(fd, upeer_sockaddr, upeer_addrlen, 0);
}
```

accept 函数的实现，印证了 socket 的原理中说的那样，原来的 socket 是监听 socket，这里会找到原来的 struct socket，并基于它去创建一个新的 newsock. 这才是连接 socket. 除此之外，还会创建一个新的 struct file 和 fd，并关联到 socket.

这里面还会调用 struct socket 的 sock->ops->accept，也即会调用 inet_stream_ops 的 accept 函数，也即 [inet_accept](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L732).

inet_accept 会调用 struct sock 的 sk1->sk_prot->accept，也即 tcp_prot 的 accept 函数 [inet_csk_accept](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/inet_connection_sock.c#L454).

inet_csk_accept 的实现，印证了上面讲的两个队列的逻辑. 如果 icsk_accept_queue 为空，则调用 inet_csk_wait_for_connect 进行等待；等待的时候，调用 schedule_timeout，让出 CPU，并且将进程状态设置为 TASK_INTERRUPTIBLE. 如果再次 CPU 醒来，会接着判断 icsk_accept_queue 是否为空，同时也会调用 signal_pending 看有没有信号可以处理. 一旦 icsk_accept_queue 不为空，就从 inet_csk_wait_for_connect 中返回，在队列中取出一个 struct sock 对象赋值给 newsk.

## 解析 connect 函数
什么情况下，icsk_accept_queue 才不为空呢？当然是三次握手完成才可以.

![](/misc/img/net/ab92c2afb4aafb53143c471293ccb2df.png)

三次握手一般是由客户端调用 connect 发起.

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1839
/*
 *	Attempt to connect to a socket with the server address.  The address
 *	is in user space so we verify it is OK and move it to kernel space.
 *
 *	For 1003.1g we need to add clean support for a bind to AF_UNSPEC to
 *	break bindings
 *
 *	NOTE: 1003.1g draft 6.3 is broken with respect to AX.25/NetROM and
 *	other SEQPACKET protocols that take time to connect() as it doesn't
 *	include the -EINPROGRESS status for such sockets.
 */

int __sys_connect_file(struct file *file, struct sockaddr_storage *address,
		       int addrlen, int file_flags)
{
	struct socket *sock;
	int err;

	sock = sock_from_file(file, &err);
	if (!sock)
		goto out;

	err =
	    security_socket_connect(sock, (struct sockaddr *)address, addrlen);
	if (err)
		goto out;

	err = sock->ops->connect(sock, (struct sockaddr *)address, addrlen,
				 sock->file->f_flags | file_flags);
out:
	return err;
}

int __sys_connect(int fd, struct sockaddr __user *uservaddr, int addrlen)
{
	int ret = -EBADF;
	struct fd f;

	f = fdget(fd);
	if (f.file) {
		struct sockaddr_storage address;

		ret = move_addr_to_kernel(uservaddr, addrlen, &address);
		if (!ret)
			ret = __sys_connect_file(f.file, &address, addrlen, 0);
		if (f.flags)
			fput(f.file);
	}

	return ret;
}

SYSCALL_DEFINE3(connect, int, fd, struct sockaddr __user *, uservaddr,
		int, addrlen)
{
	return __sys_connect(fd, uservaddr, addrlen);
}
```

connect 函数的实现一开始先根据 fd 文件描述符，找到 struct socket 结构. 接着，会调用 struct socket 结构里面 ops 的 connect 函数，根据前面创建 socket 的时候的设定，调用 inet_stream_ops 的 connect 函数，也即调用 [inet_stream_connect](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L716).

在 inet_stream_connect -> [__inet_stream_connect](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L606) 里面，会发现，如果 socket 处于 SS_UNCONNECTED 状态，那就调用 struct sock 的 sk->sk_prot->connect，也即 tcp_prot 的 connect 函数 [tcp_v4_connect](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L197).

在 tcp_v4_connect 函数中，[ip_route_connect](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/route.h#L306) 其实是做一个路由的选择. 为什么呢？因为三次握手马上就要发送一个 SYN 包了，这就要凑齐源地址、源端口、目标地址、目标端口. 目标地址和目标端口是服务端的，已经知道源端口是客户端随机分配的，源地址应该用哪一个呢？这时候要选择一条路由，看从哪个网卡出去，就应该填写哪个网卡的 IP 地址. 接下来，在发送 SYN 之前，先将客户端 socket 的状态设置为 TCP_SYN_SENT, 然后初始化 TCP 的 seq num，也即 write_seq，然后调用 [tcp_connect](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L3640) 进行发送.

在 tcp_connect 中，有一个新的结构 [struct tcp_sock](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/tcp.h#L142)，如果打开它，会发现它是 struct inet_connection_sock 的一个扩展，struct inet_connection_sock 在 struct tcp_sock 开头的位置，通过强制类型转换访问，故伎重演又一次. struct tcp_sock 里面维护了更多的 TCP 的状态，同样是遇到了再分析.

接下来 tcp_init_nondata_skb 初始化一个 SYN 包，tcp_transmit_skb 将 SYN 包发送出去，inet_csk_reset_xmit_timer 设置了一个 timer，如果 SYN 发送不成功，则再次发送.

发送网络包的过程，放到之后讲解. 这里姑且认为 SYN 已经发送出去了.

回到 __inet_stream_connect 函数，在调用 sk->sk_prot->connect 之后，inet_wait_for_connect 会一直等待客户端收到服务端的 ACK. 而我们知道，服务端在 accept 之后，也是在等待中. 网络包是如何接收的呢？对于解析的详细过程，还是在会在之后讲解，这里为了解析三次握手，简单的看网络包接收到 TCP 层做的部分事情.

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1737
/* thinking of making this const? Don't.
 * early_demux can change based on sysctl.
 */
static struct net_protocol tcp_protocol = {
	.early_demux	=	tcp_v4_early_demux,
	.early_demux_handler =  tcp_v4_early_demux,
	.handler	=	tcp_v4_rcv,
	.err_handler	=	tcp_v4_err,
	.no_policy	=	1,
	.netns_ok	=	1,
	.icmp_strict_tag_validation = 1,
};
```

kernel会通过 struct net_protocol 结构中的 handler 进行接收，调用的函数是 tcp_v4_rcv. 接下来的调用链为 tcp_v4_rcv->tcp_v4_do_rcv->tcp_rcv_state_process. [tcp_rcv_state_process](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L6183)，顾名思义，是用来处理接收一个网络包后引起状态变化的.

目前服务端是处于 TCP_LISTEN 状态的，而且发过来的包是 SYN，因而就有了上面的代码，调用 icsk->icsk_af_ops->conn_request 函数. struct inet_connection_sock 对应的操作是 [inet_connection_sock_af_ops](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/inet_connection_sock.h#L33)的[ipv4_specific](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L2113)，按照下面的定义，其实调用的是 [tcp_v4_conn_request](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L1455).

tcp_v4_conn_request 会调用 [tcp_conn_request](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L6609)，这个函数也比较长，里面调用了 send_synack，但实际调用的是 [tcp_v4_send_synack](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L963). 具体发送的过程不去管它，看注释能知道，这是收到了 SYN 后，回复一个 SYN-ACK，回复完毕后，服务端处于 TCP_SYN_RECV.

这个时候，轮到客户端接收网络包了. 都是 TCP 协议栈，所以过程和服务端没有太多区别，还是会走到 tcp_rcv_state_process 函数的，只不过由于客户端目前处于 TCP_SYN_SENT 状态，就进入了下面的代码分支.

tcp_rcv_synsent_state_process 会调用 [tcp_send_ack](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L3789)，发送一个 ACK-ACK，发送后客户端处于 TCP_ESTABLISHED 状态.

又轮到服务端接收网络包了，还是归 tcp_rcv_state_process 函数处理. 由于服务端目前处于状态 TCP_SYN_RECV 状态，因而又走了另外的分支. 当收到这个网络包的时候，服务端也处于 TCP_ESTABLISHED 状态，三次握手结束.

## 总结
![](/misc/img/net/c028381cf45d65d3f148e57408d26bd8.png)

其实socket的系统调用规律就都一样了：
- bind 第一层调用 inet_stream_ops 的 inet_bind 函数，第二层调用 tcp_prot 的 inet_csk_get_port 函数
- listen 第一层调用 inet_stream_ops 的 inet_listen 函数，第二层调用 tcp_prot 的 inet_csk_get_port 函数
- accept 第一层调用 inet_stream_ops 的 inet_accept 函数，第二层调用 tcp_prot 的 inet_csk_accept 函数
- connect 第一层调用 inet_stream_ops 的 inet_stream_connect 函数，第二层调用 tcp_prot 的 tcp_v4_connect 函数

## 解析 socket 的 Write 操作
socket 对于用户来讲，是一个文件一样的存在，拥有一个文件描述符. 因而对于网络包的发送，可以使用对于 socket 文件的写入系统调用，也就是 write 系统调用. write 系统调用对于一个文件描述符的操作，大致过程都是类似的. 对于每一个打开的文件都有一个 struct file 结构，write 系统调用会最终调用 stuct file 结构指向的 file_operations 操作. 对于 socket 来讲，它的 file_operations 定义如下：
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L149
/*
 *	Socket files have a set of 'special' operations as well as the generic file ones. These don't appear
 *	in the operation structures but are done directly via the socketcall() multiplexor.
 */

static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.sendpage =	sock_sendpage,
	.splice_write = generic_splice_sendpage,
	.splice_read =	sock_splice_read,
	.show_fdinfo =	sock_show_fdinfo,
};
```

按照文件系统的写入流程，调用的是 [sock_write_iter](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L982).

在 sock_write_iter 中，通过 VFS 中的 struct file，将创建好的 socket 结构拿出来，然后调用 sock_sendmsg. 而 sock_sendmsg 会调用 [sock_sendmsg_nosec](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L650).

sock_sendmsg_nosec里调用了 socket 的 ops 的 sendmsg，据 inet_stream_ops 的定义，这里调用的是 [inet_sendmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L807).

这里面，从 socket 结构中，可以得到更底层的 sock 结构，然后调用 sk_prot 的 sendmsg 方法即tcp_sendmsg.

## 解析 tcp_sendmsg 函数
根据 tcp_prot 的定义，实际调用的是 [tcp_sendmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L1436).

tcp_sendmsg->[tcp_sendmsg_locked](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L1185), tcp_sendmsg_locked 的实现还是很复杂的，这里面做了这样几件事情.

msg 是用户要写入的数据，这个数据要拷贝到内核协议栈里面去发送；在内核协议栈里面，网络包的数据都是由 struct sk_buff 维护的，因而第一件事情就是找到一个空闲的内存空间，将用户要写入的数据，拷贝到 struct sk_buff 的管辖范围内. 而第二件事情就是发送 [struct sk_buff](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/skbuff.h#L711).

在 tcp_sendmsg_locked 中，首先通过强制类型转换，将 sock 结构转换为 struct tcp_sock，这个是维护 TCP 连接状态的重要数据结构.

接下来是 tcp_sendmsg_locked 的第一件事情，把数据拷贝到 struct sk_buff. 先声明一个变量 copied，初始化为 0，这表示拷贝了多少数据. 紧接着是一个循环，[while (msg_data_left(msg))](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L1271)，也即如果用户的数据没有发送完毕，就一直循环. 循环里声明了一个 copy 变量，表示这次拷贝的数值，在循环的最后有 copied += copy，将每次拷贝的数量都加起来. 这里只需要看一次循环做了哪些事情:
1. 第一步，tcp_write_queue_tail 从 TCP 写入队列 sk_write_queue 中拿出最后一个 struct sk_buff，在这个写入队列中排满了要发送的 struct sk_buff，为什么要拿最后一个呢？这里面只有最后一个，可能会因为上次用户给的数据太少，而没有填满.
1. 第二步，tcp_send_mss 会计算 MSS，也即 Max Segment Size. 这是什么呢？这个意思是说，在网络上传输的网络包的大小是有限制的，而这个限制在最底层开始就有. MTU（Maximum Transmission Unit，最大传输单元）是二层的一个定义. 以以太网为例，MTU 为 1500 个 Byte，前面有 6 个 Byte 的目标 MAC 地址，6 个 Byte 的源 MAC 地址，2 个 Byte 的类型，后面有 4 个 Byte 的 CRC 校验，共 1518 个 Byte. 在 IP 层，一个 IP 数据报在以太网中传输，如果它的长度大于该 MTU 值，就要进行分片传输.

在 TCP 层有个 MSS（Maximum Segment Size，最大分段大小），等于 MTU 减去 IP 头，再减去 TCP 头. 也就是，在不分片的情况下，TCP 里面放的最大内容. 在这里，max 是 struct sk_buff 的最大数据长度，skb->len 是当前已经占用的 skb 的数据长度，相减得到当前 skb 的剩余数据空间.

1. 第三步，如果 copy 小于 0，说明最后一个 struct sk_buff 已经没地方存放了，需要调用 sk_stream_alloc_skb，重新分配 struct sk_buff，然后调用 skb_entail，将新分配的 sk_buff 放到队列尾部.

struct sk_buff 是存储网络包的重要的数据结构，在应用层数据包叫 data，在 TCP 层我们称为 segment，在 IP 层叫 packet，在数据链路层称为 frame. 在 struct sk_buff，首先是一个链表，将 struct sk_buff 结构串起来.

接下来，struct sk_buff 从中的 headers_start 开始，到 headers_end 结束，里面都是各层次的头的位置. 这里面有二层的 mac_header、三层的 network_header 和四层的 transport_header.

最后几项， head 指向分配的内存块起始地址. data 这个指针指向的位置是可变的. 它有可能随着报文所处的层次而变动. 当接收报文时，从网卡驱动开始，通过协议栈层层往上传送数据报，通过增加 skb->data 的值，来逐步剥离协议首部. 而要发送报文时，各协议会创建 sk_buff{}，在经过各下层协议时，通过减少 skb->data 的值来增加协议首部. tail 指向数据的结尾，end 指向分配的内存块的结束地址.

要分配这样一个结构，sk_stream_alloc_skb 会最终调用到 __alloc_skb. 在这个函数里面，除了分配一个 sk_buff 结构之外，还要分配 sk_buff 指向的数据区域. 这段数据区域分为下面这几个部分:
  1. 第一部分是连续的数据区域
  1. 紧接着是第二部分，一个 struct skb_shared_info 结构. 这个结构是对于网络包发送过程的一个优化，因为传输层之上就是应用层了. 按照 TCP 的定义，应用层感受不到下面的网络层的 IP 包是一个个独立的包的存在的. 反正就是一个流，往里写就是了，可能一下子写多了，超过了一个 IP 包的承载能力，就会出现上面 MSS 的定义，拆分成一个个的 Segment 放在一个个的 IP 包里面，也可能一次写一点，一次写一点，这样数据是分散的，在 IP 层还要通过内存拷贝合成一个 IP 包. 为了减少内存拷贝的代价，有的网络设备支持分散聚合（Scatter/Gather）I/O，顾名思义，就是 IP 层没必要通过内存拷贝进行聚合，让散的数据零散的放在原处，在设备层进行聚合. 如果使用这种模式，网络包的数据就不会放在连续的数据区域，而是放在 struct skb_shared_info 结构里面指向的离散数据，skb_shared_info 的成员变量 skb_frag_t frags[MAX_SKB_FRAGS]，会指向一个数组的页面，就不能保证连续了.

![](/misc/img/net/9ad34c3c748978f915027d5085a858b8.png)

1. 于是就有了第四步. 在注释 [/* Where to copy to? */](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L1314) 后面有个 if-else 分支. if 分支就是 skb_add_data_nocache 将数据拷贝到连续的数据区域. else 分支就是 skb_copy_to_page_nocache 将数据拷贝到 struct skb_shared_info 结构指向的不需要连续的页面区域.
1. 第五步，就是要发送网络包了. 第一种情况是积累的数据报数目太多了，因而需要通过调用 __tcp_push_pending_frames 发送网络包. 第二种情况是，这是第一个网络包，需要马上发送，调用 tcp_push_one. 无论 __tcp_push_pending_frames 还是 tcp_push_one，都会调用 [tcp_write_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L2426) 发送网络包.

至此，tcp_sendmsg 解析完了.

## 解析 tcp_write_xmit 函数
接下来看，[tcp_write_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L2426) 是如何发送网络包的.

这里面主要的逻辑是一个循环，用来处理发送队列，只要队列不空，就会发送.

在一个循环中，涉及 TCP 层的很多传输算法，需要一一解析.

第一个概念是 TSO（TCP Segmentation Offload）. 如果发送的网络包非常大，就像上面说的一样，要进行分段. 分段这个事情可以由协议栈代码在内核做，但是缺点是比较费 CPU，另一种方式是延迟到硬件网卡去做，需要网卡支持对大数据包进行自动分段，可以降低 CPU 负载.

在代码中，tcp_init_tso_segs 会调用 tcp_set_skb_tso_segs. 这里面有这样的语句：[DIV_ROUND_UP(skb->len, mss_now)](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L1285). 也就是 sk_buff 的长度除以 mss_now，应该分成几个段. 如果算出来要分成多个段，接下来就是要看，是在这里（协议栈的代码里面）分好，还是等待到了底层网卡再分.

于是，调用函数 tcp_mss_split_point，开始计算切分的 limit。这里面会计算 max_len = mss_now * max_segs，根据现在不切分来计算 limit，所以下一步的判断中，大部分情况下 tso_fragment 不会被调用，等待到了底层网卡来切分.

第二个概念是拥塞窗口的概念（cwnd，congestion window），也就是说为了避免拼命发包，把网络塞满了，定义一个窗口的概念，在这个窗口之内的才能发送，超过这个窗口的就不能发送，来控制发送的频率. 那窗口大小是多少呢？就是遵循下面这个著名的拥塞窗口变化图.

![](/misc/img/net/404a6c5041452c0641ae3cba5319dc1f.png)

一开始的窗口只有一个 mss 大小叫作 slow start（慢启动）. 一开始的增长速度的很快的，翻倍增长, 一旦到达一个临界值 ssthresh，就变成线性增长，就称为拥塞避免. 什么时候算真正拥塞呢？就是出现了丢包. 一旦丢包，一种方法是马上降回到一个 mss，然后重复先翻倍再线性对的过程. 如果觉得太过激进，也可以有第二种方法，就是降到当前 cwnd 的一半，然后进行线性增长.

在代码中，tcp_cwnd_test 会将当前的 snd_cwnd，减去已经在窗口里面尚未发送完毕的网络包，那就是剩下的窗口大小 cwnd_quota，也即就能发送这么多了.

第三个概念就是接收窗口rwnd 的概念（receive window），也叫滑动窗口. 如果说拥塞窗口是为了怕把网络塞满，在出现丢包的时候减少发送速度，那么滑动窗口就是为了怕把接收方塞满，而控制发送速度.

![](/misc/img/net/9791e2f9ff63a9d8f849df7cd55fe965.png)

滑动窗口，其实就是接收方告诉发送方自己的网络包的接收能力，超过这个能力，就收不了了. 因为滑动窗口的存在，将发送方的缓存分成了四个部分:
1. 第一部分：发送了并且已经确认的. 这部分是已经发送完毕的网络包，这部分没有用了，可以回收.
1. 第二部分：发送了但尚未确认的. 这部分，发送方要等待，万一发送不成功，还要重新发送，所以不能删除.
1. 第三部分：没有发送，但是已经等待发送的. 这部分是接收方空闲的能力，可以马上发送，接收方收得了.
1. 第四部分：没有发送，并且暂时还不会发送的. 这部分已经超过了接收方的接收能力，再发送接收方就收不了了.

![](/misc/img/net/b62eea403e665bb196dceba571392531.png)

因为滑动窗口的存在，接收方的缓存也要分成了三个部分:
1. 第一部分：接受并且确认过的任务. 这部分完全接收成功了，可以交给应用层了
1. 第二部分：还没接收，但是马上就能接收的任务. 这部分有的网络包到达了，但是还没确认，不算完全完毕，有的还没有到达，那就是接收方能够接受的最大的网络包数量
1. 第三部分：还没接收，也没法接收的任务. 这部分已经超出接收方能力

在网络包的交互过程中，接收方会将第二部分的大小，作为 AdvertisedWindow 发送给发送方，发送方就可以根据它来调整发送速度了. 在 tcp_snd_wnd_test 函数中，会判断 sk_buff 中的 end_seq 和 tcp_wnd_end(tp) 之间的关系，也即这个 sk_buff 是否在滑动窗口的允许范围之内. 如果不在范围内，说明发送要受限制了，就要把 is_rwnd_limited 设置为 true.

接下来，[tcp_mss_split_point](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L1826) 函数要被调用了.

tcp_mss_split_point里面除了会判断上面讲的，是否会因为超出 mss 而分段，还会判断另一个条件，就是是否在滑动窗口的运行范围之内，如果小于窗口的大小，也需要分段，也即需要调用 tso_fragment.

tcp_write_xmit在一个循环的最后，是调用 [tcp_transmit_skb](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L1251)，真的去发送一个网络包.

tcp_transmit_skb -> [__tcp_transmit_skb](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L1078), __tcp_transmit_skb 这个函数比较长，主要做了两件事情，第一件事情就是填充 TCP 头.

tcp header里面有源端口，设置为 inet_sport，有目标端口，设置为 inet_dport；有序列号，设置为 tcb->seq；有确认序列号，设置为 tp->rcv_nxt. 把所有的 flags 设置为 tcb->tcp_flags. 设置选项为 opts. 设置窗口大小为 tp->rcv_wnd.

全部设置完毕之后，就会调用 icsk_af_ops 的 queue_xmit 方法，icsk_af_ops 指向 [ipv4_specific](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L2113)，也即调用的是 ip_queue_xmit 函数.

## 解析 ip_queue_xmit 函数
从 [ip_queue_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/ip.h#L234) 函数开始，就进入 IP 层的发送逻辑了.

ip_queue_xmit -> [__ip_queue_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L451).

在 __ip_queue_xmit 中，也即 IP 层的发送函数里面，有三部分逻辑.

第一部分，选取路由，也即要发送这个包应该从哪个网卡出去. 这件事情主要由 ip_route_output_ports 函数完成.

接下来的调用链为：ip_route_output_ports->ip_route_output_flow->__ip_route_output_key->ip_route_output_key_hash->[ip_route_output_key_hash_rcu](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/route.c#L2502).

ip_route_output_key_hash_rcu 先会调用 fib_lookup. FIB 全称是 Forwarding Information Base，转发信息表, 其实就是咱们常说的路由表.

路由表可以有多个，一般会有一个主表，RT_TABLE_MAIN. 然后 fib_table_lookup 函数在这个表里面进行查找.

路由表是一个什么样的结构呢？路由就是在 Linux 服务器上的路由表里面配置的一条一条规则. 这些规则大概是这样的：想访问某个网段，从某个网卡出去，下一跳是某个 IP. 可以通过 ip route 命令查看路由表.

fib_table_lookup 的代码逻辑比较复杂，好在注释比较清楚. 因为路由表要按照前缀进行查询，希望找到最长匹配的那一个，例如 192.168.2.0/24 和 192.168.0.0/16 都能匹配 192.168.2.100/24, 但是，应该使用 192.168.2.0/24 的这一条.

为了更方面的做这个事情，使用了 Trie 树这种结构. 比如有一系列的字符串：{bcs#, badge#, baby#, back#, badger#, badness#}. 之所以每个字符串都加上 #，是希望不要一个字符串成为另外一个字符串的前缀. 然后我们把它们放在 Trie 树中，如下图所示：
![](/misc/img/net/3f0a99cf1c47afcd0bd740c4b7802511.png)

对于将 IP 地址转成二进制放入 trie 树，也是同样的道理，可以很快进行路由的查询. 找到了路由，就知道了应该从哪个网卡发出去. 然后，ip_route_output_key_hash_rcu 会调用 __mkroute_output，创建一个 struct rtable，表示找到的路由表项. 这个结构是由 [rt_dst_alloc](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/route.c#L1620) 函数分配的.

最终返回 struct rtable 实例，第一部分也就完成了.

第二部分，就是准备 IP 层的头，往里面填充内容.

在ip header里面，服务类型设置为 tos，标识位里面设置是否允许分片 frag_off. 如果不允许，而遇到 MTU 太小过不去的情况，就发送 ICMP 报错. TTL 是这个包的存活时间，为了防止一个 IP 包迷路以后一直存活下去，每经过一个路由器 TTL 都减一，减为零则“死去”. 设置 protocol，指的是更上层的协议，这里是 TCP. 源地址和目标地址由 ip_copy_addrs 设置. 最后，设置 options.

第三部分，就是调用 [ip_local_out](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L119) 发送 IP 包.

ip_local_out -> [__ip_local_out](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L98), 然后里面调用了 nf_hook. 这是什么呢？nf 的意思是 Netfilter，这是 Linux 内核的一个机制，用于在网络发送和转发的关键节点上加上 hook 函数，这些函数可以截获数据包，对数据包进行干预. 一个著名的实现，就是内核模块 ip_tables. 在用户态，还有一个客户端程序 iptables，用命令行来干预内核的规则.

![](/misc/img/net/75c8257049eed99499e802fcc2eacf4d.png)

iptables 有表和链的概念，最终要的是两个表.

filter 表处理过滤功能，主要包含以下三个链:
- INPUT 链：过滤所有目标地址是本机的数据包
- FORWARD 链：过滤所有路过本机的数据包
- OUTPUT 链：过滤所有由本机产生的数据包

nat 表主要处理网络地址转换，可以进行 SNAT（改变源地址）、DNAT（改变目标地址），包含以下三个链:
- PREROUTING 链：可以在数据包到达时改变目标地址
- OUTPUT 链：可以改变本地产生的数据包的目标地址
- POSTROUTING 链：在数据包离开时改变数据包的源地址

![](/misc/img/net/765e5431fe4b17f62b1b5712cc82abda.png)

在这里，网络包马上就要发出去了，因而是 NF_INET_LOCAL_OUT，也即 ouput 链，如果用户曾经在 iptables 里面写过某些规则，就会在 nf_hook 这个函数里面起作用.

ip_local_out 再调用 [dst_output](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/dst.h#L433)，就是真正的发送数据.

这里调用的就是 struct rtable 成员 dst 的 ouput 函数. 在 rt_dst_alloc 中，可以看到，output 函数指向的是 [ip_output](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L421).

在 ip_output 里面，又看到了熟悉的 NF_HOOK. 这一次是 NF_INET_POST_ROUTING，也即 POSTROUTING 链，处理完之后，调用 [ip_finish_output](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L309).

## 解析 ip_finish_output 函数
从 ip_finish_output 函数开始，发送网络包的逻辑由第三层到达第二层. ip_finish_output 最终调用 [ip_finish_output2](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L185).

在 ip_finish_output2 中，先找到 struct rtable 路由表里面的下一跳，下一跳一定和本机在同一个局域网中，可以通过二层进行通信，因而通过 ip_neigh_for_gw -> __ipv4_neigh_lookup_noref，查找如何通过二层访问下一跳.

__ipv4_neigh_lookup_noref 是从本地的 [ARP 表](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/arp.c#L151)中查找下一跳的 MAC 地址.

如果在 ARP 表中没有找到相应的项，则调用 [__neigh_create](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/neighbour.c#L668) 进行创建.

__neigh_create 先调用 neigh_alloc，创建一个 struct neighbour 结构，用于维护 MAC 地址和 ARP 相关的信息. 这个名字也很好理解，大家都是在一个局域网里面，可以通过 MAC 地址访问到，当然是邻居了.

在 neigh_alloc 中，先分配一个 struct neighbour 结构并且初始化. 这里面比较重要的有两个成员，一个是 arp_queue，所以上层想通过 ARP 获取 MAC 地址的任务，都放在这个队列里面. 另一个是 timer 定时器，设置成过一段时间就调用 neigh_timer_handler，来处理这些 ARP 任务.

__neigh_create 然后调用了 arp_tbl 的 constructor 函数，也即调用了 [arp_constructor](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/arp.c#L220)，在这里面定义了 ARP 的操作 [arp_hh_ops](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/arp.c#L137).

__neigh_create 最后是将创建的 struct neighbour 结构放入一个哈希表，从里面的代码逻辑比较容易看出，这是一个数组加链表的链式哈希表，先计算出哈希值 hash_val，得到相应的链表，然后循环这个链表找到对应的项，如果找不到就在最后插入一项.

回到 ip_finish_output2，在 __neigh_create 之后，会调用 [neigh_output](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/neighbour.h#L501) 发送网络包.

按照上面对于 struct neighbour 的操作函数 arp_hh_ops 的定义，output 调用的是 [neigh_resolve_output](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/neighbour.c#L1469).

在 neigh_resolve_output 里面，首先 neigh_event_send 触发一个事件，看能否激活 ARP.

在 __neigh_event_send 中，激活 ARP 分两种情况.

第一种情况是马上激活，也即 immediate_probe. 另一种情况是延迟激活则仅仅设置一个 timer, 然后将 ARP 包放在 arp_queue 上. 如果马上激活，就直接调用 neigh_probe；如果延迟激活，则定时器到了就会触发 neigh_timer_handler，在这里面还是会调用 [neigh_probe](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/neighbour.c#L1000). 就来看 neigh_probe 的实现，在这里面会从 arp_queue 中拿出 ARP 包来，然后调用 struct neighbour 的 solicit 操作.

按照上面对于 struct neighbour 的操作函数 arp_hh_ops 的定义，solicit 调用的是 arp_solicit，在这里可以找到对应于 [arp_send_dst](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/arp.c#L298) 的调用，创建并发送一个 arp 包，得到结果放在 struct dst_entry 里面.

回到 neigh_resolve_output 中，当 ARP 发送完毕，就可以调用 [dev_queue_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L4162) 发送二层网络包了.

就像在学硬盘块设备的时候学过，每个块设备都有队列，用于将内核的数据放到队列里面，然后设备驱动从队列里面取出后，将数据根据具体设备的特性发送给设备。网络设备也是类似的，对于发送来说，有一个发送队列 struct netdev_queue *txq. 这里还有另一个变量叫做 struct Qdisc，这个是什么呢？如果在一台 Linux 机器上运行 ip addr，我们能看到对于一个网卡，都有下面的输出.

这里面有个关键字 qdisc pfifo_fast 是什么意思呢？qdisc 全称是 queueing discipline，中文叫排队规则. 内核如果需要通过某个网络接口发送数据包，都需要按照为这个接口配置的 qdisc（排队规则）把数据包加入队列.

最简单的 qdisc 是 pfifo，它不对进入的数据包做任何的处理，数据包采用先入先出的方式通过队列. pfifo_fast 稍微复杂一些，它的队列包括三个波段（band）. 在每个波段里面，使用先进先出规则.

三个波段的优先级也不相同. band 0 的优先级最高，band 2 的最低. 如果 band 0 里面有数据包，系统就不会处理 band 1 里面的数据包，band 1 和 band 2 之间也是一样.

数据包是按照服务类型（Type of Service，TOS）被分配到三个波段里面的. TOS 是 IP 头里面的一个字段，代表了当前的包是高优先级的，还是低优先级的.

pfifo_fast 分为三个先入先出的队列，称为三个 Band. 根据网络包里面的 TOS，看这个包到底应该进入哪个队列. TOS 总共四位，每一位表示的意思不同，总共十六种类型.

![](/misc/img/net/ab6af2f9e1a64868636080a05cfde0d9.png)

通过命令行 tc qdisc show dev eth0，可以输出结果 priomap，也是十六个数字. 在 0 到 2 之间，和 TOS 的十六种类型对应起来. 不同的 TOS 对应不同的队列。其中 Band 0 优先级最高，发送完毕后才轮到 Band 1 发送，最后才是 Band 2.

接下来，[__dev_xmit_skb](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L3734) 开始进行网络包发送.

__dev_xmit_skb 会将请求放入队列，然后调用 __qdisc_run 处理队列中的数据. qdisc_restart 用于数据的发送. 根据注释中的说法，qdisc 的另一个功能是用于控制网络包的发送速度，因而如果超过速度，就需要重新调度，则会调用 __netif_schedule.

__netif_schedule 会调用 __netif_reschedule，发起一个软中断 NET_TX_SOFTIRQ. 学设备驱动程序的时候学过，设备驱动程序处理中断，分两个过程，一个是屏蔽中断的关键处理逻辑，一个是延迟处理逻辑. 当时说工作队列是延迟处理逻辑的处理方案，软中断也是一种方案.

在系统初始化的时候，会定义软中断的处理函数. 例如，NET_TX_SOFTIRQ 的处理函数是 net_tx_action，用于发送网络包. 还有一个 NET_RX_SOFTIRQ 的处理函数是 net_rx_action，用于接收网络包.
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L10616
/*
 *	Initialize the DEV module. At boot time this walks the device list and
 *	unhooks any devices that fail to initialise (normally hardware not
 *	present) and leaves us with a valid list of present and active devices.
 *
 */

/*
 *       This is called single threaded during boot, so no need
 *       to take the rtnl semaphore.
 */
static int __init net_dev_init(void)
...
	open_softirq(NET_TX_SOFTIRQ, net_tx_action);
	open_softirq(NET_RX_SOFTIRQ, net_rx_action);
...
}
```

这里先来解析一下 [net_tx_action](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L4838).

会发现，net_tx_action 还是调用了 qdisc_run，还是会调用 __qdisc_run，然后调用 [qdisc_restart](https://elixir.bootlin.com/linux/v5.8.1/source/net/sched/sch_generic.c#L357) 发送网络包.

再看qdisc_restart 的实现. qdisc_restart 将网络包从 Qdisc 的队列中拿下来，然后调用 [sch_direct_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/sched/sch_generic.c#L285) 进行发送.

在 sch_direct_xmit 中，调用 [dev_hard_start_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L3562) 进行发送，如果发送不成功，会返回 NETDEV_TX_BUSY. 这说明网络卡很忙，于是就调用 dev_requeue_skb，重新放入队列.

在 dev_hard_start_xmit 中，是一个 while 循环. 每次在队列中取出一个 sk_buff，调用 xmit_one 发送. 接下来的调用链为：xmit_one->netdev_start_xmit->__netdev_start_xmit.

这个时候，已经到了设备驱动层了. kernel里能看到，[drivers/net/ethernet/intel/ixgb/ixgb_main.c 里面有对于这个网卡的操作的定义](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L335).

在ixgb_main.c里面，可以找到对于 ndo_start_xmit 的定义，调用 [ixgb_xmit_frame](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L1478).

在 ixgb_xmit_frame 中，会得到这个网卡对应的适配器，然后将其放入硬件网卡的队列中.

至此，整个发送才算结束.

## 总结时刻
![](/misc/img/net/79cc42f3163d159a66e163c006d9f36f.png)

这个过程分成几个层次:
1. VFS 层：write 系统调用找到 struct file，根据里面的 file_operations 的定义，调用 sock_write_iter 函数. sock_write_iter 函数调用 sock_sendmsg 函数.
1. Socket 层：从 struct file 里面的 private_data 得到 struct socket，根据里面 ops 的定义，调用 inet_sendmsg 函数.
1. Sock 层：从 struct socket 里面的 sk 得到 struct sock，根据里面 sk_prot 的定义，调用 tcp_sendmsg 函数.
1. TCP 层：tcp_sendmsg 函数会调用 tcp_write_xmit 函数，tcp_write_xmit 函数会调用 tcp_transmit_skb，在这里实现了 TCP 层面向连接的逻辑
1. IP 层：扩展 struct sock，得到 struct inet_connection_sock，根据里面 icsk_af_ops 的定义，调用 ip_queue_xmit 函数.
1. IP 层：ip_route_output_ports 函数里面会调用 fib_lookup 查找路由表. FIB 全称是 Forwarding Information Base，转发信息表，也就是路由表
1. 在 IP 层里面要做的另一个事情是填写 IP 层的头
1. 在 IP 层还要做的一件事情就是通过 iptables 规则
1. MAC 层：IP 层调用 ip_finish_output 进行 MAC 层
1. MAC 层需要 ARP 获得 MAC 地址，因而要调用 ___neigh_lookup_noref 查找属于同一个网段的邻居，它会调用 neigh_probe 发送 ARP
1. 有了 MAC 地址，就可以调用 dev_queue_xmit 发送二层网络包了，它会调用 __dev_xmit_skb 会将请求放入队列
1. 设备层：网络包的发送会触发一个软中断 NET_TX_SOFTIRQ 来处理队列中的数据. 这个软中断的处理函数是 net_tx_action
1. 在软中断处理函数中，会将网络包从队列上拿下来，调用网络设备的传输函数 ixgb_xmit_frame，将网络包发到设备的队列上去

## 解析接收网络包的过程
### 设备驱动层
网卡作为一个硬件，接收到网络包，应该怎么通知操作系统，这个网络包到达了呢？可以触发一个中断. 但是这里有个问题，就是网络包的到来，往往是很难预期的. 网络吞吐量比较大的时候，网络包的到达会十分频繁. 这个时候，如果非常频繁地去触发中断，想想就觉得是个灾难.

比如说，CPU 正在做某个事情，一些网络包来了，触发了中断，CPU 停下手里的事情，去处理这些网络包，处理完毕按照中断处理的逻辑，应该回去继续处理其他事情. 这个时候，另一些网络包又来了，又触发了中断，CPU 手里的事情还没捂热，又要停下来去处理网络包. 能不能大家要来的一起来，把网络包好好处理一把，然后再回去集中处理其他事情呢？

网络包能不能一起来，这个我们没法儿控制，但是可以有一种机制，就是当一些网络包到来触发了中断，内核处理完这些网络包之后，可以先进入主动轮询 poll 网卡的方式，主动去接收到来的网络包. 如果一直有，就一直处理，等处理告一段落，就返回干其他的事情. 当再有下一批网络包到来的时候，再中断，再轮询 poll. 这样就会大大减少中断的数量，提升网络处理的效率，这种处理方式称为 NAPI.

在linux系统中，其网卡驱动大多通过PCI总线与系统相连，同时，内核对于所有PCI总线上的设备是通过PCI子系统来进行管理，通过PCI子系统提供各种PCI设备驱动程序共同的所有通用功能. 因此，作为linux系统中的PCI网卡设备，其加载的流程必然分为两个部分：作为PCI设备向PCI子系统注册；作为网卡设备向网络子系统注册.

以之前发送网络包时的网卡 drivers/net/ethernet/intel/ixgb/ixgb_main.c 为例子，来进行解析.

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L95
static struct pci_driver ixgb_driver = {
	.name     = ixgb_driver_name,
	.id_table = ixgb_pci_tbl,
	.probe    = ixgb_probe,
	.remove   = ixgb_remove,
	.err_handler = &ixgb_err_handler
};


MODULE_AUTHOR("Intel Corporation, <linux.nics@intel.com>");
MODULE_DESCRIPTION("Intel(R) PRO/10GbE Network Driver");
MODULE_LICENSE("GPL v2");
MODULE_VERSION(DRV_VERSION);

#define DEFAULT_MSG_ENABLE (NETIF_MSG_DRV|NETIF_MSG_PROBE|NETIF_MSG_LINK)
static int debug = -1;
module_param(debug, int, 0);
MODULE_PARM_DESC(debug, "Debug level (0=none,...,16=all)");

/**
 * ixgb_init_module - Driver Registration Routine
 *
 * ixgb_init_module is the first routine called when the driver is
 * loaded. All it does is register with the PCI subsystem.
 **/

static int __init
ixgb_init_module(void)
{
	pr_info("%s - version %s\n", ixgb_driver_string, ixgb_driver_version);
	pr_info("%s\n", ixgb_copyright);

	return pci_register_driver(&ixgb_driver);
}

module_init(ixgb_init_module);

// https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/pci.h#L1374
/* pci_register_driver() must be a macro so KBUILD_MODNAME can be expanded */
#define pci_register_driver(driver)		\
	__pci_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)

// https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L1400
/**
 * __pci_register_driver - register a new pci driver
 * @drv: the driver structure to register
 * @owner: owner module of drv
 * @mod_name: module name string
 *
 * Adds the driver structure to the list of registered drivers.
 * Returns a negative value on error, otherwise 0.
 * If no error occurred, the driver remains registered even if
 * no device was claimed during registration.
 */
int __pci_register_driver(struct pci_driver *drv, struct module *owner,
			  const char *mod_name)
{
	/* initialize common driver fields */
	drv->driver.name = drv->name;
	drv->driver.bus = &pci_bus_type;
	drv->driver.owner = owner;
	drv->driver.mod_name = mod_name;
	drv->driver.groups = drv->groups;

	spin_lock_init(&drv->dynids.lock);
	INIT_LIST_HEAD(&drv->dynids.list);

	/* register with core */
	return driver_register(&drv->driver);
}
EXPORT_SYMBOL(__pci_register_driver);
```

每个PCI设备驱动都有一个[pci_driver](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/pci.h#L848)变量，它描述了一个PCI驱动的信息. 每个PCI设备都由一组参数唯一地标识，这些参数保存在结构体pci_driver的[pci_device_id](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/mod_devicetable.h#L38)中.

ixgb_init_module ->pci_register_driver->__pci_register_driver->[driver_register](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/driver.c#L147)->[bus_add_driver](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/bus.c#L594) -> [driver_attach](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L1065) -> [bus_for_each_dev#fn(dev, data)](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/bus.c#L292) ->[__driver_attach](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L1005) -> [device_driver_attach](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L963)->[driver_probe_device](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L683)->[really_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L466)->[`dev->bus->probe(dev)`](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L466) , 因为`__pci_register_driver()`中设置了[`drv->driver.bus = &pci_bus_type;`](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L1620)->[pci_device_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L413)->[__pci_device_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L376)->[pci_call_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L332)->[ local_pci_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L287)

在网卡驱动程序初始化的时候，会调用 ixgb_init_module，注册一个驱动 ixgb_driver，并且调用它的 probe 函数 [ixgb_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L363).

在 ixgb_probe 中，会创建一个 [struct net_device](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/netdevice.h#L1843) 表示这个网络设备，并且 [netif_napi_add](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L6597) 函数为这个网络设备注册一个轮询 poll 函数 [ixgb_clean](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L1757)，将来一旦出现网络包的时候，就是要通过它来轮询了. 当一个网卡被激活的时候，会调用函数 [ixgb_open](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L598)->[ixgb_up](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L176)，在这里面注册一个硬件的中断处理函数[ixgb_intr](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L1725).

如果一个网络包到来，触发了硬件中断，就会调用 ixgb_intr，这里面会调用 [__napi_schedule](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L6278).

__napi_schedule 是处于中断处理的关键部分，在它被调用的时候，中断是暂时关闭的，但是处理网络包是个复杂的过程，需要到延迟处理部分，所以 ____napi_schedule 将当前设备放到 struct softnet_data 结构的 poll_list 里面，说明在延迟处理部分可以接着处理这个 poll_list 里面的网络设备. 然后 ____napi_schedule 触发一个软中断 NET_RX_SOFTIRQ，通过软中断触发中断处理的延迟处理部分，也是常用的手段. 软中断 NET_RX_SOFTIRQ 对应的中断处理函数是 [net_rx_action](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L6729).

在 net_rx_action 中，会得到 [struct softnet_data](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/netdevice.h#L3095) 结构，这个结构在发送的时候也遇到过. 当时它的 output_queue 用于网络包的发送，这里的 poll_list 用于网络包的接收.

在 net_rx_action 中，接下来是一个循环，在 poll_list 里面取出网络包到达的设备，然后调用 napi_poll 来轮询这些设备，napi_poll 会调用最初设备初始化的时候，注册的 poll 函数，对于 ixgb_driver，对应的函数是 ixgb_clean. ixgb_clean 会调用 [ixgb_clean_rx_irq](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L1933).

在网络设备的驱动层，有一个用于接收网络包的 rx_ring. 它是一个环，从网卡硬件接收的包会放在这个环里面. 这个环里面的 buffer_info[]是一个数组，存放的是网络包的内容. i 和 j 是这个数组的下标，在 ixgb_clean_rx_irq 里面的 while 循环中，依次处理环里面的数据. 在ixgb_clean_rx_irq里面，看到了 i 和 j 加一之后，如果超过了数组的大小，就跳回下标 0，就说明这是一个环. ixgb_check_copybreak 函数将 buffer_info 里面的内容，拷贝到 struct sk_buff *skb，从而可以作为一个网络包进行后续的处理，然后调用 netif_receive_skb.

## 网络协议栈的二层逻辑
从 netif_receive_skb 函数开始，我们就进入了内核的网络协议栈.

接下来的调用链为：netif_receive_skb->netif_receive_skb_internal->__netif_receive_skb->[__netif_receive_skb_core](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L5074). 在 __netif_receive_skb_core 中，先是处理了二层的一些逻辑, 例如，对于 VLAN 的处理，接下来要想办法交给第三层.

在网络包 struct sk_buff 里面，二层的头里面有一个 protocol，表示里面一层，也即三层是什么协议. deliver_ptype_list_skb 在一个协议列表中逐个匹配. 如果能够匹配到，就返回.

这些协议的注册在网络协议栈初始化的时候， [inet_init 函数调用 dev_add_pack(&ip_packet_type)](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L2055)，添加了 IP 协议. 协议被放在一个链表里面.

假设这个时候的网络包是一个 IP 包，则在这个链表里面一定能够找到 ip_packet_type，在 __netif_receive_skb_core 中会调用 [ip_packet_type 的 func 函数](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1940)即[ip_rcv](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_input.c#L530).

## 网络协议栈的 IP 层
从 ip_rcv 函数开始，处理逻辑就从二层到了三层，IP 层.

在 ip_rcv 中，得到 IP 头，然后又遇到了NF_HOOK，这次因为是接收网络包，第一个 hook 点是 NF_INET_PRE_ROUTING，也就是 iptables 的 PREROUTING 链. 如果里面有规则，则执行规则，然后调用 [ip_rcv_finish](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_input.c#L414).

ip_rcv_finish 得到网络包对应的路由表，然后调用 dst_input，在 dst_input 中，调用的是 struct rtable 的成员的 dst 的 input 函数. 在 rt_dst_alloc 中，可以看到，input 函数指向的是 [ip_local_deliver](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_input.c#L240).

在 ip_local_deliver 函数中，如果 IP 层进行了分段，则进行重新的组合. 接下来就是熟悉的 NF_HOOK。hook 点在 NF_INET_LOCAL_IN，对应 iptables 里面的 INPUT 链。在经过 iptables 规则处理完毕后，会调用 [ip_local_deliver_finish](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_input.c#L226).

在 IP 头中，有一个字段 protocol 用于指定里面一层的协议，在这里应该是 TCP 协议. 于是，从 [inet_protos](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/protocol.c#L27) 数组中，找出 TCP 协议对应的处理函数. 这个数组里面的内容是 [struct net_protocol](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/protocol.h#L37).

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1946
static int __init inet_init(void)
{
	...
	if (inet_add_protocol(&udp_protocol, IPPROTO_UDP) < 0)
		pr_crit("%s: Cannot add UDP protocol\n", __func__);
	if (inet_add_protocol(&tcp_protocol, IPPROTO_TCP) < 0)
		pr_crit("%s: Cannot add TCP protocol\n", __func__);
	...
}

// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1737
/* thinking of making this const? Don't.
 * early_demux can change based on sysctl.
 */
static struct net_protocol tcp_protocol = {
	.early_demux	=	tcp_v4_early_demux,
	.early_demux_handler =  tcp_v4_early_demux,
	.handler	=	tcp_v4_rcv,
	.err_handler	=	tcp_v4_err,
	.no_policy	=	1,
	.netns_ok	=	1,
	.icmp_strict_tag_validation = 1,
};

/* thinking of making this const? Don't.
 * early_demux can change based on sysctl.
 */
static struct net_protocol udp_protocol = {
	.early_demux =	udp_v4_early_demux,
	.early_demux_handler =	udp_v4_early_demux,
	.handler =	udp_rcv,
	.err_handler =	udp_err,
	.no_policy =	1,
	.netns_ok =	1,
};
```

在系统初始化的时候，网络协议栈的初始化调用的是 inet_init，它会调用 inet_add_protocol，将 TCP 协议对应的处理函数 tcp_protocol、UDP 协议对应的处理函数 udp_protocol，放到 inet_protos 数组中. 在上面的网络包的接收过程中，会取出 TCP 协议对应的处理函数 tcp_protocol，然后调用 handler 函数，也即 [tcp_v4_rcv](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L1873) 函数.

## 网络协议栈的 TCP 层
从 tcp_v4_rcv 函数开始, 处理逻辑就从 IP 层到了 TCP 层.

在 tcp_v4_rcv 中，得到 TCP 的头之后，就可以开始处理 TCP 层的事情. 因为 TCP 层是分状态的，状态被维护在数据结构 struct sock 里面，因而要根据 IP 地址以及 TCP 头里面的内容，在 tcp_hashinfo 中找到这个包对应的 struct sock，从而得到这个包对应的连接的状态.

接下来，就根据不同的状态做不同的处理，例如，tcp_v4_rcv代码中的 TCP_LISTEN、TCP_NEW_SYN_RECV 状态属于连接建立过程中. 这个在分析三次握手的时候学过了. 再如，TCP_TIME_WAIT 状态是连接结束的时候的状态，这个暂时可以不用看.

接下来，来分析最主流的网络包的接收过程，这里面涉及三个队列：
- backlog 队列
- prequeue 队列
- sk_receive_queue 队列

为什么接收网络包的过程，需要在这三个队列里面倒腾过来、倒腾过去呢？这是因为，同样一个网络包要在三个主体之间交接.

第一个主体是软中断的处理过程. 如果没忘记的话，在执行 tcp_v4_rcv 函数的时候，依然处于软中断的处理逻辑里，所以必然会占用这个软中断.

第二个主体就是用户态进程. 如果用户态触发系统调用 read 读取网络包，也要从队列里面找.

第三个主体就是内核协议栈. 哪怕用户进程没有调用 read，读取网络包，当网络包来的时候，也得有一个地方收着呀.

这时候，就能够了解上面代码中 sock_owned_by_user 的意思了，其实就是说，当前这个 sock 是不是正有一个用户态进程等着读数据呢，如果没有，内核协议栈也调用 tcp_add_backlog，暂存在 backlog 队列中，并且抓紧离开软中断的处理过程.

如果有一个用户态进程等待读取数据呢？就先调用 tcp_prequeue，也即赶紧放入 prequeue 队列，并且离开软中断的处理过程. 在这个函数里面，会看到对于 sysctl_tcp_low_latency 的判断，也即是不是要低时延地处理网络包.

如果把 sysctl_tcp_low_latency 设置为 0，那就要放在 prequeue 队列中暂存，这样不用等待网络包处理完毕，就可以离开软中断的处理过程，但是会造成比较长的时延. 如果把 sysctl_tcp_low_latency 设置为 1，还是调用 [tcp_v4_do_rcv](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L1613).

在 tcp_v4_do_rcv 中，分两种情况，一种情况是连接已经建立，处于 TCP_ESTABLISHED 状态，调用 [tcp_rcv_established](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L5588). 另一种情况，就是其他的状态，调用 [tcp_rcv_state_process](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L6183).

![](/misc/img/net/385ff4a348dfd2f64feb0d7ba81e2bc6.png)

在 tcp_rcv_state_process 中，如果对着 TCP 的状态图进行比对，能看到，对于 TCP 所有状态的处理，其中和连接建立相关的状态，已经分析过，所以重点关注连接状态下的工作模式.

在连接状态下，会调用 tcp_rcv_established. 在这个函数里面，会调用 [tcp_data_queue](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L4797)，将其放入 sk_receive_queue 队列进行处理.

在 tcp_data_queue 中，对于收到的网络包，要分情况进行处理.

第一种情况，seq == tp->rcv_nxt，说明来的网络包正是我服务端期望的下一个网络包. 这个时候判断 sock_owned_by_user，也即用户进程也是正在等待读取，这种情况下，就直接 skb_copy_datagram_msg，将网络包拷贝给用户进程就可以了.

如果用户进程没有正在等待读取，或者因为内存原因没有能够拷贝成功，tcp_queue_rcv 里面还是将网络包放入 sk_receive_queue 队列.

接下来，tcp_rcv_nxt_update 将 tp->rcv_nxt 设置为 end_seq，也即当前的网络包接收成功后，更新下一个期待的网络包.

这个时候，还会判断一下另一个队列，out_of_order_queue，也看看乱序队列的情况，看看乱序队列里面的包，会不会因为这个新的网络包的到来，也能放入到 sk_receive_queue 队列中.

例如，客户端发送的网络包序号为 5、6、7、8、9. 在 5 还没有到达的时候，服务端的 rcv_nxt 应该是 5，也即期望下一个网络包是 5. 但是由于中间网络通路的问题，5、6 还没到达服务端，7、8 已经到达了服务端了，这就出现了乱序.

乱序的包不能进入 sk_receive_queue 队列. 因为一旦进入到这个队列，意味着可以发送给用户进程. 然而，按照 TCP 的定义，用户进程应该是按顺序收到包的，没有排好序，就不能给用户进程. 所以，7、8 不能进入 sk_receive_queue 队列，只能暂时放在 out_of_order_queue 乱序队列中.

当 5、6 到达的时候，5、6 先进入 sk_receive_queue 队列. 这个时候再来看 out_of_order_queue 乱序队列中的 7、8，发现能够接上. 于是，7、8 也能进入 sk_receive_queue 队列了. [tcp_ofo_queue](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L4506) 函数就是做这个事情的.

至此第一种情况处理完毕.

第二种情况，end_seq 不大于 rcv_nxt，也即服务端期望网络包 5. 但是，来了一个网络包 3，怎样才会出现这种情况呢？肯定是服务端早就收到了网络包 3，但是 ACK 没有到达客户端，中途丢了，那客户端就认为网络包 3 没有发送成功，于是又发送了一遍，这种情况下，要赶紧给客户端再发送一次 ACK，表示早就收到了.

第三种情况，seq 不小于 rcv_nxt + tcp_receive_window. 这说明客户端发送得太猛了. 本来 seq 肯定应该在接收窗口里面的，这样服务端才来得及处理，结果现在超出了接收窗口，说明客户端一下子把服务端给塞满了.

这种情况下，服务端不能再接收数据包了，只能发送 ACK 了，在 ACK 中会将接收窗口为 0 的情况告知客户端，客户端就知道不能再发送了. 这个时候双方只能交互窗口探测数据包，直到服务端因为用户进程把数据读走了，空出接收窗口，才能在 ACK 里面再次告诉客户端，又有窗口了，又能发送数据包了.

第四种情况，seq 小于 rcv_nxt，但是 end_seq 大于 rcv_nxt，这说明从 seq 到 rcv_nxt 这部分网络包原来的 ACK 客户端没有收到，所以重新发送了一次，从 rcv_nxt 到 end_seq 时新发送的，可以放入 sk_receive_queue 队列.

当前四种情况都排除掉了，说明网络包一定是一个乱序包了. 这里有点儿难理解，还是用上面那个乱序的例子仔细分析一下 rcv_nxt=5.

假设 tcp_receive_window 也是 5，也即超过 10 服务端就接收不了了. 当前来的这个网络包既不在 rcv_nxt 之前（不是 3 这种），也不在 rcv_nxt + tcp_receive_window 之后（不是 11 这种），说明这正在我们期望的接收窗口里面，但是又不是 rcv_nxt（不是我们马上期望的网络包 5），这正是上面的例子中网络包 7、8 的情况.

对于网络包 7、8，只好调用 tcp_data_queue_ofo 进入 out_of_order_queue 乱序队列，但是没有关系，当网络包 5、6 到来的时候，会走第一种情况，把 7、8 拿出来放到 sk_receive_queue 队列中.

至此，网络协议栈的处理过程就结束了.

## Socket 层
当接收的网络包进入各种队列之后，接下来就要等待用户进程去读取它们了.

读取一个 socket，就像读取一个文件一样，读取 socket 的文件描述符，通过 read 系统调用.

read 系统调用对于一个文件描述符的操作，大致过程都是类似的，最终它会调用到用来表示一个打开文件的结构 stuct file 指向的 file_operations 操作. 对于 socket 来讲，它的 file_operations 定义如下：
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L149
/*
 *	Socket files have a set of 'special' operations as well as the generic file ones. These don't appear
 *	in the operation structures but are done directly via the socketcall() multiplexor.
 */

static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.sendpage =	sock_sendpage,
	.splice_write = generic_splice_sendpage,
	.splice_read =	sock_splice_read,
	.show_fdinfo =	sock_show_fdinfo,
};
```

按照文件系统的读取流程，调用的是 [sock_read_iter](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L960).

在 sock_read_iter 中，通过 VFS 中的 struct file，将创建好的 socket 结构拿出来，然后调用 [sock_recvmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L900)，sock_recvmsg 会调用 [sock_recvmsg_nosec](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L883).

sock_recvmsg_nosec里调用了 socket 的 ops 的 recvmsg，根据 [inet_stream_ops](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1015) 的定义，这里调用的是 [inet_recvmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L835).

在inet_recvmsg里面，从 socket 结构，可以得到更底层的 sock 结构，然后调用 sk_prot 的 recvmsg 方法, 根据 tcp_prot 的定义，调用的是 [tcp_recvmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L2015).

tcp_recvmsg 这个函数比较长，里面逻辑也很复杂，好在里面有一段注释概括了这里面的逻辑. 注释里面提到了三个队列，receive_queue 队列、prequeue 队列和 backlog 队列. 这里面，需要把前一个队列处理完毕，才处理后一个队列.

tcp_recvmsg 的整个逻辑也是这样执行的：这里面有一个 while 循环，不断地读取网络包.

这里，会先处理 sk_receive_queue 队列. 如果找到了网络包，就跳到 found_ok_skb 这里. 这里会调用 skb_copy_datagram_msg，将网络包拷贝到用户进程中，然后直接进入下一层循环.

直到 sk_receive_queue 队列处理完毕，才到了 sysctl_tcp_low_latency 判断. 如果不需要低时延，则会有 prequeue 队列. 于是，就跳到 do_prequeue 这里，调用 tcp_prequeue_process 进行处理.

如果 sysctl_tcp_low_latency 设置为 1，也即没有 prequeue 队列，或者 prequeue 队列为空，则需要处理 backlog 队列，在 release_sock 函数中处理.

[release_sock](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/sock.c#L3062) 会调用 [__release_sock](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/sock.c#L2534)，这里面会依次处理队列中的网络包.

最后，哪里都没有网络包，只好调用 sk_wait_data，继续等待在哪里，等待网络包的到来.

至此，网络包的接收过程到此结束.

## 总结
![](/misc/img/net/20df32a842495d0f629ca5da53e47152.png)

接收网络包的过程:
1. 硬件网卡接收到网络包之后，通过 DMA 技术，将网络包放入 Ring Buffer
1. 硬件网卡通过中断通知 CPU 新的网络包的到来
1. 网卡驱动程序会注册中断处理函数 ixgb_intr
1. 中断处理函数处理完需要暂时屏蔽中断的核心流程之后，通过软中断 NET_RX_SOFTIRQ 触发接下来的处理过程
1. NET_RX_SOFTIRQ 软中断处理函数 net_rx_action，net_rx_action 会调用 napi_poll，进而调用 ixgb_clean_rx_irq，从 Ring Buffer 中读取数据到内核 struct sk_buff
1. 调用 netif_receive_skb 进入内核网络协议栈，进行一些关于 VLAN 的二层逻辑处理后，调用 ip_rcv 进入三层 IP 层
1. 在 IP 层，会处理 iptables 规则，然后调用 ip_local_deliver，交给更上层 TCP 层
1. 在 TCP 层调用 tcp_v4_rcv，这里面有三个队列需要处理，如果当前的 Socket 不是正在被读；取，则放入 backlog 队列，如果正在被读取，不需要很实时的话，则放入 prequeue 队列，其他情况调用 tcp_v4_do_rcv
1. 在 tcp_v4_do_rcv 中，如果是处于 TCP_ESTABLISHED 状态，调用 tcp_rcv_established，其他的状态，调用 tcp_rcv_state_process
1. 在 tcp_rcv_established 中，调用 tcp_data_queue，如果序列号能够接的上，则放入 sk_receive_queue 队列；如果序列号接不上，则暂时放入 out_of_order_queue 队列，等序列号能够接上的时候，再放入 sk_receive_queue 队列.

至此内核接收网络包的过程到此结束，接下来就是用户态读取网络包的过程，这个过程分成几个层次:
- VFS 层：read 系统调用找到 struct file，根据里面的 file_operations 的定义，调用 sock_read_iter 函数
- sock_read_iter 函数调用 sock_recvmsg 函数
- Socket 层：从 struct file 里面的 private_data 得到 struct socket，根据里面 ops 的定义，调用 inet_recvmsg 函数
- Sock 层：从 struct socket 里面的 sk 得到 struct sock，根据里面 sk_prot 的定义，调用 tcp_recvmsg 函数
- TCP 层：tcp_recvmsg 函数会依次读取 receive_queue 队列、prequeue 队列和 backlog 队列
