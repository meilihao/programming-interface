# receive
ref:
- [coder-kung-fu](https://github.com/yanfeizhang/coder-kung-fu)
- [**Linux 网络栈接收数据（RX）：原理及内核实现（2022）**](https://arthurchiao.art/blog/linux-net-stack-implementation-rx-zh/)
- [**十年码农内功：收包（一）**](https://mp.weixin.qq.com/s?__biz=MzkzMzM5MTUwNQ==&mid=2247484076&idx=1&sn=2a43acaed3e3596bf338784ee5b09c9c&scene=21#wechat_redirect)

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
网卡驱动注册后就可以启动网卡了.

当启用一个网卡时(`ip link set eth0 up/ifconig ethO up`), net_device_ops变量中定义的ndo_open方法会被调用, 即igb_open().

igb_open主要意图:
1. `igb_setup_all_tx_resources(adapter);`: 分配传输描述符数组
1. `igb_setup_all_rx_resources(adapter);`: 分配接收描述符数组

    通过循环创建了若干接收队列, 创建单个队列见[igb_setup_rx_resources()](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L4434).

    igb_setup_rx_resources()创建了RingBuffer, 包括:
    1. igb_rx_bufter数组: 内核使用的, 通过vmalloc申请的 : 预先分配
    1. e1000_adv_rx_desc数组: 网卡硬件使用的,通过dma_alloc_coherent分配 : 先分配
    1. 众多的skb: 动态申请

    查看RingBuffer: `ethtool -g eth0`, Pre-set maximums 指的是RingBuffer的最大值,Current harcware setings指的是当前的设置.  如果`ethtool -S eth0`的rx_missed如果不为O的话(在iconig中体现为overruns指标增⻓)就表示有包因为RingBuffer 装不下而被丢弃了.

    调大RingBuffer()(`ethtool -G eth0 rx 4096 tx 4096`)可解决偶发的瞬时的丢包, 但排队的包过多会增加处理网络包的延时. 建议是加快内核消费RingBuffer中任务的速度.

1. `igb_request_irq(adapter);`: 注册中断处理函数

    在igb_request_msix中可以看到, 对于多队列的网卡, 为每一个队列都注册了硬中断, 其对应的中断处理函数是igb_msix_ring. 在msix方式下,每个Rx队列有独立的MSI-X中断, 从网卡硬件中断的层面就可以设置让收到的包被不同的CPU处理. (可以通过irqbalance或者修改/proc/irq/IRQ_NUMBER/smp_affinity,从而修改和CPU的绑定行为)

    igb_msix_ring作用: 检查有无新数据收到, 如果有, 调用 __napi_schedule()

    > e1000e是e1000_request_irq(), 硬件中断是[e1000_intr_msi](https://elixir.bootlin.com/linux/v6.8/source/drivers/net/ethernet/intel/e1000e/netdev.c#L1750)
1.  `napi_enable(&(adapter->q_vector[i]->napi));`: 启用NAPI

> Rx和Tx队列的数量和大小可以通过ethool进行配置

多队列查看: `ethtool -l eth0`, `Pre-set maximums`'s Combined表示当前网卡支持的最大队列数, `Current harddware settings`'s Combined当前开启的队列数. 或`ls /sys/class/net/eth0/queues`.
多队列修改: `ethtool -L eth0 combined 32`

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
- udp_rcv : from udp_protocol
- icmp_send

## 内核与用户进程的协助
在协议栈接收处理完输入包以后,要能通知到用户进程,让用户进程能够
收到并处理这些数据, 两者的配合有很多种方案, 常见的有:
1. 同步阻塞

    一般在client端使用, 编程友好, 但性能差

    > exmaple: java bio
1. 多路IO复用

    linux的epoll, 编写不直观, 但性能突出, 特别适用于高并发场景

    > exmaple: java nio

### socket
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

        这样后面当内核收完数据产生就绪事件的时候,就可以查找socket等待队列上的等待项,进而可以找到回调⻄数和在等待该socket就绪事件的进程了
    1. 调用sk_wait_event让出CPU,进程将进入睡眠状态,这会导致一次进程上下文的开销,而这个开销是昂贵的,大约需要消耗几个微秒的CPU时间