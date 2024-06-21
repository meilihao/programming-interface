# dev
ref:
- [SPDK官方技术文章](https://spdk.io/cn/articles/)
- [SPDK 应用编程框架](https://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653334735&idx=1&sn=b81c263cffc74cf42338d2edda371d2c)
- [使用SPDK lib搭建自己的NVMe-oF Target应用](https://blog.csdn.net/weixin_37097605/article/details/108114450)
- [搭建远端存储，深度解读SPDK NVMe-oF target](https://mp.weixin.qq.com/s/ohPaxAwmhGtuQQWz--J6WA)
- [SPDK NVMe Reservation使用简介](https://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653335852&idx=1&sn=5e08566473a1e2f14b9d1f697c4995cc)
- [SPDK块设备bdev简介](https://www.cnblogs.com/whl320124/articles/10064186.html)
- [Writing a Custom Block Device Module](https://spdk.io/doc/bdev_module.html)
- [spdk](https://rootw.github.io/tags/#SPDK-ref)
- [SPDK示例代码分析](https://blog.csdn.net/zlarm/article/details/79151138)
- [SPDK/NVMe存储技术分析1-15](https://www.cnblogs.com/vlhn/tag/NVMe/)
- [SPDK用户态 iSCSI 客户端库功能介绍](https://www.sdnlab.com/23018.html)
- [CentOS7下编译安装SPDK iSCSI Target](https://blog.csdn.net/bobpen/article/details/62218407)

> SPDK和DPDK的关系: SPDK 提供了一套环境抽象化库 (位于lib/env目录)，主要用于管理SPDK存储应用所使用的CPU资源、内存和PCIe等设备资源，其中DPDK是SPDK缺省的环境库.

> 使用SPDK application framework的应用程序, 都需要用CPU的亲和性（CPU Affinity）绑定某几个CPU, 对于SPDK的线程和应用线程之间的竞争可使用[SPDK RBD bdev的解决方案即spdk_call_unaffinitized](https://www.sdnlab.com/25330.html).

SPDK提供了一套编程框架 (SPDK Application Framework)，用于指导软件开发人员基于SPDK的用户态NVMe驱动以及用户态块设备层 (User space Bdev) 构造高效的存储应用. 用户有两种选择:
1. 直接使用SPDK应用编程框架实现应用的逻辑
2. 使用SPDK编程框架的思想，改造应用的编程逻辑，以更好的适配SPDK的用户态NVMe驱动

总体而言，SPDK的应用框架可以分为以下几部分:
1. 对CPU core和线程的管理
1. 线程间的高效通信
1. I/O的的处理模型以及数据路径(data path)的无锁化机制

## example
- [全闪分布式存储之PureFlash](https://cloud.tencent.com/developer/article/2363606)
- [ceph-nvmeof](https://github.com/ceph/ceph-nvmeof)
- [openebs/mayastor](https://github.com/openebs/mayastor)

## FAQ
### spdk repo版本
- [/include/spdk/version.h](https://github.com/spdk/spdk/blob/master/include/spdk/version.h)
