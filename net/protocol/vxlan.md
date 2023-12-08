# vxlan
ref:
- [【重识云原生】第四章云网络4.3.10节——VXLAN技术](https://cloud.tencent.com/developer/article/2020852)
- [重识云原生】第四章云网络4.3.10.2节——VXLAN Overlay网络方案设计](https://cloud.tencent.com/developer/article/2020853)
- [【重识云原生】第四章云网络4.3.10.3节——VXLAN隧道机制](https://cloud.tencent.com/developer/article/2022587)
- [【重识云原生】第四章云网络4.3.10.4节——VXLAN报文转发过程](https://cloud.tencent.com/developer/article/2024108)
- [【重识云原生】第四章云网络4.3.10.5节——VXlan组网架构](https://cloud.tencent.com/developer/article/2024109)
- [【重识云原生】第四章云网络4.5节——大二层网络](https://cloud.tencent.com/developer/article/2030941)

	大二层网络基本上都是针对云时代下数据中心场景的，因为它实际上就是为了解决数据中心的服务器虚拟化之后的虚拟机动态迁移这一特定需求而出现的。对于普通的园区网之类网络而言，大二层网络并没有特殊的价值和意义（除了某些特殊场景，例如WIFI漫游等等）.

	一旦服务器跨二层网络迁移，就需要变更IP地址，那么原来这台服务器所承载的业务就会中断，而且牵一发动全身，其他相关的服务器也要变更相应的配置，影响巨大. 为了保证虚拟机迁移过程中业务不中断，则需要保证虚拟机的IP地址、MAC地址等参数保持不变，这就要求虚拟机迁移必须发生在一个二层网络中。而传统的二层网络，将虚拟机迁移限制在了一个较小的局部范围内.
- [【重识云原生】第四章云网络4.4节——Spine-Leaf网络架构](https://cloud.tencent.com/developer/article/2026440)

vxlan是当前overlay网络的标准.

在VXLAN中，封装和解封的组件有个专有的名字叫做VTEP，VTEP之间通过组播发现对方.

## VXLAN Offload
一些新型号的网卡(Intel X540 or X710)，具备VXLAN硬件封包／解包能力。开启硬件VXLAN offload，并使用较大的MTU（如9000），可以明显提升虚拟网络的性能

## 转发
1. 同VXLAN ID内转发

	VXLAN最早依靠组播泛洪的方式来转发，但这会导致产生大量的组播流量. 在实际生产中，通常使用SDN控制器结合南向协议来避免组播问题.
1. 不同VXLAN ID转发

	大致转发过程与上面类似，所不同的是需要报文在所属vtep或者vxlan gateway处来转换vxlan id（源和目的处都需要做这个转换）
1. VXLAN与非VXLAN（如VLAN）转发

	需要VXLAN Gateway来转换vxlan vni和vlan id
