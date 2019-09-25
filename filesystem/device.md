# 设备
## `/dev/loopN`
一种伪设备，使得文件可以如同块设备一般被访问.

可通过`losetup -a`查看.

snapd的`loopN`可通过`sudo apt autoremove --purge snapd`解决.