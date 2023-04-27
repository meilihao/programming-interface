# list
## new ip
- [New IP：开拓未来数据网络的新连接和新能力](http://www.infocomm-journal.com/dxkx/article/2019/1000-0801/1000-0801-35-9-00002.shtml)
- [NEW IP Framework and Protocol for Future Applications](/misc/pdf/net/6f569c60-7045-11ea-89df-41bea055720b.pdf)

## quic
- [面向5G的阿里自研标准化协议库XQUIC](https://developer.aliyun.com/article/770062)
- [The Road to multipath QUIC: 阿里自研多路径传输技术XLINK](https://www.bilibili.com/read/cv11319597)
- [XLINK: QoE-Driven Multi-Path QUIC Transport in Large-scale Video Services](http://www.hongqiangliu.com/uploads/5/2/7/4/52747939/sigcomm2021-xlink.pdf)
- [2022龙蜥社区全景白皮书|5.5.2 面向HTTP 3.0时代的高性能网络协议栈](https://openanolis.cn/assets/static/openanoliswhitepaper.pdf)

    ExpressUDP(xudp) + ngx_xquic_module + xquic

## next
- [阿里云如何构建高性能云原生容器网络](https://yq.aliyun.com/articles/755848)
- [阿里云如何构建高性能云原生容器网络-直播](https://yq.aliyun.com/live/2626)
- [超大规模云网络数据中心创新（上/下）](https://www.51openlab.com/article/28/)

    - 大型 OTT纷纷开始在其数据中心中采用CLOS架构，利用数量众多，小规模的基本交换矩阵（CrossBar）来构建超大规模网络Fabric.
    - SRv6 Overlay是趋势
- [SRv6开启网络服务化变革之旅](https://cloud.tencent.com/developer/article/1779041)
- [从网络接入层到 Service Mesh，蚂蚁金服网络代理的演进之路](https://www.infoq.cn/article/gmyuf1cjizbyvmslpezv)
- [Linux新技术基石 | eBPF and XDP](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664613441&idx=1&sn=7badd5ed4c01706789f7547e7e2e4582)
- [云网一体化数据中心网络关键技术](http://www.infocomm-journal.com/dxkx/article/2020/1000-0801/1000-0801-36-4-00125.shtml)

    Facebook F4到F16架构的演进，再一次验证了IP-CLOS网络在超大规模组网中的优势，在可扩展性、可靠性以及节省成本上都表现突出.

    BGP是一种距离矢量路由协议，在路由控制、网络收敛时的网络稳定性更好。在中小型数据中心组网时，使用BGP和ISIS、OSPF协议性能相差不大。但是在超大规模数据中心组网中，BGP 的应用性能会更加优异.

    考虑到未来云网一体化下业务端到端需求，基于以太网的RoCE（RDMA over converged ethernet）技术已经逐步替代无限带宽（infiniband， IB）等专用技术，成为主流技术。目前最新的RoCEv2版本使用IP/UDP替代IB网络层，提供IP路由及ECMP能力，成为高性能数据中心网络主要部署协议.

    VxLAN 是目前云数据中心典型的网络虚拟化技术。本质上是一种大二层技术，通过IP网络进行二层隧道流量的传输。同时VxLAN和SDN联合部署已经成为智能化云数据中心的必要组件，VxLAN作为数据平面解耦租户网络和物理网络，SDN将租户的控制能力集成到云管平台与计算、存储资源联合调度，极大地提升了数据中心内业务承载的灵活性。

    基于IPv6的分段路由（segment routing over IPv6，SRv6）是目前承载网关注度和讨论度极高的研究热点技术，是基于源路由理念而设计的在网络上转发IPv6数据分组的一种协议，具备可编程、易部署、易维护、协议简化的特点。它通过集中控制面可实现按需路径规划与调度，同时SRv6 可以完全复用现有 IPv6 数据平面，满足网络灵活演进要求。

    **采用SRv6/EVPN可有效统一“云内网络、云间网络、用户到云网络”承载协议，提供“固移融合、云网融合、虚实网元共存”的云网一体化网络的业务综合承载方案**。SRv6 既符合国家的IPv6 战略，又符合未来技术演进方向，标准化后有可能取代现有 VxLAN 等技术，成为承载层的统一隧道协议，应用领域从骨干网、城域网向数据中心网络逐步扩展。EVPN基础标准已经完备，应用领域已经从数据中心走向广域网。SRv6/EVPN能提供云网一体化环境下的L2/L3业务高效承载.

    [单可用区的物理组网](/misc/img/net/srv6/img_92.jpg)
    整个组网设计遵照超大规模组网原则和技术，按照业务功能分Pod区规划，各区域可独立扩展:
    - 核心层：设置一对核心交换机设备（Super-Spine），提供流量高速转发，与各Pod区的 Spine 交换机交叉互联。核心交换机采用去堆叠配置，可通过设备替换或者横向增加设备的方式升级到更高带宽和更大接入规模，支持业务进一步扩展
    - 计算 Pod：承载大规模计算集群来提供租户虚拟机资源，单Pod内采用基于Spine-Leaf的典型IP-CLOS结构，Spine和Leaf节点设备均采用“去堆叠+eBGP”的组网方式，保障单Pod模块的高可扩展性。
    - 高性能计算 Pod/存储 Pod：为了增强数据中心网络承载AI、高性能计算、分布式存储等业务能力，基于eBGP的IP-CLOS组网，在高性能计算Pod、存储Pod中部署RoCEv2的无损网络技术，保障单Pod内RDMA流量的超低时延特性。经实测验证，无损网络部署可提升分布式存储IOPS 20%，单卷性能达到35万IOPS，平均时延降低 12%，并且严格保障了读取数据汇聚时“多打一”流量的零分组丢失传输。
    - 管理区：放置内部管理组件，不对外提供服务，为安全可靠区。主要承载云管理平台、SDN控制器、OpenStack、级联节点等。在Spine设备旁挂内网管理防火墙设备，保证管理网安全。
    - 扩展区：用于和多个可用区之间的互联。可用区之间利用低延迟光纤传输网络互联，要求时延小于1 ms，保障多活业务的可靠性。
    - 网络服务区：承载云内网络的服务，包括vRouter等。所有租户的业务流量直接走到这个分区，在分区内匹配相关业务对应处理。
    - 云网服务区：主要用于对外接入云网融合业务的区域，包括 Internet 网关、VPN 网关、专线网关、DCI网关等。其中，Internet网关承载Internet流量访问，VPN网关用于IPSec-VPN接入。同时，分别设立云专线接入网关、DCI 接入网关用于入云专线访问VPC、云间跨region VPC互联的连接，满足云网融合业务的各类连接需求

    [云内网络业务承载方案](/misc/img/net/srv6/img_93.jpg):
    数据中心云内网络业务承载方案如上图所示。业务承载依赖大二层网络，目前通用的做法是采用基于MP-BGP的EVPN承载的VxLAN。硬件VTEP节点包括Internet网关、VPN网关、专线接入网关、DCI接入网关的VTEP TOR、网络服务区TOR等。各VTEP节点通过网络设备间的eBGP发布并学习EVPN VxLAN所需的loopback IP。VTEP使用BGP多实例功能组建overlay网络，管理服务区汇聚作为EVPN BGP RR，与所有VTEP节点建立iBGP邻居。VTEP节点创建二层BD域，不同的VTEP 节点属于相同 VNI 的 BD 域，自动创建VxLAN隧道，实现业务流量转发。

    以入云专线承载为例，客户使用云专线产品接入云内VPC网络时，流量从专线接入网关进入，通过VTEP TOR走VxLAN到网络服务区TOR，进入vRouter。vRouter封装成VxLAN后，将报文路由到 Pod 内，通过多段 VxLAN 拼接和计算节点的虚拟交换机建立连接，VxLAN报文在虚拟交换机上解除封装进入VPC。

    VxLAN/EVPN 技术是目前大规模云数据中心网络通用且高效的业务承载方案，能够实现云内业务快速发送和自动化配置。后续随着 SRv6技术标准的成熟，SRv6/EVPN的统一承载方案会逐渐向数据中心内网络演进。目前，Linux已经支持大部分SRv6功能，Linux SRv6提供一种整合overlay和underlay的承载方案，保证underlay网络和主机叠加网络（host overlay）SLA的一致性。在数据中心中引入SRv6承载，还需进行大量的研究和实践。

    ps: 当前在云数据中心互联场景中，IP 骨干网采用 MPLS/SR-MPLS 技术，数据中心网络通常使用 VxLAN 技术，骨干网与数据网络之间通过网关设备实现 VxLAN 和 MPLS 相互映射.
- [G-SRv6技术详细介绍与应用场景解析.pdf](/misc/img/net/srv6/G-SRv6技术详细介绍与应用场景解析.pdf)
- [使用AF_XDP Socket更高效的网络传输](https://colobu.com/2023/04/17/use-af-xdp-socket/)
- [每秒1百万的包传输，几乎不耗CPU的那种](https://colobu.com/2023/04/02/support-1m-pps-with-zero-cpu-usage/)

## 实现
- [Linux 网络栈监控和调优：发送数据](https://colobu.com/2019/12/09/monitoring-tuning-linux-networking-stack-sending-data/)
- [如何用Go实现一个异步网络库？](https://zhuanlan.zhihu.com/p/544038899)
- [一文理解 OpenStack 网络](https://my.oschina.net/u/4526289/blog/5544689)

    在每个VM大门口, 增加一个Bridge网桥, 任何VM的流量，都会经过这个Bridge. 这样通过在Bridge上面, 增加iptables规则, 就可以达到给VM设置**安全组**的目的了.

    网络节点需要使用ns隔离不同租户.

    [metadata服务，就是允许每个VM去访问OpenStack平台获取自己的metadata](http://niusmallnan.com/_build/html/_templates/openstack/metadata_server.html).

    ```bash
    $ curl http://169.254.169.254 # 在OpenStack上，将OpenStack的地址固定为169.254.169.254
    $ curl http://169.254.169.254/openstack/latest/user_data # 可做VM自动化, 类似于cloud-init
    #!/bin/bash
    echo 'Extra user data here'
    ```

    网络节点上有一个proxy进程监听者9697端口，会将“访问openstack的请求”，转给了本地 unix domain socket 的监听者（即agent）, 由该agent与控制节点通信.

    网络节点的常用命令:
    ```bash
    # ip netns # 查询ns
    qdhcp-a7e512cf-1ca0-4ec7-be75-46a8998cf9ca
    qrouter-4cdb0354-7732-4d8f-a3d0-9fbc4b93a62d
    # ip netns exec qrouter-4cdb0354-7732-4d8f-a3d0-9fbc4b93a62d ip address # 查询该ns下的网卡
    # ip netns exec qrouter-4cdb0354-7732-4d8f-a3d0-9fbc4b93a62d iptables -t nat -S # 查看该ns下的iptables规则
    ```

    为了降低网络节点的负载，同时提高可扩展性(避免网络节点异常), OpenStack在Juno版本引入了DVR(Distributed Virtual Routing)特性, DVR部署在计算节点上. 计算节点上的VM使用floatingIP访问Internet, 不必经过网络节点, 直接从计算节点的DVR就可以访问. 这样网络节点只需要处理占到整体流量一部分的 SNAT （无 floating IP 的 vm 跟外面的通信）流量，大大降低了负载和整个系统对网络节点的依赖, 即实现了分布式路由.

- [**跟唐老师学云网络**](https://bbs.huaweicloud.com/blogs/109721)
- [跟唐老师学习云网络 - 什么是VLAN和VXLAN](https://bbs.huaweicloud.com/blogs/111665)

    一般情况, 一个交换机端口，都是设置为只允许一种VLAN报文通过(access模式). 但是有时候, 需要设置一个端口，允许N种VLAN报文，都可以通过, 即trunk模式, 这样可避免两个switch间需要给每个lan连线.

    > vlan的vid是12b, 也就是最大4095.
- [Nova中VIF(虚拟网卡)的实现](http://niusmallnan.com/_build/html/_templates/openstack/nova_vif.html)

## cloud
- [VXLAN vs VLAN](https://zhuanlan.zhihu.com/p/36165475)

    vxlan已成为目前网络虚拟化overlay的事实标准.
- [VXLAN 基础教程：在 Linux 上配置 VXLAN 网络](https://juejin.cn/post/6844904133430870029)
- [sdn-handbook](https://tonydeng.github.io/sdn-handbook/)
- [云原生网络数据面加速方案浅析](https://bbs.huaweicloud.com/forum/thread-95490-1-1.html)

    用户态CNI可以基于DPDK、AF_XDP两种技术实现，两种技术各有优劣，互补关系，所以需要考虑同时支持DPDK、AF_XDP两种技术.

    裸机上针对VM间通信加速，DPDK基本处于垄断地位，留给AF_XDP的空间很小，针对这种场景的性能改进工作，想象空间很小. 如果从硬件可获得角度看，可以改用AF_XDP代替DPDK，性能上略有差距，但可以弥补硬件不可获得的缺陷.
- [VXLAN](https://support.huawei.com/enterprise/zh/doc/EDOC1100023543?section=j016)
- [**IP报文在阿里云上的神奇之旅系列一：同地域内云上通信**](https://developer.aliyun.com/article/1054404)

## Protocol
- [【重识云原生】第四章云网络4.3.2节——VLAN技术](https://blog.csdn.net/junbaozi/article/details/124956412)

## SDN
- [【重识云原生】第四章云网络4.8.1节——SDN总述](https://cloud.tencent.com/developer/article/2046685)
- [【重识云原生】第四章云网络4.8.2.1节——OpenFlow概述](https://cloud.tencent.com/developer/article/2046686)
- [【重识云原生】第四章云网络4.8.2.2节——OpenFlow协议详解](https://cloud.tencent.com/developer/article/2046688)
- [【重识云原生】第四章云网络4.8.2.3节——OpenFlow运行机制](https://cloud.tencent.com/developer/article/2046692)
- [【重识云原生】第四章云网络4.8.3.1节——Open vSwitch简介](https://cloud.tencent.com/developer/article/2046695)

    vSwitch的早期代表是Linuxbridge，它在设计之初就是为了提供基本的网络连接，因此它只是模拟了ToR交换机的行为，并将自己接入到现有的物理网络中。这样实现的优点是，现有物理网络的理论和协议可以直接套用，不需要重复设计。缺点就是，作为物理网络的延伸，使得虚拟workload的网络与物理网络紧耦合，影响了虚拟化本身带来的诸如灵活，快速上线的优势。

    随着网络虚拟化（network virtualization）技术的出现，为连接虚拟workload的网络提供了另一种可能。物理网络仍然由物理网络设备管理，虚拟workload的网络，单独由vSwitch管理，并且在现有物理网络（underlay）基础之上，再定义一个独立的overlay网络（例如VxLAN）。这个overlay网络不受物理网络设备控制，完全由vSwitch控制.
- [【重识云原生】第四章云网络4.8.3.2节——Open vSwitch工作原理详解](https://cloud.tencent.com/developer/article/2046698)
- [【重识云原生】第四章云网络4.8.4节——OpenStack与SDN的集成](https://cloud.tencent.com/developer/article/2046701)
- [【重识云原生】第四章云网络4.8.5节——OpenDayLight](https://cloud.tencent.com/developer/article/2046704)

     ODL项目成立于2013年4月，是由Linux基金会管理管理的开源SDN项目。项目的目的是提供一个开放的，全功能的，厂商中立的SDN解决方案.
- [【重识云原生】第四章云网络4.8.6节——Dragonflow](https://blog.csdn.net/junbaozi/article/details/125530931)

    Dragonflow目的是提供全功能的SDN解决方案.
## offload
- [【重识云原生】第四章云网络4.9.1节——网络卸载加速技术综述](https://cloud.tencent.com/developer/article/2046707)

    OpenStack Pike版本中引入了对switchdev的支持，实现了Open vSwitch硬件卸载offloading功能.

    目前业界主流智能网卡有四种实现方案：SoC、NP、FPGA、ASIC。

    1. SoC方案在终端市场应用较成熟，硬件需要根据客户需求定制，部署周期较长，但是计算效率高，适合成熟算法及应用，功耗较低。
    1. NP方案生态封闭，主流厂商已不再发布路标，不支持重编程，难以解耦，成本高于FPGA，但是功耗较低。
    1. FPGA方案生态开放，在数据中心场景中得到广泛应用，可以重复编程实现特定应用，适合演进中的算法及应用，适用于网络转发等并行计算场景，该方案处理时延低，支持虚拟化，功耗适中。
    1. ASIC方案，其硬件根据用户需求定制，开发成本昂贵，生产周期长，不具备灵活性，但是计算效率高，功耗较低，适合大规模成熟算法及应用。


    智能网卡网络加速的技术实现:
    1. 腾讯智能网卡采用FPGA+SoC架构，网络加速技术实现方面自研vDPA，支持VIRTIO-net卸载，能够实现虚拟化性能零损耗，数据面直通，软硬结合跟踪脏页的功能。
- [【重识云原生】第四章云网络4.9.3.1节——DPDK技术综述](https://cloud.tencent.com/developer/article/2099341)
- [【重识云原生】第四章云网络4.9.3.2节——DPDK原理详解](https://cloud.tencent.com/developer/article/2099589)
- [【重识云原生】第四章云网络4.9.4.1节——智能网卡SmartNIC方案综述](https://cloud.tencent.com/developer/article/2099708)
- [【重识云原生】第四章云网络4.9.4.2节——智能网卡实现](https://cloud.tencent.com/developer/article/2099530)

    Smart NIC 的实现方式见[2021中国DPU行业发展白皮书](https://www.zhihu.com/search?q=%E8%B5%9B%E8%BF%AA%E9%A1%BE%E9%97%AE&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A2279317557%7D)
- [【重识云原生】第四章云网络4.9.4.3节——智能网卡使用场景-网络加速实现](https://cloud.tencent.com/developer/article/2101721)
- [【重识云原生】第四章云网络4.9.5.1节下一代智能网卡——DPU综述](https://cloud.tencent.com/developer/article/2101722)

    ![DPU竞争格局, 各家实现 from 赛迪顾问](https://ask.qcloudimg.com/http-save/yehe-9519988/1084dd572b6eaef39f914ac1b18c55d3.png?imageView2/2/w/1620)
- [【重识云原生】第四章云网络4.9.6节——linux switchdev技术](https://cloud.tencent.com/developer/article/2108936)

## 限速
- [网卡限速 by github.com/magnific0/wondershaper (use tc)](https://www.cnblogs.com/Dy1an/p/12170515.html)

## 测试
- [vxlan网络性能测试](https://plantegg.github.io/2018/08/21/vxlan%E7%BD%91%E7%BB%9C%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95/)

## 资源
- [www.IPv6Plus.net/resources](https://github.com/IPv6Plus/IPv6Plus.github.io)

# 概念
## 工作组 Work Group
在一个网络内，可能有成百上千台电脑，如果这些电脑不进行分组，都列在“网上邻居”内，可想而知会有多么乱. 为了解决这一问 题，Windows就引用了“工作组”这个概念，将不同的电脑一般按功能分别列入不同的组中，如财务部的电脑都列入“财务部”工作组中，人事部的电脑都 列入“人事部”工作组中. 要访问某个部门的资源，就在“网上邻居”里找到那个部门的工作组名，双击就可以看到那个部门的电脑了.

## 域 Domain
与工作组的“松散会员制”有所不同，“域”是一个相对严格的组织。“域”指的是服务器控制网络上的计算机能否加入的计算机组合.

实行严格的管理对网络安全是非常必要的。在对等网模式下，任何一台电脑只要接入网络，就可以访问共享资源，如共享ISDN上网等。尽管对等网络上的共享文件可以加访问密码，但是非常容易被破解。

在“域”模式下，至少有一台服务器负责每一台联入网络的电脑和用户的验证工作，相当于一个单位的门卫一样，称为“域控制器（Domain Controller，简写为DC）”。“域控制器”中包含了由这个域的账户密码、管理策略等信息构成的数据库。当电脑联入网络时，域控制器 首先要鉴别这台电脑是否是属于这个域的，用户使用的登录账号是否存在、密码是否正确。如果以上信息不正确，域控制器就拒绝这个用户从这台电脑登录。不能登 录，用户就不能访问服务器上有权限保护的资源，只能以对等网用户的方式访问Windows共享出来的资源，这样就一定程度上保护了网络上的资源。

想把一台电脑加入域，必须要由网络管理员进行把这台电脑加入域的相关操作。操作过程由服务器端设置和客户端设置构成.

一般情况下，域控制器集成了DNS服务，可以解析域内的计算机名称（基于TCP/IP），解决了工作组环境不同网段计算机不能使用计算机名互访的问题.

> **域就像中央集权，由一台或数台域控制器（Domain Controller, 域管理的核心/门卫）管理域内的其他计算机；工作组就像各自为政，组内每一台计算机自己管理自己，他人无法干涉.**
> 网络中运行 Windows 的计算机必须属于某个工作组或某个域.