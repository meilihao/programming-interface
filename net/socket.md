# socket
socket是通过标准的unix文件描述符和其他的程序进行通信的一种方法.

> socket 分unix socket, internet socket等, 本文socket特指internet socket.

- socket.connect() 是通信双方交换控制信息, 尝试建立一个连接
    通信操作中使用的控制信息分为两类:
    1. 头部(以太网头部, IP头部, TCP头部等)中记录的信息
    2. 套接字（协议栈中的内存空间）中记录的信息
- socket.bind() : 为新建的socket绑定ip和port.
- socket.listen() : 为新socket分配缓存空间, 并给出队列大小
- socket.accept() : 阻塞调用方, 直到有连接进入
    新连接进来时socket调用accept原语创建一个新socket并返回一个与其关联的文件描述符, 旧socket继续监听.
- socket.send() : 向socket发送数据
- socket.recv() : 从socket接收数据
- socket.close() : 释放socket

> 网络字节序(Network Byte Order)是大端序

> 在网络层, Socket 函数需要指定到底是 IPv4 还是 IPv6,分别对应设置为 AF_INET 和 AF_INET6

> 在传输层, TCP 协议是基于数据流的,所以设置为SOCK_STREAM,而 UDP 是基于数据报的,因而设置为 SOCK_DGRAM

> socket.listen用两个队列实现，一个SYN队列（或半连接队列）和一个accept队列（或完整的连接队列）: 处于SYN RECEIVED状态的连接被添加到SYN队列，并且当它们的状态改变为ESTABLISHED时，即当接收到3次握手中的ACK分组时，将它们移动到accept队列. 在这种情况下，listen syscall的backlog参数表示accept队列的大小. SYN队列的长度可以使用`/proc/sys/net/ipv4/tcp_max_syn_backlog`设置.

端口分类:
- 保留端口: 0~1023, 可通过`/etc/services`查看
- 动态分配端口(也叫非特权端口): > 1024
- 注册端口: > 1024

## 类型
socket类型即数据传输方式, 也是socket 传输层的协议.

1. 流式socket(SOCK_STREAM) : tcp

   提供可靠的, 面向连接的有序通信流
1. 数据报socket(SOCK_DGRAM) : udp

   无连接的通信, 数据通过相互独立的报文进行传输, 是无序的, 且不保证可靠, 无差错.
1. 原始socket

   主要用于协议的开发, 可以进行比较底层的操作, 很少用到.

## 协议族(protocol family)
协议族(protocol family) 即 socket ip层的通信协议, from `sys/socket.h`:
- PF_INET : ipv4协议族
- PF_INET6 : ipv6协议族
- PF_LOCAL : 本地通信的UNIX协议族
- PF_PACKET : 底层socket的协议族

## socket可选项
- IPPROTO_IP : ip相关可选项
- IPPROTO_TCP : tcp相关可选项
- SOL_SOCKET : socket相关可选项
  
  - SO_RCVBUF : 输入缓冲大小
  - SO_SNDBUF : 输出缓冲大小
  - SO_REUSEADDR : 与time-wait状态相关

## socket io
参考:
- [Redis 和 I/O 多路复用](https://draveness.me/redis-io-multiplexing)

![几种IO模式比较](/misc/img/net/20180202171329716.png)

与文件io略有不同, 分为
- 阻塞 : 与文件io相同
- 非阻塞 : 与文件io相同
- io多路复用

  `select()/poll()/epoll`
- 信号驱动io(SIGIO)

  内核：I/O能用了
　进程：接收到I/O能用的消息并执行接下来的操作

  cpu利用率很高
- 异步io

  内核：等待这个I/O有消息了，接收到数据
　进程：从缓存中得到数据

异步io与信号驱动io的区别:
1. 信号驱动io下, kernel在数据到达时向应用发送SIGIO信号, 应用收到信号后将数据从kernel复制到应用.
1. 异步io下, kernel在所有操作(包括data从kernel复制到用户缓存区)都已经被内核完成后才会通知应用程序.

## 原理
参考:
- [epoll 的本质是什么？](https://my.oschina.net/editorial-story/blog/3052308)
- [从linux源码看epoll](https://www.jishuwen.com/d/2dNT)

### 基于 TCP 协议的 Socket 函数调用过程:
![](/misc/img/net/v2-067f27c436ae555b059c4a0792e556a1_hd.jpg)

在内核中,为每个 Socket 维护两个队列: 
1. 一个是已经建立了连接的队列,这时候连接三次握手已经完毕,处于 established 状态
2. 一个是还没有完全建立连接的队列,这个时候三次握手还没完成,处于syn_rcvd 的状态

默认情况，内核会认为socket函数创建的套接字是主动套接字（active socket），它存在于一个连接的客户端. 而服务器调用listen函数告诉内核，该套接字是被服务器而不是客户端使用的，即listen函数将一个主动套接字转化为**监听socket**. 监听套接字可以接受来自客户端的连接请求. 服务器通过accept函数等待来自客户端的连接请求到达监听套接字, 处理后返回一个**已连接socket**.

socket内核描述:
![](/misc/img/net/qsrseeds2x.png)

每一个进程都有一个数据结构 task_struct，里面指向一个文件描述符数组fds，来列出这个进程打开的所有文件的文件描述符. 该数组中的内容是一个指针，指向内核中所有打开的文件列表, 而每个文件也会有一个 inode（索引节点）.

对于 Socke 而言，它是一个文件，也就有对于的文件描述符. 与真正的文件系统不一样的是，Socket 对于的 inode 并不是保存在硬盘上，而是在内存中. 在这个 inode 中，指向了 Socket 在内核中的 Socket 结构.

 在这个结构里面，主要有两个队列: 一个发送队列sk_write_queue，一个接收队列sk_receive_queue. 这两个队列里面，各保存一个 sk_buff chain作为缓存. 在这个缓存里能够看到完整的package结构.

系统会用一个四元组来标识一个 TCP 连接: `本机 IP, 本机端口, 对端 IP, 对端端口`.

服务端最大并发 TCP 连接数远不能达到理论上限(对端IP * 对端端口 = 2^32 * 2^16):
- 文件描述符限制, 因为Socket 是文件, 受到 ulimit 配置文件描述符的数目限制
- 内存, 每个 TCP 连接都要占用一定内存

提升tcp连接数量的方式:
1. 多进程方式(不推荐, 太耗资源)
    因为复制了文件描述符列表，而文件描述符都是指向整个内核统一的打开文件列表的. 因此父进程刚才因为 accept 创建的已连接 Socket 也是一个文件描述符，同样也会被子进程获得.

    接下来，子进程就可以通过这个已连接 Socket 和客户端进行通信了. 当通信完成后，就可以 退出进程. 那父进程如何知道子进程干完了项目要退出呢？父进程中 fork 函数返回的整数就是子进程的 ID，父进程可以通过这个 ID 查看子进程是否完成项目，是否需要退出.
1. 多线程方式
    在 Linux 下，通过 pthread_create 创建一个线程，也是调用 do_fork. 不同的是，虽然新的线程在 task 列表会新创建一项，但是很多资源，例如文件描述符列表、进程空间，这些还是共享的，只不过多了一个引用而已.
1. IO 多路复用，一个线程维护多个 Socket
    Socket 是文件描述符，因此某个线程盯的所有的 Socket，都放在一个文件描述符集合 fd_set 中, 然后调用 select 函数来监听文件描述符集合是否有变化(其监视数量受FD_SETSIZE 限制)，一旦有变化，就会依次轮循查看每个文件描述符. 那些发生变化的文件描述符在 fd_set 对应的位都设为 1，表示 Socket 可读或者可写，从而可以进行读写操作，然后再调用 select，接着盯着下一轮的变化.
1. IO 多路复用, select(轮循)到epoll(事件通知)
    epoll在内核中的实现不是通过轮询的方式,而是通过注册 callback 函数的方式,当某个文件描述符发送变化的时候,就会主动通知.

    ![](/misc/img/net/zqugepbmve.png)

    - epoll_create : 创建保存epoll文件描述符的空间

      epoll_create 创建一个 epoll 对象(os所有),也是一个文件,也对应一个文件描述符,同样也对应着打开文件列表中的一项. 在这项里面有一个红黑树,在红黑树里,要保存这个 epoll要监听的所有 Socket.

    - epoll_clt : 向空间注册/注销文件描述符

      当 epoll_ctl 添加一个 Socket 的时候,其实是加入这个红黑树,同时红黑树里面的节点指向一个结构, 将这个结构挂在被监听的 Socket 的事件列表中. 当一个 Socket 来了一个事件的时候,可以从这个列表中得到 epoll 对象,并调用 call back 通知它.

    - epoll_wait: 与`select()`类似, 等待文件描述符发生变化.

    这种通知方式使得监听的 Socket 数据增加的时候,效率不会大幅度降低,能够同时监听的 Socket 的数目也非常的多了, 上限就为系统定义的进程打开的最大文件描述符个数. 因而,epoll 被称为解决C10K 问题的利器

### 基于 UDP 协议的 Socket 程序函数调用过程
![](/misc/img/net/qofe1t7ocm.png)

UDP 是没有连接的,所以不需要三次握手,也就不需要调用 listen 和 connect,但是,UDP 的的交互仍然需要 IP 和端口号,因而也需要 bind. 同样因为UDP 是没有维护连接状态的,因而不需要每对连接建立一组 Socket,而是只要有一个 Socket就能够和多个客户端通信.

## FAQ
### socket 阻塞与非阻塞的区别(面试)
阻塞: 在socket中调用recv函数，如果缓冲区中没有数据，这个函数就会一直等待，直到有数据才返回.

非阻塞: 调用recv()函数读取网络缓冲区中数据，不管是否读到数据都立即返回，而不会一直挂在此函数调用上.

### epoll 和 select 的区别
1. epoll 和 select 都是 I/O 多路复用的技术,都可以实现同时监听多个 I/O 事件的状态
1. epoll 相比 select 效率更高,主要是基于其操作系统支持的 I/O事件通知机制, 而 select 是基于轮询机制
1. epoll 支持水平触发和边沿触发两种模式

### socket缓冲区已满是否会丢数据
不会. 缓冲区满后, socket无法再接收数据, 会通知对端停止传输, 即发送端会根据接收端的状态传输数据.

> 通过滑动窗口解决.

### `socket()`的第三个参数
第三个参数指定应用程序所使用的通信协议. 此参数可以用于**区分同一协议族中存在多种数据传输方式相同的协议**.在Internet通讯域中，此参数一般取值为0，系统会根据套接字的类型决定应使用的传输层协议.

### INADDR_ANY
使用相同port的所有ip地址, 用于建立监听socket.

### udp 发送端调用`sendto()`的次数与接收端调用`recvfrom()`的次数相同
udp有数据边界.

### UDP调用`connect()`
与tcp的`connect()`不同, 仅向udp socket注册了目标ip和port, 并不存在握手.

### 半关闭函数`shutdown`
`close()`是全关闭, 但某些情况下需要只关闭一部分数据流即可.

参数howto:
- SHUT_RD : 断开输入流, 此时输入缓冲即使收到数据也会忽略
- SHUT_WR : 断开输出流, 如果输出缓冲还有未发送的数据, 则也会发送出去
- SHUT_RDWR : 全部断开, 等同于`close()`

### Nagle算法
为避免因数据包过多而发生网络过载.

部分场景不需要Nagle算法: 传输大文件数据.

### 父进程通过fork出子进程处理conn, 为什么父进程fork后要close(已连接的socket)而子进程需要close(监听socket)
socket并非进程所有, 而是os的, 只有socket关联的所有文件描述符都终止后才会销毁, 因此调用fork后将无关的socket文件描述符关掉, 不会影响程序运行.

### socket编程中write、read和send、recv之间的区别
recv和send函数提供了和read和write差不多的功能, 但它们提供了第四个参数来控制读写操作:

|flag|含义|使用者|
|  ----  | ----| --- |
| MSG_PEEK | 查看输入缓冲是否有数据,并不从系统缓冲区移走数据 |recv|
| MSG_DONTROUTE | 不查找路由表,即在local中寻找目的地 |send|
| MSG_OOB | 接受或发送带外数据 |both|
| MSG_DONTWAIT | 调用io函数不阻塞,即使用非阻塞io |both|
| MSG_WAITALL | 等待任何数据 |recv|

### epoll 条件触发(level trigger, 默认)和边缘触发(edge trigger)
它们的区别在于发生事件的时间点.

条件触发: 只要满足条件(输入缓冲有数据,包括未读完)就会发生一个io事件EPOLLIN.
边缘触发: 每当状态变化时(输入缓冲收到数据)发生一个io事件EPOLLIN.

> 边缘触发中一定要以非阻塞的read/write进行, 否则有可能引起server的长期停顿.
> 边缘触发能够做到接收数据与处理数据的时间点分离.

举例: 假定经过长时间的沉默后，现在来了100个字节，这时无论边缘触发和条件触发都会产生一个read ready notification通知应用程序可读. 应用程序读了50个字节，然后重新调用api等待io事件. 这时条件触发的api会因为还有50个字节可读会再次注册事件,从而立即返回用户一个read ready notification, 而边缘触发的api会因为可读这个状态没有发生变化而陷入长期等待. 因此在使用**边缘触发的api时，要注意每次都要读到socket返回EAGAIN为止**，否则这个socket就算废了, 而使用条件触发的api 时，如果应用程序不需要写就不要关注socket可写的事件，否则就会无限次的立即返回一个write ready notification. 长期关注socket写事件会出现CPU 100%的毛病.

> redis只支持LT模式
> nginx作为高性能的通用服务器，大网络流量下, 很容易触发EPOLLOUT，则使用ET.
> ET理论上比LT要高效很多, 但对程序员的要求也多些, 必须小心使用: 因为工作在此种方式下时， 在接收数据时， 如果有数据只会通知一次， 假如read时未读完数据，那么不会再有EPOLLIN的通知了， 直到下次有新的数据到达时为止; 当发送数据时， 如果发送缓存未满也只有一次EPOLLOUT的通知， 除非你把发送缓存塞满了， 才会有第二次EPOLLOUT通知的机会， 所以在此方式下read和write时都要处理好.

### 建立socket连接的工作机制/监听队列
内核管理的每一个TCP文件描述符都是一个struct, 它记录TCP相关的信息(如序列号、当前窗口大小等等)，以及一个接收缓冲区(receive buffer,或者叫receive queue)和一个写缓冲区(write buffer,或者叫write queue)，可以在Linux内核的[net/sock.h](https://elixir.bootlin.com/linux/latest/source/include/net/sock.h)中看到socket结构的实现.

当一个新的数据包进入网络接口（NIC）时，通过被**NIC中断或通过轮询NIC**的方式通知内核获取数据. 通常，内核是由中断驱动还是处于轮询模式取决于网络通信量；当**NIC非常繁忙时，内核轮询效率更高**，但如果NIC不繁忙，则可以使用中断来节省CPU周期和电源. Linux称这种技术为NAPI，字面意思是`新的api`.

当内核从NIC获取数据包时，它会对数据包进行解码，并根据源IP、源端口、目标IP和目标端口找出与该数据包相关联的TCP连接. 此信息用于查找与该连接关联的内存中的struct sock. 假设数据包是按顺序的到来的，那么数据的有效负载就被复制到套接字的接收缓冲区中. 此时，内核将执行read(2)或使用诸如select(2)或epoll_wait(2)等I/O多路复用方式系统调用，唤醒等待此套接字的进程.

当用户态的进程实际调用文件描述符上的read(2)时，它会导致内核从其接收缓冲区中删除数据，并将该数据复制到此进程调用read(2)所提供的缓冲区中.

> 如果接收缓冲区已满，而TCP连接的另一端尝试发送更多的数据，内核将拒绝对数据包进行ACK, 这只是常规的TCP拥塞控制.

发送数据的工作原理类似. 当应用程序调用write(2)时，它将数据从用户提供的缓冲区复制到内核写入队列中, 随后内核将把数据从写队列复制到NIC中，并实际发送数据. 如果网络繁忙，如果TCP发送窗口已满，或者如果有流量整形策略等等，从用户实际调用write(2)开始，到向NIC传输数据的实际时间可能会有所延迟.

> 如果写入队列已满，并且用户调用写入write(2)），则系统调用将被阻塞.

这种设计的一个结果是，如果应用程序读取速度太慢或写入速度太快，内核的接收和写入队列可能会被填满. 因此，内核为读写队列设置最大大小, 这样可以确保行为不可控的应用程序使用有限制的内存量. 例如，内核可能会将每个接收和写入队列的大小限制在100KB, 然后每个TCP套接字可以使用的最大内核内存量大约为200KB（因为与队列的大小相比，其他TCP数据结构的大小可以忽略不计）.

实际连接超过`accept()'s backlog`时, 称为监听队列溢出, 可以通过读取`/proc/net/netstat`并检查`ListenOverflows(kernel的全局计数器)`的值来观察情况. kernel采取的两种措施是:
1. kernel不接受连接, 比如拒绝对传入的SYN包进行ACK, 更常见的情况是，内核将完成TCP三次握手，然后使用RST终止连接. **推荐**: 避免让client认为已连接但server响应不行.
1. 接受连接并为其分配一个套接字结构（包括接收/写入缓冲区），然后将套接字对象排队以备以后使用. 下次用户调用accept(2)将立即获得已分配的套接字, 而不是阻塞系统调用.

> socket tcp的backlog的上限是min(backlog,somaxconn)，其中backlog是应用程序中传递给listen系统调用的参数值，somaxconn(`/proc/sys/net/core/somaxconn`)是内核规定的最大连接数.
> Prometheus Node_exporter 的 node_netstat_TcpExt_ListenOverflows 可以监控监听队列溢出.