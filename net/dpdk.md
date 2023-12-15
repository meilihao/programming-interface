# DPDK
ref:
- [DPDK](https://tonydeng.github.io/sdn-handbook/dpdk/)

DPDK由intel支持，DPDK的加速方案原理是完全绕开内核实现的协议栈，把数据包直接从网卡拉到用户态，依靠Intel自身处理器的一些专门优化，来高速处理数据包.

Intel DPDK全称Intel Data Plane Development Kit，是intel提供的数据平面开发工具集，为Intel architecture（IA）处理器架构下用户空间高效的数据包处理提供库函数和驱动的支持，它不同于Linux系统以通用性设计为目的，而是专注于网络应用中数据包的高性能处理. DPDK应用程序是运行在用户空间上利用自身提供的数据平面库来收发数据包，绕过了Linux内核协议栈对数据包处理过程. Linux内核将DPDK应用程序看作是一个普通的用户态进程，包括它的编译、连接和加载方式和普通程序没有什么两样. DPDK程序启动后只能有一个主线程，然后创建一些子线程并绑定到指定CPU核心上运行.

![dpdk arch](/misc/img/net/dpdk/uw20zpb9p1.png)

基本组件:
- EAL（Environment Abstraction Layer）即环境抽象层，为应用提供了一个通用接口，隐藏了与底层库与设备打交道的相关细节。EAL实现了DPDK运行的初始化工作，基于大页表的内存分配，多核亲缘性设置，原子和锁操作，并将PCI设备地址映射到用户空间，方便应用程序访问。
- Buffer Manager API通过预先从EAL上分配固定大小的多个内存对象，避免了在运行过程中动态进行内存分配和回收来提高效率，常常用作数据包buffer来使用。
- Queue Manager API以高效的方式实现了无锁的FIFO环形队列，适合与一个生产者多个消费者、一个消费者多个生产者模型来避免等待，并且支持批量无锁的操作。
- Flow Classification API通过Intel SSE基于多元组实现了高效的hash算法，以便快速的将数据包进行分类处理。该API一般用于路由查找过程中的最长前缀匹配中，安全产品中根据Flow五元组来标记不同用户的场景也可以使用。
- PMD则实现了Intel 1GbE、10GbE和40GbE网卡下基于轮询收发包的工作模式，大大加速网卡收发包性能