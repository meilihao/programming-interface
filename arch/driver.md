# driver
设备驱动程序是管理系统与某种特定硬件之间交互的程序, 负责内核接口与硬件命令的翻译. 其处于性能考虑通常运行在内核空间.

对设备的用户级访问往往是通过/dev下的设备文件, 内核会把对这些文件的操作映射到对应的驱动程序的调用上.

## 用户态驱动(ULD模式驱动, User Level Driver)
将底层硬件的操作寄存器地址空间直接映射到用户态地址空间. 在用户态实现设备驱动, 协议栈, 文件系统等功能.

优势: 不需要切换到内核态, 使用polling模式, 减少中断请求, 有性能优势.
劣势: 用户态程序需要搞定一切, 无法利用内核提供的各种功能比如文件系统, 库等.

## Polling模式启动(Polling Mode Driver)
使用polling大幅度减少中断. 常见于NIC驱动.

## nvme
参考:
- [A Quick Tour of NVM Express (NVMe)](https://metebalci.com/blog/a-quick-tour-of-nvm-express-nvme/)
- [Linux中nvme驱动详解](https://developer.aliyun.com/article/596648)

NVMe相关的代码位于内核源码树的drivers/nvme中，主要有两个文件夹一个是host,一个targets.

其中targets是用于实现将本系统中的nvme设备作为磁盘导出，供其他服务器或者系统使用的功能. 而文件夹host是实现NVMe磁盘供本系统自己使用，不会对外提供，这意味着外部系统不能通过网络或者光纤来访问NVMe磁盘.

如果配置NVMe target还需要工具nvmetcli工具.

kernel mod:
- nvme
- nvme_core

NVMe命名标准描述：
- nvme0：第一个注册的设备的设备控制器
- nvme0n1：第一个注册设备的第一个名称空间, 类似与scsi lun
- nvme0n1p1：第一个注册设备的第一个名称空间的第一个分区

### NVME中断命名
nvme的队列名称是根据核数来编号的，admin的队列和第一个io队列共享同一个中断（下图示），所以它俩的中断数会相对比其他IO队列多，队列默认就是跟随cpu号而绑定的. 查看/proc/interrupt，中断名称是nvme0q0，当然类似的nvme1q0也是，以此类推，nvme0q0和nvme1q0分别是设备0与设备1的admin队列.

IO队列是nvme0q1,...,nvme0qx，其中x就是cpu的核数。nvme0q1这个队列，默认绑定在cpu0上；nvme0q30这个队列，默认绑定在cpu29上，以此类推.

## 用户态驱动
参考:
- [用户态驱动概述](https://huqianshan.github.io/Linux/Drivers/%E7%94%A8%E6%88%B7%E6%80%81%E9%A9%B1%E5%8A%A8%E6%A6%82%E8%BF%B0/)
- [Linux用户态驱动设计](https://www.cnblogs.com/wahaha02/p/8569074.html)
- `User-Space Device Drivers in Linux: A First Look`

UIO框架适用于简单设备的驱动，因为它不支持DMA，不能支持多个中断线，缺乏逻辑设备抽象能力.

VFIO(**推荐**)是一个可以安全地把设备I/O、中断、DMA等暴露到用户空间（userspace），从而可以在用户空间完成设备驱动的框架。用户空间直接设备访问，虚拟机设备分配可以获得更高的IO性能。依赖于IOMMU, vfio-pci, 相比于UIO，VFIO更为强健和安全.