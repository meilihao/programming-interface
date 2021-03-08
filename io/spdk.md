# spdk
## 背景
目前这一代的闪存存储，比起传统的磁盘设备，在性能（performance）、功耗（power consumption）和机架密度（rack density）上具有显著的优势. 而且这些优势将会继续增大，使闪存存储作为下一代设备进入市场.

因此面临一个主要的挑战：随着存储设备继续发展，它将面临远远超过正在使用的软件体系结构的风险（即存储设备受制于相关软件的不足而不能发挥全部性能）. 因为吞吐量和延迟性能比传统的磁盘好太多，现在总的处理时间中，存储软件占用了更大的比例. 换句话说，存储软件栈的性能和效率在整个存储系统中越来越重要. 

Inte为了解决该矛盾, 帮助存储OEM（设备代工厂）和ISV（独立软件开发商）整合硬件而构造了一系列驱动，以及一个完善的、端对端的参考存储体系结构，被命名为Storage Performance Development Kit（SPDK）. SPDK的目标是通过同时使用Intel的网络技术，处理技术和存储技术来突出显著地提高效率和性能.

## 架构
![SPDK Architecture 19.01](/misc/img/io/spdk/spdk_arch_1901.jpeg)

核心原理: 运行于用户态和轮询模式
- 避免内核上下文切换和中断将会节省大量的处理开销，允许更多的时钟周期被用来做实际的数据存储
- 轮询模式驱动（Polled Mode Drivers, PMDs）改变了I/O的基本模型

    在旋转设备时代（磁带和机械硬盘），中断开销只占整个I/O时间的一个很小的百分比. 然而，在固态设备的时代，持续引入更低延迟的持久化设备，中断开销成为了整个I/O时间中不能被忽视的部分. 这个问题在更低延迟的设备上只会越来越严重.


SPDK的指导原则是通过消除每一处额外的软件开销来提供最少的延迟和最高的效率. SPDK的目标是能够把硬件平台的计算、网络、存储的最新性能进展充分发挥出来.

spdk层次:
1. drivers

    - NVMe Devices：SPDK的基础组件，这个高优化无锁的驱动提供了高扩展性，高效性和高性能

        - NVMe over Fabrics（NVMe-oF）initiator：从程序员的角度来看，本地SPDK NVMe驱动和NVMe-oF启动器共享一套共同的API命令。这意味着，比如本地/远程复制非常容易实现.
    - Inter QuickData Technology：也称为Intel I/O Acceleration Technology（Inter IOAT，英特尔I/O加速技术），这是一种基于Xeon处理器平台上的copy offload引擎。通过提供用户空间访问，减少了DMA数据移动的阈值，允许对小尺寸I/O或NTB的更好利用。
1. storage services
    - Block device abstration layer（bdev）：这种通用的块设备抽象是连接到各种不同设备驱动和块设备的存储协议的粘合剂。还在块层中提供灵活的API用于额外的用户功能（磁盘阵列，压缩，去冗等等）
    - Ceph RADOS Block Device（RBD）：使Ceph成为SPDK的后端设备，比如这可能允许Ceph用作另一个存储层
    - Blobstore：为SPDK实现一个高精简的文件式语义（非POSIX）。这可以为数据库，容器，虚拟机或其他不依赖于大部分POSIX文件系统功能集（比如用户访问控制）的工作负载提供高性能基础
    - Linux Asynchrounous I/O（AIO）：允许SPDK与内核设备（比如机械硬盘）交互
    - Logical Volume：类似于内核软件栈中的逻辑卷管理，SPDK通过Blobstore的支持，同样带来了用户态逻辑卷的支持，包括更高级的按需分配、快照、克隆等功能
- storage protocols

    - iSCSI target：建立了通过以太网的块流量规范，大约是内核LIO效率的两倍。现在的版本默认使用内核TCP/IP协议栈
    - NVMe-oF target：实现了新NVMe-oF规范。虽然这取决于RDMA硬件，NVMe-oF的目标可以为每个CPU核提供高达40Gbps的流量
    - vhost-scsi target：KVM/QEMU的功能利用了SPDK NVMe驱动，使得访客虚拟机访问存储设备时延迟更低，使得I/O密集型工作负载的整体CPU负载减低

 
从流程上来看，spdk有数个子构件组成，包括网络前端、处理框架和存储后端:
- 前端由DPDK网卡驱动、用户态网络服务UNS（这是一个Linux内核TCP/IP协议栈的替代品，能够突破通用TCP/IP协议栈的种种性能限制瓶颈）组成. 

    DPDK给网卡提供一个高性能的包处理框架, 为数据从网卡到用户态空间提供一个数据快速通道；用户态网络服务则破解TCP/IP包并生成iSCSI命令
- 处理框架得到包的内容，并将iSCSI命令翻译为SCSI块级命令. 不过，在将这些命令送给后端驱动之前，SPDK提供一个API框架以加入用户指定的功能，即spcial sauce. 例如缓存，去冗，数据压缩，加密，RAID和纠删码计算等，诸如这些功能都包含在SPDK中。不过这些功能仅仅是为了帮助我们模拟应用场景，需要经过严格的测试优化才可使用。
- 数据到达后端驱动，在这一层中与物理块设备发生交互，即读与写

SPDK包括了几种存储介质的用户态轮询模式驱动：
- NVMe设备
- linux异步IO设备如传统磁盘
- 基于块地址的内存应用的内存驱动（如RAMDISKS）
- 使用Intel I/O加速技术设备

## 其他
- [SPDK用户态hotplug处理](https://blog.csdn.net/weixin_37097605/article/details/101514668)