# sdn
openflow作为sdn的主要实现, 它的发展史就是SDN的发展史. 因为基于OpenFlow为网络带来的可编程的特性，斯坦福的Nick McKeown教授和他的团队进一步提出了SDN（Software Defined Network，软件定义网络）的概念.

![OpenFlow控制转发分离架构](/misc/img/net/Image00001_net.jpg)

## openflow
### OpenFlow设计思路
OpenFlow协议的思路是网络设备维护一个FlowTable，并且只通过FlowTable对报文进行处理，FlowTable本身的生成、维护和下发完全由外置的控制器实现. 此外，OpenFlow 交换机把传统网络中完全由交换机或路由器控制的报文转发，转换为由交换机和控制器共同完成，从而实现报文转发与路由控制的分离. 控制器则通过事先规定好的接口操作OpenFlow交换机中的流表，达到数据转发的目的.

在 OpenFlow 交换机中，包含了安全通道、多级流表和组表. 通过安全通道，OpenFlow 交换机可以和控制器建立基于 OpenFlow 协议的连接；而流表则用来匹配OpenFlow交换机收到的报文；组表用来定义流表需要执行的动作.

### FlowTable
OpenFlow 通过用户定义的或预设的规则匹配和处理网络包. 一条 OpenFlow 的规则由匹配域、优先级、处理指令和统计数据等字段组成.

在一条规则中，可以根据网络包在L2、L3或者L4等网络报文头的任意字段进行匹配，比如以太网帧的源MAC地址、IP包的协议类型和IP地址或者TCP/UDP的端口号等.

所有OpenFlow的规则都被组织在不同的FlowTable中，而在同一个FlowTable中，按规则的优先级进行先后匹配。一个 OpenFlow Switch 可以包含一个或者多个FlowTable，从0开始依次编号排列.

![数据包处理流程](/misc/img/net/Image00003_net.jpg)

当网络数据包进入Switch后，必须从table 0开始依次匹配，table可以按从小到大的次序越级跳转，但不能从某一table向前跳转至编号更小的table. 当数据包成功匹配一条规则后，将首先更新该规则对应的统计数据（如成功匹配数据包总数目和总字节数等），然后根据规则中的指令进行相应操作，比如跳转至后续某一table继续处理，修改或立即执行该数据包对应的Action Set等. 当数据包已经处于最后一个table时，其对应的Action Set中的所有Action将被执行，包括转发至某一端口、修改数据包的某一字段、丢弃数据包等. OpenFlow规范对目前所支持的Instructions和Actions进行了完整详细的说明和定义.

### OpenFlow通信通道
OpenFlow 协议主要通过对不同类型消息的处理来实现控制器与交换机之间的路由控制.目前，OpenFlow 主要支持三种消息类型，分别是 Controller-to-Switch、Asynchronous（异步消息）及Symmetric（对称消息）:
- Controller-to-Switch：指由 Controller 发起，Switch 接收并处理的消息，主要包括Features、Configuration、Modify-State、Read-Stats和Barrier等消息. 这些消息主要由Controller对Switch进行状态查询和修改配置等操作.
- Asynchronous：由Switch发送给Controller，用来通知Switch上发生的某些异步事件的消息，主要包括Packet-in、Flow-Removed、Port-Status和Error等. 例如，当某一条规则因为超时而被删除时，Switch 将自动发送一条Flow-Removed消息通知Controller，以方便Controller进行相应的操作，比如重新设置相关规则.
- Symmetric：主要用来建立连接，检测对方是否在线等，都是些双向对称的消息，包括Hello、Echo与厂商自定义消息.

### OpenFlow应用
随着OpenFlow以及SDN的发展和推广，其研究和应用领域也得到了不断拓展，比如网络虚拟化、安全和访问控制、负载均衡、绿色节能，以及与传统网络设备交互和整合等. 下面重点介绍网络虚拟化和负载均衡.

1. 网络虚拟化——FlowVisor

网络虚拟化的本质是对底层网络的物理拓扑进行抽象，在逻辑上对网络资源进行分片或整合，从而满足各种应用对于网络的不同需求.

为了达到这个目的，FlowVisor实现了一个特殊的OpenFlow Controller，可以看作其他不同用户或应用的Controller与网络设备之间的一层代理. 因此，不同用户或应用可以使用自己的Controller来定义不同的网络拓扑，同时FlowVisor又可以保证这些Controller之间能够互相隔离且互不影响.

![](/misc/img/net/Image00005_net.jpg)

1. 负载均衡——Aster*x
传统的负载均衡方案一般需要在服务器集群的入口处，通过一个gateway监测、统计服务器的工作负载，并据此将用户请求动态地分配到负载相对较轻的服务器上. 既然网络中的所有网络设备都可以通过OpenFlow进行集中式的控制和管理，同时服务器的负载又可以及时地反馈给OpenFlow Controller，那么OpenFlow就非常适合做负载均衡的工作.

基于OpenFlow的负载均衡模型Aster*x通过Host Manager和Net Manager来分别监测服务器和网络的工作负载，然后将这些信息反馈给FlowManager，这样 Flow Manager 就可以根据这些实时的负载信息，重新定义网络设备上的OpenFlow规则，从而将用户请求（即网络包）按照服务器的能力进行调整和分发

![](/misc/img/net/Image00006_net.jpg)

## sdn
SDN 将控制功能从交换机中剥离出来，形成了一个统一的、集中式的**控制平面**，而交换机只保留了简单的转发功能，从而形成了**转发平面（数据平面）**. 通过控制平面对数据平面的集中化控制，SDN 为网络提供了开放的编程接口，并实现了灵活的可编程能力，从而使网络能够真正地被软件定义，达到按需定制服务、简化网络运维、灵活管理调度的目标.

在SDN中，网络设备只负责单纯的数据转发，可以采用通用的硬件. 如果将网络中所有的网络设备视为被管理的硬件资源，参考操作系统的设计原理，则可以抽象出一个网络操作系统（Network OS）的概念. 这个网络操作系统一方面抽象了底层网络设备的具体细节，负责与网络硬件进行交互，实现对硬件的编程控制和接口操作，同时还为上层应用访问网络设备提供了统一的管理视图和编程接口. 基于这个网络操作系统，用户可以开发各种网络应用程序，通过软件定义逻辑上的网络拓扑，以满足对网络资源的不同需求，而无须关心底层网络的物理拓扑结构.

![SDN基本架构](/misc/img/net/Image00007_net.jpg)

SDN 采用了上图所示的基本架构，集中式的控制平面和分布式的转发平面相互分离，控制平面利用控制器、转发通信接口对转发平面上的网络设备进行集中式管理.

sdn组件包括:
- 基础设施层（Infrastructure Layer）：主要承担数据转发功能，由各种网络设备构成，如数据中心的网络路由器，支持OpenFlow 的硬件交换机等。
- 控制层（Control Layer）：网络转发的控制管理平面，负责管理网络的基础设施，主要组成部分为SDN控制器。SDN控制器是整个网络的大脑、控制中心，主要功能是按照配置的业务逻辑，产生对应的数据平面的流转发规则，通过下发给网络设备，控制其进行数据转发。
- 应用层（Application Layer）：指商业应用。开发者可以通过SDN控制器提供的北向接口，如 REST 接口实现应用和网络的联动，例如网络拓扑的可视化、监控等。
- 南向接口（Sorthbound Interface）:SDN 控制器对网络的控制主要通过OpenFlow、NetConf等南向接口实现，包括链路发现、拓扑管理、策略制定、表项下发等。其中，链路发现和拓扑管理主要是控制其利用南向接口的上行通道对底层交换设备上报信息进行统一的监控和统计，而策略制定和表项下发则是控制器利用南向接口的下行通道对网络设备进行统一的控制。
- 北向接口（Northbound Interface）：北向接口是通过控制器向上层应用开放的接口，其目标是使得应用能够便利地调用底层的网络资源和能力。因为北向接口是直接为应用服务的，因此其设计需要密切联系应用的业务需求，比如需要从用户、运营商或产品的角度去考量。

在SDN发展初期，控制平面的表现形式更多是以单实例的控制器出现，实现SDN的协议也以 OpenFlow 为主，因此 SDN控制器更多指的是 OpenFlow 控制器. 随着SDN的发展，ONF也在白皮书中提出了SDN的架构标准. 广义的SDN支持丰富的南向协议，包括OpenFlow、NetConf、OVSDB、BGPLS、PCEP及厂商协议等，可实现灵活可编程和灵活部署，支持网络虚拟化、SR路由、智能分析和调度.

与南向接口方面已有OpenFlow等国际标准不同，目前还缺少业界公认的北向接口标准. 因此，北向接口的协议制定成为当前SDN领域竞争的一大焦点，不同的参与者或从各种角度提出了很多方案. 据悉，目前至少有20种控制器，每种控制器都会对外提供北向接口，用于上层应用开发和资源编排. 当然，对于上层的应用开发者来说，RESTful API是比较乐于采用的北向接口形式.

### SDN实现
OpenFlow并非实现SDN的唯一途径, 比如IETF定义的开放SDN架构, Overlay网络技术和基于专用接口.

#### IETF定义的开放SDN架构
IETF定义的开放SDN架构的核心思路是重用当前的技术而不是OpenFlow，比如利用Netconf和已有的设备接口. IETF的Netconf使用XML来配置设备，旨在减少与自动化设备配置有关的编程工作量. 这种架构充分地利用了现有设备，能够更大限度地保护已有的投资.

#### Overlay网络技术
Overlay网络技术是在现行的物理 IP 网络基础上建立叠加逻辑网络（Overlay Logical Network），屏蔽底层物理网络差异，实现网络资源的虚拟化，使得多个逻辑上彼此隔离的网络分区，以及多种异构的虚拟网络可以在同一共享的物理网络基础设施上共存.

![Overlay网络](/misc/img/net/Image00007_net.jpg)

Overlay 网络的主要思想可被归纳为解耦、独立、控制三个方面:
- 解耦是指将网络的控制从网络物理硬件中脱离出来，交给虚拟化的Overlay逻辑网络处理
- 独立是指Overlay网络承载于物理IP网络之上，因此只要IP可达，那么相应的虚拟化网络就可以被部署，而无须对原有物理网络架构（例如原有的网络硬件、原有的服务器虚拟化解决方案、原有的网络管理系统、原有的IP地址等）做出任何改变. Overlay网络可以便捷地在现网上部署和实施，这是它最大的优势.
- 控制是指叠加的逻辑网络将以软件可编程的方式被统一控制，网络资源可以和计算资源、存储资源一起被统一调度和按需交付.

Overlay是在现有的网络架构上叠加的虚拟化技术，本质是L2 Over IP的隧道技术. 细分Overlay的技术路线，主要可分为如下三大技术路线
- VXLAN

    VXLAN是将以太网报文封装在UDP传输层上的一种隧道转发模式，VXLAN通过将原始以太网数据头(MAC、IP、四层端口号等)的HASH值作为UDP的号. 采用24比特标识二层网络分段，称为VNI(VXLAN Network Identifier)，类似于VLAN ID作用.

    VXLAN是阿里云VPC采用的技术路线. 阿里云在硬件网关和自研交换机设备的基础上实现了VPC产品. 每个VPC都有一个独立的隧道号，一个隧道号对应着一个虚拟化网络. 一个VPC内的ECS（Elastic Compute Service）实例之间的传输数据包都会加上隧道封装，带有唯一的隧道ID标识，然后送到物理网络上进行传输. 不同VPC内的ECS实例因为所在的隧道ID不同，本身处于两个不同的路由平面，所以不同VPC内的ECS实例无法进行通信，天然地进行了隔离.
- NVGRE

    NVGRE是将以太网报文封装在GRE内的一种隧道转发模式。采用24比特标识二层网络分段，称为VSI，类似于VLAN ID作用。NVGRE在GRE扩展字段flow ID，这就要求物理网络能够识别到GRE隧道的扩展信息，并以flow ID进行流量分担.
- STT

    STT利用了TCP的数据封装形式，但改造了TCP的传输机制，数据传输不遵循TCP状态机，而是全新定义的无状态机制，将TCP各字段意义重新定义，无需三次握手建立TCP连接，因此称为无状态TCP.

这三种技术大体思路都是将以太网报文承载到某种隧道层面，差异性在于选择和构造隧道的不同，而底层均是IP转发. 总体而言，VXLAN利用了现有通用的UDP传输，成熟度高, 具有更明显的优势，L2-L4层链路HASH能力强，不需要对现有网络改造，对传输层无修改，使用标准的UDP传输流量，业界支持度最好，目前商用网络芯片大部分支持.

现在云服务商用的比较多的是基于主机(x86服务器)的Overlay技术，在服务器的Hypervisor内vSwitch上支持基于IP的二层Overlay技术. 主机的vSwitch支持基于IP的Overlay之后，虚拟机的二层访问直接构建在Overlay上，物理网络不再感知虚拟机的诸多特性，从而实现了和虚拟网络的解耦.

#### 基于专用接口
基于专用接口的方案的实现思路是不改变传统网络的实现机制和工作方式，通过对网络设备的操作系统进行升级改造，在网络设备上开发出专用的 API 接口，管理人员可以通过 API 接口实现网络设备的统一配置管理和下发，改变原先需要一台台设备登录配置的手工操作方式，同时这些接口也可供用户开发网络应用，实现网络设备的可编程. 典型的基于专用接口的SDN实现方案是的思科ONE架构.

### sdn转发平面的可编程性.
现有的SDN解决方案为用户开放的是控制平面的可编程能力，在正常情况下，对于转发设备来说，数据包的解析转发流程是由设备转发芯片固化的，所以设备在协议的支持方面并不具备扩展能力. 并且，厂商扩展转发芯片所支持的协议特性，甚至开发新的转发芯片以支持新的协议，代价非常高，需要将之前的硬件重新设计，这样势必导致更新的成本居高不下、时间周期长等一系列问题.

因此，新一代的SDN解决方案必须让转发平面也具有可编程能力，让软件能够真正定义网络和网络设备. 而P4（Programming Protocol-Independent Packet Processors, 是一门主要用于数据平面的编程语言）正是为用户提供了这种能力，打破了硬件设备对转发平面的限制，让数据包的解析和转发流程也能通过编程去控制，使得网络及设备自上而下地真正向用户开放.

因此P4解决数据平面的可编程问题，OpenFlow是解决控制平面的可编程问题.

P4编译器本质上是将在P4程序中表达的数据平面的逻辑翻译成一个在特定可编程数据包处理硬件上的具体物理配置.

## NFV (Network Function Virtualization)
NFV由运营商联盟提出，为了加速部署新的网络服务，运营商倾向于放弃笨重且昂贵的专用网络设备，转而使用标准的IT虚拟化技术拆分网络功能模块. 通过将硬件与虚拟化技术结合，NFV可以实现所有的网络功能.

NFV是网络功能虚拟化. 传统的网络设备基本都是使用专用的硬件设备实现的，比如熟悉的防火墙，F5这样的硬件设备，又如路由器和交换机. 而NFV则是希望使用**普通的x86服务器来代替专用的网络设备**，使网络功能不再依赖专用设备，而是通过x86服务器和软件来实现. NFV除了可以节约成本外，还通过开放API的方式，使得网络功能更灵活，更有弹性.

![ETSI NFV标准框架](/misc/img/net/Image00012_net.jpg)

其中，NFV infrastructure（NFVI）、MANO和VNF（Virtual Network Function）是顶层的概念实体.

NFVI包含了虚拟化层（Hypervisor或容器管理系统，如Docker）及物理资源，如交换机、存储设备等. NFVI可以跨越若干个物理位置进行部署，为这些物理站点提供数据连接的网络也成为NFVI的一部分.

VNF与NFV虽然是三个同样的字母调换了顺序，但含义截然不同。NFV是一种虚拟化技术或概念，解决了将网络功能部署在通用硬件上的问题；而VNF指的是具体的虚拟网络功能，提供某种网络服务，是一种软件，利用NFVI提供的基础设施部署在虚拟机、容器或物理机中。相对于VNF，传统的基于硬件的网元可以称为PNF。VNF和PNF能够单独或混合组网，形成Service Chain，提供特定场景下所需的E2E（End-to-End）网络服务。

MANO提供了NFV的整体管理和编排，向上接入OSS/BSS（运营支撑系统/业务支撑系统），由NFVO（NFV Orchestrator）、VNFM（VNF Manager）及VIM（Virtualised Infrastructure Manager）虚拟化基础设施管理器三者共同组成。

编排（Orchestration）指以用户需求为目的，将各种网络服务单元进行有序的安排和组织，生成能够满足用户要求的服务。在NFV架构中，凡是带“O”的组件都有一定的编排作用，各个VNF、PNF 及其他各类资源只有在合理编排下，在正确的时间做正确的事情，整个系统才能发挥应有的作用。

VIM 主要负责基础设施层虚拟化资源和硬件资源的管理、监控和故障上报，并面向上层 VNFM 和 NFVO 提供虚拟化资源池，负责虚拟机和虚拟网络的创建和管理，OpenStack和VMware都可以作为VIM。VNFM负责 VNF 的生命周期管理，如上线、下线，状态监控。VNFM基于VNFD（VNF Descriptor，描述一个VNF模块部署与操作行为的配置模板）来管理VNF。NFVO负责NS（Network Service）生命周期的管理和全局资源调度。

## 虚拟交换
1. DPDK

DPDK（Data Plane Development Kit）可提供高性能的数据包处理库和用户空间驱动程序。它不同于Linux系统以通用性设计为目的，而是专注于网络应用中数据包的高性能处理。具体体现在 DPDK 绕过了 Linux 内核协议栈对数据包的处理过程，运行在用户空间上利用自身提供的数据平面库来收发数据包。

在最近的一项研究中，使用DPDK的OVS（Open vSwitch）平均吞吐量提高了75%。该技术被英特尔公司推广，可以在多处理器上使用，并作为 EPA（Enhanced Platform Awareness，旨在加速数据平面）技术的一部分。EPA除DPDK以外的主要技术是大页、NUMA和SR-IOV：大页通过减少页面查找提高VNF的效率；NUMA确保工作负载使用处理器本地的内存；SR-IOV可以使网络流量旁路管理程序，直接转到虚拟机。

2. OVS-DPDK

DVS是一个具有工业级质量的多层虚拟交换机，它支持OpenFlow和OVSDB协议。通过可编程扩展，可以实现大规模网络的自动化（配置、管理和维护）。最初的OVS 版本是通过 Linux 内核进行数据分发的，因而用户能够得到的最终吞吐量受限于Linux网络协议栈的性能。

OVS-DPDK使用DPDK技术对Open VSwitch进行优化。OVS-DPDK是用户态的vSwitch，网络包直接在用户态进行处理。

3. FD.IO

FD.IO（Fast Data Input/Output）是Linux基金会旗下的开源项目，LFN六大创始项目之一.

FD.IO在通用硬件平台上提供了具有灵活性、可扩展、组件化等特点的高性能I/O服务框架，该框架支持高吞吐量、低延迟、高资源利用率的用户空间I/O服务，并可适用于多种硬件架构和部署环境。

FD.IO的关键组件来自Cisco捐赠的商用VPP（Vector Packet Processing，矢量分组处理引擎）库。VPP和FD.IO其他子项目如NSH_ SFC、Honeycomb、ONE等一起用于加速数据平面。

所谓 VPP 向量报文处理是与传统的标量报文处理相对而言的。传统报文处理方式的逻辑是：按照到达先后顺序来处理，第一个报文处理完，处理第二个，依次类推；函数会频繁嵌套调用，并最终返回。相比而言，向量报文处理则是一次并行处理多个报文，相当于一次处理一个报文数组，扩展了整个数据包集合的查找和计算开销，从而提高了效率。

## Linux操作系统网络栈
在Linux中，网络分为两个层次，分别是网络协议栈，以及接收和发送网络协议的设备驱动程序. 网络协议栈是硬件中独立出来的部分，主要用来支持TCP/IP等多种协议，而网络设备驱动程序连接了网络协议栈和网络硬件设备. Linux中与网络有关的实现主要有：
- 网络驱动程序
- Linux VLAN：一种虚拟设备，只有绑定一个真实网卡才能完成实际的数据发送和接收
- Linux Bridge（网桥）：工作于二层的虚拟网络设备，功能类似于物理的交换机. 其他Linux网络设备可以被绑定到Bridge上作为从设备，并被虚拟化为端口. 当一个从设备被绑定到Bridge上时，就相当于真实网络中的交换机端口插入了一个连接有终端的网线.
- Linux TCP/IP协议栈：可以处理IP、ICMP、ARP、TCP/UDP/SCTP等协议。
- Linux Socket函数库：从Berkeley大学开发的BSD UNIX系统中移植而来。网络的Socket数据传输是一种特殊的I/O。
- Linux应用层协议：处理更高层的协议，常用的有DNS、HTTP、SSH、Telnet等。

## 网络操作系统
1. OpenDaylight

ODL（OpenDaylight）是由 Linux 基金会和多家行业巨头如 Cisco、Juniper 和Broadcom等公司一起创立的开源项目，其目的在于推出一个通用的SDN控制平台。

ODL支持OpenFlow、Netconf和OVSDB等多种南向接口，是一个广义的SDN控制平台。ODL 支持分布式集群，不仅可以管理更大的网络，性能更好，还可以相互容灾备份，提升系统的可靠性。它包括一系列功能模块，可以动态地组合，提供不同的服务。

ODL 主要的功能模块有拓扑管理、转发管理、主机监测、交换机管理等。ODL控制平台引入了模型驱动的设计思想，构建了服务抽象层 MD-SAL，是控制器模块化的核心，能够自动适配底层不同的设备，使开发者专注于业务应用的开发。

2. ONOS

ONOS（Open Network Operating System）顾名思义就是要定义一个开放的网络操作系统，其核心的服务对象是服务提供商。既然服务对象要达到运营商的级别，那么其重点就需要考虑可靠性与性能，并能够在白盒系统上创建高性能可编程的运营商网络。

ONOS 的北向接口抽象层和 API 可以使得应用开发变得更加简单，而通过南向接口抽象层和API则可以管控OpenFlow或传统设备。北向接口基于具有全局网络视图的框架，南向接口包括 OpenFlow 和 Netconf，以便能够管理虚拟和物理交换机。ONOS的核心是分布式的，因此可以水平扩展，架构如图1-15所示。

ONOS 在诞生之初就是为了对抗 ODL，希望能成为控制器的主流。目前主要的参与者包括 AT&T、CIENA、VERIZON、NTT、爱立信、华为、NEC、INTEL、富士通等。

3. Tungsten Fabric
Tungsten Fabric是由OpenContrail（由Juniper开源的SDN控制器）向Linux基金会迁移并更名而来的。Tungsten Fabric是一个可扩展的多云网络平台，能够与包括Kubernetes和OpenStack在内的多个云平台集成，并且支持私有云、混合云和公有云部署。

## 云平台
1. OpenStack

OpenStack社区将Nova项目中的网络模块和块存储模块剥离出来，成立了两个新的核心项目，分别是 Neutron 和 Cinder.

Neutron通过插件的方式对众多的网络设备提供商进行支持，比如Cisco、Juniper等，同时也支持很多流行的技术，比如Openvswitch、OpenDaylight和SDN等.

Neutron的插件分为Core Plugin和Service Plugin两类。Core Plugin负责管理和维护Neutron的Network、Subnet和Port三类核心资源的状态信息，这些信息是全局的，只需要也只能由一个Core Plugin管理。Havana版本中实现了ML2（Modular Layer 2）Core Plugin用于取代原有的Core Plugin。对三类核心资源进行操作的REST API被neutron-server看作Core API，由Neutron原生支持。
- Network：代表一个隔离的二层网段，是为创建它的租户而保留的一个广播域。Subnet和Port始终被分配给某个特定的Network。Network的类型包括Flat、VLAN、VxLAN、GRE等。
- Subnet：代表一个IPv4/v6的CIDR地址池，以及与其相关的配置，如网关、DNS等，该Subnet中的 VM 实例随后会自动继承该配置。Sunbet必须关联一个Network。
- Port：代表虚拟交换机上的一个虚机交换端口。VM 的网卡 VIF 连接 Port后，会拥有 MAC 地址和 IP 地址。Port 的 IP 地址是从 Subnet 地址池中分配的。

Service Plugin 即为除 Core Plugin 以外其他的插件，包括 l3 router、firewall、loadbalancer、VPN、metering 等，主要实现 L3~L7的网络服务。这些插件要操作的资源比较丰富，对这些资源进行操作的 REST API 被 neutron-server 看作 Extension API，需要厂家自行进行扩展。

OpenStack已被Kubernetes打败.

2. Kubernetes

Kubernetes，又称为k8s（首字母为k、首字母与尾字母之间有8个字符、尾字母为 s，所以简称为k8s），或者简称为“kube”，设计初衷是在主机集群之间提供一个能够自动化部署、可扩展、应用容器可运营的平台。在整个k8s生态系统中，能够兼容大多数的容器技术实现，比如Docker与Rocket(已淘汰)。

## 网络编排

NFV 给网络带来极大的灵活性和敏捷性，但它们的实现依赖于新的管理系统和自动化编排系统。在 NFV 体系中，引入了全新的管理和编排系统——NFV MANO（NFV Management and Orchestration）系统，编排器作为其中的核心部件，是网络灵活调整和资源动态调度的关键，是下一代网管系统的核心。

NFV 编排器由两层构成：服务编排和资源编排，可以控制新的网络服务，并将VNF集成到虚拟架构中，NFV编排器还能验证并授权NFV基础设施（NFVI）的资源请求。VNF管理器能够管理VNF的生命周期。VIM能够控制并管理NFV基础设施，包括了计算、存储和网络等资源。为了使NFV MANO行之有效，它必须与现有系统中的应用程序接口（API）集成，以便跨多个网络域使用多厂商技术。同样，OSS/BSS也需要与MANO实现互操作。

## 网络数据分析
1. PNDA

2016年8月16日，Linux基金会发布了一个网络数据分析平台PNDA（Platform for Network Data Analytics）.

PNDA旨在通过集成、缩放和管理一组公开的数据处理技术，并提供部署分析应用和服务的端到端平台来降低复杂性。PNDA 能够支持批量的实时数据流探索和分析，甚至可以达到每秒数百万消息的规模。

2. SNAS

SNAS（Streaming Network Analytics System）是一个实时跟踪和分析网络路由拓扑数据的框架。系统将从网络的第2层和第3层挖掘和收集数据，包括IP信息、服务质量及物理和设备规范。

## 网络集成
OPNFV（Open Platform for NFV）是运营商级的开源网络集成参考平台。NFV架构里包含多个开源组件，不同开源组件间的集成和测试非常关键。OPNFV则提供了多组件持续开发、集成和测试的开源方案，并不断地向上游组织输出电信级 NFV平台的增强特性。

OPNFV 目前已经集成了 OpenStack、ODL、ONOS、DPDK、ONAP、FD.IO 等多个关键组件，发布了7个版本、超过60多个集成套件和几十个自动化测试工具，为NFV集成和测试提供了大量开源参考方案和自动化框架。

# linux virtual network
![虚拟网络结构](/misc/img/net/Image00019_net.jpg)

虚拟机的网络功能由虚拟网卡（vNIC）提供，Hypervisor可以为每个虚拟机创建一个或多个 vNIC. 站在虚拟机的角度，这些 vNIC 等同于物理的网卡。为了实现与传统物理网络等同的网络结构，与 NIC 一样，Switch 也被虚拟化为虚拟交换机（vSwitch）. 各个vNIC连接在vSwitch的端口上，最后这些vSwitch通过物理Server的物理网卡访问外部的物理网络. 由此可见，一个虚拟的二层网络结构，主要是完成两种网络设备的虚拟化：NIC硬件与交换设备.

## MACVTAP
传统的Linux网络虚拟化技术采用的是TAP+Bridge方式，将虚拟机连接到虚拟的TAP网卡，然后将TAP网卡绑定到Linux Bridge. 这种解决方案实际上就是使用软件，用服务器的CPU模拟网络，但这种技术主要有三个缺点：
- 每台宿主机内都存在Bridge会使网络拓扑变得复杂，相当于增加了交换机的级联层数
- 同一宿主机上虚拟机之间的流量直接在Bridge完成交换，使流量监控、监管变得困难
- Bridge是软件实现的二层交换技术，会加大服务器的负担

针对云计算中的复杂网络问题，业界主要提出了两种技术标准进行扩展：802.1Qbg与802.1Qbh.

802.1Qbh Bridge Port Extension主要由VMware与 Cisco 提出，尝试从接入层到汇聚层提供一个完整的虚拟化网络解决方案，尽可能达到通过软件定义一个可控网络的目的. 它扩展了传统的网络协议，因此需要新的网络设备支持，成本较高.

802.1Qbg Edge Virtual Bridging（EVB）主要由 HP 等公司提出，尝试以较低成本利用现有设备改进软件模拟的网络. 802.1Qbg 的一个核心概念是 VEPA，它通过端口汇聚和数据分类转发，把宿主机上原来由 CPU 和软件来做的网络处理工作转移到接入层交换机上，减轻宿主机的CPU 负载. 同时，使得在一级的交换机上做虚拟机网络流量监控成为可能.

为支持这种新的虚拟化网络技术，Linux 引入了新的网络设备模型——MACVTAP，用来简化虚拟化环境下的桥接网络，代替传统的TAP+Bridge组合，同时支持新的虚拟化网络技术，如 802.1 Qbg. 和 TAP 设备一样，每一个 MACVTAP设备都拥有一个对应的 Linux 字符设备，因此能直接被 KVM/QEMU使用，方便完成网络数据交换工作.

MACVTAP的实现基于传统的 MACVLAN. MACVLAN允许在主机的一个网络接口上配置多个虚拟的网络接口，这些网络接口有自己独立的 MAC 地址，也可以配置 IP 地址进行通信. MACVLAN 下的虚拟机或者容器和主机在同一个网段中，共享同一个广播域. MACVLAN 和 Bridge 比较相似，但因为它省去了 Bridge，所以配置和调试起来比较简单，而且效率也相对更高.

同一个物理网卡上的各个MACVTAP设备，都可以拥有属于自己的 MAC地址和IP地址。使用MACVTAP，管理员不再需要建立网桥br0，并且同时把物理网卡eth0、连接虚拟机的TAP设备tap0和tap1加入网桥br0中，而是只需要在物理网卡eth0上建立两个MACVTAP设备，并让虚拟机直接使用这两个MACVTAP设备就可以了.

MACVTAP设备支持3种操作模式：
- VEPA模式：VEPA模式是默认模式. 在这种模式下，两个在同一个物理网卡上的 MACVTAP 设备（都处于 VEPA 模式）通信，网络数据会从一个MACVTAP 设备通过底层的物理网卡发往外界的交换机. 此时，外界交换机必须支持Hairpin模式，只有这样才可以把网络数据重新送回物理网卡，传送给此物理网卡上的另一个MACVTAP设备.
- 桥接模式：在桥接模式下，同一个物理网卡上的所有桥接模式的MACVTAP设备直接两两互通，它们之间的通信，网络数据不会经过外界交换机
- 私有模式：在私有模式时，类似于VEPA模式时外界交换机不支持Hairpin模式的情况. 此时，同一个物理设备上的MACVTAP设备之间不能通信

## Open vSwitch
Open vSwitch是一个具有产品级质量的虚拟交换机，它使用C语言进行开发，从而充分考虑了在不同虚拟化平台间的移植性，同时它遵循Apache2.0许可，因此对商用也非常友好.

对于虚拟网络来说，交换设备的虚拟化是很关键的一环，vSwitch负责连接vNIC与物理网卡，同时也桥接同一物理Server内的各个vNIC. Linux Bridge已经能够很好地充当这样的角色，为什么还需要Open vSwith呢？

在传统数据中心中，网络管理员通过对交换机的端口进行一定的配置，可以很好地控制物理机的网络接入，完成网络隔离、流量监控、数据包分析、Qos配置、流量优化等一系列工作.

但是在云环境中，仅凭物理交换机的支持，管理员无法区分被桥接的物理网卡上流淌的数据包属于哪个VM、哪个OS及哪个用户，Open vSwitch的引入则使云环境中虚拟网络的管理以及对网络状态和流量的监控变得容易.

比如，可以像配置物理交换机一样，将接入到Open vSwitch（Open vSwitch同样会在物理Server上创建一个或多个vSwitch供各个虚拟机接入）上的各个VM分配到不同的VLAN中实现网络的隔离. 也可以在Open vSwitch端口上为VM配置Qos，同时Open vSwitch也支持包括NetFlow、sFlow很多标准的管理接口和协议，可以通过这些接口完成流量监控等工作.

此外，Open vSwitch也提供了对Open Flow的支持，可以接受Open Flow Controller的管理.

总之，Open vSwitch在云环境中的各种虚拟化平台上（比如KVM）实现了分布式的虚拟交换机，一个物理 Server 上的 vSwitch 可以透明地与另一个 Server上的vSwitch连接在一起.

![Open vSwitch软件结构](/misc/img/net/Image00023_net.jpg)

其中ovs-vswitchd是最重要的模块，实现了虚拟机交换机的后台，负责与远程的Controller进行通信，例如通过OpenFlow协议与OpenFlow Controller通信，通过sFlow协议同sFlow Trend通信. 此外，ovs-switchd也负责同内核态模块通信，基于netlink机制下发具体的规则和动作到内核态的 datapath。datapath 负责执行数据交换，也就是把从接收端口收到的数据包在流表（Flow Table）中进行匹配，并执行匹配到的动作。每个datapath都和一个流表关联，当datapath接收数据后，会在流表中查找可以匹配的Flow，执行对应的动作，比如转发数据到另外的端口。ovsdb-server是一个轻量级的数据库服务器，主要用来记录被ovs-switchd的配置信息。

Open vSwitch还包括了一系列的命令行工具，主要包括：
- ovs-vsctl：查询和更新ovs-vswitchd的配置信息
- ovsdb-client:ovsdb-server的客户端命令行工具
- ovs-appctl：用来配置运行中的Open vSwitch daemon
- ovs-dpctl：用来配置内核模块中的datapath
- ovs-ofctl：通过OpenFlow协议查询和控制OpenFlow交换机和控制器

## Linux Network Namespace
如果不考虑内存、CPU等其他共享的资源，仅从网络的角度来看，Network Namespace就和一台虚拟机一样，它可以在一台机器上模拟出多个完整的协议栈.

每个新的Network Namespace都默认有一个本地回环LO接口，此外，所有的其他网络设备，包括物理/虚拟网络接口、网桥等，只能属于一个Network Namespace，每个Socket也只能属于一个Network Namespace.

创建 Network Namespace 也非常简单，使用 ip netns add 后面跟着要创建的Namespace 名称，如果相同名字的 Namespace 已经存在，会产生"Cannot create namespace"的错误.

ip netns命令创建的Network Namespace会出现在/var/run/netns/目录下，如果需要管理其他不是ip netns创建的Network Namespace，只要在这个目录下创建一个指向对应 Network Namespace 文件的链接就可以.

ip命令提供了ip netns exec子命令，可以在对应的Network Namespace中执行任意命令(但和网络无关的命令执行结果和在外部执行没有区别)，比如要看Network Namespace 中有哪些网卡: `ip netns exec ns1 ip addr`.

有了不同的Network Namespace后，也就有了网络的隔离，而要把两个网络连接起来，Linux提供了VETH pair. 可以把VETH pair当作双向的管道，从一端发送的网络数据，可以直接被另外一端接收到，也可以想象成两个 Namespace 直接通过一个特殊的虚拟网卡连接起来，可以直接通信.

可以使用ip link add type veth 创建一对VETH pair，系统自动生成VETH0和VETH1两个网络接口，如果需要指定它们的名字，则可以使用ip link add vethfoo type veth peer name vethbar，此时创建出来的两个名字就是vethfoo和vethbar. 需要记住的是**VETH pair无法单独存在，删除其中一个，另一个也会自动消失**.

然后，可以把这对VETH pair分别放到创建的两个Namespace里，可以使用ip link set DEV netns NAME来实现.

虽然 VETH pair 可以实现两个 Network Namespace 之间的通信，但是当多个Namespace需要通信的时候，就无能为力了. 涉及多个网络设备之间的通信，首先想到的是交换机和路由器. 因为这里要考虑的只是同一个网络，所以只用到交换机的功能，也就是brigde. 即先创建VETH pair，比如veth0与veth1，再将一个VETH接口veth0加入Network Namespace，另一个VETH接口veth1加入Bridge即可.

## NAT
> 在ipv6逐渐普及的现今, nat不再是重点, 因此这里不会很深入.

网络地址转换（NAT,Network Address Translation）是一种在IP数据包通过路由器或防火墙时重写来源IP地址或目的IP地址的技术. 这种技术被普遍使用在有多台主机通过一个公有IP地址访问外部网络的私有网络中，NAT也可以被称作IP伪装（IP Masquerading），可以分为目的地址转换（DNAT）和源地址转换（SNAT）两类.

DNAT 主要用在从公网访问内部私网的网络数据流中. 比如从公网访问地址为IP1的公网IP地址，NAT网关根据设定的DNAT规则，把IP数据报文包头内的目的IP地址IP1修改为内部的私网IP地址192.168.1.10，从而把IP数据报文发送给地址为192.168.1.10的内部服务器. DNAT可以用来将内部私网服务器上的服务暴露给公网使用.

SNAT 主要应用在从内部私网访问公网的网络数据流中. 比如内部私网 IP 地址为192.168.1.20的机器想访问外部公网IP地址为IP2的服务，NAT网管根据设定的SNAT规则，把IP数据报文包头内的源IP地址192.168.1.20修改为NAT网关自己的公网IP地址IP1。这样内网中没有公网IP地址的机器也能访问外部公网中的服务了.

Linux中的NAT功能一般通过iptables/nftables实现. iptables/nftables是基于Linux内核的功能强大的防火墙. eBPF也实现了NAT.

## 虚拟网络隔离技术
### 1. 虚拟局域网（VLAN）

LAN（Local Area Network，本地局域网）中的计算机通常使用Switch连接. 一般来说，两台计算机连入同一个Switch时，它们就在同一个 LAN 中. 一个 LAN 表示一个广播域，含义就是：LAN 中的所有成员都会收到任意一个成员发出的广播包.

VLAN（Virtual LAN，虚拟局域网）表示一个带有VLAN功能的Switch能将自己的端口划分出多个LAN. 计算机发出的广播包可以被同一个LAN中的其他计算机收到，但位于其他LAN的计算机则无法收到。简单地说，**VLAN将一个交换机在逻辑上分成了多个交换机，限制了广播的范围，在二层将计算机隔离到不同的VLAN中**.

需要注意的是，VLAN实现的只是二层的隔离，比如二层广播包arp无法跨越 VLAN 的边界，但在三层上（比如IP地址）是可以通过路由器让两个VLAN互通的.

使用VLAN，能够更好地控制广播风暴，提高网络整体的安全性，也能使网络管理更加简单直观. 不同的VLAN由VLAN tag（VID）标明，IEEE 802.1Q规定了VLAN tag的格式. 在Linux上要使用VLAN，需要加载8021q的内核模块.

现在的交换机几乎都是支持 VLAN 的. 交换机的端口通常有两种配置模式：Access和Trunk:

其中，Access 端口被打上了 VLAN tag，表明该端口属于哪个 VLAN,Access口只能属于一个 VLAN。Access 端口都是直接与计算机网卡相连的，这样从该网卡出来的数据包流入Access端口后就被打上了所在的 VLAN tag。

Trunk 端口一般用于交换机之间的连接，可以允许多个 VLAN 通过，可以接收和发送多个 VLAN 的报文。如果划分了 VLAN，但是没有配置 Trunk，那么交换机之间的每个VLAN 间通信都要通过一条线路来实现。

![Linux VLAN](/misc/img/net/Image00042_net.jpg)

在Linux中可以通过Linux Bridge、物理网卡等模拟交换机的VLAN环境。上图中, eth0 是宿主机上的物理网卡，有一个命名为eth0.10的子设备与之相连。eth0.10 就是 VLAN 设备，其 VLAN ID 就是 VLAN 10。eth0.10 挂在命名为 brvlan10 的 Linux Bridge 上，虚拟机 VM1 的虚拟网卡 vent0也挂在brvlan10 上。

这样配置的效果等同于宿主机用软件实现了一个虚拟交换机，上面定义了一个VLAN10, eth0.10和vnet0 都分别接到 VLAN10 的 Access端口上。而 eth0 就相当于一个 Trunk 端口，VM1 通过 vnet0 发出来的数据包会被打上 VLAN10 的标签。

但是Linux在VLAN模拟上有一个不足，即需要多少个VLAN，就得创建多少个Bridge,Trunk端口也需要创建同样数量的类似eth0.10的虚口。这是由于**Bridge的转发表没有VLAN tag的维度**，要实现不同VLAN独立转发，只能使用多个Bridge实例实现转发表的隔离.

## 2. VxLAN
VxLAN（Virtual Extensible Local Area Network，虚拟局域网扩展）是基于隧道（Tunnel）的一种网络虚拟化技术.

隧道是一个虚拟的点对点的连接，提供了一条通路使封装的数据报文能够在这个通路上传输，并且在通路的两端分别对数据报文进行封装及解封装。某个协议的报文要想穿越IP网络在隧道中传输，必须要经过封装与解封装两个过程。隧道提供了一种某一特定网络技术的PDU穿过不具备该技术转发能力的网络的手段，如组播数据包穿过不支持组播的网络.

**VxLAN将二层报文用三层协议进行封装，可以对二层网络在三层范围内进行扩展**, 比如把二层网络的整个数据帧封装在UDP报文中，送到VxLAN隧道对端。隧道对端的虚拟或者物理网络设备再将其解封装，取出里面的二层网络数据帧发送给真正的目的节点.

VxLAN协议头使用了24bit表示VLAN ID，可以支持1600多万个VLAN ID。RFC协议7348号中定义了VxLAN协议。

VxLAN应用于数据中心内部，使虚拟机可以在互相连通的三层网络范围内迁移，而不需要改变IP地址和MAC地址，保证业务的连续性。

## 3. 通用路由封装GRE
GRE（RFC1701）也是基于隧道的一种网络虚拟化技术. 与VxLAN相比，GRE使用的是IP报文而非UDP作为传输协议。同时，不同于VxLAN只能封装二层以太网数据帧，GRE可以封装多种不同的协议，包括IP报文（RFC2784,RFC2890）、ARP、以太网帧（NVGRE,RFC 7637）等.

相比于VxLAN,GRE更加灵活，可以支持的协议也更多。但是目前物理网卡支持GRE协议的还不是很多，大部分GRE协议的处理还要依靠主机CPU，会增加CPU的负载。

## 4. 通用网络虚拟化封装（Geneve）
为了应对 VLAN 只能有4094的上限，利用隧道技术，产生了诸如 VxLAN、NVGRE、STT（无状态传输隧道）等多种技术来实现虚拟网络的隔离要求。但是这类技术互相不能兼容，所以提出了通用网络虚拟化封装 Geneve（Generic Network Virtualization Encapsulation）.

Geneve技术的RFC正式标准还没产生，还处于IETF草案的阶段。Geneve主要的目的是适应虚拟网络技术的发展和隔离要求，定义一种通用的网络虚拟化隧道封装协议，能够尽可能地兼容目前的VxLAN、NVGRE等正式RFC标准的功能，并且提供高可扩展性来应对以后虚拟网络技术的发展。

Geneve综合了VxLAN和NVGRE两者的特点，首先使用了UDP作为传输协议，同时吸收了GRE可以封装多种不同类型的数据包的优点。