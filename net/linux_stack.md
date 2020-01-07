# linux net stack
- ![linux收包](/misc/img/net/深度截图_选择区域_20191126214406.png)
- ![linux发包](/misc/img/net/深度截图_选择区域_20191126215757.png)


## nic
```
$ ls -al /sys/class/net # ip addr 获取的所有网络接口均在/sys/class/net下(因为部分文件不是接口,比如`bonding_masters`)且是均会链接到`/sys/devices/xxx`下, 通过`/sys/devices/virtual/net`排除虚拟接口即可得到真实网卡
lrwxrwxrwx  1 root 0 1月   3 14:04 enp2s0 -> ../../devices/pci0000:00/0000:00:1c.1/0000:02:00.0/net/enp2s0
lrwxrwxrwx  1 root 0 1月   3 14:04 lo -> ../../devices/virtual/net/lo # 虚拟接口会链接到`/sys/devices/virtual/net`下
```