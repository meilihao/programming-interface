# macvtap
ref:
- [基于libvirt kvm macvtap的虚拟化解决方案](https://blog.huiyiqun.me/2016/11/24/virtualization-with-libvirt-kvm-and-macvtap.html)
- [Qemu 使用 macvtap 桥接网络](https://runsisi.com/2022/08/27/qemu-macvtap/)
- [macvtap与vhost-net技术](https://www.cnblogs.com/echo1937/p/7249812.html)
- [Using the MacVTap driver](https://www.ibm.com/docs/en/linux-on-systems?topic=choices-using-macvtap-driver)

libvirt可使用macvtap来做网络端口复用, 相比于以前的bridge方案确实有一些优势，如果虚拟机的流量确实很大，可以用这套方案，来减少物理机的CPU和 网卡的压力.

macvtap与macvlan实际上是内核里面的两个特性，用于在物理网卡后面接一些虚拟端口，复用物理端口，但是利用了网卡 的一个较新的特性，所以从性能上来说比纯虚拟交换性能更高，属于一种半虚拟化方案.

MacVlan的功能是给同一个物理网卡配置多个MAC地址，可以在软件上配置多个以太网口，属于物理层的功能. MacVTap是用来替代TUN/TAP和Bridge内核模块的。MacTap是基于MacVlan这个模块，提供TUN/TAP中TAP设备使用的接口，使用MACVTap以太网口的虚拟机能够通过TAP设备接口，直接将数据传递到内核中对应的MacVTap以太网中.

macvtap实际上是在macvlan创建的虚拟端口后面接了一个字符设备，方便某些场景（比如虚拟机）.