# list

## io_uring
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

## 性能
- [【冬瓜哥手绘】它保你上线性能也吊炸天！](https://mp.weixin.qq.com/s?__biz=MzAwNzU3NzQ0MA==&mid=2652088576&idx=1&sn=af2557735037e254b2f1a5b6ad93e541)
- [IO协议栈前沿技术研究动态](https://www.eda365.com/article-109723-1.html)
- 数据布局Append Only

    对付随机写入的最好的办法就是append only模式，将所有写操作顺序记录写入，然后修改指针，这一招对SSD屡试不爽, 实际上SSD内部也是这么做的.

## 可靠性
- 阿里云对于SSD/ESSD云盘，所有写到后台的数据是直接下盘，并非到RAM，正因如此才会加持到9个9的可靠性.

## target
- TCM( Target Core Module )是LINUX LIO 的另一个名称， TCMU（TCM in Userspace） 是 TCM 的 用户态实现