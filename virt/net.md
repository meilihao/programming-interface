# net
## 网络模型
总的来说，目前有四种常见的网络模型：
- 桥接（Bridge Adapter）

	使用虚拟交换机（Linux Bridge）, 将虚拟机和物理机连接起来，它们处于同一个网段, IP 地址是一样的.

	在这种网络模型下，虚拟机和物理机都处在一个二层网络里面，所以有：
	- 虚拟机之间彼此互通
    - 虚拟机与主机彼此可以互通
    - 只要物理机可以上网，那么虚拟机也可以

    桥接网络的好处是简单方便，但也有一个很明显的问题，就是一旦虚拟机太多，广播就会很严重. 所以, 桥接网络一般也只适用于桌面虚拟机或者小规模网络这种简单的形式.
- NAT

	网络地址转换（Network Address Translatation）, 这种模型严格来讲，又可以分为 NAT 和 NAT 网络两种: 
	- NAT：主机上的虚拟机之间是互相隔离的，彼此不能通信（它们有独立的网络栈，独立的虚拟 NAT 设备）

	                                    |->虚拟nat设备-> vm(10.0.2.15)
    	物理网卡(192.168.108.34)->switch -> 虚拟dhcp
    									|->虚拟nat设备-> vm(10.0.3.15)

    - NAT 网络：虚拟机之间共享虚拟 NAT 设备，彼此互通

                                           |-> vm(10.0.2.15)
    	物理网卡(192.168.108.34)->虚拟nat设备-> 虚拟dhcp
    									   |-> vm(10.0.2.4)

	根据 NAT 的原理，虚拟机所在的网络和物理机所在的网络不在同一个网段，虚拟机要访问物理所在网络必须经过一个地址转换的过程，也就是说在虚拟机网络内部需要内置一个虚拟的 NAT 设备来做这件事.

	NAT 网络模式中一般还会内置一个虚拟的 DHCP 服务器来进行 IP 地址的管理.

	总结一下，以上两种 NAT 模式，如果不做其他配置，那么有：
    - 虚拟机可以访问主机，反之不行
    - 如果主机可以上外网，那么虚拟机也可以
    - 对于 NAT，同主机上的虚拟机之间不能互通
    - 对于 NAT 网络，虚拟机之间可以互通

	PS：如果做了 端口映射 配置，那么主机也可以访问虚拟机

- 主机（Host-only Adapter）

	只限于主机内部访问的网络，虚拟机之间彼此互通，虚拟机与主机之间彼此互通。但是默认情况下虚拟机不能访问外网（注意：这里说的是默认情况下，如果稍作配置，也是可以的）.

	它的网络模型是相对比较复杂的，可以说前面几种模式实现的功能，在这种模式下，都可以通过虚拟机和网卡的配置来实现，这得益于它特殊的网络模型.

	```bash
											 |-> vm(192.168.56.103)
	host网卡---(两者默认断开)虚拟网卡(192.168.56.1) -> switch-
											 |-> vm(192.168.56.104)
	```

	主机网络模型会在主机中模拟出一块虚拟网卡供虚拟机使用，所有虚拟机都连接到这块网卡上，这块网卡默认会使用网段 192.168.56.x（在主机的网络配置界面可以看到这块网卡).

	默认情况下，虚拟机之间可以互通，虚拟机只能和主机上的虚拟网卡互通，不能和不同网段的网卡互通，更不能访问外网. 如果vm想访问外网，那么需要将按上图将host网卡和虚拟网卡桥接或共享即可.
- 内部网络（Internal）

	这种模型是相对最简单的一种，虚拟机与外部环境完全断开，只允许虚拟机之间互相访问，这种模型一般不怎么用，所以在 VMware 虚拟机中是没有这种网络模式的.

VirtualBox 支持的四种模型，对于 VMware，则只有前三种.

可访问性:
<table>
<thead>
<tr>
<th style="text-align: center">Model</th>
<th style="text-align: center">VM -&gt; host</th>
<th style="text-align: center">host -&gt; VM</th>
<th style="text-align: center">VM &lt;-&gt; VM</th>
<th style="text-align: center">VM -&gt; Internet</th>
<th style="text-align: center">Internet -&gt; VM</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center">Bridged</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
</tr>
<tr>
<td style="text-align: center">NAT</td>
<td style="text-align: center">+</td>
<td style="text-align: center">Port Forwarding</td>
<td style="text-align: center">-</td>
<td style="text-align: center">+</td>
<td style="text-align: center">Port Forwarding</td>
</tr>
<tr>
<td style="text-align: center">NAT Network</td>
<td style="text-align: center">+</td>
<td style="text-align: center">Port Forwarding</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">Port Forwarding</td>
</tr>
<tr>
<td style="text-align: center">Host-only</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">+</td>
<td style="text-align: center">-</td>
<td style="text-align: center">-</td>
</tr>
<tr>
<td style="text-align: center">Internal</td>
<td style="text-align: center">-</td>
<td style="text-align: center">-</td>
<td style="text-align: center">+</td>
<td style="text-align: center">-</td>
<td style="text-align: center">-</td>
</tr>
</tbody>
</table>

## 网络虚拟化
ref:
- [【重识云原生】第四章云网络4.7.9节——NFV](https://www.jianshu.com/p/ff1cd84079c4)

软件定义数据中心（Soft ware Define Data Center，SDDC）通过底层硬件架构上加载的一个虚拟基础架构层，提供了数据中心应用程序所需的运行环境，并能管理存储、服务器、交换机和路由器等多种设备。SDDC 使数据中心由传统的以架构为中心改为以应用/业务为中心。SDDN 使得硬件平台及其相应的操作对上层应用完全透明。管理人员只需定义应用所需的资源（包括计算、存储、网络）和可用性要求，SDDC 可从硬件资源中提取出“逻辑资源”，供应用使用。未来，SDDC的发展将改变传统云计算的概念，提高数据中心的健壮性、运营效率以及资源利用率.

### 1. 网络设备虚拟化
#### a. 网卡虚拟化
软件网卡虚拟化主要通过软件控制各个虚拟机共享同一块物理网卡实现.

软件虚拟出来的网卡可以有单独的MAC 地址、IP 地址.

虚拟网卡包括e1000，virtio等实现技术。virtio是目前最为通用的技术框架。virtio提供了虚拟机和物理服务器数据交换的通用机制，得到了大多数hypervisor的支持，成为事实上的标准.

vDPA（硬件） 是virtio的半硬件虚拟化实现。该框架由Redhat提出，实现了virtio数据平面的硬件卸载。控制平面仍然采用原来的控制平面协议.

越来越多的硬件厂商开始原生支持virtio协议，将虚拟化功能Offload到硬件上，把嵌入式CPU集成到SmartNIC中，网卡处理所有网络中的数据，而嵌入式CPU控制路径的初始化并处理异常情况。这种具有Offload的SmartNIC则称为DPU，专门处理数据中心内的各种计算机和存储之间的数据移动和处理.

硬件网卡虚拟化主要用到的技术是单根I/O 虚拟化（Single Root I/O Virtulization，SR-IOV）.

#### b. 硬件设备虚拟化
硬件设备虚拟化主要有两个方向:
1. 在传统的基于x86 架构机器上安装特定操作系统，实现路由器的功能
1. 传统网络设备硬件虚拟化

### 2. 链路虚拟化
链路虚拟化是日常使用最多的网络虚拟化技术之一。常见的链路虚拟化技术有链路聚合和隧道协议。这些虚拟化技术增强了网络的可靠性与便利性.

链路聚合（Port Channel）是最常见的二层虚拟化技术。链路聚合将多个物理端口捆绑在一起，虚拟成为一个逻辑端口。当交换机检测到其中一个物理端口链路发生故障时，就停止在此端口上发送报文，根据负载分担策略在余下的物理链路中选择报文发送的端口。链路聚合可以增加链路带宽、实现链路层的高可用性。

隧道协议（Tunneling Protocol）指一种技术/协议的两个或多个子网穿过另一种技术/协议的网络实现互联。使用隧道传递的数据可以是不同协议的数据帧或包。隧道协议将其他协议的数据帧或包重新封装然后通过隧道发送。新的帧头提供路由信息，以便通过网络传递被封装的负载数据。隧道可以将数据流强制送到特定的地址，并隐藏中间节点的网络地址，还可根据需要，提供对数据加密的功能。一些典型的使用到隧道的协议包括Generic Routing Encapsulation（GRE）和 Internet Protocol Security（IPsec）。

### 3. 虚拟网络
虚拟网络（Virtual Network）是由虚拟链路组成的网络。虚拟网络节点之间的连接并不使用物理线缆连接，而是依靠特定的虚拟化链路相连。典型的虚拟网络包括层叠网络、v*n 网络以及在数据中心使用较多的虚拟二层延伸网络。


#### a. 层叠网络
层叠网络（Overlay Network）简单说来就是在现有网络的基础上搭建另外一种网络。层叠网络可以充分利用现有资源，在不增加成本的前提下，提供更多的服务。比如ADSL Internet 接入线路就是基于已经存在的PSTN 网络实现。

#### b. vpn
虚拟专用网（Virtual Private Network，v*n）是一种常用于连接中、大型企业或团体与团体间的私人网络的通信方法。虚拟专用网通过公用的网络架构（比如互联网）来传送内联网的信息。利用已加密的隧道协议来达到保密、终端认证、信息准确性等安全效果。这种技术可以在不安全的网络上传送可靠的、安全的信息.

#### c. 虚拟二层延伸网络
虚拟化从根本上改变了数据中心网络架构的需求。虚拟化引入了虚拟机动态迁移技术，要求网络支持大范围的二层域。一般情况下，多数据中心之间的连接是通过三层路由连通的。而要实现通过三层网络连接的两个二层网络互通，就要使用到虚拟二层延伸网络（VirtualL2 Extended Network）。

传统的VPLS（MPLS L2v*n）技术，以及新兴的Cisco OTV、H3C EVI 技术，都是借助隧道的方式，将二层数据报文封装在三层报文中，跨越中间的三层网络，实现两地二层数据的互通。也有虚拟化软件厂商提出了软件的虚拟二层延伸网络解决方案。例如VXLAN、NVGRE，在虚拟化层的vSwitch 中将二层数据封装在UDP、GRE 报文中，在物理网络拓扑上构建一层虚拟化网络层，从而摆脱对底层网络的限制.

### 4. 基于SDN的网络虚拟化
SDN 改变了传统网络架构的控制模式， 将网络分为控制层（Control Plane）和数据层（Data Plane）。网络的管理权限交给了控制层的控制器软件， 通过OpenFlow 传输通道，统一下达命令给数据层设备。数据层设备仅依靠控制层的命令转发数据包。由于SDN的开放性，第三方也可以开发相应的应用置于控制层内，使得网络资源的调配更加灵活。网管人员只需通过控制器下达命令至数据层设备即可，无需一一登录设备，节省了人力成本，提高了效率。可以说，SDN 技术极大地推动了网络虚拟化的发展进程。

SDN 有三种主流实现方式，分别是OpenFlow 组织主导的开源软件（包括Google，IBM，Citrix 等公司支持），思科主导的应用中心基础设施（Application CentricInfrastructure，ACI），以及VMware 主导的NSX.

#### a. OpenFlow
OpenFlow 的核心是将原本完全由交换机/路由器控制的数据包转发，转化为由支持OpenFlow 特性的交换机和控制服务器分别完成的独立过程。OpenFlow 交换机是整个OpenFlow 网络的核心部件，主要管理数据层的转发。OpenFlow 交换机至少由三部分组成：流转发表（Flow Table），告诉交换机如何处理流；安全通道（Secure Channel），连接交换机和控制器；OpenFlow协议，一个公开的、标准的供OpenFlow 交换机和控制器通信的协议。OpenFlow 交换机接收到数据包后，首先在本地的流表上查找转发目标端口，如果没有匹配，则把数据包转发给Controller，由控制层决定转发端口.

基于OpenFlow 的SDN 协议是公开的，网络设备可以有更多的选择.