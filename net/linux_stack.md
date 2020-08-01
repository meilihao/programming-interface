# linux net stack
- ![linux收包](/misc/img/net/深度截图_选择区域_20191126214406.png)
- ![linux发包](/misc/img/net/深度截图_选择区域_20191126215757.png)


## nic
```
$ ls -al /sys/class/net # ip addr 获取的所有网络接口均在/sys/class/net下(因为部分文件不是接口,比如`bonding_masters`)且是均会链接到`/sys/devices/xxx`下, 通过`/sys/devices/virtual/net`排除虚拟接口即可得到真实网卡
lrwxrwxrwx  1 root 0 1月   3 14:04 enp2s0 -> ../../devices/pci0000:00/0000:00:1c.1/0000:02:00.0/net/enp2s0
lrwxrwxrwx  1 root 0 1月   3 14:04 lo -> ../../devices/virtual/net/lo # 虚拟接口会链接到`/sys/devices/virtual/net`下
```

## devices
### ifb
和tun一样，ifb也是一个虚拟网卡，和tun一样，ifb也是在数据包来自的地方和去往的地方做文章.

ifb驱动模拟一块虚拟网卡，它可以被看作是一个**只有TC过滤功能的虚拟网卡，说它只有过滤功能**，是因为它并不改变数据包的方向，即对于往外发的数据包被重定向到ifb之后，经过ifb的TC过滤之后，依然是通过重定向到之前的网卡再发出去；对于一个网卡接收的数据包，被重定向到ifb之后，经过ifb的TC 过滤之后，依然被重定向到之前的网卡再继续进行接收处理，不管是从一块网卡发送数据包还是从一块网卡接收数据包，重定向到ifb之后，都要经过一个经由ifb 虚拟网卡的dev_queue_xmit操作.

![原理](/misc/img/net/20141101151854140.jpg)

### inter
`interN`是vmware的虚拟网卡, 通过`/sys/class/net/inter0/device/vendor`和[`#define 	PCI_VENDOR_ID_VMWARE   0x15AD`](https://doc.dpdk.org/api-1.6/rte__pci__dev__ids_8h.html)得知.

结合服务器型号[SuperStorage 2028R-DE2CR24L](https://www.supermicro.org.cn/en/products/system/2U/2028/SSG-2028R-DE2CR24L.cfm)和`ethtool inter0`的`Speed:  100Mb/s`, 推测是其中的`100Mb private ethernet between controller nodes`, 即双控内连互通网卡.

### 光纤网卡
光纤网卡的网络接口命名与普通网卡相同, 但`ethtool xxx`的`Supported ports`是`FIBRE`.

### tun/tap 
tun/tap是tun(tunnel)设备和tap设备的总称.

tun是点对点设备,在tcp/ip中的ip层(即三层)处理数据包, 在linux中用`tun<N>`表示.

tap是虚拟的以太网设备, 在tcp/ip中的数据传输层(即2层)处理以太网帧, 在linux中用`tap<N>`表示.

Linux内核通过TAP/TUN设备向绑定该设备的用户空间应用发送数据；反之，用户空间应用也可以像操作硬件网络设备那样，通过TAP/TUN设备发送数据.

基于TAP驱动，即可以实现虚拟网卡的功能，虚拟机的每个vNIC都与Hypervisor中的一个TAP设备相连. 当一个TAP设备被创建时，Linux设备文件目录下将会生成一个与之对应的字符设备文件，用户空间应用可以像打开普通文件一样打开这个文件进行读写.

当对TAP设备文件执行write（）操作时，对于Linux网络子系统来说，相当于位于内核空间的TAP设备收到了数据，Linux内核收到此数据后将根据网络配置进行后续处理，处理过程类似于普通的物理网卡从外界收到数据. 当用户空间应用执行read（）请求时，相当于向内核查询TAP设备上是否有数据需要被发送，有的话则取出到用户空间里，从而完成 TAP 设备发送数据的功能. 在这个过程中，TAP 设备可以被当成本机的一个网卡，而操作TAP设备的应用程序相当于另外一台计算机，它通过read/write系统调用，和本机进行网络通信.

实际上，除了虚拟网卡的驱动，TAP/TUN 驱动程序还包括一个字符设备驱动，内核通过字符设备/dev/net/tun与用户空间应用传递网络数据，同时，利用网卡驱动部分接收并发送来自TCP/IP协议栈的网络数据，或反过来将收到的网络数据传给协议栈处理.

### bridge
bridge是一个虚拟交换机(工作在二层), 可将多个网络接口连接到同一个网段上, 在linux中用`br<N>`表示.

Bridge可以绑定其他Linux网络设备作为从设备，并将这些从设备虚拟化为端口，当一个从设备被绑定到 Bridge 上时，就相当于真实网络中的交换机端口插入了一个连接有终端的网线.

![Linux Bridge结构](/misc/img/net/Image00021_net.jpg)

上图的Bridge设备br0绑定了真实设备eth0与虚拟设备tap0/tap1. 此时，对于Hypervisor的网络协议栈来说，只看得到br0，并不会关心桥接的细节. 当这些从设备接收数据包时，会将其提交给br0决定数据包的去向，br0会根据MAC地址与端口的映射关系进行转发.

因为Bridge工作在第二层，所以绑定在br0上的从设备eth0、tap0与tap1**均不需要再设置IP地址**，对上层路由器来说，它们都位于同一子网，因此只需为br0设置IP地址（Bridge设备虽然工作于二层，但它只是Linux网络设备抽象的一种，能够设置IP地址也可以理解），比如10.0.1.0/24. 此时，eth0、tap0与tap1均通过br0处于10.0.1.0/24网段.

因为具有自己的IP地址，br0可以被加入路由表，并利用它发送数据，而最终实际的发送过程则由某个从设备来完成. 此时**相当于Linux拥有了另外一个隐藏的虚拟网卡和Bridge绑定，IP地址可以看成是这个网卡的**，当有符合此IP地址的数据到达Bridge 时，内核协议栈认为收到了一包目标为本机的数据，此时应用程序可以通过Socket接收它.

Bridge的实现有一个限制：**当一个设备被绑定到Bridge上时，该设备的IP地址会变得无效，Linux不再使用该IP地址在三层接收数据**. 比如，如果eth0本来具有自己的IP地址192.168.1.1，在绑定到br0之后，它的IP地址会失效，用户程序不再能接收或发送到这个IP地址的数据，只有目的地址为br0 IP的数据包才会被Linux接收.

## 相关内核文件
- `/sys/class/net/{interface}` : 网络接口对应的目录
- `/sys/class/net/{interface}/bonding/slaves` : bond使用的底层网络接口

## 丢包监控
因为非法数据包, 或**超过接收缓冲区大小**等情况的丢包是不会在日志中输出.

dropwatch是监控kernel的网络栈丢包的工具.

## FAQ
### nginx listen `0.0.0.0`能否对之后添加的ipv4生效?
```bash
# systemctl restart nginx
# ip addr add 192.168.10.161/24 dev ens3
# curl 192.168.10.161 # 能正常返回, 说明会生效
```