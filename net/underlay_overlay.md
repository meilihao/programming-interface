# Underlay 和 Overlay
ref:
- [【重识云原生】第四章云网络4.6节——Underlay 和 Overlay概念](https://www.jianshu.com/p/e6ac700c224a)
- [**IP报文在阿里云上的神奇之旅系列一：同地域内云上通信**](https://developer.aliyun.com/article/1054404)

Overlay意为叠加，即通过在现有网络上叠加一个软件定义的逻辑网络，原有网络尽量不做改造，通过定义其上的逻辑网络来实现业务逻辑，解决原有数据中心的网络问题. Overlay是一种将（业务的）二层网络构架在（传统网络的）三层/四层报文中进行传递的网络技术.

Overlay技术实际上是一种隧道封装技术，主要有VXLAN、NVGRE等，基本原理都是通过隧道封装的方式将二层报文进行封装后在现有网络中进行透明传输，到达目的地之后再解封装得到原始报文，相当于一个大二层网络叠加（overlay）在现有的网络之上。

Underlay是一张承载网，由各类物理设备构成，如TOR交换机、汇聚交换机、核心交换机、负载均衡设备与防火墙设备等。

实施Overlay技术后，会在Underlay网络基础上形成一张逻辑网.

Overlay网络是建立在Underlay网络基础上的虚拟网，由逻辑节点和逻辑链路构成。

Overlay网络具有独立的控制和转发平面，对于连接在Overlay边缘设备之外的终端来说，物理网络是透明的。

在华为CloudFabric解决方案中，选用VXLAN技术来构建Overlay网络，业务报文运行在VXLAN Overlay网络上，与物理承载网络解耦。

根据承担Overlay边缘设备（VXLAN NVE）属性的不同，基于VXLAN的Overlay又可以分为：

- Network Overlay：所有NVE全部由物理交换机承担（传统网络设备厂家主推，使用专用的网络设备，性能较好）。
- Host Overlay：所有NVE全部由vSwitch承担（VMware主推，大规模公有云也会使用：使用定制的服务器硬件，提高vSwitch性能，网络设备仅做IP转发。与网络设备商解耦）。
- Hybrid Overlay：NVE一部分部署在物理交换机上，另一部分部署在vSwitch上。