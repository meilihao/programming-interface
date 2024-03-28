# device mapper
device mapper和md一样, 也是一个虚拟块设备层, 属于块I/O子系统的块设备驱动层, 代码在`include/linux`和`drivers/md`. 它主要是支持lvm管理. 管理工具是dmsetup.

device mapper将虚拟块设备的逻辑范围分段进行映射, 将一段范围按照特定的规则映射到"低层设备".