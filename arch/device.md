# device

## /sys
`/sys`是一种在内存中的虚拟文件系统sysfs的mount point, 提供了有关系统上可用设备, 设备的配置及其状态的信息. sysfs的指定原则之一是`/sys`里的每个文件都只表示下层设备的一个属性.

可通过`udevadm`查询设备信息, 触发事件, 控制vdevd守护进程, 以及监视udev和内核的事件, 比如查看ssd信息`udevadm info -a -n nvme0n1`.

`/sys`目录:
- block : 有关硬盘之类的块设备信息
- bus : 总线: pci-e, scsi, usb等
- class : 按设备的功能类型(比如声卡, 显卡, 输入设备, 网卡)组织的一颗树
- dev : 区分块设备和字符设备的设备信息
- devices : 正确表示所有找到的设备
- firmware : 特定于平台的子系统的接口, 如ACPI
- fs : 内核知道的一些但不是全部文件系统的目录
- hypervisor
- kernel : 内核的内部信息, 比如高速缓存和虚拟内存状态
- module : 内核动态加载的模块
- power : 系统电源状态的几种详细信息

## udev
在用户空间实现的一种设备管理系统, 它会将设备文件自动放在/dev下. 它依赖sysfs来了解系统的设备变化, 并使用一系列udev特有的规则来指定其对设备的管理以及命令, 默认规则在`/lib/udev/rules.d`下, 自定义规则在`/etc/udev/rules.d`下.

## `/dev/loopN`
一种伪设备，使得文件可以如同块设备一般被访问.

可通过`losetup -a`查看.

snapd的`loopN`可通过`sudo apt autoremove --purge snapd`解决.