# list

## base
- [itworld123](https://www.zhihu.com/people/zhang-shu-zhu-69)
- [存储系统的快照技术](https://zhuanlan.zhihu.com/p/64595897)
- [快照原理](https://www.haxi.cc/archives/%E5%BF%AB%E7%85%A7%E5%8E%9F%E7%90%86.html)
- [Linux IO 之 文件系统的架构.md](https://github.com/0voice/linux_kernel_wiki/blob/main/%E6%96%87%E7%AB%A0/%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/Linux%20IO%20%E4%B9%8B%20%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F%E7%9A%84%E6%9E%B6%E6%9E%84.md)
- [Linux Storage Stack Diagram](https://www.thomas-krenn.com/en/wiki/Linux_Storage_Stack_Diagram)

## next
- [唬人的“零拷贝”技术，也就那么回事](https://developer.51cto.com/art/202011/633030.htm)

    零拷贝技术的几个实现手段包括：mmap+write、sendfile、sendfile+DMA 收集、splice 等.
- [下一代数据存储技术研究报告(2021年)-210715.pdf](https://pdf.dfcfw.com/pdf/H3_AP202107151503981150_1.pdf)
- [<<阿里云存储产品及应用白皮书>>]
- [Advanced Format](https://wiki.archlinux.org/title/Advanced_Format)

    - [教你把 NVMe SSD 切换到4Kn模式](https://www.zhihu.com/tardis/zm/art/355590811)

## io_uring
- [新一代异步 IO 框架 io_uring ｜ 得物技术](https://my.oschina.net/u/5783135/blog/8657262)
- [Linux 异步 I/O 框架 io_uring：基本原理、程序示例与性能压测](https://arthurchiao.art/blog/intro-to-io-uring-zh/)
- [Alibaba Cloud Linux 2 LTS 率先提供支持 io_uring](https://kernel.taobao.org/2020/06/io_uring-in-Alibaba-Cloud-Linux-2-LTS/)
- [下一代异步 IO io_uring 技术解密](https://kernel.taobao.org/2020/08/Introduction_to_IO_uring/)
- [io_uring，高并发网络编程新利器](https://kernel.taobao.org/2020/09/New_Weapon_for_High_Concurrency_Network_Programming/)
- [面对疾风吧！io_uring 优化 nginx 实战演练](https://kernel.taobao.org/2020/09/IO_uring_Optimization_for_Nginx/)

## 超融合
- [The Nutanix Bible](https://toutiao.io/posts/v28zs0/preview)

## spdk
- [SPDK用户态 iSCSI 客户端库功能介绍](https://www.sdnlab.com/23018.html)
- [阿里云单盘百万IOPS的背后](https://zhuanlan.zhihu.com/p/33593012)
- [基于SPDK的用户态存储引擎：FusionEngine 2.0](/misc/pdf/io/02_Presentation_03_FusionEngine_2.0--Alibaba_User-Space_Full_Stack_Solution_for_Storage_Alibaba_Zhengyong_Yi.pdf)
- [用户态本地存储引擎 — 百万IOPS背后的故事](/misc/pdf/io/f0f8a513fb12402fa52ff9772c3c8f79.pdf)
- [使用 SPDK 技术优化虚拟机本地存储的 IO 性能](https://zhuanlan.zhihu.com/p/52970477)
- [基于SPDK的UDisk全栈优化](https://ci.spdk.io/download/2019-summit-prc/02_Presentation_06_Full_Stack_Optimization_for_Udisk_with_SPDK_UCloud_Yutian.pdf)
- [利用SPDK配合QLogic HBA创建FC target系统](https://qlogicbj.github.io/2020/09/08/qfc-spdk-target-v2/)

    但互联网上找不到QFC 4.5.3a
- [SPDK iSCSI Target加速方案设计](https://blog.csdn.net/junbaozi/article/details/124001718)
- [SPDK NVMe-oF Target](https://blog.csdn.net/junbaozi/article/details/124001718)
- [H3C 全闪（NVMe）存储使用SPDK 加速前后端I/O](https://www.intel.cn/content/dam/www/public/cn/zh/documents/case-studies/h3c-nvme-storage-expedite-front-back-io-cn.pdf)
- [基于SPDK的高效NVMe-oF target](https://www.sdnlab.com/21082.html)

    总体而言iSCSI target更加通用，NVMe-oF target的设计初衷则是考虑性能, 当然在兼容性和通用性方面，NVMe-oF target也在持续进步: NVMe-oF 协议一样可以export 出非PCIe SSD，即SPDK 的NVMe-oF target提供了后端存储的简单抽象, 可以虚拟出相应的NVMe盘(在SPDK 中可以用malloc的bdev或者基于libaio的bdev来模拟出NVMe 盘，把NVMe协议导入到SPDK通用bdev的语义，远端看到的依然是NVMe的盘), 这解决了通用性的问题.

    SPDK NVMe-oF target代码的架构说明.

- [NVMe-oF不只是RDMA，还有TCP](https://www.cnblogs.com/whl320124/articles/11358669.html)

    NVMe-oF协议在某种程度上希望替换掉iSCSI 协议.

    Software RoCE，把网络设备模拟成一个具有RDMA功能的设备.

    NVMe/TCP PDU(Protocol Data Unit)的定义说明.

    **基于SPDK的NVMe-oF TCP 的transport 的实现以及使用介绍(包含"测试 SPDK NVMe-oF target")**.

- [通过 SPDK NVMe/TCP 进行的应用设备队列 (ADQ) 性能测试](https://www.intel.cn/content/www/cn/zh/customer-spotlight/cases/performance-testing-adq-nvme-tcp-spdk.html)

    部分测试配置(没有测试步骤)

## ntb
- [SmartIO: Zero-overhead Device Sharing through PCIe Networking](/misc/pdf/io/ntb_SmartIO.pdf)
- [Flexible device compositions and dynamic resource sharing in PCIe interconnected clusters using Device Lending](/misc/pdf/io/Flexible_device_compositions_and_dynamic_resource_.pdf)
- [存储中的镜像技术](https://blog.csdn.net/linjiasen/article/details/104531417)

    早期也有厂商通过SAS或者FC做镜像通道的，现在很多厂商也会使用网卡或者带RDMA功能的网卡作为存储的镜像通道。使用NTB做为镜像好处和坏处都显而易见。NTB做镜像通道对软硬件的工程能力的挑战远远高于RDMA，但是一旦稳定，成本和性能优势就转换成整个产品的护城河。当然如果整机厂商有自己的RDMA芯片也可以变成本做到很低.
- [【冬瓜哥手绘】从多控缓存管理到集群锁](https://www.chinastor.com/jishu/SAN/0P6164392015.html)
- [功能PCIE交换机之六：基于NTB夸节点的读写](https://developer.aliyun.com/article/506809)
- [科技观察：神威·太湖之光超级计算机](https://solution.zhiding.cn/2016/0623/3079588.shtml)

## 性能
- [【冬瓜哥手绘】它保你上线性能也吊炸天！](https://mp.weixin.qq.com/s?__biz=MzAwNzU3NzQ0MA==&mid=2652088576&idx=1&sn=af2557735037e254b2f1a5b6ad93e541)
- [IO协议栈前沿技术研究动态](https://www.eda365.com/article-109723-1.html)
- 数据布局Append Only

    对付随机写入的最好的办法就是append only模式，将所有写操作顺序记录写入，然后修改指针，这一招对SSD屡试不爽, 实际上SSD内部也是这么做的.

## 可靠性
- 阿里云对于SSD/ESSD云盘，所有写到后台的数据是直接下盘，并非到RAM，正因如此才会加持到9个9的可靠性.

## target
- TCM( Target Core Module )是LINUX LIO 的另一个名称， TCMU（TCM in Userspace） 是 TCM 的 用户态实现

## 用户态
- [DatenLord : 王璞博士开源的云原生分布式存储系统, 用Rust实现了高性能用户态存储，通过绕过Linux内核，避免进程上下文切换以及自行调度IO任务，从而更好地发挥出存储硬件的性能](https://github.com/datenlord/datenlord)
- [打造用户态存储利器，基于SPDK的存储引擎Blobstore & BlobFS](https://www.sdnlab.com/22880.html)
- [基于SPDK的用户态存储引擎：FusionEngine 2.0](https://ci.spdk.io/download/2019-summit-prc/02_Presentation_03_FusionEngine_2.0--Alibaba_User-Space_Full_Stack_Solution_for_Storage_Alibaba_Zhengyong_Yi.pdf)

## 生态
- [浪潮AS13000-H并行存储系统是专门针对高性能计算开发和优化的并行文件存储系统。它基于BeeGFS文件系统商业版本开发，为提高应用的扩展性能和灵活性实现了分布式元数据架构](https://www.inspur.com/lcjtww/2527583/2527584/2527588/2527688/index.html)
- [软件定义存储，看这一篇就够了！](https://www.sohu.com/a/397070625_505795)
- SDS市场占比 from `2019年中国SDS软件定义存储行业市场研究`

    头部厂商: 华为(文件存储得政府,广电和电信行业认可), 曙光, H3C新华三, 浪潮, 联想, IBM以及初创公司(XSKY星辰天合, 杉岩数据, 大道云行等以块存储和对象存储发力).

## cdp
- [linux CDP（连续性数据保护）实现方案](https://blog.csdn.net/weixin_34621309/article/details/116324494)

## 缓存
- [缓存那些事](https://tech.meituan.com/2017/03/17/cache-about.html)

## 分布式
- [一文看懂分布式存储架构](https://stor.51cto.com/art/202005/616378.htm)
- [分布式块存储系统Ursa的设计与实现](https://tech.meituan.com/2016/03/11/block-store.html)
- [深入浅出分布式存储性能优化方案](https://xie.infoq.cn/article/e258215d7bfdf1c89cc319e04)
- [**华为存储 OceanStor SmartMatrix 架构**](https://zhuanlan.zhihu.com/p/81871403)

    多控前后端的全互联共享架构. 4控制器通过PCIe交换机互联.

    四个控制器放在一个控制框, 可通过PCIE交换机连接控制框, 实现Scale Out.

    > SVP（service processor）与KVM（keyboard, video, and mouse）配套使用，是OceanStor 18500 V3/18800 V3存储系统管理、配置、维护等的核专心部件。其上安属装了OceanStor 18500 V3/18800 V3存储系统所需的维护、管理等工具，可以在本地或远程轻松完成全套的管理、配置、鉴等一系列工作. 其本质推测应是一个台刀片服务器.
- [面向核心业务的全闪分布式存储架构设计与实践 - QingStor NeonSAN](https://new.qq.com/omn/20210324/20210324A0B49G00.html)
- [**FASS白皮书|探秘分布式全闪存储原理架构**](https://mp.weixin.qq.com/s/V5bb6fvg5n2DBqVI69sfUw)

    FASS目前支持NVMeoF运行在RDMA transport(IB or RoCE v2)之上, 分为host端和target端，host端采用工具nvme-cli，target端采用SPDK的NVMeoFtarget，完成协议解析后，对接到后端suzaku存储系统.
- [MinIO技术白皮书](https://mp.weixin.qq.com/s?__biz=MzAwMzgyMDk1Mw==&mid=2649276023&idx=1&sn=58ea3118c4cd30868084571047972dce)
- [**突破硬件瓶颈(一)：Intel体系架构的发展与瓶颈挖掘**](https://mp.weixin.qq.com/s?__biz=MzAwMzgyMDk1Mw==&mid=2649276427&idx=2&sn=63d0268972c27dd721517a5c08e1fe27)
- [搜索"华为 OceanStor   Data Sheet/技术白皮书"]

    - [华为OceanStor 6800 V5高端混合闪存存储系统技术白皮书](https://e.huawei.com/cn/material/datacenter/storage/cb4e8571742743498c1012a8190e64c3)
    - [Huawei OceanStor 18500 and 18800 V5 Mission-Critical Hybrid Flash Storage Systems Technical White Paper](https://actfornet.com/ueditor/php/upload/file/20190104/1546532119794412.pdf)
- [【重识云原生】第三章云存储3.3节——Ceph统一存储方案](https://blog.csdn.net/junbaozi/article/details/124003270)
- [gitee.com/explore/distributed-storage](https://gitee.com/explore/distributed-storage)

### 实现
- [openeuler/fastblock](https://gitee.com/openeuler/fastblock)
- [opencurve/curve](https://github.com/opencurve/curve)

    - [CURVE是网易自主设计研发的高性能、高可用、高可靠分布式存储系统](https://opencurve.github.io/)和[了解Curve](https://zhuanlan.zhihu.com/p/338343002)
- [FastCFS 是一款基于块存储的通用分布式文件系统，可以作为MySQL、PostgresSQL、Oracle等数据库和云平台的后端存储](https://www.oschina.net/p/fastcfs)
- [Open vStorage 是一个开源的虚拟机存储路由器](https://blog.csdn.net/sinat_27186785/article/details/52060441)和[openvstorage / volumedriver](https://github.com/openvstorage/volumedriver)
- [WFS 文件存储系统 v1.0.1 发布](https://www.oschina.net/news/281930/wfs-go-1-0-1-released)

## 多活
- [双活数据中心解决方案技术白皮书](https://www-file.huawei.com/~/media/CNBG/Downloads/Product/IT/cn/%E5%8D%8E%E4%B8%BA%E4%B8%9A%E5%8A%A1%E8%BF%9E%E7%BB%AD%E6%80%A7%E5%AE%B9%E7%81%BE%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88%20%E5%8F%8C%E6%B4%BB%E6%95%B0%E6%8D%AE%E4%B8%AD%E5%BF%83%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88%E6%8A%80%E6%9C%AF%E7%99%BD%E7%9A%AE%E4%B9%A6_HyperMetro.pdf)

## bio
- [Linux 通用块层 bio 详解](https://www.byteisland.com/linux-%E9%80%9A%E7%94%A8%E5%9D%97%E5%B1%82-bio-%E8%AF%A6%E8%A7%A3/)
- [Linux block 层 - BIO拆分分析 (V5.4内核)](https://zhuanlan.zhihu.com/p/164884780)

    磁盘单次传输的IO Size也有限制max_hw_sectors_kb （单位KB），block层单次下发的IO大小也有一个限制max_sectors_kb, max_sectors_kb支持动态修改, 但是不能超过max_hw_sectors_kb

## 白皮书
- [华为OceanStor 18000F V5高端全闪存存储系统技术白皮书](https://e.huawei.com/cn/material/datacenter/7e3a286b1f49493f996dff420374395f)

    基于PCIE Switch互联, 4坏3

    - [华为存储 OceanStor SmartMatrix 架构](https://zhuanlan.zhihu.com/p/81871403)
- [OceanStor Dorado 6000, Dorado 18000 系列 6.1.2 产品技术白皮书](https://carrier.huawei.com/~/media/cnbgv2/download/products/it-new/oceanstor-dorado-180006800.pdf)

    基于RDMA互联, 8坏7(OceanStor Dorado V6)

    - [构建新型数据基础设施 华为OceanStor Dorado V6重定义存储架构](https://server.zhiding.cn/server/2019/0814/3120409.shtml)