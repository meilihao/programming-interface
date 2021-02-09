# vhost
参考:
- [VIRTIO & VHOST](https://zhuanlan.zhihu.com/p/38370357)
- [virtIO之VHOST工作原理简析](https://www.cnblogs.com/ck1020/p/7204769.html)
- [*Virtio和Vhost介绍*](https://forum.huawei.com/enterprise/zh/thread-465473.html)
- [Virtio网络的演化之路](https://cloud.tencent.com/developer/article/1540284)
- [详解vhost-user协议及其在OVS DPDK、QEMU和virtio-net驱动中的实现](http://www.jeepxie.net/article/378987.html)
- [Vhost-user详解](https://www.jianshu.com/p/ae54cb57e608)
- [UCloud云盘是全链路的改造: client端用了VHOST，网络端用RDMA，后端提交用SPDK - UCan下午茶武汉站，为你全面挖宝分布式存储](http://www.ciotimes.com/index.php?m=content&c=index&a=app_show&catid=67&id=163237)
- [基于SPDK的UDisk全栈优化](/misc/pdf/io/02_Presentation_06_Full_Stack_Optimization_for_Udisk_with_SPDK_UCloud_Yutian.pdf)

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