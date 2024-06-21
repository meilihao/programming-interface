# nbd
ref:
- [tools source code](https://github.com/NetworkBlockDevice/nbd)

  - `nbd/maketr`: example for how to use tools
  - [周边生态](https://www.cnblogs.com/suv789/p/17545477.html)
  - [nbdkit go examples](https://gitlab.com/nbdkit/libnbd/tree/master/golang/examples)
- [jsnbd](https://blog.csdn.net/jiangwei0512/article/details/134388491), [web jsnbd](https://blog.csdn.net/jiangwei0512/article/details/119770137)

  通过jsnbd, Web浏览器成为了服务器端, 存放着文件, 而本地系统通过/dev/nbdx来访问Web浏览器文件
- [BDEV: 基于Append-Only存储实现的块设备](https://gitee.com/mphyatyh/bdev/)

    使用nbd-client无法连接10809, 必须使用[go-nbd/cmd/go-nbd-example-client](github.com:pojntfx/go-nbd.git)

## example
```bash
$ sudo modprobe nbd [max_part=8] # 默认创建nbd设备是16个
$ lsmod |grep nbd
nbd                    65536  0
$ sudo apt install nbd-server nbd-client
$ sudo nbd-server 9999 /usr/lib/memtest86+/memtest86+x64.iso

** (process:136881): WARNING **: 16:22:05.988: Specifying an export on the command line no longer uses the oldstyle protocol.
$ ss -anltp |grep 9999
LISTEN 0      10                 *:9999             *:*
$ sudo kill 136882 # pid from `ps -ef|grep nbd`, 也可通过nbd-server -p指定pid file来获取
```

other terminal:
```bash
$ ls /dev/nbd*
/dev/nbd0   /dev/nbd11  /dev/nbd14  /dev/nbd3  /dev/nbd6  /dev/nbd9
/dev/nbd1   /dev/nbd12  /dev/nbd15  /dev/nbd4  /dev/nbd7
/dev/nbd10  /dev/nbd13  /dev/nbd2   /dev/nbd5  /dev/nbd8
$ sudo nbd-client localhost 9999 /dev/nbd0
$ sudo mkdir /mnt/test
$ sudo mount /dev/nbd0 /mnt/test
$ ls test
boot  boot.catalog  EFI
$ ll /sys/devices/virtual/block/nbd0/pid # 存在pid文件即该设备已使用, pid是nbd-client命令fork出的子进程的pid. 类似也可判断/sys/devices/virtual/block/nbd0/size是否为0
$ sudo umount /mnt/test # 解除占用
$ sudo nbd-client -d /dev/nbd0 # 之前需要解除nbd0的占用, 否则执行没效果
```

## io路径
ref:
- [rbd-nbd的设计实现及应用](https://itdks.su.bcebos.com/0a7c50c9e6424363b02b18d02cd451c5.pdf)
- [The NBD protocol](https://github.com/NetworkBlockDevice/nbd/blob/master/doc/proto.md)
- [【块存储block源码分析】 linux内核模块ceph nbd源码分析](https://blog.csdn.net/zhonglinzhang/article/details/103815508)

`/dev/nbd<X>`<->nbd-client<->nbd-server

> nbd-server, nbd-client均在用户态

各组件作用:
- nbd kernel module
  - 导出块设备文件/dev/nbdx
- nbd client
  - 创建一个socket与nbd server建立连接
  - 对/dev/nbdx调用NBD_SET_SOCK ioctl接口将该socket信息传入kernel
  - 调用NBD_DO_IT ioctl接口，进入nbd kernel module的代码, 创建一个kernel thread，然后在内核监听socket, 收取处理nbd server响应

  参数:
  - swap: 将device作为swap, 见[访问远程存储作为本地交换内存区域](https://bbs.huaweicloud.com/blogs/313524)

- nbd kernel thread
  - 接收块设备访问请求，然后通过socket将请求发送给server
- nbd server
  - 将数据访问请求转化为对服务器本地磁盘或文件的访问
  - 将响应通过socket发送给nbd client

  > handle_modern_connection: 处理连接
  > nbd server实现可参考[curve-nbd](https://github.com/opencurve/curve-nbd/blob/master/src/NBDServer.cpp)

## impl
nbd驱动的初始化在[nbd_init](https://elixir.bootlin.com/linux/v6.6.23/source/drivers/block/nbd.c#L2527), 添加设备在[nbd_dev_add](https://elixir.bootlin.com/linux/v6.6.23/source/drivers/block/nbd.c#L1787).

nbd_dev_add使用了add_disk()将nbd设备加入系统.

linux通过块设备文件系统来管理块设备. 块设备文件系统的入口是bdev_cache_init, 它把块设备文件系统注册到内核.

bdev_sops是块设备文件系统超级块的操作函数.

打开块设备时实际使用的是块设备文件系统提供的blkdev_open by def_blk_fops.
