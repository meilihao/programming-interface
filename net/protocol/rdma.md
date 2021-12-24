# rdma
远程直接内存访问(即RDMA)是一种直接内存访问技术, 它将数据直接从一台计算机的内存传输到另一台计算机, 无需双方操作系统的介入. 它是一种为了解决网络传输中服务器端数据处理延迟而产生的技术.

RDMA最早在Infiniband传输网络上实现, 后来业界厂家把RDMA移植到传统Ethernet以太网上, 降低了RDMA的使用成本, 推动RDMA技术普及.

在Ethernet以太网上, 根据协议栈融合度的差异, 分为iWARP和RoCE两种技术, 而RoCE又包括RoCEv1和RoCEv2两个版本(RoCEv2的最大改进是支持IP路由).

![rdma_protocol](/misc/img/net/rdma_protocol.png)

因此RDMA 实现有:
1. InfiniBand (IB)

    基于 InfiniBand 架构的 RDMA 技术, 由 IBTA(InfiniBand Trade Association)提出, 需要专用的 IB 网卡和 IB 交换机, 是一种专为RDMA设计的网络，从硬件级别保证可靠传输 ，技术先进，但是成本高昂.

    目前EMC全系产品已经切换到Infiniband组网，IBM/TMS的FlashSystem系列，IBM的存储系统XIV Gen3，DDN的SFA系列都采用Infiniband网络.

    相比FC的优势主要体现在性能是FC的3.5倍，Infiniband交换机的延迟是FC交换机的1/10，支持SAN和NAS.

    Mellanox OFED软件堆栈是承载在InfiniBand硬件和协议之上, 由开源OpenFabrics组织维护.

1. RDMA over Converged Etherent (RoCE)

    基于以太网的 RDMA 技术，也是由 IBTA提出. RoCE 与 InfiniBand技术有相同的软件应用层及传输控制层, 仅网络层及以太网链路层存在差异. RoCE支持在标准以太网基础设施上使用RDMA技术, 但是需要交换机支持无损以太网传输, 需要服务器使用 RoCE 网卡.

    RoCE消耗的资源比 iWARP 少，支持的特性比 iWARP 多.

    测试环境可使用SoftRoCE模拟rdma网卡.

    RoCE v1协议：基于以太网承载 RDMA，只能部署于二层网络，它的报文结构是在原有的 IB架构的报文上增加二层以太网的报文头，通过 Ethertype 0x8915 标识 RoCE 报文.

    RoCE v2协议：基于 **UDP/IP** 协议承载 RDMA，可部署于三层网络，它的报文结构是在原有的 IB 架构的报文上增加 UDP 头、IP 头和二层以太网报文头，通过 UDP 目的端口号 4791 标 识RoCE 报文. RoCE v2 支持基于源端口号 hash，采用 ECMP 实现负载分担，提高了网络的利用率.

    RoCE需要支持无损以太网络，这是由于 IB 的丢包处理机制中, 任意一个报文的丢失都会造成大量的重传, 严重影响数据传输性能.

1. iWARP

    基于 **TCP/IP** 协议的 RDMA 技术，由IETF 标 准定义. iWARP 支持在标准以太网基础设施上使用 RDMA 技术, 但服务器需要使用支持iWARP 的网卡.

    > 大规模数据中心或大规模应用程序中使用 iWARP 时，大量的 TCP 连接与可靠传输将导致凄惨的性能, 可参考intel的"Understanding iWARP".


虽然IB、以太网RoCE、以太网iWARP这三种RDMA技术使用统一的API(libverbs), 但它们有着不同的物理层和链路层. 在以太网解决方案中，RoCE相对于iWARP来说有着明显的优势，这些优势体现在延时、吞吐率和 CPU负载, RoCE被很多主流的方案所支持.

InfiniBand 与 Ethernet 之间的区别:
- InfiniBand 模式的延时更低，带宽更高

    - ConnectX-4 Lx EN （Ethernet）提供 1、10、25、40 和 50GbE 带宽 : 亚微秒级延迟
    - ConnectX-5 具备 Virtual Protocol Interconnect®, 支持具有 100Gb/s InfiniBand 和以太网连接、小于 600纳秒的延迟
- InfiniBand 采用 Cut-Through 转发模式，减少转发时延，基于 Credit 流控机制，保证无丢包. RoCE 性能与 IB 网络相当，DCB 特性保证无丢包，需要网络支持 DCB 特性，但时延比 IB 交换机时延稍高一些.

- Ethernet 模式可能存在丢包，而导致数据重传的延时