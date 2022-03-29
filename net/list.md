# list
## new ip
- [New IP：开拓未来数据网络的新连接和新能力](http://www.infocomm-journal.com/dxkx/article/2019/1000-0801/1000-0801-35-9-00002.shtml)
- [NEW IP Framework and Protocol for Future Applications](/misc/pdf/net/6f569c60-7045-11ea-89df-41bea055720b.pdf)

## quic
- [面向5G的阿里自研标准化协议库XQUIC](https://developer.aliyun.com/article/770062)
- [The Road to multipath QUIC: 阿里自研多路径传输技术XLINK](https://www.bilibili.com/read/cv11319597)
- [XLINK: QoE-Driven Multi-Path QUIC Transport in Large-scale Video Services](http://www.hongqiangliu.com/uploads/5/2/7/4/52747939/sigcomm2021-xlink.pdf)

## next
- [阿里云如何构建高性能云原生容器网络](https://yq.aliyun.com/articles/755848)
- [阿里云如何构建高性能云原生容器网络-直播](https://yq.aliyun.com/live/2626)
- [超大规模云网络数据中心创新（上/下）](https://www.51openlab.com/article/28/)

    - 大型 OTT纷纷开始在其数据中心中采用CLOS架构，利用数量众多，小规模的基本交换矩阵（CrossBar）来构建超大规模网络Fabric.
    - SRv6 Overlay是趋势
- [SRv6开启网络服务化变革之旅](https://cloud.tencent.com/developer/article/1779041)

## 实现
- [Linux 网络栈监控和调优：发送数据](https://colobu.com/2019/12/09/monitoring-tuning-linux-networking-stack-sending-data/)

## cloud
- [VXLAN vs VLAN](https://zhuanlan.zhihu.com/p/36165475)
- [sdn-handbook](https://tonydeng.github.io/sdn-handbook/)
- [云原生网络数据面加速方案浅析](https://bbs.huaweicloud.com/forum/thread-95490-1-1.html)

    用户态CNI可以基于DPDK、AF_XDP两种技术实现，两种技术各有优劣，互补关系，所以需要考虑同时支持DPDK、AF_XDP两种技术.

    裸机上针对VM间通信加速，DPDK基本处于垄断地位，留给AF_XDP的空间很小，针对这种场景的性能改进工作，想象空间很小. 如果从硬件可获得角度看，可以改用AF_XDP代替DPDK，性能上略有差距，但可以弥补硬件不可获得的缺陷.

## 限速
- [网卡限速 by github.com/magnific0/wondershaper (use tc)](https://www.cnblogs.com/Dy1an/p/12170515.html)

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