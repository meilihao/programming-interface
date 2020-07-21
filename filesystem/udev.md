# udev
> hotplug(热插拔)狠推了一把udev.

配置文件: /etc/udev/udev.conf
默认的udev规则在/etc/udev/rules.d下. udev规则文件的文件名通常以两个数字开头, 表明系统应用该规则的顺序.

udev规则文件的一行就是一条规则. 规则以kv形式表示, 由多个kv组成并以`,`分隔.