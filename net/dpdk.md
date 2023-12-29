# DPDK
ref:
- [DPDK](https://tonydeng.github.io/sdn-handbook/dpdk/)
- [DPDK：v22.03.0原理浅析](https://blog.csdn.net/hhd1988/article/details/124449273)
- [DPDK架构图.pdf](/misc/pdf/net/DPDK架构图.pdf)
- [dpdk_engineer_manual](https://github.com/0voice/dpdk_engineer_manual)

DPDK由intel支持，DPDK的加速方案原理是完全绕开内核实现的协议栈，把数据包直接从网卡拉到用户态，依靠Intel自身处理器的一些专门优化，来高速处理数据包.

Intel DPDK全称Intel Data Plane Development Kit，是intel提供的数据平面开发工具集，为Intel architecture（IA）处理器架构下用户空间高效的数据包处理提供库函数和驱动的支持，它**不同于Linux系统以通用性设计为目的，而是专注于网络应用中数据包的高性能处理**. DPDK应用程序是运行在用户空间上利用自身提供的数据平面库来收发数据包，绕过了Linux内核协议栈对数据包处理过程. Linux内核将DPDK应用程序看作是一个普通的用户态进程，包括它的编译、连接和加载方式和普通程序没有什么两样. DPDK程序启动后只能有一个主线程，然后创建一些子线程并绑定到指定CPU核心上运行.

![dpdk arch](/misc/img/net/dpdk/uw20zpb9p1.png)

基本组件:
- EAL（Environment Abstraction Layer）即环境抽象层，为应用提供了一个通用接口，隐藏了与底层库与设备打交道的相关细节。EAL实现了DPDK运行的初始化工作，基于大页表的内存分配，多核亲缘性设置，原子和锁操作，并将PCI设备地址映射到用户空间，方便应用程序访问。
- Buffer Manager API通过预先从EAL上分配固定大小的多个内存对象，避免了在运行过程中动态进行内存分配和回收来提高效率，常常用作数据包buffer来使用。
- Queue Manager API以高效的方式实现了无锁的FIFO环形队列，适合与一个生产者多个消费者、一个消费者多个生产者模型来避免等待，并且支持批量无锁的操作。
- Flow Classification API通过Intel SSE基于多元组实现了高效的hash算法，以便快速的将数据包进行分类处理。该API一般用于路由查找过程中的最长前缀匹配中，安全产品中根据Flow五元组来标记不同用户的场景也可以使用。
- PMD则实现了Intel 1GbE、10GbE和40GbE网卡下基于轮询收发包的工作模式，大大加速网卡收发包性能

## 注意
- [DPDK-22.11.2 [二] 使用建议和注意事项](https://www.cnblogs.com/studywithallofyou/p/17633727.html)

	建议使用vfio-pci，依赖系统的vfio

	igb_uio从DPDK v20.02开始禁止编译。可以通过CONFIG_RTE_EAL_IGB_UIO打开编译。igb_uio计划迁移到其他项目。

	uio_pci_generic是linux系统提供的，不支持virtual function (VF)。

	如果想支持virtual function (VF)，请使用igb_uio，依赖系统的uio。

	由于igb_uio不安全，提供了vfio，更安全，功能更多。

	如果BIOS开启了UEFI，就无法使用UIO

## FAQ
### Run to Completion 和 Pipeline 两种报文处理模式的区别
DPDK 支持 Run to Completion 和 Pipeline 两种报文处理模式，用户可
以依据需求灵活选择，或者混合使用。

Run to Completion 是一种水平调度方式，利用网卡的多队列，将报文分发给多个 CPU 核处理，每个核均独立处理到达该队列的报文，资源分配相对固定，减少了报文在核间的传递开销，可以随着核的数
目灵活扩展处理能力.

Pipeline 模式则通过共享环在核间传递数据报文或消息，将系统处理任务分解到不同的 CPU 核上处理，通过任务分发来减少处理等待时延.

### hugepage对普通的 page(4K)的优势
hugepage 相对于普通的 page(4K)来说有以下几个个特点:
- 1. hugepage 页面不受虚拟内存管理影响，不会被替换出内存，而普通的 4kpage
有换出问题
- 2. 同样的内存大小， hugepage 产生的页表项数目远少于 4kpage

	如：用户进程需要使用 4M 大小的内存， 若采用 4Kpage， 需要 1K 的页表项存放虚拟地址到物理地址的映射关系， 而采用 hugepag 则只产生 2 条页表项。 这样会带来两个好处：
	1. 是使用 hugepage 的内存产生的页表比较少，这对于数据库系统等动不动就需要映射非常大的数据到进程的应用来说， 可以有效减少页表的开销，所以很多数据库系统都采用 hugepage 技术
	1. 是 TLB 冲突率会大大减少， TLB 驻留在 cpu 的 1 级 cache 里，是芯片访问最快的缓存，一般只能容纳 100 多条页表项，如果采用 hugepage，则可以极大减少 TLB miss 导致的开销（以 64 位为例）：

		- TLB 命中，立即就获取到物理地址，这个命中率会比 4Kpage 要高很多
		- TLB 不命中， 传统 4Kpage 需要查 CR3->PML4E->PDPTE->PDE->PTE->物理内存，如果这页框被虚拟内存系统替换到交互区，则还需要交互区 load 回内存，而 hugepage 则可以忽略这一步。 总之， TLB miss是性能大杀手，而采用 hugepage 可以有效降低 tlb miss