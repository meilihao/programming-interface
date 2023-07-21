# list

## base
- [虚拟化调试和优化指南](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html/virtualization_tuning_and_optimization_guide)

## driver
- [qemu backend block device driver in practice by qingcloud](/misc/pdf/virt/qingcloud_block_device_driver.pdf)

## image
- [vm镜像常见格式](https://support.huaweicloud.com/productdesc-ims/zh-cn_topic_0089615820.html)
- [KVM虚拟机导出最小化体积的qcow2镜像文件](https://www.moonfly.net/archives/50.html)

## disk
- [virtio-blk vs virtio-scsi](https://mpolednik.github.io/2017/01/23/virtio-blk-vs-virtio-scsi/)

    aliyun仍在使用virtio-blk(2020.12). 个人推测: 虽然virtio-scsi功能更丰富, 但因为架构原因, 性能比virtio-blk略差.

    virtio-blk:

    guest: app -> Block Layer -> virtio-blk
    host: QEMU -> Block Layer -> Block Device Driver -> Hardware

    virtio-scsi (using QEMU as SCSI target):

    guest: app -> Block Layer -> SCSI Layer -> scsi_mod
    host: QEMU -> Block Layer -> SCSI Layer -> Block Device Driver -> Hardware

    virtio-SCSI的优势在于**它能够处理数百个设备**，而virtio-blk只能处理大约30个设备并耗尽PCI插槽.

## vm
- [oVirt 是个管理虚拟化的应用程序, 是Red Hat 虚拟化产品的上游, 且据说华为的fusionsphere6.x版本的虚拟化中的vrm也是基于它](https://wiki.centos.org/zh/HowTos/oVirt)
- [云集技术学社 | 一文带您了解深信服aSV服务器虚拟化功能及原理](https://www.51cto.com/article/686566.html)和[深信服超融合架构技术白皮书](https://jsits-image.oss-cn-beijing.aliyuncs.com/sangfor-hci.pdf)

    KVM的vDISK有两种格式：RAW 和 QCOW2格式。RAW格式性能更高些，但相比QCOW2，RAW不支持快照、精简分配等特性，故而深信服采用的是QCOW2格式.

    对于QCOW2文件，有三种模式：精简分配、动态分配（需要底层存储支持空洞文件）、预分配模式。其中“预分配”性能最好，接近于RAW格式的性能，“精简分配”性能最差，“动态分配”居中（注意：目前超融合中动态分配已接近于预分配性能、aSAN有优化）.

    无论vCPU数量配置多大，总的物理主机CPU资源是恒定的，因而：
    1. 单个虚拟机最大的配置不要超过物理CPU的核心数量
    1. 主机上运行所有虚拟机的总vCPU数量不能太多，否则调度消耗会增大. **生产环境最佳实践为不超过CPU的逻辑核心的2倍**，主要参考真实生产中物理CPU占用一般不超过20%；超配2倍以后，物理CPU占用40%左右，超配要考虑峰值预留，且物理CPU占用超过50%以上，已经比较繁忙了.

    热迁移分为两种形式，一种是共享存储热迁移，此种热迁移形式，需要虚拟机镜像在共享存储上，此种迁移类型，只需要通过网络发送客户机的vCPU执行状态、内存中的内容、虚机设备的状态到目的主机上。另一种是跨主机跨存储热迁移，与跨主机不跨存储热迁移类似。不同的是其需要在目的存储创建相同配置的虚拟机镜像（空白的，没有数据），之后仍然是在目的宿主机上启动目的端Qemu进程， 目的端Qemu镜像打开新创建的镜像文件。另外还需要传送源端虚拟机的磁盘数据到目的端。

## virt
- [virt spec v1.1](http://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.html)/[latest virt spec](https://github.com/oasis-tcs/virtio-spec)

    virtio1.1 关键的最大改动点就是引入了packed virtqueues(新的 virtqueues 内存布局, 旧的布局叫split virtqueues). packed的关键变化是desc ring的变化，为了更好的利用cache和硬件的亲和性（方便硬件实现virtio）, 将split方式中的三个ring（desc，avail，used）打包成一个packed desc ring.
- [^Virtio Spec Overview](https://kernelgo.org/virtio-overview.html)

## next
- [浅谈Linux设备虚拟化技术的演进之路](https://www.modb.pro/db/110904)

## 迁移
- [Nova实现虚拟机块迁移](http://niusmallnan.com/_build/html/_templates/openstack/block_migration.html)