# kvm
ref:
- [虚拟化技术 — Linux Kernel 虚拟化网络设备](https://mp.weixin.qq.com/s?__biz=MzI3MDM0NjU3MA==&mid=2247486139&idx=1&sn=4ef45d1ab3ba391d0fa783bd1a075130&chksm=ead33a9edda4b3882d68c125906fc09102e57ef05a14334af3b809c2760a50d6ad194692df8d&cur_album_id=2372160472910839809&scene=189#wechat_redirect)
- [虚拟化技术 — 应用 Bridge 和 VLAN 子接口构建 KVM 多平面网络](https://mp.weixin.qq.com/s?__biz=MzI3MDM0NjU3MA==&mid=2247486161&idx=1&sn=f3d8438f6f68eaf282aa9da8132706b8&chksm=ead33af4dda4b3e20e141f2133c992b71fcdf32d6e2199482ea9a89461e5ccc6a7d582a39a02&cur_album_id=2372160472910839809&scene=189#wechat_redirect)
- [虚拟化技术 — Libvirt 异构虚拟化管理组件](https://mp.weixin.qq.com/s?__biz=MzI3MDM0NjU3MA==&mid=2247486297&idx=1&sn=046c2b4c363668a4d22af38d81bef17e&chksm=ead33b7cdda4b26ac7cd2b62f76de0bcf160f59a37f0f171eacbb21f1c3ea72da11c03ec21d2&cur_album_id=2372160472910839809&scene=189#wechat_redirect)

## tap/tun
Tap 和 Tun 是 Linux Kernel 2.4.x 版本之后引入的虚拟网卡设备, 简称 vNIC，完全由软件实现，是一种让 User Application 可以和 Kernel Network Stack 双向传输数据包的虚拟设备.

Tap（虚拟以太网卡）工作在数据链路层：实现了 Ethernet 协议，只能处理以太网数据帧，可以与物理网卡做 Bridge，支持二层广播
Tun（虚拟隧道网卡）工作在网络层：支持 IP 路由转发，但无法与物理网卡做 Bridge。实现了 Overlay 隧道封装协议（e.g. VxLAN、GRE etc..），用于建立基于 IP 协议的点对点（Peer To Peer）隧道.

## Veth-pair（虚拟网线）
Veth-pair 虚拟网线，和 Tap 搭配一起使用，用于连接两个虚拟网卡.

Veth-pair 的实现原理的本质是一种 “反转数据传输方向“ 的实现，即在 Kernel 中将需要被发送的数据包 “反转" 为新接收到的数据包，重新进入 Kernel Network Stack 处理. 这和 lo 回环设备的工作方式类似.

## bridge
ref:
- [brctl 指令](https://mp.weixin.qq.com/s?__biz=MzI3MDM0NjU3MA==&mid=2247486139&idx=1&sn=4ef45d1ab3ba391d0fa783bd1a075130&chksm=ead33a9edda4b3882d68c125906fc09102e57ef05a14334af3b809c2760a50d6ad194692df8d&cur_album_id=2372160472910839809&scene=189#wechat_redirect)

    **veth0 因为没有了 IP 地址，所以不再接入到 Network Stack**

Linux Bridge 是一个虚拟网桥设备，提供二层数据帧交换功能，支持绑定若干个 Ethernet 设备（包括物理网卡和虚拟网卡设备），并根据设备发出的数据帧中的 MAC 地址进行广播、转发或过滤处理.

Bridge 属于 Kernel 中的 L2 sub-system，当一个 Netdev（Ethernet 设备）挂载到 Bridge 时，就调用了 netdev_rx_handler_register() 函数，注册一个 Rx 收包函数，用于从 Netdev 接收数据包. 以后每当 Netdev 接收到数据包后，都会调用这个 Rx 收包函数，将数据包发送到 Bridge.

而当 Bridge 接收到数据包后，就会调用 br_handle_frame() 函数，执行二层交换功能, 包括：
1. 判断数据帧的类型（e.g. 广播包、单播包）
1. 查询 MAC-Port 映射表，确定转发目标端口或丢弃
1. 执行交换转发
1. 完成 MAC-Port 映射表自学习
1. etc

区别于物理交换机，Linux Bridge 的本质是一个 Kernel 中的虚拟网络设备，也具备虚拟网卡的特性，即：可以配置数据面的 MAC/IP 地址（区别于物理交换机的管理面 IP 地址）。通过这个 IP 地址，User Application 就可以通过 L3 sub-system 路由表，将数据报文直接发送至 Bridge 中进行二层交换.

当将物理网卡（e.g. eth0、eth1）挂载到 Bridge（e.g. br0）之后，Bridge 就成为了上层 Network Stack 和底层设备驱动层之间的中间层。此时对于 Network Stack 而言，只能看见 Bridge 设备，而看不见物理网卡，物理网卡对应的 IP 地址也会失效。所以，通常会将物理网卡原本的 IP 地址赋予 Bridge，以此来建立 Bridge 和外部网络进行通信的物理链路:
- Network Stack 收包时：物理网卡先将数据包送入 Bridge，并在 Bridge 上执行 br_handle_frame() 函数，判断数据包应该被转发、丢弃或者提交到 Network Stack
- Network Stack 发包时：通过 Bridge 将数据包发出，Bridge 会判断数据包应该通过 eth0 还是 eth1 还是 Both 发送到外部网络