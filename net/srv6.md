# SRv6
参考:
- [SRv6浅谈](https://cloud.tencent.com/developer/article/1420074)
- [kernel support srv6 status](https://www.segment-routing.net/open-software/linux/)


## [从MPLS到SR，再到SRv6](https://mp.weixin.qq.com/s?__biz=MzIwMTMzMDU5Nw==&mid=2456798735&idx=1&sn=d2d9050d20b22bb64e2080e5a6ab257c)
MPLS全称是Multi-Protocol Label Switching，直译过来就是多协议标签交换，它是操作在OSI的2层（数据链路层）和3层（网络层）之间的数据转发技术.

MPLS对技术是做了加法，MPLS是通过在原有IGP协议基础上增加LDP协议来实现标签的分发，又因为LDP不具有流量工程，增加RSVP-TE. 然而RSVP信令非常复杂，同时还得维护庞大的链路信息，因此信息交互效率低下，扩展也非常困难.

也正是在这样的背景下，SR（Segment Routing）技术应运而生.

> SR已经成为事实上的SDN架构标准

SR简单总结就是"一减一集中":
- 减

    既然LDP不维护状态信息，只对IGP中的目的IP和MPLS标签做了一层映射，本质上是依靠IGP协议制定标签转发，SR技术干脆直接使用对IGP协议扩展SR属性，不再部署LDP协议，由IGP来分发标签

- 集中

    既然每个节点都要通过RSVP进行大量的交互以维护全网的状态信息，干脆把RSVP功能集中起来，不用每个节点都计算交互, 这也正谙合了SDN的思想，集中控制管理，可以说SR天然的支持SDN.

但在这需要强调的是，上面的SR在数据平面仍然是基于MPLS的，无论控制面分发标签是基于IPv4还是IPv6，从本质上来说还是MPLS下的Segment Routing，也就是SR-MPLS.

与SR-MPLS相比，传统的SR-MPLS是在MPLS的基础上运用了减法和集中的思想，减去LDP集中RSVP. 而SRv6则是在传统SR-MPLS基础上，给我们带来了大一统和编程的思想:
1. 与传统SR-MPLS的3层类型标签（VPN/BGP/SR）相比，SRv6在标签分层上更为简单，只有一种IPv6头，以此实现统一的转发. 另外，由于SRv6帧头的标准性，使得它更能兼容现网的IPv6设备，当中间节点不支持SRv6功能时，也可以根据IPv6路由方式来转发报文.
1. SRv6 SRH扩展中128位SID特殊的帧结构中定义了Function字段

    Function字段支持编程自定义，可以根据业务需要灵活地定义任意功能和业务. 比如说，可以定义Function为END，表示此SID标识的是一个节点，当定义为END.X时，表示此SID标识的是一段路径信息.

总结一下，从MPLS到SR（SR-MPLS），通过IGP扩展SR属性省略了LDP协议，并实现基于源地址标签转发的集中控制; 从SR到SRv6，通过在IPv6中增加SRH字段，实现基于IPv6的标签转发，替代传统的MPLS下的标签转发功能.

至少在目前来说，**SRv6提供了可预见的网络业务变革的最终形态**.

## SRv6

SRv6是一种网络转发技术，其中SR是Segment Routing的缩写，v6顾名思义是指IPv6.

它是一种源路由技术，基于SDN理念，构成面向路径连接的网络架构，支撑未来网络多层次的可编程需求，可以满足5G超大连接和切片的应用场景下的连接需求.

SRv6是直接在IPv6的IP扩展头中进行新的扩展，这个扩展部分称为[SRH, Segment Routing Header](https://datatracker.ietf.org/doc/rfc8754/)，而这部分扩展没有破坏标准的IP头，因此可以认为SRv6是一种native的IPv6技术.

> [SRv6 uSID在任何情况下的协议开销都要低于VxLAN over SR-MPLS.](https://www.cisco.com/c/dam/global/zh_cn/solutions/service-provider/segment-routing/pdf/usid_srv6.pdf)

Linux SRv6可以完美整合Overlay和Underlay，因为无论是Overlay还是Underlay，本质上都是对应着不同的SRv6 Segment（操作）而已；如果需要提高Linux SRv6性能，可以优化DPDK或者使用FD.IO(VPP)，其中FD.IO已经内置了完善的SRv6支持，在使用上会更为方便.

## SRv6 Segment
SRv6的Segment有128bits，而且分成了三部分:
1. Locator（位置标识）

    网络中分配给一个网络节点的标识，可以用于路由和转发数据包. Locator有两个重要的属性，可路由和聚合. 在SRv6 SID中Locator是一个可变长的部分，用于适配不同规模的网络.
2. Function（功能）

    设备分配给本地转发指令的一个ID值，该值可用于表达需要设备执行的转发动作，相当于计算机指令的操作码. 在SRv6网络编程中，不同的转发行为由不同的功能ID来表达. 一定程度上功能ID和MPLS标签类似，用于标识VPN转发实例等.
3. Args（变量）

    转发指令在执行的时候所需要的参数，这些参数可能包含流，服务或任何其他相关的可变信息

从SRv6 SID的组成来看，SRv6同时具有路由和MPLS两种转发属性，可以融合两种转发技术的优点.

## Segment Routing Header（SRH）
参考:
- [SRv6技术课堂（一）：SRv6概述](https://www.sdnlab.com/23735.html)

为了在IPv6报文中实现SRv6转发，引入了一个SRv6扩展头（Routing Type为4），叫Segment Routing Header（SRH），用于进行Segment的编程组合形成SRv6路径.

SRv6的报文封装格式: 绿色的是IPv6报文头，棕色部分是SRH，蓝色是报文负荷:
![](/misc/img/net/srv6/11SRv602.png)

IPv6 Next Header字段取值为43，表示后接的是IPv6路由扩展头. Routing Type = 4，表明这是SRH的路由扩展头，这个扩展头里字段解释如下:
![](/misc/img/net/srv6/11SRv6biao1.jpeg)

### SRv6三层编程空间
SRv6具有比SR-MPLS更强大的网络编程能力. [SRv6的网络可编程性](https://datatracker.ietf.org/doc/draft-ietf-spring-srv6-network-programming/)体现在SRH扩展头中. SRH中有三层编程空间:
![](/misc/img/net/srv6/11SRv603.png)

第一部分是Segment序列. 它可以将多个Segment组合起来，形成SRv6路径, 这跟MPLS标签栈比较类似.

第二部分是对SRv6 SID的128比特的运用.

    众所周知，MPLS标签封装主要是分成四个段，每个段都是固定长度（包括20比特的标签，8比特的TTL，3比特的Traffic Class和1比特的栈底标志）, 而SRv6的每个Segment是128比特长，可以灵活分为多段，每段的长度也可以变化，由此具备灵活编程能力.

第三部分是是紧接着Segment序列之后的可选TLV（Type-Length-Value）.

    报文在网络中传送时，需要在转发面封装一些非规则的信息，它们可以通过SRH中TLV的灵活组合来完成.

SRv6通过三层编程空间，具备了更强大的网络编程能力，可以更好地满足不同的网络路径需求.

### SRv6报文转发流程
![](/misc/img/net/srv6/11SRv604.png)
上图展示了SRv6转发的一个范例. 在这个范例中，结点R1要指定路径（需要通过R2-R3、R4-R5的链路转发）转发到R6，其中R1、R2、R4、R6为有SRv6能力的的设备，R3、R5为不支持SRv6的设备.

步骤一：Ingress结点处理：R1将SRv6路径信息封装在SRH扩展头，指定R2和R4的END.X SID，同时初始化SL = 2，并将SL指示的SID A2::11拷贝到外层IPv6头目的地址. R1根据外层IPv6目的地址查路由表转发到R2.

步骤二：End Point结点处理：R2收到报文以后，根据外层IPv6地址A2::11查找本地Local SID表，命中END.X SID，执行END.X SID的指令动作：SL—，并将SL指示的SID拷贝到外层IPv6头目的地址，同时根据END.X关联的下一跳转发.

步骤三：Transit结点处理：R3根据A4::13查IPv6路由表进行转发，不处理SRH扩展头, 具备普通的IPv6转发能力即可.

步骤四：End Point结点处理：R4收到报文以后，根据外层IPv6地址A4::13查找本地Local SID表，命中END.X SID，执行END.X SID的指令动作：SL—，并将SL指示的SID拷贝到外层IPv6头目的地址，由于SL = 0, 弹出SRH扩展头，同时根据END.X关联的下一跳转发.

步骤5：弹出SRH扩展头以后，报文就变成普通的IPv6头，由于A6::1是1个正常的IPv6地址，遵循普通的IPv6转发到R6.

从上面的转发可以看出，对于支持SRv6转发的节点，可以通过SID指示经过特定的链路转发，对于不支持SRv6的节点，可以通过普通的IPv6路由转发穿越, 这个特性使得SRv6可以很好地在IPv6网络中实现增量部署.

## SRv6的标准和产业进展
SRv6的标准化基本上分为两大部分：
第一部分是SRv6基础特性，包括SRv6网络编程框架、报文封装格式SRH以及IGP、BGP/VPN、BGP-LS、PCEP等基础协议扩展支持SRv6，主要提供VPN、TE、FRR等应用。所有SRv6基本特性文稿均由华为和思科共同引领，并有Bell Canada、SoftBank、Orange等运营商参与.

第二部分是SRv6面向5G和云的新应用，这些应用包括网络切片、确定性时延（DetNet）、OAM、IOAM（In-situ OAM）、SFC、SD-WAN、组播/BIER等。这些应用都对网络编程提出了新的需求，需要在转发面封装新的信息。SRv6可以很好地满足这些需求，充分体现了其在网络编程能力方面具备的独特优势。

## [uSID](https://datatracker.ietf.org/doc/draft-filsfils-spring-net-pgm-extension-srv6-usid/)
参考:
- [uSID：SRv6新范式](https://www.sdnlab.com/23390.html)

uSID兼容既有的SRv6框架，将极大地改变SRv6的设计、实现和部署方式，成为SRv6的新范式.

SRv6在网络可编程性方面有着巨大的优势，但要发挥其优势，需要迫切解决SRv6在协议开销、承载效率、MTU和对硬件要求方面的问题. 而uSID可以彻底解决上述的协议开销、承载效率、MTU和对硬件要求高方面的问题.

SRv6则是天生整合了Overlay和Underlay，无须 在Overlay和Underlay间进行复杂的交接，这极大地降低了网络复杂性和业务复杂性.

## SRv6操作(不全)
参考:
- [看看有趣的SRv6 Ⅰ：SRv6网络编程](https://www.jianshu.com/p/6fd84ec5d06a)

- End : 该操作要求Segment Left不为0（不是最后一跳），会将Segment Left减1，并更新IPv6数据包的目的地址为下一个Segment，这是最常见的SRv6操作。相当于SR MPLS中的Prefix-SID
- End.X：该操作和End操作基本一致，区别是可以将处理过的数据包发送到指定的下一跳地址。相当于SR MPLS中的Adj-SID
- End.DX4：该操作要求Segment Left为0且数据包内封装了IPv4数据包，会去掉外层的IPv6报头，并将内部的IPv4数据包转发给指定的下一跳地址。相当于VPNv4 Per-CE标签
- End.DX6：该操作要求Segment Left为0且数据包内封装了IPv6数据包，会去掉外层的IPv6报头，并将内部的IPv6数据包转发给指定的下一跳地址。相当于VPNv6 Per-CE标签
- End.B6：该操作会在已有的SRH的基础上，插入一个新的SRH，并可以定义新的Segment列表，数据包将首先按照插入的新的SRH进行转发。相当于Binding-SID
- End.B6.Encaps：该操作和End.B6基本一致，区别是该操作将在数据包外层新增一个新的IPv6报头和SRH，而不是仅仅添加一个SRH的路由报头