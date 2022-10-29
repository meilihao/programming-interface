# device
在Linux中实现虚拟网络的方法中比较常用的工具有两个：bridge-utils 和 openvswitch, 它们创建的虚拟网络设备是不能相互使用的.

## 路由器
路由器实质上是一种利用协议转换技术将异种网进行互联的设备, 路由器在TCP/IP中又称为IP网关.

路由器转发数据包过程:
1. 拆除数据包二层帧头，查看三层报头中的目的IP地址。
1. 将目的IP地址和本地路由条目进行与运算，得出结果。
1. 如果一致则按相应的接口转发，否则丢弃。

## 三层交换机
三层交换机同时具备交换机与路由器功能的强大网络设备, 即三层交换机=路由器(三层)+交换机(二层) .

在云计算和虚拟化中，数据中心的流量(以路由器为参照物)分为两种：
1. 南北向流量: 数据中心与外部通信的流量, 经过路由器的流量
1. 东西向流量: 数据中心内部的流量, 不经过路由器的流量

## 分布式虚拟交换机
虚拟交换机分两种类型，一种是普通虚拟交换机，一种是分布式虚拟交换机, 区别：
1. 普通虚拟交换机只运行在一台单独的物理主机上，所有与网络相关的配置只适用于此物理服务器上的虚拟机
1. 分布式虚拟交换机分布在不同的物理主机上，通过虚拟化管理工具，可以对分布式虚拟交换机进行统一地配置

	VMware虚拟机化解决方案支持普通和分布式虚拟交换机
	华为虚拟化解决方案只支持分布式虚拟交换机

分布式虚拟交换机优点:
1. 多台Host Machine共享一台虚拟交换机，只需要对分布式虚拟交换机配置做修改，其他所有Host Machine上的配置都会更新；
1. 虚拟机能够进行热迁移的必要条件之一就是要有分布式虚拟交换机，热迁移，即虚拟机直接迁移到另一台服务器上。不需要关机；
扩展：

## 虚拟网络接口
在linux的网络设备驱动框架里面, 使用一个net_device来代表一个网络设备接口, 因此, 一个物理网卡对应着一个net_device结构.

虚拟网络接口是指一个网络接口的net_device没有直接对应的物理设备.

## hub
不管信号从哪个口子进来，其他口子都能收到, 类似一个广播.

一般用来把 tap/tun、veth-pair网线连到Bridge上面，这样可以把一组容器，或者一组虚机连在一起。比如著名的Docker就是用Bridge(docker0)把Host里面的容器都连在一起，使得这些容器可以互相访问。也可以把Host上面的物理网卡也加入到Bridge，这样主机的VM就可以和外界通信了.

## bridge
网桥是Linux中用于表示一个能连接不同网络设备的虚拟设备, linux中传统实现的网桥类似一个hub设备, 而ovs管理的网桥一般类似交换机.

```bash
# brctl show
# ip link add tsj-0 type veth peer name tsj-1
# brctl addbr tsj-br # 创建bridge
# brctl addif tsj-br tsj-0 # 将tsj-0加入bridge
# brctl show tsj-br # 查看bridge tsj-br 
```

当前ip命令支持操作bridge:
```bash
# ip link add br0 type bridge
# ip link set br0 up
```

容器的 Bridge 网络通常配置成内网形式，要出外网需要走 NAT，所以它的数据传输不像虚拟机的桥接形式可以直接跨过协议栈，而是必须经过协议栈，通过 NAT 和 ip_forward 功能从物理网卡转发出去，因此，从性能上看，Bridge 网络虚拟机要优于容器.

## Veth
ref:
- [《跟唐老师学习云网络》 - Veth网线](https://bbs.huaweicloud.com/blogs/149798)

Veth是Linux中一种虚拟出来的网络设备, veth设备总是成对出现，所以一般也叫veth-pair. 其作用是反转数据流方向, 即把数据通过一头`复制`到另一头.

Veth可理解为就是一根`网线`, 且两头是一样的.

演练:
```bash
# ip netns add tsj
# ip netns list
# ip link add tsj-0 type veth peer name tsj-1 # ip link add 【网线一头名字】 type veth peer name 【网线另一头名字】
# ip link set tsj-0 up
# ip link set tsj-1 netns tsj # 将`tsj-1`的网线头子，放入`tsj`这个虚拟空间中, 此时`tsj-0`仍然在Host主机空间中的
# ip netns exec tsj ip link set tsj-1 name eth0 # 把网卡命名为 eth0
# ip netns exec tsj ip link set eth0 up
# ip netns exec tsj ip addr add 10.254.1.1/24 dev eth0
# ip addr add 10.254.1.2/24 dev tsj-0
# ip netns exec tsj ping 10.254.1.2
# --- Docker创建容器网络，过程原理跟上面是一样的，只是把cli改为使用系统调用，并且做了自动化
# --- 进阶
# ip netns exec tsj nc -lp 1234
# curl -v 10.254.1.1:1234
# ---- 当ns非常多，veth也非常多（比如在OpenStack的网络节点上），怎么查找Veth网线的另一头在哪里:
# ip netns exec tsj ethtool -S eth0 # 对端的index的149
NIC statistics:
     peer_ifindex: 149
# ip link list # 找idnex为 149 的网卡即可
# --- 删除Veth和NS
# ip link delete tsj-0 # peer veth也会同时删除
# ip netns delete tsj # 直接删除ns，会连带veth一起删除
```

## TUN/TAP
ref:
- [详解云计算网络底层技术——虚拟网络设备 tap/tun 原理解析](https://www.cnblogs.com/bakari/p/10450711.html)

TUN/TAP是Linux中一种虚拟出来的网络设备. 它也是一种`网线`, 只是这种网线和Veth有点不同. Veth网线的2头是一样的，都是"水晶头". TUN/TAP的2头长得不一样, 一头是"水晶头", 另一头是"USB".

更正式一点是: 它是一种用户空间和内核空间传输报文用的网线, 一头是普通的网卡，跟eth0一样, Host主机可以用; 另一头则是一个文件描述符, 给用户空间的程序用的.

演练:
```bash
# modinfo tun # 当前内核版本应该都支持tun/tap
# modinfo tap
# ip tuntap add dev tap0 mode tap # 创建tap, 应用open文件句柄`/dev/net/tun`即可发送&接收报文
# ip tuntap add dev tun0 mode tun # 创建 tun
# ip tuntap del dev tap0 mode tap # 删除 tap 
# ip tuntap del dev tun0 mode tun # 删除 tun
# ip link set tun0 up
# ip addr add 10.0.0.1/24 dev tun0 # 给tun0分配IP地址
```

tap/tun 最常见的应用也就是用于隧道通信，比如 VPN，包括 tunnel 和应用层的 IPsec 等，其中比较有名的两个开源项目是 openvpn 和 VTun.

TAP设备：模拟一个二层的网络设备，只接受和发送二层网包, 对应`/dev/tapX`. TUN设备：模拟一个三层的网络设备，只接受和发送三层网包, 对应`/dev/net/tunX`.

tap/tun 驱动程序包括两个部分，一个是字符设备驱动，一个是网卡驱动. 这两部分驱动程序分工不太一样，字符驱动负责数据包在内核空间和用户空间的传送，网卡驱动负责数据包在 TCP/IP 网络协议栈上的传输和处理.

设备文件即充当了用户空间和内核空间通信的接口。当应用程序打开设备文件时，驱动程序就会创建并注册相应的虚拟设备接口，一般以 tunX 或 tapX 命名。当应用程序关闭文件时，驱动也会自动删除 tunX 和 tapX 设备，还会删除已经建立起来的路由等信息。

tap/tun 设备文件就像一个管道，一端连接着用户空间，一端连接着内核空间。当用户程序向文件 /dev/net/tun 或 /dev/tap0 写数据时，内核就可以从对应的 tunX 或 tapX 接口读到数据，反之，内核可以通过相反的方式向用户程序发送数据.

> tun/tap的内核代码位于`driver/net/tun.c`, tun/tap 的添加通过使用ioctl 发送 TUNSETIFF 命令来实现.

## OVS交换机(Open vSwtich)
ref:
- [《跟唐老师学习云网络》 - OVS交换机](https://bbs.huaweicloud.com/blogs/358029)

交换机是hub的进阶版, 功能更多.


```bash
# apt-get install -y openvswitch-switch
# ovs-vsctl add-br br-tsj # 添加ovs. 当一个新的交换机被创建的时候，会自带一个同名的端口，也会有一个同名的网卡插到这个口子上，并且这个网卡也会加入到Host中(状态是down).
# ip link set br-tsj up # 将 br-tsj 网卡up
# ip addr add 192.168.0.3/24 dev br-tsj
# ip link add tsj-0 type veth peer name tsj-1
# ip netns add ns-tsj
# ovs-vsctl add-port br-tsj tsj-1 # 把网线的一个"水晶头（tsj-1）", 给插到交换机的网口上面
# ip link set tsj-1 up
# ip link set tsj-0 netns ns-tsj # 把网线另一头放入这个ns（可以把这个ns假装看作是一台VM）
# ip netns exec ns-tsj ip link set tsj-0 up
# ip netns exec ns-tsj ip addr add 192.168.0.2/24 dev tsj-0
# ip netns exec ns-tsj ping 192.168.0.3
# ip netns exec ns-tsj ip route add default via 192.168.0.3 # 给VM加上路由规则，那么它就可以访问主机的网络
# ip netns exec ns-tsj ping 172.17.169.146 # 记得给对方主机配置回来的路由。不然报文能过去到达对方主机，但对方报文不知道怎么回来（即对方Host不知道怎么给这个192.168.0.2的VM地址，找路由。）
# --- 其他命令
# ovs-vsctl show # 查询ovs
# ovs-ofctl dump-flows br-tsj # 查询各个端口的转发规则. 这个规则都是用来控制报文的接下来的行为（从哪个口出去），所以规则基本长这样：符合xx条件 ==》执行zz动作
cookie=0x0, duration=145616.013s, table=0, n_packets=160, n_bytes=10664, priority=0 actions=NORMAL # 除了优先级和统计信息之外。有2个关键的字段：table 和 action. 
# ovs-vsctl list-ports br-tsj # 查询交换机上面的端口信息
# ovs-vsctl set Port tsj-1 tag=100 # 给某个网口设置VLAN标签
# ovs-vsctl remove Port tsj-1 tag 100 # 移除某个网口上面的VLAN tag配置
# ovs-vsctl set Port tsj-1 trunks=100,200 # 设置某个网口为Trunk模式
# ovs-vsctl remove Port tsj-1 trunks 100,200 # 移除某个网口允许通过的的VLAN tag配置
# ovs-vsctl del-port br-tsj tsj-1 # 移除交换机上的某根网线
# ovs-vsctl del-br br-tsj
```

两个ovs互联, 使用一种新的patch网线, 就是一种特殊的端口类型, 专门用来连接2个ovs的端口的. Interface的type显示为patch.

```bash
# ovs-vsctl add-br ovs1
# ovs-vsctl add-br ovs2
# --- ovs1这边
# ovs-vsctl add-port ovs1 patch-to-ovs2 -- set Interface patch-to-ovs2 type=patch -- set Interface patch-to-ovs2 options:peer=patch-to-ovs1
# ovs-vsctl show
# --- ovs2这边
# ovs-vsctl add-port ovs2 patch-to-ovs1 -- set Interface patch-to-ovs1 type=patch -- set Interface patch-to-ovs1 options:peer=patch-to-ovs2
```

### OpenFlow
1. 常见ovs流表条件
xx条件基本有以下几类：
- 哪个ovs端口

	- in_port=port, 比如`priority=100,tcp,in_port=1 actions=resubmit:4000`

- MAC地址（dl_xxx， 即data link 的缩写）

	- dl_src=xx:xx:xx
	- dl_dst=xx:xx:xx:xx

	比如`priority=8,in_port=4,dl_dst=52:4b:14:90:74:46 actions=output:1`

- IP地址（nw_xxx， 即network的缩写）


	- nw_src=ip[/netmask]
	- nw_dst=ip[/netmask]

	举例：`priority=5,in_port=1,ip,nw_dst=192.168.0.2,actions=output:2`

- 端口号（tcp_xxx/udp_xxx）

	- tcp_src=port
	- udp_dst=port

	比如匹配tcp端口号179: `tcp,tcp_src=179/0xfff0,actions=output:2`

 
2. 常见ovs流表动作

zz动作（Action）常见有以下几类：

- 从哪个ovs端口出去

	- output:port

	举例：`priority=8,in_port=4,dl_dst=52:4b:14:90:74:46 actions=output:1`

- 将包丢弃

	- drop

	举例：条件=xx,yy actions=drop

- 修改VLAN头

	- 修改vlanID: mod_vlan_vid:vlan_vid
	- 剥离vlan头: strip_vlan

- 修改源/目的IP+端口等

	- 修改源IP: mod_nw_src:ip
	- 修改目的IP: mod_nw_dst:ip
	- 修改目的端口: mod_tp_dst:port

### ovs-dpdk
ref:
- [ovs-dpdk](https://www.cnblogs.com/goldsunshine/p/14260941.html#ovs-dpdk)

DPDK加速的OVS与原始OVS的区别在于，从OVS连接的某个网络端口接收到的报文不需要openvswitch.ko内核态的处理，报文通过DPDK PMD驱动直接到达用户态ovs-vswitchd里. 即主要区别在于流表的处理: 普通ovs流表转发在内核态, 而ovs-dpdk流表转发在用户态.

## FC光模块与以太网光模块
FC光模块与以太网光模块区别:
- 协议与安全

	FC光模块属于光纤通道协议，不遵循OSI模型分层，而以太网光模块符合IEEE 802.3标准，在局域网中实现基于数据包的物理通信。它是TCP/IP堆栈中的数据链路层协议，属于OSI模型。

	存储区域网络（SAN）与外界隔离，可以降低存储网络受到攻击和数据泄漏的风险，因此在存储网络中使用FC光模块要更加安全。相反的，以太网模块运行的TCP/IP协议存在一系列的安全缺陷。由于尾端管理的介入是发生在网络上的，整个系统更容易受到攻击。

- 可靠性

	受到协议的影响，不同传输模式也导致了传输结果的差异。相比以太网，光纤通道的显著优势是其更好的可靠性。由于无损特性这一出色性能，光纤通道长期以来被应用在存储网络中。与其相反，以太网模块就没有光纤通道模块这种有序无损传输无损的特点。此外，SAN通常使用光纤通道，而以太网适用于NAS系统。FC模块是为那些追求高速，低延迟来进行区块存储的用户而设计的。如果用户需要进行文件层级的存储访问，则优先考虑以太网模块。

- 传输速度

	正如上文介绍部分提到的，光纤通道和以太网模块的传输速度范围是不同的。具体而言，FC模块目前可以运行在1Gbps/2Gbps/8Gbps/16Gbps/32Gbps/128Gbps，以太网模块可以支持更大范围的传输速度：10/100/1000Mbps和10Gbps/25Gbps/50Gbps/40Gbps/100Gbps/400Gbps。

	此外，每代FC模块的更迭速率通常是以2次方来计的，从1Gbps到32Gbps。很明显以太网模块在吞吐量上的提升远远超过了光纤通道模块，新推出的400G以太网QSFP-DD模块已经是初始1G SFP模块的近400倍。显然，**以太网光模块相比FC光模块更符合目前日益增长的高带宽需求**。

- 应用领域

	光纤通道光模块与以太网光模块之间的差异还在于它们的应用领域。光纤通道是在服务器和存储设备之间传输大量数据的最佳方法之一。因此，连接到FC交换机的FC模块主要用于光纤通道，存储网络和以太网应用中。通过基于以太网的光纤通道协议（FCoE），光纤通道通信可以在以太网上运行。显然，FC模块早已应用于大型企业和数据中心。

	以太网光模块通常应用于局域网上，有时也应用于广域网中。与FC模块的工作场景相比，以太网模块的应用环境则基于用户的带宽需要，要更加灵活多样，从小型办公室到超大规模数据中心，各种场合都能见到它的身影。

- 搭配设备

	在实现上述应用场景时，保持光模块与交换机之间的稳定连接至关重要。一般情况下FC模块安装到到FC交换机上，而以太网模块则匹配到以太网交换机，不会出现混合使用的情况。

	传统的光纤通道网络包括FC交换机和光纤卡（FC HBAs），是SAN的主要选择之一。FC交换机将存储连接到SAN，而光纤卡将交换机连接到服务器。以太网网络交换机具有多样性，体现在可堆叠性、端口数、传输速率等方面。当最新的400G以太网光模块安装到400G网络交换机上时，就可以实现400G网络。

## 裸金属
ref:
- [华为云裸金属方案](https://blog.csdn.net/junbaozi/article/details/123834314)

裸金属服务器（Bare Metal Server）类似云上的专属物理服务器，在拥有弹性灵活的基础上，具有高性能的计算能力。计算性能与传统物理机无差别，具有安全物理隔离的特点.

裸金属云服务器技术层面要解决两个问题，第一是物理服务器在云上的自动化管理，第二是管理功能封装成API，方便云管理平台和租户用户调用.

## 裸金属网络
ref:
- [升级全新网络方案，给你低成本、高性能的裸金属体验](https://bbs.huaweicloud.com/blogs/380724)

裸金属服务器是云上的租户级专属物理服务器，它具有性能卓越、高效部署、安全可靠、自动运维等优势.

### 裸金属服务器当前网络解决方案
云上的VPC采用的是overlay技术实现互通，而裸金属服务器本质上是运行在underlay上的，想要让裸金属服务器实现VPC网络，需要对它进行overlay封装.

业界头部云厂商采用的是智能网卡方案，也就是在裸金属服务器上插入特殊的智能网卡，使用智能网卡中的vSwitch进行封装，这样只会消耗智能网卡的CPU和内存，不会消耗裸金属自身的资源.

智能网卡形态的裸金属服务器也存在一些不足。比如：智能网卡比普通网卡价格高；和云平台的紧耦合导致服务器的成熟度和生态方面有所欠缺；升级换代成本代价大；开发适配复杂度和门槛比较高；对云平台的稳定度有影响等等。由此可见，智能网卡，并不是政企客户最佳的方案。

![](https://bbs-img.huaweicloud.com/blogs/img/20221020/1666265000345734828.png)

### 裸金属服务器增强型网络解决方案
华为云 Stack 新推出一个既不依赖智能网卡，也不需要额外引入专有裸金属网关，同时又可以给裸金属服务器提供类云服务器网络能力的高性能、高可靠、低成本的方案 —— 增强型裸金属网络方案。

该方案通过复用裸金属的 TOR，做封装和解封装来实现 VPC 网络接入。在数据面上，裸金属服务器发送 / 接收的原始报文由 TOR 进行封装 / 解封装；在控制面上，自动配置 TOR 的转发表项、自动创建 TOR 和其它节点的隧道.

![](https://bbs-img.huaweicloud.com/blogs/img/20221021/1666332730710589627.png)

裸金属服务器使用的TOR不局限在华为CloudEngine交换机，而是可以采用第三方厂商的交换机来给客户提供相同的功能和体验。华为云Stack提供的[云网络南向生态框架](https://bbs.huaweicloud.com/blogs/364622)，支持快速集成第三方厂商硬件设备.

由于支持VXLAN的交换机设备功能和规格稳定，交换机OS软件升级和云平台松耦合，可以克服上面智能网卡裸金属服务器的形态方案缺点，给客户带来更加低成本、高性能的方案体验.