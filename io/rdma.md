# rdma
ref:
- [一文了解 NVMe、NVMe-oF 和 RDMA](https://www.smartx.com/blog/2021/12/nvme-nvme-of-rdma/)

直接内存访问 (DMA) 指设备无需 CPU 干预即可直接访问主机内存的能力。远程直接内存访问 (RDMA) 也就是在不中断远程机器系统 CPU 处理的情况下对该机器上的内存执行访问（读取和写入）的能力。

RDMA 主要优势:
- 零复制：应用程序无需网络软件栈的参与即可执行数据传输。数据可以直接发送和接收到缓冲区，无需在网络层之间复制。
- 内核旁路：应用程序可以直接从用户空间执行数据传输，无需内核参与。
- 无 CPU 参与：应用程序无需在远程服务器内耗用任何 CPU 时间即可访问远程内存。无需任何远程进程（或处理器）的干预即可读取远程内存服务器。此外，远程 CPU 的高速缓存不会被访问的内存内容填满。

RDMA多用于对性能要求较高的领域. 由于其性能优势，所以基于RDMA的NVMe over RDMA也成为了很多超算中心、科研机构、互联网公司的首选。[华为主导的NoF+存储网络解决方案](https://support.huawei.com/enterprise/zh/doc/EDOC1100211900)，就是基于NVMe over RoCE的一种增强方案

## 使用 RDMA
要使用 RDMA，需要具备 RDMA 功能的网络适配器:
1. 支持 RDMA 的以太网 NIC (rNIC)，如 Broadcom NetXtreme E 系列、Marvell / Cavium FastLinQ 或 Nvidia / Mellanox Connect-X 系列
1. InfiniBand 领域内的 InfiniBand 主机通道适配器 (HCA)（同样以 Nvidia / Mellanox Connect-X 为例）

由此可以推断出，网络的链路层协议既可以是以太网，也可以是 InfiniBand。这两种协议均可用于传输基于 RDMA 的应用程序.

## RDMA 的三种类型：
1. InfiniBand：InfiniBand 网络架构原生支持 RDMA

	由于这是一种新的网络技术，因此需要支持该技术的网卡和交换机
2. RoCE（基于聚合以太网的 RDMA，读作“Rocky”）：这种类型基本上就是基于以太网网络的 RDMA 的实现。其方式是通过以太网来封装 InfiniBand 传输包。RoCE 有两种版本：

	- RoCEv1：以太网链路层协议（以太网 0x8915），支持在相同以太网广播域内任意两个主机之间进行通信。因此，仅限第 2 层网络，不可路由。
	- RoCEv2：利用 UDP/IP（IPv4 或 IPv6）报头增强 RoCEv1，因此增加了第 3 层网络可路由性。NVMe/RoCEv2 默认使用 UDP 目标端口  4791。

	允许在标准以太网基础架构(交换机)上使用RDMA, 只有NIC应该是特殊的，需支持RoCE

	基于以太网的 RoCE 目前已成为 RDMA 的主流网络承载方式
3. iWARP（互联网广域 RDMA 协议）：按 IETF 标准拥塞感知协议（如 TCP 和 SCTP）分层。具有卸载 TCP/IP 流量控制和管理功能。

	允许在标准以太网基础架构(交换机)上使用RDMA，只有NIC应该是特殊的，需支持iWARP(如果使用CPU卸载)，否则所有iWARP堆栈都可以在SW中实现，并且丢失了大部分的RDMA性能优势

即使 iWARP 和 RoCE 都使用相同的 RDMA 软件谓词和同类以太网 RDMA-NIC (rNIC)，由于第 3 层/第 4 层网络之间存在的差异，两者之间仍无法以 RDMA 来通信。现如今，RoCEv2 是供应商最常用的选择

## RDMA对于NVMe over Fabrics协议的便利性
1. 提供了低延迟、低抖动和低CPU使用率的传输层协议
1. 最大限度利用硬件加速，避免软件协议栈的开销
1. 依赖于开放互联联盟组织维护的Verbs和代码库，RDMA定义了丰富的可异步访问的接口机制，这对于提高IO性能是至关重要的

NVMe over Fabric 作为集中存储网络的下一跳，会形成两个主要市场：一个是以 NVMe over RoCE/FC 为主，主打高性能和高可靠的市场；另一个是以 NVMe over TCP 为主，主打扩展性和兼容性的市场.