# vpn
ref:
- [VPN](https://tonydeng.github.io/sdn-handbook/secure/vpn/)

根据不同的划分标准，VPN可以按几个标准进行分类划分:
1. 按协议

	VPN的隧道协议主要有四种，PPTP、L2TP、IPSec和SSL，其中PPTP和L2TP协议工作在OSI模型的第二层，又称为二层隧道协议；IPSec是第三层隧道协议；而SSL是工作在OSI会话层之上的协议，如果按照TCP/IP协议模型划分，即工作在应用层
1. 按VPN的应用

	- Access VPN（远程接入VPN）：客户端到网关，使用公网作为骨干网在设备之间传输VPN数据流量；
	- Intranet VPN（内联网VPN）：网关到网关，通过公司的网络架构连接来自同公司的资源；
	- Extranet VPN（外联网VPN）：与合作伙伴企业网构成Extranet，将一个公司与另一个公司的资源进行连接
1. 按所用的设备类型

	网络设备提供商针对不同客户的需求，开发出不同的VPN网络设备，主要为交换机、路由器和防火墙:
	- 路由器式VPN：路由器式VPN部署较容易，只要在路由器上添加VPN服务即可；
	- 交换机式VPN：主要应用于连接用户较少的VPN网络；
	- 防火墙式VPN：防火墙式VPN是最常见的一种VPN的实现方式，许多厂商都提供这种配置类型
1. 按实现原理

	- 重叠VPN：此VPN需要用户自己建立端节点之间的VPN链路，主要包括：GRE、L2TP、IPSec等众多技术。
	- 对等VPN：由网络运营商在主干网上完成VPN通道的建立，主要包括MPLS、VPN技术

L2TP和SSL VPN确实更适合远端PC, 特别是SSL VPN就是专门为远端PC开发的VPN技术; IPSec更适合在分支和总部之间搭建VPN隧道.