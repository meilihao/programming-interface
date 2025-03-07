# receive
ref:
- [coder-kung-fu](https://github.com/yanfeizhang/coder-kung-fu)
- [**Linux 网络栈接收数据（RX）：原理及内核实现（2022）**](https://arthurchiao.art/blog/linux-net-stack-implementation-rx-zh/)
- [**十年码农内功：收包（一）**](https://mp.weixin.qq.com/s?__biz=MzkzMzM5MTUwNQ==&mid=2247484076&idx=1&sn=2a43acaed3e3596bf338784ee5b09c9c&scene=21#wechat_redirect)
- [**Linux 路由实现原理**](https://blog.csdn.net/CoderTnT/article/details/124856179)

kernel: 6.8.0/ubuntu 24.04.1

![receive.png](/misc/img/net/receive.png)

数据包接收过程:
1. 当网卡收到数据以后,以DMA的方式把网卡收到的帧写到内存里
1. 向CPU发起一个中断,以通知CPU有数据到达
1. 当CPU收到中断请求后,会去调用网络设备驱动注册的中断处理函数
1. 网卡的中断处理函数并不做过多工作, 发出软中断请求, 然后尽快释放CPU资源
1. ksoftirad内核线程检测到有软中断请求到达,调用poll开始轮询收包,收到后交由各级协议栈处理.

    对于TCP包来说,会被放到用户socker的接收队列中.

## 1. ksoftirqd
ref:
- [中断和中断处理(九)](https://www.cntofu.com/book/114/Interrupts/interrupts-9.md)

Linux的软中断都是在专⻔的内核线程(ksoftirqd)中进行的, 每个cpu逻辑core一个ksoftirqd.

ksoftirqd创建过程:
1. [early_initcall(spawn_ksoftirqd)](https://elixir.bootlin.com/linux/v6.8/source/kernel/softirq.c)

    `smpboot_register_percpu_thread(&softirq_threads)`

当ksoftirad被创建出来以后,它就会进入自己的线程循环函数ksoftirad_should_run和run_ksoftirad了, 接下来判断有没有软中断需要处理.

软中断在 Linux 内核编译时就静态地确定了, 见[include/linux/interrupt.h#](https://elixir.bootlin.com/linux/v6.8/source/include/linux/interrupt.h#L548).

## 2. 网络子系统
在网络子系统的初始化过程中,会为每个CPU初始化softnet_data(简写sd),也会为RXSOFTIRQ和TX SOFTIRQ注册处理函数.

网络子系统初始化过程:
1. [subsys_initcall(net_dev_init)](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L11750)

    > Linux内核通过调用subsys_initcall来初始化各个子系统

    1. `struct softnet_data *sd = &per_cpu(softnet_data, i);`: 为每个cpu初始化[softnet_data](https://elixir.bootlin.com/linux/v6.8/source/include/linux/netdevice.h#L3273), 其poll_list用于等待驱动程序将其 pol函数注册进来

        每个 CPU 都有一个 backlog queue，其加入到 NAPI 变量的方式和驱动差不多，都是注册一个 poll() 方法，在软中断的上下文中处理包.

        > backlog: 为了统一两种处理方式，为所有非napi设备设计了一个napi_struct结构，与每个napi设备都有一个napi_struct结构共同在软中断的处理框架中生效

    1. 注册net_tx_action, net_rx_action, 即将软中断和处理函数的映射保存到softirq_vec

      ```c
      open_softirq(NET_TX_SOFTIRQ, net_tx_action);
	  open_softirq(NET_RX_SOFTIRQ, net_rx_action);
      ```
      
      > open_softirg用于注册软中断的处理函数, 即将每一种软中断与一个处理函数关联.
      > ksottirad线程收到软中断的时候,也会使用softirq_vec来找到每一种软中断对应的处理函数.

## 3. 协议栈注册
过程:
1. [`fs_initcall(inet_init);`](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L2078)

    - `inet_add_protocol(&udp_protocol, IPPROTO_UDP)`: 注册udp_rcv()
    - `inet_add_protocol(&tcp_protocol, IPPROTO_TCP)`: 注册tcp_v4_rcv()
    - `dev_add_pack(&ip_packet_type);`: 注册ip_rcv()

    inet_add_protocol将tcp,udp相应的函数注册到inet_protos数组
    dev_add_pack()将ip_rcv() 函数的处理地址保存在ptype_base哈希表里

    软中断中会通过ptype_base找到ip_rev()的地址,进而将IP包正确地送到ip_rcv()中执行. 在ip_rcv()中将会通过inet_protos找到TCP或者UDP的处理函数, 再把包转发给udp_rev()或tcp_v4_rcv().

    > ip_ rev中会处理iptable netfilter过滤. udp_rcv中会判断socket接收队列是否满了, 对应的相关内核参数是net.core.rmern_max 和net.core.mem_default.

## 4. 网卡驱动初始化
ref:
- [Linux网络协议栈：NAPI机制与处理流程分析（图解）](https://blog.csdn.net/Rong_Toa/article/details/109401935)

驱动使用module_init向kernel注册一个初始化函数, 当驱动被加载时, 内核会调用该函数.

igb网卡驱动注册在[module_init(igb_init_module);](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L670).

驱动的`pci_register_driver(&igb_driver);`调用完成后, 内核就知道了该驱动的相关信息. 比如igb网卡驱动的igb_driver_namne和igb_probe函数地址等等. 当网卡设备被识别以后,内核会调用其驱动的probe方法(igb_driver的probe方法是[igb_probe](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L3201)). 驱动的probe方法执行的目的就是让设备处于ready状态.

igb_probe主要意图:
1. DMA初始化
1. 注册ethtool实现函数

    网卡驱动实现了ethtool所需要的接口. 当ethtool发起一个系统调用之后,内核会找到对应操作的回调函数. 对于igb网卡来说, 其实现在`drivers/net/etheret/intel/igb/igb_ ethtool.c`
1. 注册net_device_opt, netdev等变量

    注册net_device_opt用的是igb_netdev_ops变量,其中包含igb_open()等函数, 该函数在网卡启动的时候会被调用.
1. NAPI初始化, 注册poll函数

    igb_alloc_q_vector注册了一个NAPI机制必需的poll函数 by [netif_napi_add(adapter->netdev, &q_vector->napi, igb_poll);](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L1218), 对于igb网卡驱动来说, 这个函数就是igb_poll.

    NAPI设备均对应一个napi_struct结构. 如果驱动愿意，可以创建多个 napi_struct，因为现在越来越多的硬件已经开始支持多接收队列 (multiple receive queues)，这样，多个 napi_struct 的实现使得多队列的使用也更加的有效.

    > 网速越来越快，2.6之前的中断收包模式已经无法适应千兆，万兆的带宽了, 因为CPU会一直陷入硬中断而没有时间来处理别的事情. NAPI机制就是混合中断和轮询的方式来收包，核心概念就是不采用中断的方式读取数据，而代之以首先采用中断唤醒数据接收的服务程序，然后 POLL 的方法来轮询数据. 即当有中断来了，驱动关闭中断，通知内核收包，内核软中断轮询当前网卡，在规定时间尽可能多的收包。时间用尽或者没有数据可收，内核再次开启中断，准备下一次收包.

    > napi使能: `void napi_enable(struct napi *napi); /void napi_disable(struct napi *napi);`

## 5. 启用网卡
ref:
- [intel网卡状态统计疑问](https://lp007819.wordpress.com/2013/05/23/intel%E7%BD%91%E5%8D%A1%E7%8A%B6%E6%80%81%E7%BB%9F%E8%AE%A1%E7%96%91%E9%97%AE/)

网卡驱动注册后就可以启动网卡了.

当启用一个网卡时(`ip link set eth0 up/ifconig ethO up`), net_device_ops变量中定义的ndo_open方法会被调用, 即igb_open().

[igb_open](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L4234)主要意图:
1. `igb_setup_all_tx_resources(adapter);`: 分配传输描述符数组
1. `igb_setup_all_rx_resources(adapter);`: 分配接收描述符数组

    通过循环创建了若干接收队列, 创建单个队列见[igb_setup_rx_resources()](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L4434).

    igb_setup_rx_resources()创建了RingBuffer, 包括:
    1. igb_rx_bufter数组: 内核使用的, 通过vmalloc申请的 : 预先分配
    1. e1000_adv_rx_desc数组: 网卡硬件使用的,通过dma_alloc_coherent分配 : 先分配
    1. 众多的skb: 动态申请

    查看RingBuffer: `ethtool -g eth0`, Pre-set maximums 指的是RingBuffer的最大值,Current harcware setings指的是当前的设置.  如果`ethtool -S eth0`的rx_no_buffer_count如果不为O的话(在ifconfig中体现为overruns指标增⻓)就表示有包因为RingBuffer 装不下而被丢弃了.

    调大RingBuffer()(`ethtool -G eth0 rx 4096 tx 4096`)可解决偶发的瞬时的丢包, 但排队的包过多会增加处理网络包的延时. 建议是加快内核消费RingBuffer中任务的速度.

1. `igb_request_irq(adapter);`: 注册中断处理函数

    在[igb_request_msix](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L930)中可以看到, 对于多队列的网卡, 为每一个队列都注册了硬中断, 其对应的中断处理函数是igb_msix_ring. 在msix方式下,每个Rx队列有独立的MSI-X中断, 从网卡硬件中断的层面就可以设置让收到的包被不同的CPU处理. (可以通过irqbalance或者修改/proc/irq/IRQ_NUMBER/smp_affinity,从而修改和CPU的绑定行为)

    igb_msix_ring作用: 检查有无新数据收到, 如果有, 调用 __napi_schedule()

    > e1000e是e1000_request_irq(), 硬件中断是[e1000_intr_msi](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/e1000e/netdev.c#L1750)
1.  `napi_enable(&(adapter->q_vector[i]->napi));`: 启用NAPI
1.  `netif_tx_start_all_queues()`: 开启全部队列

> Rx和Tx队列的数量和大小可以通过ethool进行配置

ethtool常见操作:
- 多队列查看: `ethtool -l eth0`, `Pre-set maximums`'s Combined表示当前网卡支持的最大队列数, `Current harddware settings`'s Combined当前开启的队列数. 或`ls /sys/class/net/eth0/queues`.
- 多队列修改: `ethtool -L eth0 combined 32`

> `ethtool -S`企业级网卡比消费级可能多很多统计项, 比如消费级可能没有rx_no_buffer_count

> (`ethtool -S` igb统计源头igb_gstrings_stats)[https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_ethtool.c#L32]

每个队列有自己的中断号见`cat /proc/interrupts`, 根据该中断号通过`/proc/irq/<N>/smp_affinity`可查看其亲和的cpu(亲和性是通过二进制中的比特位来标记的).

**在网络包的接收处理过程中, 主要工作集中在硬中断和软中断上,二者的消耗都可以通过top命令来查看**.

## 数据进入
### 1. 硬中断
当数据帧从网线到达网卡上的时候, 首先是网卡的接收队列. 网卡在分配给自己的RingBufler中寻找可用的内存位置, 找到后DMA引擎会把数据DMA到网卡之前关
联的内存里, 到这个时候CPU都是无感的. 当DMA操作完成以后,网卡会向CPU发起一个硬中断,通知CPU有数据到达. 内核调用驱动里注册的硬中断处理函数来启动NAPI, 继而触发软中断.

> 当RingBuffer满的时候,新来的数据包将被去弃. 使用`ip -s link ls dev eno1/ifconfig`命令查看网卡的时候, 可以看到里面有个missed/overrun, 表示因为环形队列满被丢弃的包数. 如果发现有丢包, 可能需要通过ethtool命令来加大环形队列的长度.

硬中断igb_msx_ring()意图:
1. `napi_schedule()`
1. `__napi_schedule()`
1. `____napi_schedule()`

    1. `list_add_tail(&napi->poll_list, &sd->poll_list);`

        list_add_tail修改了CPU变量softnet_data里的poll_list, 将驱动napi_struct传过来的poll_list添加了进来. softnet_data中的poll_list是一个双向列表, 其中的设备都带有输入帧等着被处理.
    2. `__raise_softirq_irqoff(NET_RX_SOFTIRQ);`

        触发了一个软中断 NET_RXSOFTIRQ, 触发过程只是对一个变量进行了一次或运算而已: `or_softirq_pending(1UL << nr);`

可以看出, 硬中断处理过程真的非常短,只是记录了一个寄存器, 修改了一下CPU的poll_list,然后发出一个软中断.

### 1. 软中断
软中断处理过程:
1. run_ksoftirqd()

    通过`local_softirq_pending()`判断, 硬中断(比如igb_msx_ring())写入的标记, 是否需要执行软中断

    > 硬中断中的设置软中断标记,和ksoftirgd中的判断是否有软中断到达, 都是基于smp_processor_id()的. 即只要硬中断在哪个CPU上被响应, 那么软中断也是在这个CPU上处理的. 因此**如果linux软中断的CPU消耗都集中在一个核上, 正确的做法应该是调整硬中断的CPU亲和性, 将硬中断打散到不同的CPU核上去**.
1. `__do_softirq()`

    在_do_softirq中, 判断根据当前CPU的软中断类型, 调用其注册的action方法

    > 软中断统计: `/proc/softirqs`
1. `net_rx_action()`

    函数开头的time_limit和budget(可调整)是用来控制net_rx_action()主动退出的,目的是保证网络包的接收 不霸占CPU不放, 等下次网卡再有硬中断过来的时候再处理剩下的接收数据包

    `local_irq_disable();`: 关闭硬中断, 避免硬中断重复将设备添加到poll_list.

    核心逻辑是获取当前CPU变量softnet_data, 对其poll_list进行遍历, 然后执行到网卡驱动注册到的poll(), 对于igb网卡来说即igb_poll().
1. [igb_poll()](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L8212)

    在读取操作中,igb_poll的重点工作是对igb_clean_rx_irq的调用.
1. [igb_clean_rx_irq()](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L8892)

    igb_fetch_rx_buffer和igb_is_non_eop的作用就是把数据帧从 RingButfer取下来.

    skb被从RingBuffer取下来以后, 会通过igb_alloc_rx_buffers申请新的skb再重新挂上去. 所以不要担心后面新包到来的时候没有skb可用.
    数据帧有可能要占多个RingBuffer, 需要在一个循环中获取的,直到帧尾部. 获取的一个数据帧用一个sk_buff 来表示. 收取完数据后,对其进行一些校验,然后开始设置sbk变量的timestamp、 VLAN_id、protocol等字段. 接下来进[napi_gro_receive()](https://elixir.bootlin.com/linux/v6.8/source/net/core/gro.c#L600).

    > dev_gro_receive()代表的是网卡GRO特性, 可以简单理解成把相关的小包合并成一个大包, 目的是减少传送给网络栈的包数, 这有助于减少对CPU的使用量.
    
    直接看napi_gro_receive()里面的[napi_skb_finish()](https://elixir.bootlin.com/linux/v6.8/source/net/core/gro.c#L573), 它主要就是调用了gro_normal_one()将skb放入 napi 的接收队列. 为了效率，接收队列中的数据，希望尽量批量处理.

    > gro_normal_batch 默认是 8，可以通过 `sysctl net.core.gro_normal_batch` 配置. 

    [gro_normal_list()](https://elixir.bootlin.com/linux/v6.8/source/include/net/gro.h#L435) 将数据从给协议栈，然后清空 napi 的接收队列.

1. [netif_receive_skb_list_internal()](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L5739)  

    它会进行 rps 检查，如果使能了 rps(`CONFIG_RPS=y`)，那么会为 skb 计算其他的 cpu，然后将 skb 放入那个 cpu 的 sd->backlog 中. sd->backlog 的类型也是 napi，因此在将数据放入 backlog 后，调用 schedule_napi，以 trigger 另一个 cpu 在自己的 软中断中处理从当前 cpu 转移过去的数据.

    > 多个 CPU 同时处理从网卡来的中断，处理收包过程。 这个特性被称作 RSS（Receive Side Scaling，接收端水平扩展）.
    > RPS（Receive Packet Steering，接收包控制，接收包引导）是 RSS 的一种软件实现.

    net_timestamp_check()根据设置的一个接收时间戳选项（`sysctl netdev_tstamp_prequeue`），这个选项决定在 packet 在到达 backlog queue 之前还是之后打时间戳:
    1. 如果启用，那立即打时间戳，在 RPS 之前（CPU 和 backlog queue 绑定）. 这个时间戳记录了数据包进入设备的瞬间，有助于更精确地测量数据包的延迟
    1. 否则，那只有在它进入到 backlog queue 之后才会打时间戳
    
    RPS 开启了，那这个选项可以将打时间戳的任务分散个其他 CPU，但会带来一些延迟.

    rps逻辑分叉:
    1. 未启用

        执行到 __netif_receive_skb_list()，它会继续调用 __netif_receive_skb_list_core()，后者做一些 bookkeeping 工作， 进而对每个 skb 调用 __netif_receive_skb_core()，将数据送到网络栈
    1. 启用 : 先执行enqueue_to_backlog(), 后面的逻辑和 `未启用` 一样.

        如果 RPS 启用了，它会做一些计算，判断使用哪个 CPU 的 backlog queue，这个过程由 get_rps_cpu() 完成. 它会考虑 RFS 和 aRFS 设置，以此选出一个合适的 CPU.

        通过 enqueue_to_backlog() 将 skb 加入 CPU 的 sd->input_pkt_queue, 再执行`napi_schedule_rps(sd)`(会将目标 CPU 的sd挂到本 CPU 的sd的rps_ipi_list便于后续向目标 CPU 发送 IPI 信号).

        > enqueue_to_backlog()先从远端 CPU 的 struct softnet_data 获取 backlog queue 长度. 如果 backlog 大于 netdev_max_backlog，或者超过了 flow limit，直接 drop，并更新 远端cdpu softnet_data 的 drop 统计.

        > flow limit 功能默认是关掉的. 要打开 flow limit，需要指定一个 bitmap（类似于 RPS 的 bitmap）

        napi_schedule_rps()将 backlog 的 poll(process_backlog()) 加入 poll_list 里.

        当软中断来临, 到 net_rx_action() 中执行process_backlog(), 如果存在sd_has_rps_ipi_waiting() (检查是否有待处理的 ipi), 如果存在先执行net_rps_action_and_irq_enable()处理ipi.
        
        > net_rps_action_and_irq_enable -> net_rps_send_ipi -> smp_call_function_single_async 远程激活sd->rps_ipi_list中的其他 CPU 的软中断，使其他 CPU 执行初始化时注册的软中断函数 csd = rps_trigger_softirq 来处理数据包. rps_trigger_softirq 函数将 backlog（napi）加入 poll_list 里，然后发出软中断信号 NET_RX_SOFTIRQ.

        napi_schedule_rps加了两次poll_list: 第一次在当前cpu执行, 让其执行process_backlog(), 第二次是在远端cpu添加.

        process_backlog 最终会消费 CPU 的input_pkt_queue队列数据包，经过 __netif_receive_skb()->__netif_receive_skb_one_core()，最终也调用 __netif_receive_skb_core()把数据包递交网络协议栈.

1. [__netif_receive_skb_core()](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L5322) : 完成将数据送到协议栈

    > 注释可参考[这里](https://mp.weixin.qq.com/s?__biz=MzkzMzM5MTUwNQ==&mid=2247484119&idx=1&sn=969c1e5b066c787781852fd6a625cb68&chksm=c24c7f29f53bf63fa3b3534631fe0001465c4f43807f632469621f24c12b8b4b027e03bd2de3&cur_album_id=2510886692103536641&scene=189#wechat_redirect)

    ![](/misc/img/net/netif_receive_skb_list_internal.png)

    它的主逻辑:
    1. 处理 skb 时间戳
    1. Generic XDP：软件执行 XDP 程序（XDP 是硬件功能，本来应该由硬件网卡来执行）

        Generic XDP 是软件层面的实现，当硬件网卡不支持 offload 模式的 XDP，可以选择 Generic 模式的 XDP.

        > XDP 的目的是将部分逻辑下放（offload）到网卡执行，通过硬件处理提高效率. 如果网卡不支持 XDP， 那 XDP 程序就会推迟到这里来执行。它并不能提升效率，所以主要用来测试功能.

        硬件XDP在igb_clean_rx_irq()执行.
    1. 处理 VLAN header
    1. TAP 处理：例如 tcpdump 抓包、流量过滤

        1. `list_for_each_entry_rcu(ptype, &ptype_all, list)`: 遍历全局变量链表ptype_all，对链接的每个packet_type结构调用函数deliver_skb这一步与下一步一同构成了将skb提交到taps嗅探设备，比如libpcap使用的PF_PACKET. tcpdump在这里抓包.
        1. `list_for_each_entry_rcu(ptype, &skb->dev->ptype_all, list)`: 遍历skb->dev->extended->ptype_all，对链接的每个packet_type结构调用函数deliver_skb这一步与上一步一同构成了将skb提交到taps嗅探设备，比如libpcap使用的PF_PACKET。区别在于这里的是只嗅探特定net_device的

        tcpdump是通过虛拟协议的方式工作的,它会将抓包西数以协议的形式挂到ptype all上, 遍历所有的`协议`, 这样就能抓到数据包了.

        > [dev_add_pack()](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L573)会把tcpdump用到的`协议`挂到ptype_all上, 见`grep -R "dev_add_pack(" net/{ipv4,packet}/*`里的`po->prot_hook`和`f->prot_hook`.  tcpdump 抓包一般通过 libpcap 库埋的探测点（TAP）.

        > 包类型: `cat /proc/net/ptype`

        每个packet_type结构就是数据包的一个可能去向.

    1. TC：TC 规则或 TC BPF 程序

        TC（Traffic Control）是 Linux 的流量控制子系统，通过调用 sch_handle_ingress 函数进入 TC ingress 处理

        1. 以前主要用于限速
        1. 5.10 版本之后，可以使用 TC BPF 编程来做流量的透明拦截和负载均衡
    1. Netfilter：处理 iptables 规则等

        Netfilter 是 Linux 的包过滤子系统，iptables 是其用户空间的客户端。通过调用 nf_ingress 函数进入 Netfilter ingress 处理.
    1. 递交给协议栈


        ```c
        if (likely(!deliver_exact)){
            // 如果没有设置精确匹配，将调用 deliver_ptype_list_skb() 函数传递数据包给指定的注册的协议处理函数处理 by 全局定义的协议
            deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type, &ptype_base[ntohs(type) & PTYPE_HASH_MASK]);
        }
        // 调用 deliver_ptype_list_skb() 函数传递数据包给指定的协议处理函数处理 by 设备上注册的协议进行处理
        deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type, &orig_dev->ptype_specific); 
        if (unlikely(skb->dev != orig_dev)) {
            // 如果数据包的网络设备与接收时的网络设备不一致，将调用 deliver_ptype_list_skb() 函数传递数据包给指定的协议处理函数处理 by 针对新设备的注册协议进行处理
            deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type, &skb->dev->ptype_specific);
        }
        ```

        > IP 层在函数 inet_init() 中将自身注册到 ptype_base 哈希表.

        deliver_ptype_list_skb()->deliver_skb(skb, pt_prev, orig_dev)->`pt_prev->func(skb, skb->dev, pt_prev, orig_dev)`

        `pt_prev->func`就调到了协议层的处理函数. 对于ip包就是ip_rcv; arp是arp_rcv.

        __netif_receive_skb_list_ptype???

## IP层处理
ref:
- [十年码农内功：收包（三）](https://mp.weixin.qq.com/s?__biz=MzkzMzM5MTUwNQ==&mid=2247484136&idx=1&sn=97d8c21299cea474446bc5a7a0e9ef4c&chksm=c24c7f16f53bf600c633c9ef5ce02c8b3749cb3e6ca801222c326d6e5a075676919aaecda609&cur_album_id=2510886692103536641&scene=189#wechat_redirect)

![](/misc/img/net/l3-processing-stack.png)

[ip_rcv()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_input.c#L560)

NF_HOOK是一个钩子函数. 它就是iptables netfiter过滤. 如果有很多或者很复杂的netiler规则,会在这里消耗过多的CPU资源, 加大网络
延迟. 另外,使用NF_HOOK在源码中搜索(`grep -r "NF_HOOK"`)可以搜到很多filter的过滤点, 想深入研究netfilter可以从这些NF_HOOK的这些引用处入手. 通过搜索结果可以看到, **主要是在IP、APP等层实现**的.

> netfilter 或 iptables 规则都是在软中断上下文中执行的，数量很多或规则很复杂时会导致网络延迟.

> TC BPF 也是在软中断上下文中， 但要比 netfilter/iptables 规则高效地多，也发生在更前面（能提前返回），所以应尽可能用 BPF.

> **tcpdump工作在设备层, 将包送到IP层以前就能处理. 而netilter工作在IP、ARP等层. 即netfilter是在tcpdump后面工作的, 所以iptables封禁规则影响不到tcpdump抓包. 发包过程恰恰相反,发包的时候,netfilter在协议层就被过滤掉了,所以tcpdump什么也看不到.**

当执行完注册的钩子后就会执行到最后一个参数指向的[ip_rcv_finish()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_input.c#L435).

ip_rcv_finish():
1. 调用 ip_rcv_finish_core 函数完成对 skb->dst_entry 的设置;
1. 调用 dst_input 函数，根据上一步设置的 skb->dst_entry 来跳到下一个处理该 skb 的函数

ip_rcv_finish_core()->  ip_route_input_noref()-> ip_route_input_rcu()-> ip_route_input_mc()-> rt_dst_alloc(). rt_dst_allo()如果 packet 的最终目的地是本机（local system），路由子系统会将 [ip_local_deliver](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_input.c#L242) 赋给 input.

> ip_early_demux 是一个优化项，通过检查相应的变量是否缓存在 socket 变量上，提前获得 skb 的 dst_entry（目标入口）. 默认 early_demux(`net.ipv4.ip_early_demux = 1`) 是打开的.

> 在某些场景下，特别是**大量短连接**的情况下，开启该机制可能会导致性能损耗，比如，对于那些在 TCP ESTABLISHED 表中找不到对应 Socket 的数据包，会经历 Early Demux 过程、查询「路由子系统」和 TCP 层又要再查一次 Socket 表，增加总体开销(最大5%).

> Early Demux（早期解复用）和查询 IP Route System（路由子系统）目的都是为了设置 skb->dst，如果 skb 是发给本机器，那么 Early Demux 和查询 IP Route System 获得的 dst_entry 会是同一个函数 ip_local_deliver；如果不是本机器，那么会转发出去

路由子系统完成工作后，会更新计数器，然后调用 dst_input(skb)，后者会进一步调用 dst_entry 变量中的 input 方法，这个方法是一个函数指针，由路由子系统初始化. 例如 ，如果 packet 的最终目的地是本机（local system），路由子系统会将 ip_local_deliver 赋给 input.

ip_local_deliver()中只要 packet 没有在 netfilter 被 drop，就会调用 ip_local_deliver_finish().

ip_local_deliver_finish 从数据包中读取协议，寻找注册在这个协议上的 struct net_protocol 变量，并调用该变量中的回调方法。这样将包送到协议栈的更上层. 根据上层协议类型选择不同的 callback 函数把数据收走：
- tcp_v4_rcv : from tcp_protocol
- [udp_rcv : from udp_protocol](https://mp.weixin.qq.com/s?__biz=MzkzMzM5MTUwNQ==&mid=2247484136&idx=1&sn=97d8c21299cea474446bc5a7a0e9ef4c&chksm=c24c7f16f53bf600c633c9ef5ce02c8b3749cb3e6ca801222c326d6e5a075676919aaecda609&cur_album_id=2510886692103536641&scene=189#wechat_redirect)
- icmp_send

## 内核与用户进程的协助
在协议栈接收处理完输入包以后,要能通知到用户进程,让用户进程能够
收到并处理这些数据, 两者的配合有很多种方案, 常见的有:
1. 同步阻塞

    一般在client端使用, 编程友好, 但性能差

    > exmaple: java bio

    现在有一些封装得很好的网络框架,例如Sogou Workllow、Golang的net包等, 在client端也早已摒弃了这种低效的模式
1. 多路IO复用

    linux的epoll, 编写不直观, 但性能突出, 特别适用于高并发场景

    epoll高性能最根本的原因是极大程度地减少了无用的进程上下文切换,让进程更专注地处理网络请求.

    > exmaple: java nio

> 阻塞: 进程因为等待某个事件而主动让出CPU挂起的操作

### socket
ref:
- [十年码农内功：收包（四）](https://mp.weixin.qq.com/s?__biz=MzkzMzM5MTUwNQ==&mid=2247484137&idx=1&sn=aa07fbc012b8a4ea6f6c3bebfc3847ae&chksm=c24c7f17f53bf60176c3a4c1a89c7dc20bdb9444042c8af65d2ffdf6c4b2dd05d6d4bffe9101&cur_album_id=2510886692103536641&scene=189#wechat_redirect)

`int sk = socket(AF_INET, SOCK_STREAM, 0);`调用成功后, 用户层面看到返回的是一个整数型的句柄,但其实内核在内部创建了一系列的socket相关的内核对象.

![](/misc/img/net/socket_objects.png)

实现:
1. [`SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)`](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1718)
1. [`sock_create(family, type, protocol, &sock);`](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1659)

    sock create是创建socket的主要位置,其中sock_create 又调用了__sock_create
1. [__sock_create](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1500)

    首先调用sock_alloc来分配一个struct sock内核对象, 接着获取协议族的操作函数表 by `rcu_dereference(net_families[family])`, 并调用其create方法. 对于AF_INET协议族来说,执行到的是inet_create()

1. [inet_create()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L251)

    ```c
    ...
    sock->ops = answer->ops; // 将inet_stream_ops 赋到socket->ops
	answer_prot = answer->prot; // 获得 tcpprot
	answer_flags = answer->flags;
	rcu_read_unlock();

	WARN_ON(!answer_prot->slab);

	err = -ENOMEM;
	sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot, kern); // 分配 sock对象,并把 tCp_prot 赋到sock->sk_prot 上
	if (!sk)
		goto out;

	err = 0;
	if (INET_PROTOSW_REUSE & answer_flags)
		sk->sk_reuse = SK_CAN_REUSE;

	if (INET_PROTOSW_ICSK & answer_flags)
		inet_init_csk_locks(sk);

	inet = inet_sk(sk);
	inet_assign_bit(IS_ICSK, sk, INET_PROTOSW_ICSK & answer_flags);

	inet_clear_bit(NODEFRAG, sk);

	if (SOCK_RAW == sock->type) {
		inet->inet_num = protocol;
		if (IPPROTO_RAW == protocol)
			inet_set_bit(HDRINCL, sk);
	}

	if (READ_ONCE(net->ipv4.sysctl_ip_no_pmtu_disc))
		inet->pmtudisc = IP_PMTUDISC_DONT;
	else
		inet->pmtudisc = IP_PMTUDISC_WANT;

	atomic_set(&inet->inet_id, 0);

	sock_init_data(sock, sk); // 对sock对象进行初始化
    ...
    ```

    在inet_create中,根据类型SOCK_STREAM查找到对于TCP定义的操作方法实现集合inet_stream_ops和tcp_prot, 并把它们分别设置到socket->ops和sock->sk_prot上.

    [sock_init_data()](https://elixir.bootlin.com/linux/v6.8/source/net/core/sock.c#L3427)将sock中的sk_data_ready函数指针进行了初始化, 设置为默认sock_defr_eadable.

    当软中断上收到数据包时会通过调用sk_data_ready函数指针来唤醒在sock上等待的进程.
    
    至此,一个tcp对象(AF_INET协议族下的SOCK_STPEAM对象)就算创建完成了.

### 同步阻塞
![](/misc/img/net/BIO.png)

![](/misc/img/net/recvfrom.png)

libc recv会执行recvfrom系统调用. 进入系统调用后,用户进程就进入了内核态,执行一系列的内核协议层函数,然后到socket对象的接收队列中查看是否有数据,没有的话就把自己添加到socket对应的等待队列里. 最后让出CPU,操作系统会选择下一个就绪状态的进程来执行.

> 重点是recvfrom最后是怎么把自己的进程阻塞掉的(假如没有使用O NONBLOCK标记)

实现:
- [`SYSCALL_DEFINE6(recvfrom, ...)`](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L2256)

    ```c
    ...
    sock = sockfd_lookup_light(fd, &err, &fput_needed); // 根据用户传入的fd找到socket对象
	if (!sock)
		goto out;

	if (sock->file->f_flags & O_NONBLOCK)
		flags |= MSG_DONTWAIT;
	err = sock_recvmsg(sock, &msg, flags);
    ...
    ```
1. [sock_recvmsg_nosec](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1043)

    调用socket对象Ops里的recvmsg即inet_recvmsg()
1. [inet_recvmsg()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L872)

    `INDIRECT_CALL_2(sk->sk_prot->recvmsg,...)`: 调用socket对象里的sk_prot下的recvmsg方法, 即tcp_recvmsg()
1. [tcp_recvmsg()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp.c#L2562)

1. [tcp_recvmsg_locked()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp.c#L2317)

    ```c
    ...
    do {
		...
		skb_queue_walk(&sk->sk_receive_queue, skb) { // 遍历接收队列接收数据
			...
		}

        ...

		if (copied >= target) {
			/* Do not sleep, just process backlog. */
			__sk_flush_backlog(sk);
		} else { // 没有收到足够数据, 启用sk_wait_data阻塞当前进程
			tcp_cleanup_rbuf(sk, copied);
			err = sk_wait_data(sk, &timeo, last);
			if (err < 0) {
				err = copied ? : err;
				goto out;
			}
		}
    ...
    ```

    [sk_wait_data](https://elixir.bootlin.com/linux/v6.8/source/net/core/sock.c#L3013):
    ```c
    int sk_wait_data(struct sock *sk, long *timeo, const struct sk_buff *skb)
    {
        DEFINE_WAIT_FUNC(wait, woken_wake_function); // 当前进程(current)关联到所定义的等待队列项上
        int rc;

        add_wait_queue(sk_sleep(sk), &wait); // 调用sk_sleep获取sock对象下的wait
        sk_set_bit(SOCKWQ_ASYNC_WAITDATA, sk); // 准备挂起,将进程状态设置为可打断 (INTERRUPTIBLE)
        rc = sk_wait_event(sk, timeo, skb_peek_tail(&sk->sk_receive_queue) != skb, &wait); // 通过调用schedule timeout让出CPU,然后进行睡眠
        sk_clear_bit(SOCKWQ_ASYNC_WAITDATA, sk);
        remove_wait_queue(sk_sleep(sk), &wait);
        return rc;
    }
    ```

    ![/misc/img/net/](socket_block.png)

    sk_wait_data实现:
    1. 首先用DEFINE_WAIT_FUNCT宏定义了一个等待队列项wait, 在这个新的等待队列项上, 注册了回调函数woken_wake_function, 并把当前进程描述符current关联到其private成员上
    1. 调用[sk_sleep](https://elixir.bootlin.com/linux/v6.8/source/include/net/sock.h#L2062)获取sock对象下的等待队列列表头wait_queue_head_t.
    1. 调用add_wait_queue来把新定义的等待队列项wait插入sock对象的等待队列

        这样后面**当内核收完数据产生就绪事件的时候,就可以查找socket等待队列上的等待项,进而可以找到回调函数和在等待该socket就绪事件的进程了**
    1. 调用sk_wait_event让出CPU,进程将进入睡眠状态,这会导致一次进程上下文的开销,而**这个开销是昂贵的,大约需要消耗几个微秒的CPU时间**

### tcp_v4_rcv
![](/misc/img/net/tcp_v4_rcv.png)

[tcp_v4_rcv](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L2161)流程:
1. tcp_v4_rcv
    ```c
    int tcp_v4_rcv(struct sk_buff *skb)
    {
        ...
        th = (const struct tcphdr *)skb->data; // 获取tcp header
        iph = ip_hdr(skb); // 获取ip header
    lookup:
        sk = __inet_lookup_skb(net->ipv4.tcp_death_row.hashinfo,
                    skb, __tcp_hdrlen(th), th->source,
                    th->dest, sdif, &refcounted); // 根据数据包 header中的ip、端口信息查找到对应的socket
        ...
        if (sk->sk_state == TCP_LISTEN) {
            ret = tcp_v4_do_rcv(sk, skb);
            goto put_and_return;
        }

        sk_incoming_cpu_update(sk);

        bh_lock_sock_nested(sk);
        tcp_segs_in(tcp_sk(sk), skb);
        ret = 0;
        if (!sock_owned_by_user(sk)) { // socket未被用户锁定
            ret = tcp_v4_do_rcv(sk, skb);
        } else {
            if (tcp_add_backlog(sk, skb, &drop_reason))
                goto discard_and_relse;
        }
        ...
    }
    ```
1. [tcp_v4_do_rcv](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L1885)

    ```c
    int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
    {
        enum skb_drop_reason reason;
        struct sock *rsk;

        if (sk->sk_state == TCP_ESTABLISHED) { /* Fast path */
            struct dst_entry *dst;

            dst = rcu_dereference_protected(sk->sk_rx_dst,
                            lockdep_sock_is_held(sk));

            sock_rps_save_rxhash(sk, skb);
            sk_mark_napi_id(sk, skb);
            if (dst) {
                if (sk->sk_rx_dst_ifindex != skb->skb_iif ||
                    !INDIRECT_CALL_1(dst->ops->check, ipv4_dst_check,
                            dst, 0)) {
                    RCU_INIT_POINTER(sk->sk_rx_dst, NULL);
                    dst_release(dst);
                }
            }
            tcp_rcv_established(sk, skb); // 执行连接状态下的数据处理
            return 0;
        }
        ..
    }
    ```
1. [tcp_rcv_established](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_input.c#L5998)

    ```c
    eaten = tcp_queue_rcv(sk, skb, &fragstolen); // 将接收到的数据放到socket的接收队列上
    ...
	tcp_data_ready(sk) // 来唤醒在socket上等待的用户进程
    ```

    tcp_data_ready()->sk_data_ready(): 在创建socket的流程里执行到的sock_init_data函数已经把sk_data_ready指针设置成sock_def_readable函数了, 它是默认的数据就绪处理函数.
1. [sock_def_readable](https://elixir.bootlin.com/linux/v6.8/source/net/core/sock.c#L3333)

    获取sock->sk_wq下的wait, 再调用wake_up_interruptible_sync_poll来唤醒在socket 上因为等待数据而被阻塞掉的进程了
1. [wake_up_interruptible_sync_poll](https://elixir.bootlin.com/linux/v6.8/source/include/linux/wait.h#L243)

    [__wake_up_sync_key](https://elixir.bootlin.com/linux/v6.8/source/kernel/sched/wait.c#L167)->[__wake_up_common_lock](https://elixir.bootlin.com/linux/v6.8/source/kernel/sched/wait.c#L99)->[__wake_up_common](https://elixir.bootlin.com/linux/v6.8/source/kernel/sched/wait.c#L73)

    传入__wake_up_common_lock的nr_exclusive=1: 即使有多个进程都阻塞在同一个socket 上,也只唤醒一个进程, 其作用是为了避免"惊群", 而不是把所有的进程都唤醒.

    __wake_up_common的`curr->func(curr, mode, wake_flags, key);`: 找出一个等待队列项curr,然后调用其curr->func(). 上面在recv()执行的时候, 内核已经报curr->func设置成了[woken_wake_function](woken_wake_function)

    woken_wake_function()->[default_wake_function()](https://elixir.bootlin.com/linux/v6.8/source/kernel/sched/core.c#L7055)->[try_to_wake_up()](https://elixir.bootlin.com/linux/v6.8/source/kernel/sched/core.c#L4222)

    调用try_to_wake_up时传人的task_struct是curr->private, 这个就是当时因为等待而被阻塞的进程项. 当这个函数执行完的时候, **在socket上等待而被阻塞的进程就被推入可运行队列里了, 这又将产生一次进程上下文切换的开销**

进程等待socket数据而进入sleep, 到数据已准备, 唤醒睡眠进程共产生2次进程上下文切换. 根据业界的测试,每一次切换大约花费3-5微秒左右浮动.

### epoll
在用户进程调用epoll_create时,内核会创建一个struct eventpol的内核对象,并把它关联到当前进程的已打开文件列表中

![](/misc/img/net/epoll_task.png)

![eventpol](/misc/img/net/epoll.png)

[epoll_create](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L2072)创建的[eventpoll](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L178)结构:
- wq:等待队列链表. 软中断数据就绪的时候会通过wq来找到阻塞在epoll对象上的用户进程
- rbr:一棵红黑树. 为了支持对海量连接的高效查找、插入和删除, eventpol内部使用了一棵红黑树. 通过这棵树来管理用户进程下添加进来的所有sockel连接
- rdlist:就绪的描述符的链表. 当有连接就绪的时侯,内核会把就绪的连接放到rdlist链表里. 这样应用进程只需要判断链表就能找出就绪连接,而不用去遍历整棵树.

它的初始化工作由[ep_alloc](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L975)中完成

使用[epoll_ctl](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L2266)注册每一个socket的时候,内核会做如下事情:
1. 根据传入的epfd, fd找到eventpoll、socket相关的内核对象, 并进入`case EPOLL_CTL_ADD`分支
1. ep_insert

    1. 分配一个红黑树节点对象[epitem](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L130) by kmem_cache_zalloc
    2. ep_rbtree_insert: 将epitem插入epoll对象的红黑树
    3. [ep_item_poll](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L883): 将等待事件添加到socket的等待队列中,其回调函数是[ep_poll_callback](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L1166)

![通过epoll_ctl添加两个socket](/misc/img/net/epoll_ctl_2socket.png)

![](/misc/img/net/epitem_init.png)
![](/misc/img/net/epoll_set_sockt_wq.png)

[ep_item_poll](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L883):
```c
static __poll_t ep_item_poll(const struct epitem *epi, poll_table *pt,
				 int depth)
{
	struct file *file = epi->ffd.file;
	__poll_t res;

	pt->_key = epi->event.events;
	if (!is_file_epoll(file))
		res = vfs_poll(file, pt);
	else
		res = __ep_eventpoll_poll(file, pt, depth);
	return res & epi->event.events;
}
```

`vfs_poll:file->f_op->poll(file, pt)`即[sock_poll](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1391)->`ops->poll`即[tcp_poll](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp.c#L497)->[sock_poll_wait](https://elixir.bootlin.com/linux/v6.8/source/include/net/sock.h#L2365)->[poll_wait](https://elixir.bootlin.com/linux/v6.8/source/include/linux/poll.h#L46)

poll_wait的`p->_qproc`是一个函数指针, 已在[ep_insert的init_poll_funcptr()中设置了ep_ptable_queue_proc()](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L1551)

在[ep_ptable_queue_proc](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L1271)中, 新建了一个等待队列项,并注册其回调函数为ep_poll_callback函数,然后再将这个等待项添加到socket的等待队列中.

[init_waitqueue_func_entry](https://elixir.bootlin.com/linux/v6.8/source/include/linux/wait.h#L88):
1. `wq_entry->private	= NULL`: 因为这里的socket是交给epol来管理的,不需要在一个socket就绪的时候就唤醒进程, 即不需要像recvfrom那样将等待对象项的private设置成当前用户进程描述符current
1. `wq_entry->func		= func;`: 将回调函数设置为ep_poll_callback. 软中断将数据收到socket的接收队列后, 会通过注册的这个ep_poll_callback()回调, 进而通知epoll对象

[epoll_wait](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L2324)逻辑: 观察eventpoll->rdllist链表里有没有数据, 有数据就返回,没有数据就创建一个等待队列项,将其添加到eventpol的等待队列上,然后把自己阻塞掉完事.

![](/misc/img/net/epoll_wait.png)

> epoll_ctl添加socket时也创建了等待队列项. 不同的是epoll_wait的等待队列项是挂在epoll对象上的,而是epoll_ctl的是挂在socket对象上的.

[epoll_wait](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L2324)->[ep_poll](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L1824):
```c
...
ep_events_available() // 判断就绪队列上有没有事件就绪
...
init_wait(&wait);
wait.func = ep_autoremove_wake_function;

write_lock_irq(&ep->lock);
/*
    * Barrierless variant, waitqueue_active() is called under
    * the same lock on wakeup ep_poll_callback() side, so it
    * is safe to avoid an explicit barrier.
    */
__set_current_state(TASK_INTERRUPTIBLE); // 当没有IO事件时, epoll也会阻塞掉当前进程. 这个是合理的,因为没有事情可做了占着CPU也没什么意义

/*
    * Do the final check under the lock. ep_scan_ready_list()
    * plays with two lists (->rdllist and ->ovflist) and there
    * is always a race when both lists are empty for short
    * period of time although events are pending, so lock is
    * important.
    */
eavail = ep_events_available(ep);
if (!eavail)
    __add_wait_queue_exclusive(&ep->wq, &wait); // 将上面定义的等待任务wait添加到了epol对象的等待队列中

write_unlock_irq(&ep->lock);

if (!eavail)
    timed_out = !schedule_hrtimeout_range(to, slack,
                            HRTIMER_MODE_ABS); // 通过__set_current_state把当前进程设置为可打断, 再调用schedule_hrtimeout_range让出CPU,主动进入睡眠状态
__set_current_state(TASK_RUNNING);
```

> [schedule_hrtimeout_range](https://elixir.bootlin.com/linux/v6.8/source/kernel/time/hrtimer.c#L2355)->[schedule()](https://elixir.bootlin.com/linux/v6.8/source/kernel/sched/core.c#L6807)

当数据接收到任务队列:
tcp_v4_rcv()->tcp_v4_do_rcv()->tcp_rcv_established()tcp_data_ready()->sk_data_ready(), 最终调到[ep_poll_callback](https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L1166).

ep_poll_callback:
```c
static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
    int pwake = 0;
	struct epitem *epi = ep_item_from_wait(wait); // 获取wait对应的epitem
	struct eventpoll *ep = epi->ep; // 获取epitem对应的eventpoll
	__poll_t pollflags = key_to_poll(key);
	unsigned long flags;
	int ewake = 0;
    ...
    if (READ_ONCE(ep->ovflist) != EP_UNACTIVE_PTR) {
		if (chain_epi_lockless(epi))
			ep_pm_stay_awake_rcu(epi);
	} else if (!ep_is_linked(epi)) {
		/* In the usual case, add event to ready list. */
		if (list_add_tail_lockless(&epi->rdllink, &ep->rdllist)) // 当前epitem添加到eventpoll的就绪队列中
			ep_pm_stay_awake_rcu(epi);
	}

	/*
	 * Wake up ( if active ) both the eventpoll wait list and the ->poll()
	 * wait list.
	 */
	if (waitqueue_active(&ep->wq)) { // 查看eventpo11的等待队列上是否有等待
		if ((epi->event.events & EPOLLEXCLUSIVE) &&
					!(pollflags & POLLFREE)) {
			switch (pollflags & EPOLLINOUT_BITS) {
			case EPOLLIN:
				if (epi->event.events & EPOLLIN)
					ewake = 1;
				break;
			case EPOLLOUT:
				if (epi->event.events & EPOLLOUT)
					ewake = 1;
				break;
			case 0:
				ewake = 1;
				break;
			}
		}
		wake_up(&ep->wq);
	}
	if (waitqueue_active(&ep->poll_wait))
		pwake++;
    ...
}
```

ep_poll_callback做的第一件事就是把自己的epitem添加到epoll的就绪队列中. 接着它又会查看eventpoll对象上的等待队列里是否有等待项(epoll wait执行的时候会设置). 如果没有等待项,软中断的事情就做完了. 如果有等待项,那就找到等待项里设置的回调函数(wake_up()-> __wake_up()->__wake_up_common_lock()->[__wake_up_common()](https://elixir.bootlin.com/linux/v6.8/source/kernel/sched/wait.c#L73)).

在 `__wake_up_common`里,调用curr->func这里的func是在epoll_wait时传入的ep_autoremove_wake_function(https://elixir.bootlin.com/linux/v6.8/source/fs/eventpoll.c#L1794), 再执行到[default_wake_function()](https://elixir.bootlin.com/linux/v6.8/source/kernel/sched/core.c#L7055).

在detault wake function中找到等待队列项里的进程描述符,然后唤醒它.

等待队列项cur->private指针是在epoll对象上等待而被阻塞掉的进程.

将epoll_walt进程推入可运行队列,等待内核重新调度进程. 当这个进程重新运行后, 从epoll_wait阻塞时暂停的代码处继续执行, 把rdlist中就绪的事件返回给用户进程.

```c
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, struct timespec64 *timeout)
{
	...

	while (1) {
		if (eavail) {
			/*
			 * Try to transfer events to user space. In case we get
			 * 0 events and there's still timeout left over, we go
			 * trying again in search of more luck.
			 */
			res = ep_send_events(ep, events, maxevents); // 给用户进程返回就绪事件
			if (res)
				return res;
		}

		...

		if (!eavail)
			timed_out = !schedule_hrtimeout_range(to, slack,
							      HRTIMER_MODE_ABS);
		__set_current_state(TASK_RUNNING);

		/*
		 * We were woken up, thus go and try to harvest some events.
		 * If timed out and still on the wait queue, recheck eavail
		 * carefully under lock, below.
		 */
		eavail = 1;

		if (!list_empty_careful(&wait.entry)) {
			write_lock_irq(&ep->lock);
			/*
			 * If the thread timed out and is not on the wait queue,
			 * it means that the thread was woken up after its
			 * timeout expired before it could reacquire the lock.
			 * Thus, when wait.entry is empty, it needs to harvest
			 * events.
			 */
			if (timed_out)
				eavail = list_empty(&wait.entry);
			__remove_wait_queue(&ep->wq, &wait);
			write_unlock_irq(&ep->lock);
		}
	}
}
```

从用广角度来看,epolt wrait只是多等了一会儿而已,但执行流程还是顺序的.

![](/misc/img/net/epoll_arch.png)

其中软中断回调时的回调⻄数调用关系整理如下:
sock_def_readable: sock对象初始化时设置的
    => ep_poll_callback: 调用epoll_ctl时添加到socket 上的
        => ep_poll_callback:调用epoll_wait时设置到epol上的

总结一下,epoll相关的函数里内核运行环境分两部分:
1. 用户进程内核态. 调用epoll_wait等函数时会将进程陷入内核态来执行. 这部分代码负责查看接收队列, 以及负责把当前进程阻塞掉, 让出CPU
1. 硬/软中断上下文. 在这些组件中,将包从网卡接收过来进行处理,然后放到socket的接收队列. 对于epoll来说, 再找到socket关联的epitem, 并把它添加到
epoll对象的就绪链表中. 这个时候再捎带检查一下epoll上是否有被阻塞的进程, 如果有唤醒它

在实践中,只要活儿足够多,epoll_weit根本不会让进程阻塞. 用户进程会一直干活儿, 直到epoll_wait里实在没活儿可干的时候才主动让出CPU. 这就是epoll高效的核心原因所在!

# send
ref:
- [十年码农内功：网络发包详细过程(一)](https://mp.weixin.qq.com/s?__biz=MzkzMzM5MTUwNQ==&mid=2247484152&idx=1&sn=8e097a2198bdaedd85ff41196770de38&chksm=c24c7f06f53bf6107ff404a1da6d728b4783f29686321d6754e5d21ab815164cb461f54405c5#wechat_redirect)

    udp发送过程

![](/misc/img/net/send_arch.png)

send过程
1. 应用层

    `send(fd, buf, sizeof(buf), 0)`
1. 系统调用

    [sendto](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L2199)->[__sock_sendmsg](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L740)->[sock_sendmsg_nosec()](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L728)的`READ_ONCE(sock->ops)->sendmsg`
1. 协议栈

    传输层: [inet_sendmsg](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L843)的`sk->sk_prot->sendmsg`->[tcp_sendmsg](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp.c#L1336)->[tcp_write_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_output.c#L2700)->[tcp_transmit_skb](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_output.c#L1477)-> __tcp_transmit_skb的`icsk->icsk_af_ops->queue_xmit`

    网络层: [ip_queue_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_output.c#L547)->[ip_local_out](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_output.c#L123)->[dst_output()](https://elixir.bootlin.com/linux/v6.8/source/include/net/dst.h#L449)->[ip_output()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_output.c#L426)->[ip_finish_output()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_output.c#L316)->[ip_finish_output2](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_output.c#L198)->neigh_output()

1. 邻居子系统

    [neigh_output()](https://elixir.bootlin.com/linux/v6.8/source/include/net/neighbour.h#L529)->[neigh_hh_output()](https://elixir.bootlin.com/linux/v6.8/source/include/net/neighbour.h#L489)/[neigh_resolve_output](https://elixir.bootlin.com/linux/v6.8/source/net/core/neighbour.c#L1543)->dev_queue_xmit()
1.  网络设备子系统

    [dev_queue_xmit()](https://elixir.bootlin.com/linux/v6.8/source/include/linux/netdevice.h#L3166) ->[__dev_queue_xmit()](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L4259)
    
    __dev_queue_xmit:
    - 有队列: [__dev_xmit_skb](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L3748)->[__qdisc_run](https://elixir.bootlin.com/linux/v6.8/source/net/sched/sch_generic.c#L410)->[qdisc_restart](https://elixir.bootlin.com/linux/v6.8/source/net/sched/sch_generic.c#L388)->[sch_direct_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/sched/sch_generic.c#L314)->[dev_hard_start_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L3553)
    - 没有队列: [dev_hard_start_xmit()](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L3553)->xmit_one()->[netdev_start_xmit](https://elixir.bootlin.com/linux/v6.8/source/include/linux/netdevice.h#L4994)->[__netdev_start_xmit](https://elixir.bootlin.com/linux/v6.8/source/include/linux/netdevice.h#L4981)的`ops->ndo_start_xmit(skb, dev)`

1. 驱动

    igb_netdev_ops的ndo_start_xmit=[igb_xmit_frame](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L6559)->[igb_xmit_frame_ring](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L6460)->[igb_tx_map()](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L6207)

1. 硬件

释放缓存队列等资源:
1. 硬件
1. 硬中断: [igb_msix_ring](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L6207)的`napi_schedule(&q_vector->napi)` ->[napi_schedule](https://elixir.bootlin.com/linux/v6.8/source/include/linux/netdevice.h#L519) ->[____napi_schedule](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L4439)->[__raise_softirq_irqoff](https://elixir.bootlin.com/linux/v6.8/source/kernel/softirq.c#L691)

    发送数据在硬中断最終触发的软中断却是NET_RX_SOFTIRQ, 而并不是NET_TX_SOFTIRQ. 这是/proc/softirqs能看到NET_RX更多的原因之一
1. 软中断

    [net_rx_action()](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L6741)->[napi_poll](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L6635)->__napi_poll()的`work = n->poll(n, weight)`
1. 驱动

    [igb_poll](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L8212)->[igb_clean_tx_irq](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L8892)

## 网卡就位
`__igb_open()`调用igb_setup_all_tx_resources 分配所有的传输 RingBuffer by [igb_setup_tx_resources](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L4285).

一个传输 RingBuffer的内部也不仅仅是一个环形队列数组:
- igb_tx_butfer数组: 这个数组是内核使用的, 通过vmalloc申请
- e1000_adv_tx_desc数组: 这个数组是网卡硬件使用的, 通过dma_alloc_coherent分配

在发送的时候,这两个环形数组中相同位置的指针都将指向同一个skb. 这样,内核和硬件就能共同访问同样的数据了,内核往skb写数据,网卡硬件负责发送.

## 数据从用户态到网卡
send()对应`sendto`系统调用, 它主要只干了两件简单的事情:
1. 在内核中把真正的socket 找出来,在这个对象里记录着各种协议栈的函数地址

    `sock = sockfd_lookup_light(fd, &err, &fput_needed)`
1. 构造一个struct msghdr对象,把用户传入的数据,比如butfer地址、数据长度什么的,都装进去

之后交给协议栈的inet_sendmsg()(AF_INET协议族提供的通用发送函数), 该函数是从socket内核对象里的ops成员([inet_stream_ops](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L1051))中找到的.

> 在用户态使用的send函数和sendto西数其实都是sendto系统调用实现的.

在进入协议栈inet_sendrmsg以后, 内核接着会找到socket上的具体协议发送函数. 对于TCP协议来说, 那就是tcp_sendmsg(同样也是通过socket內核对象找到的).

tcp_sendmsg的tcp_sendmsg_locked:
```c
int tcp_sendmsg_locked(struct sock *sk, struct msghdr *msg, size_t size)
{
	...

restart:
	mss_now = tcp_send_mss(sk, &size_goal, flags);

	err = -EPIPE;
	if (sk->sk_err || (sk->sk_shutdown & SEND_SHUTDOWN))
		goto do_error;

	while (msg_data_left(msg)) { // msg_data_left: 获取未拷贝的数据块数
		ssize_t copy = 0;

		skb = tcp_write_queue_tail(sk); // 获取socket发送队列中的最后一个skb. 
		if (skb)
			copy = size_goal - skb->len;

		if (copy <= 0 || !tcp_skb_can_collapse_to(skb)) {
			bool first_skb;

new_segment:
			if (!sk_stream_memory_free(sk))
				goto wait_for_space;

			if (unlikely(process_backlog >= 16)) {
				process_backlog = 0;
				if (sk_flush_backlog(sk))
					goto restart;
			}
			first_skb = tcp_rtx_and_write_queues_empty(sk);
			skb = tcp_stream_alloc_skb(sk, sk->sk_allocation,
						   first_skb); // 申请skb, 并添加到发送队列的尾部
			if (!skb)
				goto wait_for_space;

			process_backlog++;

			tcp_skb_entail(sk, skb); // 把skb挂到socket的发送队列上
			copy = size_goal;

			/* All packets are restored as if they have
			 * already been sent. skb_mstamp_ns isn't set to
			 * avoid wrong rtt estimation.
			 */
			if (tp->repair)
				TCP_SKB_CB(skb)->sacked |= TCPCB_REPAIRED;
		}

		...

		if (forced_push(tp)) { // 发送判断
			tcp_mark_push(tp, skb);
			__tcp_push_pending_frames(sk, mss_now, TCP_NAGLE_PUSH);
		} else if (skb == tcp_send_head(sk))
			tcp_push_one(sk, mss_now);
		continue;

...
}
EXPORT_SYMBOL_GPL(tcp_sendmsg_locked);
```

用户的发送队列就是skb(struct sk_buff)组成的一个链表.

条件都不满足的话,这次用户要发送的数据只是拷贝到内核就算完事了.

只有满足 forced_push(tp)或者 skb == tcp_send_head(sk)成立的时候,内核才会真正启动发送数据包, 其中 forced_push(tcp) 判断的是未发送的数据是否已经超过最大窗口的一半了. 无论调用的是__tcp_push_pending_frames 还是tcp_push_one, 最终都会实际执行到[tcp_write_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_output.c#L2700).

tcp_write_xmit()处理了传输层的拥塞控制、滑动窗又相关的工作. 满足窗口要求的时候, 调用[tcp_transmit_skb](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_output.c#L1477)真正开启发送.

[__tcp_transmit_skb](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_output.c#L1281):
```c
static int __tcp_transmit_skb(struct sock *sk, struct sk_buff *skb,
			      int clone_it, gfp_t gfp_mask, u32 rcv_nxt)
{
	...
	if (clone_it) {
		oskb = skb;

		tcp_skb_tsorted_save(oskb) {
			if (unlikely(skb_cloned(oskb)))
				skb = pskb_copy(oskb, gfp_mask);
			else
				skb = skb_clone(oskb, gfp_mask); // 克隆skb
		} tcp_skb_tsorted_restore(oskb);

		if (unlikely(!skb))
			return -ENOBUFS;
		/* retransmit skbs might have a non zero value in skb->dev
		 * because skb->dev is aliased with skb->rbnode.rb_left
		 */
		skb->dev = NULL;
	}

	...

	/* Build TCP header and checksum it. */ // 封装TCP头
	th = (struct tcphdr *)skb->data;
	th->source		= inet->inet_sport;
	th->dest		= inet->inet_dport;
	th->seq			= htonl(tcb->seq);
	th->ack_seq		= htonl(rcv_nxt);
	*(((__be16 *)th) + 6)	= htons(((tcp_header_size >> 2) << 12) |
					tcb->tcp_flags);

	th->check		= 0;
	th->urg_ptr		= 0;

	...

	err = INDIRECT_CALL_INET(icsk->icsk_af_ops->queue_xmit, // 调用网络层发送接口
				 inet6_csk_xmit, ip_queue_xmit,
				 sk, skb, &inet->cork.fl);

	...
}
```

__tcp_transmit_skb克隆skb的理由: 因为 skb 后续在调用网络层,最后到达网卡发送完成的时候,这个 skb 会被释放掉. 而TCP协议是支持丢失重传的,在收到对方的ACK之前,这个skb不能被删除. 所以内核的做法就是每次调用网卡发送的时候,实际上传递出去的是skb的一个拷贝, 等收到ACK再真正删除.

设置skb中的TCP头的细节: skb内部其实包含了网络协议中所有的头(header), 在设置TCP头的时候,只是把指针指向skb的合适位置. 后面设置ip头的时候,再把指针挪一挪就行, 这样可避免频繁的内存申请和拷贝,效率更高.

icsk->icsk_af_ops->queue_xmit的icsk_af_ops=ipv4_specific即queue_xmit=[ip_queue_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_output.c#L547).

在网络层主要处理路由项查找、IP头设置、netilter过滤、skb切分(大于MTU的话)等几项工作,处理完这些工作后会交给更下一层的邻居子系统来处理.

__ip_queue_xmit:
```c
int __ip_queue_xmit(struct sock *sk, struct sk_buff *skb, struct flowi *fl,
		    __u8 tos)
{
	...

	/* Make sure we can route this packet. */
	rt = (struct rtable *)__sk_dst_check(sk, 0); // 检查socket是否有缓存的路由表
	if (!rt) {
		__be32 daddr;

		/* Use correct destination address if we have options. */
		daddr = inet->inet_daddr;
		if (inet_opt && inet_opt->opt.srr)
			daddr = inet_opt->opt.faddr;

		/* If this fails, retransmit mechanism of transport layer will
		 * keep trying until route appears or the connection times
		 * itself out.
		 */
		rt = ip_route_output_ports(net, fl4, sk,
					   daddr, inet->inet_saddr,
					   inet->inet_dport,
					   inet->inet_sport,
					   sk->sk_protocol,
					   RT_CONN_FLAGS_TOS(sk, tos),
					   sk->sk_bound_dev_if); // 查找路由项(`route -n`), 并缓存到socket中
		if (IS_ERR(rt))
			goto no_route;
		sk_setup_caps(sk, &rt->dst);
	}
	skb_dst_set_noref(skb, &rt->dst); // 为skb设置路由表

    ...
	iph = ip_hdr(skb); // 设置ip头
	*((__be16 *)iph) = htons((4 << 12) | (5 << 8) | (tos & 0xff));
	if (ip_dont_fragment(sk, &rt->dst) && !skb->ignore_df)
		iph->frag_off = htons(IP_DF);
	else
		iph->frag_off = 0;
	iph->ttl      = ip_select_ttl(inet, &rt->dst);
	iph->protocol = sk->sk_protocol;
	ip_copy_addrs(iph, fl4);

	...

	res = ip_local_out(net, sk, skb); // 发送
	...
}
EXPORT_SYMBOL(__ip_queue_xmit);
```

ip_local_out->__ip_local_out的nf_hook的过程中会执行netfilter过滤. 如果使用iptables配置了一些规则,那么这里将检测是否命中规则. 如果设置了非常复杂的netfilter 规则, 那该函数将会导致CPU开销大增.

nf_hook的dst_output会执行`skb_dst(skb)->output`即此函数会找到这个skb的路由表(dst条目),然后调用路由表的output方法, 该函数指针指向的是ip_output方法.

在ip_output中进行一些简单的统计工作,再次执行netilter过滤, 过滤通过之后回调ip_finish_output().

在ip_finish_output中可以看到,如果数据大于MTU,是会执行分片的. 分片的话是执行`ip_fragment(net, sk, skb, mtu, ip_finish_output2);`

分片缺点:
1. 需要进行额外的切分处理,有额外性能开销
2. 只要一个分片丢失, 整个包都要重传

所以避免分片既杜绝了分片开销,也大大降低了重传率.

ip_finish_output2通过[ip_neigh_for_gw()](https://elixir.bootlin.com/linux/v6.8/source/include/net/route.h#L373)->[__ipv4_neigh_lookup_noref](https://elixir.bootlin.com/linux/v6.8/source/include/net/arp.h#L22)(其第二个参数传入的是路由下一跳ip信息)查找arp, 如果找不到,则调用__neigh_create创建一个邻居, 然后将包发给邻居子系统的neigh_output.

邻居子系统(`/net/core/neighbour.c`)是位于网络层和数据链路层中间的一个系统,其作用是为网络层提供个下层的封装,让网络层不必关心下层的地址信息,让下层来决定发送到哪个 MAC地址.

在邻居子系统里主要查找或者创建邻居项,在创建邻居项的时候,有可能会发出实际的arp请求, 然后封装MAC 头,将发送过程再传递到更下层的网络设备子系统.

有了邻居项以后, 此时仍然不具备发送ip报文的能力,因为目的MAC地址还未获取. 调用neigh_output继续传递skb by `READ_ONCE(n->output)(n, skb)`.

n->output实际指向的是[neigh_resolve_output](https://elixir.bootlin.com/linux/v6.8/source/net/core/neighbour.c#L1543). 在这个函数内部有可能发出arp请求(by neigh_event_send)

当获取到硬件MAC地址以后,就可以封装skb的MAC头了. 最后调用dev_queue_xmit将skb传递给Linux网络设备子系统.

[dev_queue_xmit()](https://elixir.bootlin.com/linux/v6.8/source/include/linux/netdevice.h#L3166)->[__dev_queue_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L4259).

__dev_queue_xmit:
```c
int __dev_queue_xmit(struct sk_buff *skb, struct net_device *sb_dev)
{
	...

	if (!txq)
		txq = netdev_core_pick_tx(dev, skb, sb_dev); // 选择发送队列

	q = rcu_dereference_bh(txq->qdisc); // 获取与此队列关联的排队规则

	trace_net_dev_queue(skb);
	if (q->enqueue) { // 如果有队列, 则__dev_xmit_skb继续处理
		rc = __dev_xmit_skb(skb, q, dev, txq);
		goto out;
	}

    // 没有队列的是回环设备和隧道设备
	...
}
EXPORT_SYMBOL(__dev_queue_xmit);
```

netdev_core_pick_tx()发送队列的选择受XPS等配置的影响,而且还有缓存.

大部分的设备都有队列(回环设备和隧道设备除外),所以现在进入[__dev_xmit_skb](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L3748).

__dev_xmit_skb有两套逻辑: 一种是可以bypass(绕过)排队系统,另外一种是正常排队(可通过tc命令查看).

正常排队:
```c
rc = dev_qdisc_enqueue(skb, q, &to_free, txq); // 入队
__qdisc_run(q); // 开始发送
```

[__qdisc_run](https://elixir.bootlin.com/linux/v6.8/source/net/sched/sch_generic.c#L410):
```c
void __qdisc_run(struct Qdisc *q)
{
	int quota = READ_ONCE(dev_tx_weight);
	int packets;

    //循环从队列取出一个skb并发送
	while (qdisc_restart(q, &packets)) {
		quota -= packets;
		if (quota <= 0) {
			if (q->flags & TCQ_F_NOLOCK)
				set_bit(__QDISC_STATE_MISSED, &q->state);
			else
				__netif_schedule(q); //将触发一次NET_TX_SOFTIRQ类型softirq

			break;
		}
	}
}
```

__qdisc_run()占用的是用户进程的系统态时间(sy). 只有当quota用尽的时候才触发软中断进行发送.

所以这就是为什么`/proc/softirqs`,一般NET_RX都要比NET_TX大得多的第二个原因. 对于接收来说, 都要经过 NET_RX软中断,而对于发送来说,只有系统
态配额用尽才让软中断上.

[qdisc_restart](https://elixir.bootlin.com/linux/v6.8/source/net/sched/sch_generic.c#L388) 通过`skb = dequeue_skb(q, &validate, packets)`取出skb再调用[sch_direct_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/sched/sch_generic.c#L314)->[dev_hard_start_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L3553)

因此__qdisc_run有两种发送途径: 1, 直接发送; 2, 通过软中断发送

发送网络包的时候系统态CPU用尽了, 会调用[__netif_schedule](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L3100)触发一个软中断. 该函数会进入[__netif_reschedule](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L3086), 此时软中断能访问到的softnet_data设置了要发送的数据队列,添加到output_queue里, 紧接着触发了NET_TX_SOFTIPRQ类型的软中断.

软中断是由内核线程来运行的, 会进入net_tx_action(), 在该函数中能获取发送队列, 并也最终调用到驱动程序里的入口函数dev_hard_start_xmit.

进入[net_tx_action()](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L5125)后发送数据消耗的CPU就都显示在si里, 不会消耗用户进程的系统时间.

net_tx_action:
```c
static __latent_entropy void net_tx_action(struct softirq_action *h)
{
	struct softnet_data *sd = this_cpu_ptr(&softnet_data); // 通过softnet_data获取发送队列

	...

	if (sd->output_queue) { //如果output_queue上有qdisc. 之前进程内核态在调用__netif_reschedlule的时候把发送队列写到softnet_data的output_queue里了
		struct Qdisc *head;

		local_irq_disable();
		head = sd->output_queue; // 将head指向第一个qdisc
		sd->output_queue = NULL;
		sd->output_queue_tailp = &sd->output_queue;
		local_irq_enable();

		rcu_read_lock();

		while (head) { // 遍历qdsics列表
			struct Qdisc *q = head;
			spinlock_t *root_lock = NULL;

			head = head->next_sched;

			/* We need to make sure head->next_sched is read
			 * before clearing __QDISC_STATE_SCHED
			 */
			smp_mb__before_atomic();

			if (!(q->flags & TCQ_F_NOLOCK)) {
				root_lock = qdisc_lock(q);
				spin_lock(root_lock);
			} else if (unlikely(test_bit(__QDISC_STATE_DEACTIVATED,
						     &q->state))) {
				/* There is a synchronize_net() between
				 * STATE_DEACTIVATED flag being set and
				 * qdisc_reset()/some_qdisc_is_busy() in
				 * dev_deactivate(), so we can safely bail out
				 * early here to avoid data race between
				 * qdisc_deactivate() and some_qdisc_is_busy()
				 * for lockless qdisc.
				 */
				clear_bit(__QDISC_STATE_SCHED, &q->state);
				continue;
			}

			clear_bit(__QDISC_STATE_SCHED, &q->state);
			qdisc_run(q); // 发送数据
			if (root_lock)
				spin_unlock(root_lock);
		}

		rcu_read_unlock();
	}

	xfrm_dev_backlog(sd);
}
```

qdisc_run()->_ qdisc_ run(), 之后逻辑同上.

dev_hard_start_xmit->[xmit_one](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L3536)->[netdev_start_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L3536)->[__netdev_start_xmit](https://elixir.bootlin.com/linux/v6.8/source/include/linux/netdevice.h#L4981)的`ops->ndo_start_xmit(skb, dev)`

ndo_start xmit是网卡驱动要实现的一个函数,是在net_device_ops中定义的. 在igb网卡驱动源码中找到了net_device_ops函数是igb_xmit_frame. 在该函数里, 会将skb挂到RingBuffer上, 驱动调用完毕,数据包将真正从网卡发
送出去.

igb_xmit_frame->[igb_xmit_frame_ring](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L6460)->[igb_tx_map](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L6207)

igb_tx_map将skb数据映射到网卡可访问的内存DMA区域.

igb_tx_map:
```c
static int igb_tx_map(struct igb_ring *tx_ring,
		      struct igb_tx_buffer *first,
		      const u8 hdr_len)
{
	...

	tx_desc = IGB_TX_DESC(tx_ring, i); // 获取下一个可用描述符指针

	igb_tx_olinfo_status(tx_ring, tx_desc, tx_flags, skb->len - hdr_len);

	size = skb_headlen(skb);
	data_len = skb->data_len;

	dma = dma_map_single(tx_ring->dev, skb->data, size, DMA_TO_DEVICE); // 为skb->data构造内存映射, 以允许设备通过DMA从ram中读取数据

	tx_buffer = first;

	for (frag = &skb_shinfo(skb)->frags[0];; frag++) { // 遍历该数据包的所有分片, 为skb的每个分片生成有效映射
		...
	}

	/* write last descriptor with RS and EOP bits */ // 设置最后一个descriptor
	cmd_type |= size | IGB_TXD_DCMD;
	tx_desc->read.cmd_type_len = cpu_to_le32(cmd_type);

	...
}
```

当所有需要的描述符都已建好,且skb的所有数据都映射到DMA地址后,驱动就会进入到它的最后一步,触发真实的发送.

当数据发送完以后,其实工作并没有结束, 因为内存还没有清理. 当发送完成的时候,网卡设备会触发一个硬中断来释放内存.

**无论硬中断是因为有数据要接收,还是发送完成通知, 从硬中断触发的软中断都是NET_RX_SOFTIRQ. 它是软中断统计中RX要高于TX的一个原因**.

[igb_clean_tx_irq](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L8255)清理了skb, 解除了DMA映射, 到了这一步,传输才算是基本完成了. 因为传输层需要保证可靠性,所以其实skb还没有删除. 它得等收到对方的ACK之后才会真正删除,那个时候才算彻底发送
完毕.

相关疑问:
1. 在监控内核发送数据消耗的CPU时,应该看sy还是si?

    在网络包的发送过程中,用户进程(在内核态) 完成了绝大部分的工作,甚至连调用驱动的工作都干了. 只当内核态进程被切走前才会发起软中断. 发送过程中,绝大部分(90%)以上的开销都是在用户进程内核态消耗掉的.

    只有一少部分情况才会触发软中断 (NET_TX类型),由软中断 ksoftirad内核线程来发送.

    所以,在监控网络IO对服务器造成的CPU开销的时候,不能仅看si,而是应该把sisy都考虑进来.
1. `/proc/softirqs`,为什么NET_RX要比NET_TX大得多的多?

    1. 当数据发送完以后,通过硬中断的方式来通知驱动发送完毕. 但是硬中断无论是有数据接收,还是发送完毕,触发的软中断都是NET_RX_SOFTIRQ,并不是NET_TX_SOFTIRQ.
    1. 对于读来说,都是要经过NET_Rx软中断的,都走ksoftiraa内核线程. 而对于发送来说,绝大部分工作都是在用户进程内核态处理了,只有系统态配额用尽才会发出NET_TX,让软中断上.
1. 发送网络数据的时候都涉及哪些内存拷⻉操作?

    这里的内存拷贝,只特指待发送数据的内存拷贝.

    第一次拷贝操作是在内核申请完skb之后,这时候会将用户传递进来的buffer里的数据内容都拷贝到skb. 如果要发送的数据量比较大,这个拷贝操作开销还是不小的.

    第二次拷贝操作是从传输层进入网络层的时候,每一个skb都会被克隆出来一个新的副本. 目的是保存原始的skb,当网络对方没有发回ACK的时候,还可以重新发送,以实现TCP中要求的可靠传输. 不过这次只是浅拷贝,只拷贝skb描述符本身,所指向的数据还是复用的.

    第三次拷贝不是必需的,只有当ip层发现skb大于MTU时才需要进行. 此时会再申请额外的skb,并将原来的skb拷贝为多个小的skb

    TCP为了保证可靠性,第二次的拷贝根本就没法省. 如果包大于MTU,分片时的拷贝同样避免不了.
1. 零拷贝到底是怎么回事?

    拿sendfile系统调用来举例.

    如果想把一个文件通过网络发送出去, 需先用 read系统调用把文件读取到内存,然后再调用send把文件发送出去.

    假设文件之前从来没有读取过,那么read 硬盘上的数据需要经过两次拷贝才能到用户进程的内存: 第一次是从硬盘DMA到Page Cache, 第二次是从Page Cache拷贝到用户内存. 之后再调用send发送数据.

    如果要发送的数据量比较大,那需要花费不少的时间在大量的数据拷贝上. sendfile就是内核提供的一个可用来减少发送文件时拷贝开销的一个技术方案. 在sendtile系统调用里, 数据不需要拷贝到用户空间, 在内核态就能完成发送处理, 这就显著减少了需要拷贝的次数.

    sendfile: 硬盘->page cache->socket发送缓冲区->网卡

    katka高性能的原因有很多,其中的重要原因之一就是采用了sendfile系统调用来发送网络数据包,减少了内核态和用户态之间的频繁数据拷贝.

# local网络
![](/misc/img/net/cross_host_net.png)

发送数据进入协议栈到达网络层的时候,网络层入口函数是ip_queue_xmit. 在网络层里会进行路由选择,路由选择完毕,再设置一些IP头,进行一些netfilter的过滤,将包交给邻居子系统.

对于本机网络IO来说,特殊之处在于在local路由表中就能找到路由项,对应的设备都将使用loopback网卡,也就是常说的lo设备.

[ip_queue_xmit](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/ip_output.c#L547)->__ip_queue_xmit->[ip_route_output_ports](https://elixir.bootlin.com/linux/v6.8/source/include/net/route.h#L159)->[ip_route_output_flow](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/route.c#L2869)->[__ip_route_output_key](https://elixir.bootlin.com/linux/v6.8/source/include/net/route.h#L131)->[ip_route_output_key_hash](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/route.c#L2629)->[ip_route_output_key_hash_rcu](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/route.c#L2651)->[fib_lookup](https://elixir.bootlin.com/linux/v6.8/source/include/net/ip_fib.h#L310)

在fib_lookup中将会对local和main两个路由表展开查询,并且先查询local后查main. 命令`ip route list table xxx`可以查看到这两个路由表, 这里只看local路由表, 因为本机网络IO查询到这个表就终止了. 因此对于目的是127.0.0.1的路由在local路由表中就能够找到.

ip_route_output_key_hash_rcu中显示(by `res->type == RTN_LOCAL`), 对于本机的网络请求,设备将全部使用net->loopback_dev,也就是lo虚拟网卡.

接下来的网络层仍然和跨机网络IO一样,最终会经过 ip_finish_output.

本机网络IO需要进行IP分片吗? 因为和正常的网络层处理过程一样,会经过ip_finish_output(). 在这个函数中,如果skb大于MTU,仍然会进行分片. 只不过lo虛拟网卡的MTU比Ethernet要大很多. 通过`ip link/ifconfig`命令就可以查到, 物理网卡MTU一般为1500, 而lo虚拟接口是65535.

用本机IP(例如192.168.x.y)和用127.0.0.1在性能上有差别吗? 结论是没区别. 其实选用哪个设备是路由相关函数ip_route_output_key_hash_rcu决定的. 在fib_lookup()里会查询到local路由表, 比如`local 192.168.88.236 dev eno1 proto kernel scope host src 192.168.88.236`. 看到eno1不要被它迷惑, 因为**内核在初始化local路由表的时候,把local路由表里所有的路由项都设置成了RTN_LOCAL, 而不只是127.0.0.1**. 这个过程是在设置本机IP的时候,调用[fib_inetaddr_event](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/fib_frontend.c#L1432)->[fib_add_ifaddr](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/fib_frontend.c#L1109)的`fib_magic(RTM_NEWROUTE, RTN_LOCAL, addr, 32, prim, 0);`
里完成设置的.

所以即使本机IP不用127.0.0.1,内核在路由项查找的时候判断类型是RTN_LOCAL,仍然会使用net->loopback_dev,也就是lo虛拟网卡. 该场景可通过在lo上用tcpdump抓包验证.

### 网络设备子系统
网络设备子系统的入口函数是dev_queue_xmit(). 跨机发送过程时, 对于真的有队列的物理设备,该函数进行了一系列复杂的排队等处理后, 才调用dev_hard_start_xmit(), 从这个函数再进入驱动程序来发送. 在这个过程中,甚至还有可能触发
软中断进行发送.

但是对于启动状态的回环设备(q->enqueve 判断为false)来说,就简单多了. 没有队列的问题, 直接进入dev_hard_start _xmit. 接着进入回环设备的“驱动”里发送回调[loopback_xmit()](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/loopback.c#L69), 将skb"发送"出去.

loopback_dev的回调函数集合是[loopback_ops](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/loopback.c#L156)

所以对dev_hard_start_xmit调用实际上执行的是loopback 驱动里的[loopback_xmit()](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/loopback.c#L69).

loopback_xmit()先在skb_orphan()中把skb上的socket指针去掉, 在调用__netif_rx()发送skb.

注意: 在本机网络lo发送的过程中, 传输层下面的skb就不需要释放了, 直接给接收方传过去就行, 是省了一点点开销. 不过传输层的skb是节约不了, 还是要频繁地申请和释放.

__netif_rx->netif_rx_internal()->enqueue_to_backlog(), 在enqueve_to_backog函数中,把要发送的skb插入 softnet_data->input_pkt_queue队列中并调用napi_schedule_rps来触发软中断.

在跨机的网络包的接收过程中,需要经过硬中断,然后才能触发软中断. 而在本机的网络lo过程中, 由于其并不是真的过网卡, 所以网卡的发送过程、硬中断就都省去了,直接从软中断开始.

发送过程触发软中断后,会进入软中断处理函数net_rx_action() ->[napi_poll](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L6635)->__napi_poll()的`work = n->poll(n, weight)`. 对于igb网卡, poll实际调用的是igb_ poll函数, 那么loopback网卡的poll函数是[process_backlog](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L5956), 它是在net_dev_init 中, [struct softnet_data 默认的poll 在初始化的时候设置成了 process_backlog()](https://elixir.bootlin.com/linux/v6.8/source/net/core/dev.c#L11718).

process_backlog的__skb_dequeue()会从sd->process_queue取下来包交给__netif_receive_skb()发往协议栈. 这样和前面发送过程的结尾处就对上了, 发送过程是把包放到了input_pkt_queue队列. 当`__skb_dequeue()`结束后还会通过skb_queue_splice_tail_init()把 sd->input_pkt_queue 里的skb合并到sd->process_queue 链表继续循环处理.

__netif_receive_skb()之后的过程就又和跨机网络IO一致了, 发往协议栈的调用链是_netif_receive_skb ->__netif_receive_skb_core -> deliver_skb, 然后将数据包送入ip_rcv中. 网络层再往后是传输层,最后唤醒用户进程.

### 总结
1. 127.0.0.1的本地网络IO不需要经过网卡
1. 本地网络数据包在内核中是什么走向,和外网发送相比流程上有什么差别?

	总的来说,本机网络IO和跨机网络IO比较起来,确实是节约了驱动上的一些开销. 发送数据不需要进 RingBuffer的驱动队列,直接把skb 传给接收协议栈(经过软中断). 但是在内核其他组件上,可是一点儿都没少,系统调用、协议栈(传输层、网络层等)、设备子系统, 驱动整个走了一遍. 所以即使是本机网络IO,切忌误以为没啥开销就滥用.

	如果想在本机网络IO上绕开协议栈的开销,也不是没有办法,但是要动用eBPF. 使用eBPF的sockmap和sk_redirect 可以达到真正不走协议栈的目的.