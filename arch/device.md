# device

## udev
在用户空间实现的一种设备管理系统, 它会将设备文件自动放在/dev下. 它依赖sysfs来了解系统的设备变化, 并使用一系列udev特有的规则来指定其对设备的管理以及命令, 默认规则在`/lib/udev/rules.d`下, 自定义规则在`/etc/udev/rules.d`下.

## `/dev/loopN`
一种伪设备，使得文件可以如同块设备一般被访问.

可通过`losetup -a`查看.

snapd的`loopN`可通过`sudo apt autoremove --purge snapd`解决.
