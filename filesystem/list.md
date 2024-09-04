# list
## base
- [Linux文件系统十问](https://cloud.tencent.com/developer/article/1963183)

## zfs
- [ZFS源代码剖析](https://docs.google.com/document/d/1mEp7qxAPCpjqjXQDaYtOQNMk8d5DSbob1DxlK9SIbkY/edit?pli=1)
- [ZFS源代码之旅——ZAP模块分析（一）](https://blog.csdn.net/nofrish/article/details/7470972)
- [zfs源码分析及动态](http://www.360doc.com/content/16/0106/09/10671613_525828017.shtml)

## 盘古
- [盘古（Pangu）能够为用户提供大规模、高可靠、高可用、高吞吐量和可扩展的存储服务](https://help.aliyun.com/apsara/agile-data/v_2_5_0_20200506/apsara_stack_platform/insight-cloud-platform-o-m-guide/pangu-platform-o-m-0001.html)
- [阿里云单盘百万IOPS的背后](https://zhuanlan.zhihu.com/p/33593012)

## dfs
- [分布式文件系统架构对比](https://mp.weixin.qq.com/s/ZwfRZW42e_2dh0u6vH0psg)
- [PolarFS: An Ultra-low Latency and Failure Resilient Distributed File System for Shared Storage Cloud Database](http://www.vldb.org/pvldb/vol11/p1849-cao.pdf)
- [PolarFS论文部分翻译](https://nan01ab.github.io/2018/09/PolarFS.html)
- [PolarFS论文完整翻译](https://blog.csdn.net/qq_38289815/article/details/108944408)

## fuse
- [ceph存储 FUSE原理总结](https://blog.csdn.net/skdkjzz/article/details/42299751)

## net fs
- [Linux网络文件系统的实现与调试](https://www.cnblogs.com/wahaha02/p/9559345.html)
- [NFS文件系统协议及解析概要](https://zhuanlan.zhihu.com/p/58095846)
- [NFS文件锁一致性设计原理解析](https://developer.aliyun.com/article/771329)
- [Datenlord | 重新思考Rust Async - 如何实现高性能IO](https://juejin.cn/post/7036993784210685965)

## fs
- [第4章 ext文件系统机制原理剖析](https://www.cnblogs.com/f-ck-need-u/p/7016077.html)
- [大话EXT4文件系统完整版](http://www.ssdfans.com/?p=8136)
- [Stratis 是基于XFS 和LVM 的组合构建的卷管理器](https://stratis-storage.github.io/)
- [NVFS面向基于DAX的设备（直接访问）的fs]()
- [NOVA: a new file system for persistent memory](https://lwn.net/Articles/749009/)

## FAQ
### ext4格式化慢
ext4格式化的时候是真的慢，它在格式化的时候就将inode和block就已经分好了，而xfs与他的区别在于xfs格式化的时候不分配，而当数据来了的时候再分发inode和block.

### ext4 inode耗尽
XFS 动态分配 inode. 只要文件系统上有可用空间，XFS 文件系统就不会耗尽 inode.