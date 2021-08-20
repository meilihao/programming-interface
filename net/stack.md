# 内核协议栈

![内核协议栈](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6nj6QFbU1dhIQWqgwbG7hdO6LiaDTLXrYBBrLKicTMBU9Gc9ibNo3mXMEdwIia1XaYFuQFP83om89s9zg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

随着互联网流量越来愈大, 网卡性能越来越强, Linux内核协议栈在10Mbps/100Mbps网卡的慢速时代是没有任何问题的, 那个时候应用程序大部分时间在等网卡送上来数据. 现在到了1000Mbps/10Gbps/40Gbps网卡的时代, 数据被很快地收入, 协议栈复杂处理逻辑, 效率捉襟见肘, 把大量报文堵在内核里.

问题有:
- 各类链表在多CPU环境下的同步开销
- 不可睡眠的软中断路径过长
- sk_buff的分配和释放
- 内存拷贝的开销
- 上下文切换造成的cache miss
- …

于是, 内核协议栈各种优化措施应着需求而来：
- 网卡RSS, 多队列
- 中断线程化
- 分割锁粒度
- Busypoll

但却都是见招拆招, 治标不治本. 问题的根源不是这些机制需要优化, 而是这些机制需要推倒重构. 重构的思路很显然有两个：
- upload方法：别让应用程序等内核了, 让应用程序自己去网卡直接拉数据
- offload方法：别让内核处理网络逻辑了, 让网卡自己处理

总之, 绕过内核就对了, 内核协议栈背负太多历史包袱.

DPDK让用户态程序直接处理网络流, bypass掉内核, 使用独立的CPU专门干这个事.
XDP让灌入网卡的eBPF程序直接处理网络流, bypass掉内核, 使用网卡NPU专门干这个事.

如此一来, 内核协议栈就不再参与数据平面的事了, 留下来专门处理诸如路由协议, 远程登录等控制平面和管理平面的数据流.

改善iptables/netfilter的规模瓶颈, 提高Linux内核协议栈IO性能, 内核需要提供新解决方案, 那就是eBPF/XDP框架.