# vhost
参考:
- [VIRTIO & VHOST](https://zhuanlan.zhihu.com/p/38370357)
- [virtIO之VHOST工作原理简析](https://www.cnblogs.com/ck1020/p/7204769.html)
- [**【重识云原生】第四章云网络4.7.2节——virtio网络半虚拟化简介**](https://www.jianshu.com/p/0775347d86d4)
- [【重识云原生】第四章云网络4.7.3节——Vhost-net方案](https://cloud.tencent.com/developer/article/2030947)
- [【重识云原生】第四章云网络4.7.4节vhost-user方案——virtio的DPDK卸载方案](https://cloud.tencent.com/developer/article/2030949)
- [【重识云原生】第四章云网络4.7.5节vDPA方案——virtio的半硬件虚拟化实现](https://cloud.tencent.com/developer/article/2035805)

    为了解决高性能SRIOV网络的热迁移问题, 提出了vDPA.

    vDPA(vhost Data Path Acceleration)即是让virtio数据平面不需主机干预的解决方案。该框架由Redhat提出，实现了virtio数据平面的硬件卸载。控制平面仍然采用原来的控制平面协议，当控制信息被传递到硬件中，硬件完成数据平面的配置之后，数据通信过程由硬件设备（智能网卡）完成，虚拟机与网卡之间直通。中断信息也由网卡直接发送至虚拟机不需要主机的干预。这种方式，控制面比较复杂，硬件难以实现.

    Redhat提出的硬件vDPA架构，目前在DPDK和内核程序中均有实现，基本是未来的标准架构。Qemu支持两种方式的vDPA，一种是vhost-user，配合DPDK中的vDPA运行，DPDK再调用厂商用户态vDPA驱动；另一种方式是vhost-vdpa，通过ioctl调用到内核通用vDPA模块，通用vDPA模块再调用厂商硬件专有的vDPA驱动。

    1) 软件vDPA: 软件vDPA也叫VF relay，由于需要软件把VF上接收的数据通过virtio转发给虚拟机（VM），如Mellanox在OVS-DPDK实现了这个relay，OVS流表由硬件卸载加速，性能上与SR-IOV VF直通（passthrough）方式比略有降低，不过实现了虚拟机（VM）的热迁移特性。

    2) 硬件vDPA: 硬件vDPA实际上是借助virtio硬件加速，以实现更高性能的通信。由于控制面复杂，所以用硬件难以实现。厂商自己开发驱动，对接到用户空间DPDK的vDPA和内核vDPA架构上，可以实现硬件vDPA。目前Mellanox mlx5和Intel IFC对应的vDPA适配程序都已经合入到DPDK和kernel社区源码.

    ![](https://ask.qcloudimg.com/http-save/yehe-9519988/57ca602b94b2d57e1e427e6ce4300ac1.png?imageView2/2/w/1620)

    在硬件vDPA场景下，通过OVS转发的流量首包依然由主机上的OVS转发平面处理，对应数据流的后续报文由硬件网卡直接转发。

    后来在Bluefield-2上，由于集成了ARM核，所以NVIDIA在与UCloud的合作中，将OVS的控制面也完全卸载到网卡到ARM核上，这样主机上就可以将OVS完全卸载到网卡上。
- [*Virtio和Vhost介绍*](https://forum.huawei.com/enterprise/zh/thread-465473.html)
- [Virtio网络的演化之路](https://cloud.tencent.com/developer/article/1540284)
- [详解vhost-user协议及其在OVS DPDK、QEMU和virtio-net驱动中的实现](http://www.jeepxie.net/article/378987.html)
- [Vhost-user详解](https://www.jianshu.com/p/ae54cb57e608)
- [UCloud云盘是全链路的改造: client端用了VHOST，网络端用RDMA，后端提交用SPDK - UCan下午茶武汉站，为你全面挖宝分布式存储](http://www.ciotimes.com/index.php?m=content&c=index&a=app_show&catid=67&id=163237)
- [基于SPDK的UDisk全栈优化](/misc/pdf/io/02_Presentation_06_Full_Stack_Optimization_for_Udisk_with_SPDK_UCloud_Yutian.pdf)
- [从linux设备驱动模型看virtio初始化](http://blog.chinaunix.net/uid-28541347-id-5820032.html)
- [【重识云原生】第四章云网络4.7.6节——virtio-blk存储虚拟化方案](https://cloud.tencent.com/developer/article/2032186)

vhost是一种 virtio 高性能的后端驱动实现. 原有virtio后端驱动的i/o要进过vmm(qemu)和host kernel, 但它进一步缩短了i/o路径, 不再经过vmm.

它是位于host kernel的一个模块，用于和guest直接通信，所以数据交换就在guest和host kernel间进行，减少了上下文的切换. vhost相对与virto之前架构，把virtio驱动后端驱动从用户态放到了内核态中（vhost的内核模块充当virtiO后端驱动).

![virtio和vhost（vhost-net时vhost架构中的网卡实现）架构下内核的不同工作流程](/misc/img/virt/4nry30ay3h.jpeg)

![vhost工作原理](/misc/img/virt/905w5l04dt.jpeg)

## vhost-user
在 vhost 的方案中，由于 vhost 实现在内核中，guest 与 vhost 的通信，相较于原生的 virtio 方式性能上有了一定程度的提升，从 guest 到 kvm.ko 的交互只有一次用户态的切换以及数据拷贝. 这个方案对于不同 host 之间的通信，或者 guest 到 host nic 之间的通信是比较好的，但是**对于某些用户态进程间的通信(因此vhost-user不适合所有情况)**，比如数据面的通信方案，openvswitch 和与之类似的 SDN 的解决方案，guest 需要和 host 用户态的 vswitch 进行数据交换，如果采用 vhost 的方案，guest 和 host 之间又存在多次的上下文切换和数据拷贝，为了避免这种情况，业界就想出将 vhost 从内核态移到用户态. 这就是 vhost-user 的实现.

vhost-user和vhost类似，只是使用一个用户态进程vhost-user代替了内核中的vhost模块. vhost-user进程和Guset之间时通过共享内存的方式进行数据操作. 

vhost-user 和 vhost 的实现原理是一样，都是采用 vring 完成共享内存，eventfd 机制完成事件通知. 不同在于 vhost 实现在内核中，而 vhost-user 实现在用户空间中，用于用户空间中两个进程之间的通信，其采用共享内存的通信方式.

vhost-user 基于 C/S 的模式，采用 UNIX 域套接字（UNIX domain socket）来完成进程间的事件通知和数据交互，相比 vhost 中采用 ioctl 的方式，vhost-user 采用 socket 的方式大大简化了操作.

vhost-user 基于 vring 这套通用的共享内存通信方案，只要 client 和 server 按照 vring 提供的接口实现所需功能即可，常见的实现方案是 client 实现在 guest OS 中，一般是集成在 virtio 驱动上，server 端实现在 qemu 中，也可以实现在各种数据面中，如 OVS，Snabbswitch 等虚拟交换机.

在软件实现的网络I/O半虚拟化中，vhost-user在性能、灵活性和兼容性等方面达到了近乎完美的权衡.

![vhost-user的工作原理](/misc/img/virt/s5hebv5w16.jpeg)

> guest通过`lspci|grep -i virtio`查询是否存在virtio设备.

## virtio
virtio实现了VIRTIO API， 减少了VM-Exit次数， 提高了客户机I/O执行效率.

![Qemu模拟IO的架构和流程](/misc/img/virt/4f62ea32826daa12d8622f2e4379fe86.jpg)
使用QEMU模拟I/O的流程:
1. 当客户机中的设备驱动程序（Device Driver） 发起I/O操作请求时， KVM模块（Module） 中的I/O操作捕获代码会拦截这次I/O请求， 然后在
经过处理后将本次I/O请求的信息存放到I/O共享页（sharing page） ， 并通知用户空间的QEMU程序.
1. QEMU模拟程序获得I/O操作的具体信息之后， 交由硬件模拟代码（Emulation Code） 来模拟出本次的I/O操作， 完成之后， 将结果放回到I/O共享页， 并通
知KVM模块中的I/O操作捕获代码。
1. 最后， 由KVM模块中的捕获代码读取I/O共享页中的操作结果， 并把结果返回客户机中

当然， 在这个操作过程中， 客户机作为一个QEMU进程在等待I/O时也可能被阻塞。 另外，当客户机通过DMA（Direct Memory Access） 访问大块I/O时，QEMU模拟程序将不会把操作结果放到I/O共享页中， 而是通过内存映射的方式将结果直接写到客户机的内存中去， 然后通过KVM模块告诉客户机DMA操作已经完成.

QEMU模拟I/O设备的方式的优点: 可以通过软件模拟出各种各样的硬件设备， 包括一些不常用的或很老很经典的设备（比如intel的e1000网卡） ， 而且该方式不用修改客户机操作系统， 就可以使模拟设备在客户机中正常工作. 在KVM客户机中使用这种方式， 对于解决手上没有足够设备的软件开发及调试有非常大的好处。 而QEMU模拟I/O设备的方式的缺点是， 每次I/O操作的路径比较长， 有较多的VMEntry、 VMExit发生， 需要多次上下文切换（context switch） ， 也需要多次数据复制， 所以它的性能较差.

![virtio](/misc/img/virt/0a47027fa2416e84dd46fd68d6402181.jpg)
其中前端驱动（frondend， 如virtio-blk、 virtio-net等） 是在客户机中存在的驱动程序模块, 而后端处理程序（backend） 是在QEMU中实现的. 在前后端驱动之间， 还定义
了两层来支持客户机与QEMU之间的通信. 其中， “virtio”这一层是虚拟队列接口， 它在概念上将前端驱动程序附加到后端处理程序. 一个前端驱动程序可以使用0个或多个队列，
具体数量取决于需求. 例如， virtio-net网络驱动程序使用两个虚拟队列（一个用于接收，另一个用于发送） ， 而virtio-blk块驱动程序仅使用一个虚拟队列。 虚拟队列实际上被实现为跨越客户机操作系统和Hypervisor的衔接点， 但该衔接点可以通过任意方式实现， 前提是客户机操作系统和virtio后端程序都遵循一定的标准， 以相互匹配的方式实现它。 而virtio-ring实现了环形缓冲区（ring buffer） ， 用于保存前端驱动和后端处理程序执行的信息。 该环形缓冲区可以一次性保存前端驱动的多次I/O请求， 并且交由后端驱动去批量处理， 最后实际调用宿主机中设备驱动实现物理上的I/O操作， 这样做就可以根据约定实现批量处理而不是客户机中每次I/O请求都需要处理一次， 从而提高客户机与Hypervisor信息交换的效率.

virtio半虚拟化驱动的方式， 可以获得很好的I/O性能， 其性能几乎可以达到与native（即非虚拟化环境中的原生系统） 差不多的I/O性能. 所以, 在使用KVM之时， 如
果宿主机内核和客户机都支持virtio， 一般推荐使用virtio， 以达到更好的性能。 当然，virtio也是有缺点的， 它要求客户机必须安装特定的virtio驱动使其知道是运行在虚拟化环
境中， 并且按照virtio的规定格式进行数据传输。 客户机中可能有一些老的Linux系统不支持virtio， 还有一些主流的Windows系统需要安装特定的驱动才支持virtio。 不过， 较新的一些Linux发行版（如RHEL 6.3、 Fedora 17以后等） 默认都将virtio相关驱动编译为模块，可直接作为客户机使用，然而主流Windows系统中都有对应的virtio驱动程序可供下载使用(`dnf install virtio-win`, Fedora在[virtio-win](https://fedorapeople.org/groups/virt/virtio-win/virtio-win.repo)).

> Windows OS本身是没有安装virtio驱动的， 所以直接分配给Windows客户机以半虚拟化设备的话， 是无法识别加载驱动的，需要安装windows os时选择"加载驱动程序"来事先安装驱动, 安装该驱动后才可看到半虚拟化设备.

### virtio_balloon
通常来说， 要改变客户机占用的宿主机内存， 要先关闭客户机， 修改启动时的内存配置， 然后重启客户机才能实现. 而内存的ballooning（气球） 技术可以在客户机运行时动态地调整它所占用的宿主机内存资源， 而不需要关闭客户机.

ballooning技术形象地在客户机占用的内存中引入气球（balloon） 的概念。 气球中的内存是可以供宿主机使用的（但不能被客户机访问或使用） ， 所以， 当宿主机内存紧张，
空余内存不多时， 可以请求客户机回收利用已分配给客户机的部分内存， 客户机就会释放其空闲的内存。 此时若客户机空闲内存不足， 可能还会回收部分使用中的内存， 可能会将
部分内存换出到客户机的交换分区（swap） 中， 从而使内存“气球”充气膨胀， 进而使宿主机回收气球中的内存用于其他进程（或其他客户机） 。 反之， 当客户机中内存不足时， 也
可以让客户机的内存气球压缩， 释放出内存气球中的部分内存， 让客户机有更多的内存可用.

目前很多虚拟机， 如KVM、 Xen、 VMware等， 都对ballooning技术提供支持.

KVM中ballooning的工作过程主要有如下几步：
1. Hypervisor（即KVM） 发送请求到客户机操作系统， 让其归还一定数量的内存给Hypervisor
1. 客户机操作系统中的virtio_balloon驱动接收到Hypervisor的请求
1. virtio_balloon驱动使客户机的内存气球膨胀， 气球中的内存就不能被客户机访问. 如果此时客户机中内存剩余量不多（如某应用程序绑定/申请了大量的内存） ， 并且不能让内存气球膨胀到足够大以满足Hypervisor的请求， 那么virtio_balloon驱动也会尽可能多地提供内存使气球膨胀，尽量去满足Hypervisor所请求的内存数量（即使不一定能完全满足）.
1. 客户机操作系统归还气球中的内存给Hypervisor
1. Hypervisor可以将从气球中得来的内存分配到任何需要的地方
1. 即使从气球中得到的内存没有处于使用中， Hypervisor也可以将内存返还给客户机. 这个过程为： Hypervisor发送请求到客户机的virtio_balloon驱动； 这个请求使客户机
操作系统压缩内存气球； 在气球中的内存被释放出来， 重新由客户机访问和使用.

ballooning在节约和灵活分配内存方面有明显的优势， 其好处有如下3点。
1. 因为ballooning能够被控制和监控， 所以能够潜在地节约大量的内存。 它不同于内存页共享技术（ KSM是内核自发完成的， 不可控） ， 客户机系统的内存只有在通过命令
行调整balloon时才会随之改变， 所以能够监控系统内存并验证ballooning引起的变化.
1. ballooning对内存的调节很灵活， 既可以精细地请求少量内存， 也可以粗犷地请求大量的内存
1. Hypervisor使用ballooning让客户机归还部分内存， 从而缓解其内存压力。 而且从气球中回收的内存也不要求一定要被分配给另外某个进程（ 或另外的客户机）.

从另一方面来说， KVM中ballooning的使用不方便、 不完善的地方也是存在的， 其缺点如下：
1. ballooning需要客户机操作系统加载virtio_balloon驱动， 然而并非每个客户机系统都有该驱动（ 如Windows需要自己安装该驱动）
1. 如果有大量内存需要从客户机系统中回收， 那么ballooning可能会降低客户机操作系统运行的性能. 一方面， 内存的减少可能会让客户机中作为磁盘数据缓存的内存被放到
气球中， 从而使客户机中的磁盘I/O访问增加； 另一方面， 如果处理机制不够好， 也可能让客户机中正在运行的进程由于内存不足而执行失败
1. 目前没有比较方便的、 自动化的机制来管理ballooning， 一般都采用在QEMU monitor中执行balloon命令来实现ballooning. 没有对客户机的有效监控， 没有自动化的ballooning机制， 这可能会不便于在生产环境中实现大规模自动化部署.
1. 内存的动态增加或减少可能会使内存被过度碎片化， 从而降低内存使用时的性能。 另外， 内存的变化会影响客户机内核对内存使用的优化， 比如， 内核起初根据目前状
态对内存的分配采取了某个策略， 而后由于balloon的原因突然使可用内存减少了很多， 这时起初的内存策略就可能不是太优化了.

> KVM启用ballooning通过`config_virtio_balloon=m`

qemu启用ballooning: `-balloon virtio[,addr=addr]`, 也可使用较新的`-device`统一参数来分配balloon设备, 如`-device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x4`. QEMU monitor可通过`info balloon`查看guest balloon信息;`balloon <N>`设置balloon为<N>MB(不能超过设置的内存大小). guest通过`lspci|grep -i balloon`查询是否存在balloon设备.

`virsh setmem <domain-id or domain-name> <Amount of memory in KB>`可基于virtio_balloon来调整guest memory.

### virtio_net
### virtio_blk
guest通过`grep -i virtio_blk /boot/config-3.10.0-514.el7.x86_64`得到`CONFIG_VIRTIO_BLK=m`说明已支持virtio_blk

### vhost-scsi
- [vhost加速方案演进](https://blog.csdn.net/junbaozi/article/details/124001718)

目前主要有3种virtio-scsi后端的解决方案:
1. QEMU virtio-scsi

    是virtio-scsi最早的实现
1. Kernel vhost-scsi

    这个方案是QEMU virtio-scsi的后续演进，基于LIO在内核空间实现为虚拟机服务的SCSI设备。实际上vhost-kernel方案并没有完全模拟一个PCI设备，QEMU仍然负责对该PCI设备的模拟，只是把来自virtqueue的数据处理逻辑拿到内核空间了.

    相比QEMU virtio-scsi方案在具体的SCSI命令处理时减少了数据的内存复制过程，从而提高了性能.
1. SPDK vhost-user-scsi

    虽然Kernel vhost-scsi方案在数据处理时已经没有数据的复制过程，但是当Guest有新的请求时，仍然需要QEMU通过系统调用通知内核工作线程，这里存在两方面的开销：Guest内核需要更新PCI配置空间，QEMU需要捕获Guest的VMM自陷，然后通知Kernel vhost-scsi工作线程。

    SPDK vhost-user-scsi方案消除了这两方面的影响，后端的I/O处理线程在轮询所有的virtqueue，因此不需要Guest在添加新的请求到virtqueue后更新PCI的配置空间。SPDK vhost-user-scsi的后端I/O处理模块轮询机制加上零拷贝技术基本解决了前面提到的阻碍QEMU virtio-scsi性能提升的两个关键点.

    QEMU Guest和SPDK vhost target是两个独立的进程，vhost-user方案一个核心的实现就是队列在Guest和SPDK vhost target之间是共享的.

### vhost-net
virtio在宿主机中的后端处理程序（backend） 一般是由用户空间的QEMU提供的. 而vhost-net作为一个内核级别的后端处理程序， 将virtio-net的后端处理任务放到内核空间中执行, 从而提高效率. 即vhost是为了减少网络数据交换过程中的多次上下文切换， 让guest
与host kernel直接通信， 从而提高网络性能.

`-netdev tap,[,vnet_hdr=on|off][,vhost=on|off][,vhostfd=h][,vhostforce=on|off][,queues=n]`:
- vnet_hdr=on|off， 设置是否打开TAP设备的“IFF_VNET_HDR”标识

    “vnet_hdr=off”表示关闭这个标识， 而“vnet_hdr=on”表示强制开启这个标识。 如果没有这个标识的支持，则会触发错误.

    IFF_VNET_HDR是tun/tap的一个标识， 打开这个标识则允许在发送或接收大数据包时仅做部分的校验和检查。 打开这个标识, 还可以提高virtio_net驱动的吞吐量.
- vhost=on|off， 设置是否开启vhost-net这个内核空间的后端处理驱动， 它只对使用MSI-X(MSI, Message Signaled Interrupts)中断方式的virtio客户机有效
- vhostforce=on|off， 设置是否强制使用vhost作为非MSI-X中断方式的Virtio客户机的后端处理程序
- vhostfs=h， 设置去连接一个已经打开的vhost网络设备
- queues=n， 设置创建的TAP设备的多队列个数

在-device virtio-net-pci参数中， 也有几个参数与网卡多队列相关：
- mq=on/off， 分别表示打开和关闭多队列功能
- vectors=2*N+2， 设置MSI-X中断矢量个数

    假设队列个数为N， 那么这个值一般设置为2*N+2， 是因为N个矢量给网络发送队列， N个矢量给网络接收队列， 1个矢量用于配置目的， 1个矢量用于可能的矢量量化控制.

host查看vhost后端线程的方法: `ps -ef | grep 'vhost-'`, 其中vhost-xxx-0线程用于客户机的数据包接收， 而vhost-xxx-1线程用于客户机
的数据包发送.

guest通过`ethtool -l eth0 |grep Combined`可查看最多支持的队列数, `ethtool -L eth0 combined <N>`可用于启用未生效的队列.

xml如下:
```xml
<interface type='network'>
    <mac address='54:52:00:1b:ea:47'/>
    <source network='default'/>
    <target dev='vnet1'/>
    <model type='virtio'/>
    <driver name='vhost' queues='2'/>
</interface>
```

在客户机中， virtio-net多队列网卡也可以将网卡中断打散到多个CPU上由它们并行处理， 从而避免单个CPU处理网卡中断可能带来的瓶颈， 从而提高整个系统的网络处理能力.

一般来说， 使用vhost-net作为后端处理驱动可以提高网络的性能。 不过， 对于一些使用vhost-net作为后端的网络负载类型， 可能使其性能不升反降。 特别是从宿主机到其客户
机之间的UDP流量， 如果客户机处理接收数据的速度比宿主机发送数据的速度要慢， 这时就容易出现性能下降。 在这种情况下， 使用vhost-net将会使UDP socket的接收缓冲区更快
地溢出， 从而导致更多的数据包丢失。 因此， 在这种情况下不使用vhost-net， 让传输速度稍微慢一点， 反而会提高整体的性能.

使用qemu命令行时， 加上“vhost=off”（或不添加任何vhost选项） 就会不使用vhost-net作为后端驱动。 而在使用libvirt时， 默认会优先使用vhost_net作为网络后端驱动， 如果要选QEMU作为后端驱动， 则需要对客户机的XML配置文件中的网络配置部分进行如下的配置， 指定后端驱动的名称为“qemu”（而不是“vhost”）:
```xml
<interface type="network">
    ...
    <model type="virtio"/>
    <driver name="qemu"/>
    ...
</interface>
```

#### 用户态vhost-user
在大规模使用KVM虚拟化的云计算生产环境中， 通常都会使用Open vSwitch或与其类似的SDN方案， 以便可以更加灵活地管理网络资源。 通常在这种情况下， 在宿主机上会运行一个虚拟交换机（vswitch） 用户态进程， 这时如果使用vhost作为后端网络处理程序， 那么也会存在宿主机上用户态、 内核态的上下文切换. vhost-user的产生就是为了解决这样的问题， 它可以让客户机直接与宿主机上的虚拟交换机进程进行数据交换.

vhost-user协议实现了在同一个宿主机上两个进程建立共享的虚拟队列（virtqueue） 所需要的控制平面. 控制逻辑的信息交换是通过共享文件描述符的UNIX套接字来实现的；而在数据平面是通过两个进程间的共享内存来实现的.

vhost-user协议定义了master和slave作为通信的两端， master是共享自己virtqueue的一端， slave是消费virtqueue的一端。 在QEMU/KVM的场景中， master就是QEMU进程，slave就是虚拟交换机进程（如： Open vSwitch、 Snabbswitch等）.

DPDK（Data Plane Development Kit） 项目提供了网卡运行在polling模式的用户态驱动， 它可以和vhost-user、 Open vSwitch等结合起来使用， 可以让网络数据包都在用户态进行交换， 消除了用户态、 内核态的上下文切换开销，从而降低网络延迟、 提高网络吞吐量.

### kvm_clock
在保持时间的准确性方面， 虚拟化环境似乎天生就面临几个难题和挑战。 由于在虚拟机中的中断并非真正的中断， 而是通过宿主机向客户机注入的虚拟中断， 因此中断并不总
是能同时且立即传递给一个客户机的所有虚拟CPU（vCPU） 。 在需要向客户机注入中断时， 宿主机的物理CPU可能正在执行其他客户机的vCPU或在运行其他一些非QEMU进
程， 这就是说中断需要的时间精确性有可能得不到保障.

QEMU/KVM通过提供一个半虚拟化的时钟[1]， 即kvm_clock， 为客户机提供精确的System time和Wall time， 从而避免客户机中时间不准确的问题。 kvm_clock使用较新的硬
件（如Intel SandyBridge平台） 提供的支持， 如不变的时钟计数器（Constant Time StampCounter） 。 constant TSC的计数频率， 即使当前CPU核心改变频率（如使用了一些省电策略） ， 也能保持恒定不变。 CPU有一个不变的constant TSC频率是将TSC作为KVM客户机时钟的必要条件.

> 查看物理CPU对constant TSC的支持`cat /proc/cpuinfo | grep flags | uniq | grep constant_tsc`. guest通过`dmesg|grep clock|grep kvm-clock`验证是否使用了kvm_clock.

> kvm_clock的linux编译选项: `CONFIG_PARAVIRT_CLOCK=y`, 旧版是`CONFIG_KVM_CLOCK=y`

qemu默认使用kvm_clock作为时钟源.

Intel的一些较新的硬件还向时钟提供了更高级的硬件支持， 即TSC Deadline Timer, 比如Broadwell平台的CPU信息时已经有“tsc_deadline_timer”的标识.

TSC deadline模式， 不是使用CPU外部总线的频率去定时减少计数器的值， 而是用软件设置了一个“deadline”（最后期限） 的阈值， 当CPU的时间戳计数器的值大于或等于这
个“deadline”时， 本地的高级可编程中断控制器（Local APIC） 就产生一个时钟中断请求（IRQ） 。 正是由于这个特点（CPU的时钟计数器运行于CPU的内部频率而不依赖于外部
总线频率） ， TSC Deadline Timer可以提供更精确的时间， 也可以更容易避免或处理竞态条件（Race Condition）.

KVM模块对TSC Deadline Timer的支持开始于Linux 3.6版本， QEMU对TSC DeadlineTimer的支持开始于QEUM/KVM 0.12版本。 而且在启动客户机时， 在qemu命令行使用“-
cpu host”参数才能将这个特性传递给客户机， 使其可以使用TSC Deadline Timer.

### virtio对guest windows的优化
在QEMU/KVM中， 同样开发了与Windows客户机类似的半虚拟化优化的支持， 这部分代码主要在QEMU源码中的target-i386/hyperv.c和target-i386/hyperv.h文件中, 主要包括以下几个优化特性：
1. hv_relaxed这个特性关闭了在Windows客户机做严格的完整性检查从而避免一些Windows蓝屏死机， 因为在虚拟化环境中， 宿主机的负载可能很高， 对客户机的中断注入
可能会延迟
1. hv_vapic这个特性提供了对几个频繁访问的APIC寄存器的快速的M访问通道，如： HV_X64_MSR_EOI提供了对APIC EOI寄存器的快速访问
1. hv_spinlocks让KVM可以感知Windows客户机中尝试获取spinlock的次数， 超过这个次数才会被认为进入过度自旋等待（spin） 的状态
1. hv_time与前面提到的kvmclock类似， 提供了针对Windows客户机的半虚拟化时钟计数器， 在客户机中访问Timer频繁时提高性能

启用该优化的方法:
- qemu

    ```bash
    -cpu ...,hv_relaxed,hv_spinlocks=0x1fff,hv_vapic,hv_time
    ```
- libvirt

    ```xml
    <features>
        <hyperv>
        <relaxed state='on'/>
        <vapic state='on'/>
        <spinlocks state='on' retries='8191'/>
        </hyperv>
    <features/>
    <clock offset='localtime'>
        <timer name='hypervclock' present='yes'/>
    </clock>
    ```

目前主流的Windows系统都支持这些优化， 包括： Windows Vista、 Windows 7、Windows 10、 Windows 2008、 Windows 2012、 Windows 2016等。 即使对于不支持这些特性的Windows XP、 Windows 2003等， 打开这些优化特性一般来说也不会影响客户机的正常运行.

## Virtio网络的演化之路 
参考:
- [Virtio网络的演化之路](https://www.sdnlab.com/24468.html)
- [VirtIO and TC](https://hackmd.io/@ebhFqyF8QryV_ZWAFURDhQ/HkQLV90yv)
- [DPDK vhost-user详解 -  VFIO, Virtio-pmd,  IOMMU](https://www.sdnlab.com/24470.html)
- [devconf 19′: virtio 硬件加速](http://tech.mytrix.me/2019/05/devconf-19-virtio-%E7%A1%AC%E4%BB%B6%E5%8A%A0%E9%80%9F/)
- [浅谈网络I/O全虚拟化、半虚拟化和I/O透传](https://ictyangye.github.io/virtualized-network-io/2019/03/31/virtualized-IO.html)

纵观virtio网络的发展，控制平面由最原始的virtio到vhost-net协议，再到vhost-user协议，逐步得到了完善与扩充。而数据平面上，从原先集成在QEMU中或内核模块的中，到集成了DPDK数据平面优化技术的vhost-user，最终到使用硬件加速数据平面。在保留virtio这种标准接口的前提下，达到了SR-IOV设备直通的网络性能.

## 1. virtio-net驱动与设备:最原始的virtio网络, 不推荐, 不赘述.

## 2. vhost-net：处于内核态的后端
QEMU实现的virtio网络后端带来的网络性能并不如意，究其原因是因为频繁的上下文切换，低效的数据拷贝、线程间同步等. 于是，内核实现了一个新的virtio网络后端驱动，名为vhost-net.

与之而来的是一套新的vhost协议. vhost协议可以将允许VMM将virtio的数据面offload到另一个组件上，而这个组件正是vhost-net. 在这套实现中，QEMU和vhost-net内核驱动使用ioctl来交换vhost消息，并且用eventfd来实现前后端的通知. 当vhost-net内核驱动加载后，它会暴露一个字符设备在/dev/vhost-net. 而QEMU会打开并初始化这个字符设备，并调用ioctl来与vhost-net进行控制面通信，其内容包含virtio的特性协商，将虚拟机内存映射传递给vhost-net等. 对比最原始的virtio网络实现，控制平面在原有的基础上转变为vhost协议定义的ioctl操作（对于前端而言仍是通过PCI传输层协议暴露的接口），基于共享内存实现的Vring转变为virtio-net与vhost-net共享，数据平面的另一方转变为vhost-net，并且前后端通知方式也转为基于eventfd的实现.

### 3. vhost-user:使用DPDK加速的后端
vhost-user就是结合DPDK的各方面优化技术得到的用户态virtio网络后端, 这些优化技术包括：处理器亲和性，巨页的使用，轮询模式驱动等. 除了vhost-user，DPDK还有自己的virtio PMD作为高性能的前端.

基于vhost协议，DPDK设计了一套新的用户态协议，名为vhost-user协议，这套协议允许qemu将virtio设备的网络包处理offload到任何DPDK应用中（例如OVS-DPDK）.

vhost-user协议和vhost协议最大的区别其实就是通信信道的区别: Vhost协议通过对vhost-net字符设备进行ioctl实现; 而vhost-user协议则通过unix socket进行实现.

基于DPDK的Open vSwitch(OVS-DPDK)一直以来就对vhost-user提供了支持，可以通过在OVS-DPDK上创建vhost-user端口来使用这种高效的用户态后端.

### 4. vDPA:使用硬件加速数据面
Virtio作为一种半虚拟化的解决方案，其性能一直不如设备的pass-through，即将物理设备（通常是网卡的VF）直接分配给虚拟机，其优点在于数据平面是在虚拟机与硬件之间直通的，几乎不需要主机的干预。而virtio的发展，虽然带来了性能的提升，可终究无法达到pass-through的I/O性能，始终需要主机（主要是软件交换机）的干预.

vDPA(vhost Data Path Acceleration)即是让virtio数据平面不需主机干预的解决方案.

总体来看，vDPA的数据平面与SR-IOV设备直通的数据平面非常接近，并且在性能数据上也能达到后者的水准. 更重要的是vDPA框架保有virtio这套标准的接口，使云服务提供商在不改变virtio接口的前提下，得到更高的性能.

需要注意的是，vDPA框架中利用到的硬件必须至少支持virtio ring的标准，否则可想而知，硬件是无法与前端进行正确通信的. 另外，原先软件交换机提供的交换功能，也转而在硬件中实现.

### ps
对于性能的追求是永无止境的，I/O透传可让物理设备穿过宿主机、虚拟化层，直接被客户机使用，这种方式通常可以获取近乎native的性能. 但这种方式主要缺点是： 1.硬件资源昂贵且有限。 2.动态迁移问题，宿主机并不知道设备的运行的内部状态，状态无法迁移或恢复.

DPDK针对这两点问题都做了一定程度的解决, 另外还提供了一种基于硬件的PF（物理功能）转VF（虚拟功能），这相当于在网卡层面上就已经有了虚拟化的概念，把一个网卡的PF虚拟成几十上百个VF，这样可以把不同的VF透传给不同的虚拟机，这就是我们最熟悉的SR-IOV.

对于I/O透传在虚拟化环境中最严重的问题不是性能了，而是灵活性. 客户机和网卡之间没有任何软件中间层过度，也就意味着不存在负责交换转发功能的I/O栈，也就不会有软件交换机. 那么如果要想有一台server内部的软件交换功能如何实现呢. 业界的主要做法是把交换功能完全下沉到网卡，直接在智能网卡上实现虚拟交换功能, 这又带来了另一个问题，成本和性能的权衡.

而DPDK 18.05以后的版本似乎也解决了这一灵活性问题，为了充分发掘标准网卡（区别于智能网卡）在flow（流）层面上的功能，推出了VF representer, 可以直接将OVS上的流表规则下发到网卡上，实现网卡在VF之间的交换功能，这样就实现了高效灵活的虚拟化网络配置.