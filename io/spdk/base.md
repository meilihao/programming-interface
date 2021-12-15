# spdk
越来越多公有云服务提供商采用SPDK技术作为其高性能云存储的核心技术之一.

SPDK(全称Storage Performance Development Kit)，提供了一整套工具和库，以实现高性能、扩展性强、全用户态的存储应用程序. 它是继DPDK之后，intel在存储领域推出的又一项颠覆性技术，旨在大幅缩减存储IO栈的软件开销，从而提升存储性能，可以说它就是为了存储性能而生.

SPDK能实现高性能，得益于以下三个关键技术：
- 全用户态，它把所有必要的驱动全部移到了用户态，避免了系统调用的开销并真正实现内存零拷贝
- 轮循模式，针对高速物理存储设备，采用轮循的方式而非中断通知方式判断请求完成，大大降低时延并减少性能波动
- 无锁机制，在IO路径上避免采用任何锁机制进行同步，降低时延并提升吞吐量

SPDK整体分为三层：

- 存储协议层(Storage Protocols)，指SPDK支持存储应用类型。iSCSI Target对外提供iSCSI服务，用户可以将运行SPDK服务的主机当前标准的iSCSI存储设备来使用；vhost-scsi或vhost-blk对qemu提供后端存储服务，qemu可以基于SPDK提供的后端存储为虚拟机挂载virtio-scsi或virtio-blk磁盘；NVMF对外提供基于NVMe协议的存储服务端。注意，图中vhost-blk在spdk-18.04版本中已实现，后面我们主要基于此版本进行代码分析。
- 存储服务层(Storage Services)，该层实现了对块和文件的抽象。目前来说，SPDK主要在块层实现了QoS特性，这一层整体上还是非常薄的。
- 驱动层(drivers)，这一层实现了存储服务层定义的抽象接口，以对接不同的存储类型，如NVMe，RBD，virtio，aio等等。图中把驱动细分成两层，和块设备强相关的放到了存储服务层，而把和硬件强相关部分放到了驱动层。

## ref
- [spdk](https://rootw.github.io/tags/#SPDK-ref)
- [SPDK示例代码分析](https://blog.csdn.net/zlarm/article/details/79151138)
- [SPDK/NVMe存储技术分析1-15](https://www.cnblogs.com/vlhn/tag/NVMe/)
- [SPDK用户态 iSCSI 客户端库功能介绍](https://www.sdnlab.com/23018.html)
- [CentOS7下编译安装SPDK iSCSI Target](https://blog.csdn.net/bobpen/article/details/62218407)

## 编译
```bash
# --- SPDK使用了DPDK中一些通用的功能和机制，因此首先需要下载DPDK的源码并完成编译和安装
# make config T=x86_64-native-linuxapp-gcc
# make
# make install # (默认安装到/usr/local，包括.a库文件和头文件)

# --- SPDK的编译, 进入spdk/script目录，执行pkgdep.sh脚本，自动下载安装相关依赖文件
# ./configure –with-dpdk=/usr/local
# make
```