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

## virt
- [virt spec v1.1](http://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.html)/[latest virt spec](https://github.com/oasis-tcs/virtio-spec)

    virtio1.1 关键的最大改动点就是引入了packed virtqueues(新的 virtqueues 内存布局, 旧的布局叫split virtqueues). packed的关键变化是desc ring的变化，为了更好的利用cache和硬件的亲和性（方便硬件实现virtio）, 将split方式中的三个ring（desc，avail，used）打包成一个packed desc ring.
- [^Virtio Spec Overview](https://kernelgo.org/virtio-overview.html)

## next
- [浅谈Linux设备虚拟化技术的演进之路](https://www.modb.pro/db/110904)

## 迁移
- [Nova实现虚拟机块迁移](http://niusmallnan.com/_build/html/_templates/openstack/block_migration.html)