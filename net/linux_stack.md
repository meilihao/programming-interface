# linux net stack
- ![linux收包](/misc/img/net/深度截图_选择区域_20191126214406.png)
- ![linux发包](/misc/img/net/深度截图_选择区域_20191126215757.png)


## nic
```
$ ls -al /sys/class/net # ip addr 获取的所有网络接口均在/sys/class/net下(因为部分文件不是接口,比如`bonding_masters`)且是均会链接到`/sys/devices/xxx`下, 通过`/sys/devices/virtual/net`排除虚拟接口即可得到真实网卡
lrwxrwxrwx  1 root 0 1月   3 14:04 enp2s0 -> ../../devices/pci0000:00/0000:00:1c.1/0000:02:00.0/net/enp2s0
lrwxrwxrwx  1 root 0 1月   3 14:04 lo -> ../../devices/virtual/net/lo # 虚拟接口会链接到`/sys/devices/virtual/net`下
```

## devices
### ifb
和tun一样，ifb也是一个虚拟网卡，和tun一样，ifb也是在数据包来自的地方和去往的地方做文章.

ifb驱动模拟一块虚拟网卡，它可以被看作是一个**只有TC过滤功能的虚拟网卡，说它只有过滤功能**，是因为它并不改变数据包的方向，即对于往外发的数据包被重定向到ifb之后，经过ifb的TC过滤之后，依然是通过重定向到之前的网卡再发出去；对于一个网卡接收的数据包，被重定向到ifb之后，经过ifb的TC 过滤之后，依然被重定向到之前的网卡再继续进行接收处理，不管是从一块网卡发送数据包还是从一块网卡接收数据包，重定向到ifb之后，都要经过一个经由ifb 虚拟网卡的dev_queue_xmit操作.

![原理](/misc/img/net/20141101151854140.jpg)