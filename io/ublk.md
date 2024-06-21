# ublk
ref:
- [从内核到用户空间(2) — 初探 ublk](https://blog.jcix.top/2024-02-08/ublk-in    tro/)
  
      ublk 是一个 6.X 内核全新的实现用户态块设备驱动的内核框架 
- [ublk：来自Linux社区的新热点，基于io_uring的全新高性能用户态块设备](ht    tps://developer.aliyun.com/article/989552)
- [ublk: the new and improved way to serve SPDK storage locally!](https:    //spdk.io/news/2023/03/28/ublk/)
    nbd存在巨大的可扩展性和性能问题, 因此SPDK nbd 支持主要用于非性能用例
- [UBLK](https://people.redhat.com/minlei/docs/ublk_intro.pdf)
- [ublk-qcow2: ublk-qcow2 is available](https://lore.kernel.org/lkml/b15    c2ff8-ae30-8e8e-4178-c9976346afde@manuel-bentele.de/t/)
- [基于SPDK的Ublk和Vduse的用户空间块服务](https://blog.csdn.net/weixin_37097605/article/details/137362344)

    ublk vs uduse

**blk server is on localhost.**

## example
ref:
- [Testing ublk on Ubuntu 22.04](https://dev.to/amarjargal/testing-ublk-on-ubuntu-2204-9pe)
- [nbdublk - connect network block device to a local device](https://libguestfs.org/nbdublk.1.html)
- [[Libguestfs] [PATCH libnbd] ublk: Add new nbdublk program
](https://listman.redhat.com/archives/libguestfs/2022-August/029655.html)

```bash
$ modprobe ublk_drv
...
```

待补充example: ubuntu 24.04 repos找不到nbdublk; federa最新版里有.
