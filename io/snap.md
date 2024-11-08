# sanp
ref:
- [快照是什么？揭秘存储快照的实现](https://cloud.tencent.com/developer/article/1158686)
- [COW、ROW快照技术原理](https://support.huawei.com/enterprise/zh/doc/EDOC1100196336)
- [存储数据保护技术——HyperSnap快照与HyperCDP高密快照技术讲解](https://blog.csdn.net/m0_49864110/article/details/123989551)
- [快照原理](https://www.tencentcloud.com/zh/document/product/362/31640)
- [快照原理](https://www.haxi.cc/archives/%E5%BF%AB%E7%85%A7%E5%8E%9F%E7%90%86.html)

    Vmware 快照的实现机制就是 RoW.

    RoW最典型的就是虚拟机的快照了，各虚拟机、虚拟化平台所使用的快照技术一般都基于 RoW 来实现的.
- [超融合灾备方案方面有使用快照复制技术么？如何解决COW性能不足的问题？](https://www.smartx.com/blog/2020/06/hci-cow/)

    通常认为 ROW 在连续读的场景影响较大，因为数据块打散了，不连续了，这主要是传统存储利用 HDD 磁盘介质的时候会出现这种情况. 在分布式存储的情况下，ROW 不存在这种问题: 用户在业务层看到连续存储，实际上是分布在不同的服务器的不同硬盘中，数据越是分散，系统性能越高. 同时 ROW 把源数据卷中的原始数据打散之后，对性能反而有好处，而且，分布式块存储的场景下，多数会利用 SSD 进行缓存加速，更好地规避了超融合灾备的性能问题.


