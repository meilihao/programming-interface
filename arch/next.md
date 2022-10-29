# 趋势
高带宽、低延迟是目前数据中心应用的基本需求. NVM（Non-Volatile Memory）和 RDMA（Remote Direct Memory Access）可以称得上加速数据中心应用的两架马车，分别从存储和网络方面满足高带宽、低延迟的需求.

## 硬件
### Bare metal（裸机服务器）方案
参考:
- [裸金属服务器业务（BMS）的现状调研](http://www.brofive.org/?p=4512)
- [阿里云弹性裸金属服务器-神龙架构（X-Dragon）揭秘](http://cnzzidc.com/cnzz1372.html)
- [继AWS Nitro和阿里云神龙，京东造起了“京刚”](https://finance.sina.com.cn/tech/2021-03-09/doc-ikkntiak6464795.shtml)
- [阿里云张献涛：自主最强DPU神龙的秘诀](https://developer.aliyun.com/article/867094)

一般而言，实现“软件的硬件化处理”，是将这个硬件做成一个网卡，插在卡槽上，硬件层面还有一个层做在Bios之上，操作系统之下，负责硬件资源的切分。云主机的性能扩展不需要占用CPU资源，显著提升性能的可扩展性.

裸金属服务器是对物理主机进行云计算场景下的资源管理与调度, 为用户提供专属的物理服务器, 满足核心应用场景对高性能及稳定性的需求. 它的特点是:
1. 性能无损

  所有号称性能损耗小的VM技术，实际上还是有5-15%甚至更高的损耗，这种损耗在VM规格增大的时候，也会增大
1. 安全隔离
1. 自动化交付
1. 自动化运维

> zstack软件裸金属技术: IPMI(可通过它进入裸金属设备bios设置PXE等) + PXE/TFTP DHCP/Kickstart, 本质是直接将物理机转为裸金属服务器.

思路是做offload——把原本许多需要系统调用、内核操作的工作交给物理设备来做，绕开系统、CPU，简化流程和操作.

当前实现:
- AWS的`Nitro架构`
  ref：
  - [AWS Nitro架构简介](https://blog.csdn.net/lindahui2008/article/details/109397778)
  - [AWS Nitro System介绍](https://aijishu.com/a/1060000000215894)

  参考:
  - [裸金属服务器业务（BMS）的现状调研1：AWS Nitro System](http://www.brofive.org/?p=4395)

  Nitro系统主要由三部分组成:
  1. 以PCIe卡形式呈现的Nitro卡，主要包括支持网络功能的VPC（Virtual Private Cloud）卡，支持存储功能的EBS（Elastic Block Store）、Instance Storage卡和支持系统控制的Nitro Controller卡。
  1. Nitro安全芯片，该芯片提供Hardware Root of Trust，防止运行于通用服务器上的软件对non-volatile storage进行修改，比如虚拟机的UEFI程序。
  1. 运行于通用服务器的Nitro Hypervisor，这是个基于kvm的轻量级hypervisor，主要提供CPU和内存的管理功能，不提供设备的模拟（因为所有的设备都是通过透传的方式添加到虚拟机中）。

  Nitro系统的主要职责，它将存储、网络、管理和安全能力都offload到专有的硬件之上，免去了与通用计算设备抢占资源的各种麻烦，节省资源的同时提升效率.

  有无Nitro架构上发生的变化:
  ![](/misc/img/arch/279072a9d5c1422c94a495997a59b884.jpeg)

  实际实现中，Nitro分为三大方向，彼此相互独立:
  - NitroHypervisior：专有硬件(ASIC芯片)上承载hypervisior(基于KVM的Hypervisior)，实现近似裸机服务器的性能表现
    直接接管原本需要在CPU上运行的Hypervisor做的事
  - NitroCards：专有硬件承载存储、网络功能，以及控制EC2实例的业务逻辑.
    ![](/misc/img/arch/79d2775165d246078905972ebb94c0d4.jpeg)
    Nitro Card 控制器是一个大管家，可以管理Nitro的Hypervisior、存储、网络以及安全功能. AWS通过它在后台进行系统管理, 比如:
    1. 充当NVMe控制器的角色，从Hypervisior做offload.
    1. VPC网络方面，Nitro card充当网卡、ENA控制器的角色，是具备SDN功能的专有硬件. 同样从Hypervisior做offload，让服务器CPU无需预留资源处理网络事务
  - Nitro安全芯片：硬件层的安全验证能力, 包括记录服务器上各种控制器firmware的IO操作，同时还能升级管理这些firmware

- Aliyun的`X-Dragon架构`
  参考:
  - [从 VMWare 到阿里神龙，虚拟化技术 40 年演进史](https://blog.csdn.net/csdnnews/article/details/105548972)
  - [突破性能极限，阿里云神龙最新ASPLOS论文解读](https://www.sohu.com/a/380803236_115128)
  - [NBJL 2020论文导读13： High-density Multi-tenant Bare-metal Cloud](https://nbjl.nankai.edu.cn/2020/0407/c12124a268844/page.htm)

  ![](/misc/img/arch/25c7936befd17a4230afacb6bae45164fcc35ef2.png)

  ![](/misc/img/arch/09148045f2d6738deba330df18df6ff4dda9d696.png)

  ![](/misc/img/arch/e7f5c50902b2d79d6c13c8e92e96ad40c757b02e.png)

  阿里云的神龙架构是通过一个PCIe的MoC卡（实际是一个包含CPU、内存、磁盘、智能网卡的系统）**把Hypervisor、带外管理功能都卸载到这个卡上**。MoC卡对外可以连接EBS存储和VPC网络，通过virtio（一种半虚拟化的设备抽象接口规范）访问IO设备，实现裸金属和虚拟化同样的扩展和管理功能，和现有云环境可以通过私有接口或Open API无缝集成.

  [神龙Moc卡网上有文章说是智能网卡](https://blog.csdn.net/junbaozi/article/details/123834314), 这个更形象, 毕竟它只是弹性裸金属服务器上的一个pcie设备.

  ![X-Dragon MOC卡架构详解](https://yqfile.alicdn.com/25c7936befd17a4230afacb6bae45164fcc35ef2.png)解读:
  1. 上面是弹性裸金属的一个实例，包含CPU、内存、moc卡等资源，并且这些资源都是物理的
  1. 中间的VirtlO-NIC，从名字推断，应该就是基于KVM的I/O虚拟化通用框架Virtio实现的定制化网卡驱动，用于配合网卡的DPDK特性来大幅加速网络转发性能
  1. VirtlO-Blk，应该就是基于KVM的I/O虚拟化通用框架Virtio实现的定制化存储驱动，用于结合SPDK特性提升对块设备的访问性能. moc卡充当virtio后端.
  1. 因VirtlO-NIC、VirtlO-Blk其实均是KVM通用I/O框架的实现，与云主机KVM技术同宗同源，所以神龙裸金属便能与云上的所有镜像、云上的所有系统、虚拟机和物理机之间完全兼容；
  1. 除了上述功能，其他外部设备，例如键盘、鼠标、显示器等，基本可以判断也是基于KVM的I/O虚拟化框架来虚拟实现。这样就可以实现和KVM虚拟机一样的对外接口，使得运营的操作系统不需要做任何的修改，在虚拟机上拿过来在X-Dragon MOC卡上直接用。
  1. 最下面的整个X-Dragon Hypervisor层，基本也可以判断，应该也是基于KVM来实现的，完完全全运行在这张Moc卡上；
  由此，Moc可以无缝支持阿里云的云盘、VPC网络、存储/网络设备热插拔、32块弹性物理网卡，同时对X86、ARM、Power等CPU兼容。

  > 感觉moc卡有点像xen中的物理dom0.

  > 推测moc卡应该和[腾讯智能网卡(最新版代号银杉,上一版代号水杉)一样采用FPGA模拟virtio设备](https://blog.csdn.net/junbaozi/article/details/123834314). github上类似的有[
virtio-fpga](https://github.com/RSPwFPGAs/virtio-fpga)

  `X-Dragon架构`性能未损失原因: 将原先跑在cpu上的Hypervisior offload到了`X-Dragon架构`上.

  在阿里云神龙高密裸金属架构中，每个裸金属实例都运行在一个单独设计的计算子板上，该计算子板带有**物理的CPU和内存**模块. BM-Hive为每个计算子板配备了硬件/软件混合 virtio I/O 系统，使客户实例能够直接访问阿里云网络和存储服务. BM-Hive 可在单个物理服务器中托管多达 16 个裸金属实例，显著提高裸金属服务器的实例密度. 此外，BM-Hive 在硬件级别严格隔离每个裸金属实例，以提高安全性和隔离性.

  神龙裸金属BM-Hive由于采用了计算子板直接运行实例，避免了任何传统CPU/内存虚拟化的开销. 因为当前虚拟化的基本原理决定了CPU必须要在vCPU环境与物理CPU环境下来回切换（VM-Exit）, 频繁的切换会导致严重的VM性能问题.
- OpenStack的Ironic项目
- 京东的京刚

  京刚智能芯片卸载网络转发和存储IO功能，让硬件性能不受损，支持了业界标准的SRIOV虚拟化技术，保证设备虚拟化无开销；此外实现了云原生的Virtio-Net和Virtio-Blk设备模型，兼容性更好；同时芯片级的硬件隔离技术，更加安全可靠。

  而自主研发的智能网卡，则是基于英特尔® C5000X-PL 和英特尔® 至强® D芯片，全面利用英特尔®FPGA的可编程能力，能够大幅提高端口速度，提升网卡的业务灵活性，通过 FPGA 和至强 D 芯片协同来定制并优化网络负载。

### 指令集
AVX 512指令集强化的向量和浮点计算

## 网络
- 网卡offload, 比如checksum offload.
- HPCC (High Precision Congestion Control- 高精度拥塞控制)
- QSFP-DD 400G已成熟，QSFP-DD800G正在进行.

### RDMA Vs DPDK
参考:
- [RDMA](https://yq.aliyun.com/articles/394912)
- [RDMA学习路线总结](https://my.oschina.net/SysuHuyh5LoveHqq/blog/842767)
- [借助RDMA功能的互连实现您企业软件定义的数据中心基础设施的效率最大化](https://yq.aliyun.com/articles/137416)
- [UCloud高性能RoCE网络设计](http://blog.ucloud.cn/archives/4364)
- [RDMA技术原理分析、主流实现对比和解析](https://www.sohu.com/a/229080366_632967)
- [RDMA-远程直接内存访问-01-RDMA 协议 iWARP 和 RoCE](https://houbb.github.io/2019/11/20/rdma-01-protocol)

DPDK思路:
- 网络层：硬件中断->放弃中断流程
- 用户层通过设备映射取包->进入用户层协议栈->逻辑层->业务层

核心技术：
1. 将协议栈上移到用户态，利用UIO技术直接将设备数据映射拷贝到用户态
1. 利用大页技术，降低TLB cache miss，提高TLB访问命中率
1. 通过CPU亲和性，绑定网卡和线程到固定的core，减少cpu任务切换
1. 通过无锁队列，减少资源的竞争

优势：
1. 减少中断次数
1. 减少内存拷贝次数
1. 绕过linux的协议栈，用户获得协议栈的控制权，能够定制化协议栈以降低复杂度

劣势：
1. 内核栈转移至用户层增加了开发成本

> [全用户态网络开发套件 F-Stack 架构分析](https://cloud.tencent.com/developer/article/1005218)

RDMA思路:
网卡硬件收发包并进行协议栈封装/解析，然后将数据存放到指定内存地址，而不需要CPU干预. 即消除主机CPU 中不必要的频繁数据传输，减少通信对CPU的占用, RDMA就是由此来优化通信性能. RDMA 通过 Memory Region 机制使得网卡能够直接读写用户态的内存数据，避免了数据拷贝和上下文切换.

RDMA三大特性：CPU offload 、kernel bypass、zero-copy.

核心技术：协议栈硬件offload

优势：
1. 协议栈offload，解放cpu
1. 减少了中断和内存拷贝，降低时延
1. 高带宽

劣势：
1. 特定网卡才支持，成本开销相对较大
1. RDMA提供了完全不同于传统网络编程的API，一般需要对现有APP进行改造，引入额外开发成本

目前支持RDMA的网络协议有：
- InfiniBand(IB): 从一开始就支持RDMA的新一代网络协议. 由于这是一种新的网络技术，因此需要支持该技术的网卡和交换机.
- RDMA过融合以太网(RoCE): 即RDMA over Ethernet, 允许通过以太网执行RDMA的网络协议, 这允许在标准以太网基础架构(交换机)上使用RDMA，只不过网卡必须是支持RoCE的特殊的NIC.

  RoCE又包括RoCEv1和RoCEv2两个版本(RoCEv2的最大改进是支持IP路由)
- 互联网广域RDMA协议(iWARP): 即RDMA over TCP, 允许通过TCP执行RDMA的网络协议. 这允许在标准以太网基础架构(交换机)上使用RDMA，只不过网卡要求是支持iWARP(如果使用CPU offload的话)的NIC; 否则，所有iWARP栈都可以在软件中实现，但是失去了大部分的RDMA性能优势.

总结
相同点：
1. 两者均为kernel bypass技术，可以减少中断次数，消除内核态到用户态的内存拷贝

相异点：
1. DPDK是将协议栈上移到用户态，而RDMA是将协议栈下沉到网卡硬件，DPDK仍然会消耗CPU资源
1. DPDK的并发度取决于CPU核数，而RDMA的收包速率完全取决于网卡的硬件转发能力
1. DPDK在低负荷场景下会造成CPU的无谓空转，RDMA不存在此问题
1. DPDK用户可获得协议栈的控制权，可自主定制协议栈；RDMA则无法定制协议栈

因为**RDMA以前只能运行在InfiniBand网络下，为了将这种技术用在以太网环境下，就逐步发展出了RoCE/iWarp两种协议**, 可参考[深入浅出全面解析RDMA](https://zhuanlan.zhihu.com/p/37669618). RoCE目前主要是由Mellonax主导，和TCP协议无关，性能更好. iWarp主要由Chelsio主导，下层会依赖TCP协议，性能和可扩性行都差一些，优点是考虑了对广域网的支持. 目前来看RoCE比iWarp前景更好，实际使用也更广泛.

## 存储
参考:
- [AIO 的新归宿：io_uring](https://zhuanlan.zhihu.com/p/62682475)
- [Alibaba Cloud Linux 2 LTS 率先提供支持 io_uring](https://developer.aliyun.com/article/764720)
- [网络编程的未来，io_uring？](https://www.sdnlab.com/24474.html)

io_uring和SPDK

> [对于SPDK中VPP的整合, 目前的状况是在SPDK中的集成, 将会在20.07停止, 20.10中把VPP的sock实现从SPDK 中删除. 其主要原因是基于VPP的sock实现并没有体现出相应的性能优势](https://lists.01.org/hyperkitty/list/spdk@lists.01.org/thread/L7FST3E5CKUCUK4SX24IYXMDWBHH4VAA/)

SPDK 中使能io_uring:
1. 至少满足内核版本大于5.4.3. 当然越高版本的内核对于io_uring的支持越好，比如Linux kernel 版本5.7-rc1以上的特性应该丰富，诸如支持IORING_FEAT_FAST_POLL
1. 下载和安装liburing的库：https://github.com/axboe/liburing
1. 编译SPDK， 打开如下开关：./configure --with-uring. 如果liburing没有安装在系统指定的目录, 需要自己指定. 这样编译出的SPDK可执行文件, 会优先使用SPDK的uring socket实现, 而不是POSIX. 比如启动SPDK NVMe-oF tcp target, 就会采用SPDK uring 的socket实现.

## 媒体系统
- [Pipewire代替PulseAudio, 并使用WirePlumber管理Pipewire](https://www.oschina.net/news/151623/fedora-may-use-wireplumber).
- [BAT 共同创建新视频流标准 - 超低延迟直播协议信号标准](https://linux.cn/article-14351-1.html)

## FAQ
### 驱动趋势
1. 将设备驱动程序移动到用户空间（而不是内核）
2. 使用轮询（而不是中断）

与之前基于内核相比，这可以实现更高的性能，因为它消除了：
- 代价高昂的系统调用和中断处理
- 数据副本(zero copy)
- 上下文切换

> 英特尔存储性能开发套件（SPDK）和数据平面开发套件（DPDK）就是很好的例子.