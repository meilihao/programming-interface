# pcie
参考:
- [PCIe学习笔记之pcie结构和配置空间](https://www.codenong.com/cs106676528/)

PCI Express，是计算机总线PCI的一种，它沿用现有的PCI编程概念及通信标准，但建基于更快的串行通信系统. 最新版本是5.0, 但6.0已在制定中.

任何厂商生产的设备只要符合PCIE规范, 都能通过PCI插槽和CPU进行通信. 每个PCI设备都有唯一的识别编号, 其内容包括:总线号（bus number）、设备号
（device number）、功能号（function number）. 识别编号可用于在PCIE设备枚举阶段快速识别设备.

PCIe采用的是树形拓扑结构， 它的体系架构一般由root complex，switch，endpoint等类型的PCIe设备组成:
- root complex: 根桥设备，是PCIe最重要的一个组成部件； root complex主要负责PCIe报文的解析和生成.

    RC接受来自CPU的IO指令，生成对应的PCIe报文，或者接受来自设备的PCIe TLP报文，解析数据传输给CPU或者内存.

    如果CPU想读PCIe外设的数据，先叫RC通过TLP把数据从PCIe外设读到Host内存，然后CPU从Host内存读数据；如果CPU要往外设写数据，则先把数据在内存中准备好，然后叫RC通过TLP写入到PCIe设备.
- switch: PCIe的转接器设备，目的是扩展PCIe总线.

    和PCI并行总线不同，PCIe的总线采用了高速差分总线，并采用端到端的连接方式, 因此在每一条PCIe链路中两端只能各连接一个设备， 如果需要挂载更多的PCIe设备，那就需要用到switch转接器. switch在linux下不可见，软件层面可以看到的是switch的上行口（upstream port， 靠近RC的那一侧)和下行口(downstream port).

    一般而言，一个switch 只有一个upstream port， 可以有多个downstream port.

- PCIe endponit: PCIe终端设备，是PCIe树形结构的叶子节点.

    比如网卡，NVME卡，显卡都是PCIe ep设备.

``lspci -tv`命令可以显示pcie的总线拓扑.  一个PCIe设备的ID由以下几个部分组成, 以`0000:00:00.0`为例，分别对应PCI域，总线号，设备号，功能号:
- PCI域：PCI域ID，目的是为了突破pcie256条总线的限制
- PCI总线号：pci设备的总线ID，占用8位，所以PCIe总线最多支持256个子总线
- PCI设备号：指定总线上，pci的设备ID，Device Number占用5位， 所以每个子总线最多支持32个设备
- PCI功能号：指定设备上，pci设备的功能ID， 一个pci 物理设备可以实现多个功能设备，且逻辑功能相互独立，Function Number占用3位，所以每个物理设备最多支持8个功能

BDF（Bus，device，function）构成了每个PCIe设备节点的身份标识.

PCI/pcie有三个相互独立的物理地址空间:(设备寄存器)memory地址空间、I/O地址空间和配置空间. 这三个地址空间都是采用唯一的地址进行寻址.

配置空间是PCI所特有的一个物理空间. 由于PCI支持设备即插即用，所以PCI设备不占用固定的内存地址空间或I/O地址空间，而是由操作系统决定其映射的基址, 这就是配置空间的作用.

PCI总线规范定义的配置空间总长度为256个字节，配置信息按一定的顺序和大小依次存放. 前64个字节的配置空间称为配置头，对于所有的设备都一样，配置头的主要功能是用来识别设备、定义主机访问PCI卡的方式（I/O访问或者存储器访问，还有中断信息）. 其余的192个字节称为本地配置空间（设备有关区），主要定义卡上局部总线的特性、本地空间基地址及范围等.

> 对应pci设备,中断相关信息保存在设备的配置空间里.

> 设备本身是挂载在pcie总线上的, 设备使用的内存地址就是pcie总线可访问的地址, 成为总线地址. 在x86中, 总线地址和内存物理地址相同, 设备直接使用物理地址访问系统内存. 这种方式就是DMA(Direct Memory Access, 直接获取内存). 从设备的配置空间可以发现, 设备的寄存器中有一个可保存DMA地址. 驱动设置该寄存器内容后, 设备就可根据该地址启动DMA, 访问host内存了.

> PCI标准配置空间分type0和type1两种. type0主要是针对PCI的endpoint设备，type1主要是针对PCI bridge(即pcie switch).

> 系统加电时，BIOS检测PCIE总线，确定所有连接在PCIE总线上的设备以及它们的配置要求，并进行系统配置. 所以，所有的PCIE设备必须实现配置空间，从而能够实现参数的自动配置，实现真正的即插即用. CPU要访问该PCIe设备空间，只需访问对应的内存空间. RC检查该内存地址，如果发现该内存空间地址是某个PCIe设备空间的映射，就会触发其产生TLP，去访问对应的PCIe设备，读取或者写入PCIe设备.

> 对Endpoint Configuration（Type 0），提供了最多6个BAR，而对Switch（Type 1）来说，只有2个.

> 一个PCIe设备，可能有若干个内部空间（属性可能不一样，比如有些可预读，有些不可预读）需要映射到内存空间，设备出厂时，这些空间的大小和属性都写在Configuration BAR寄存器里面，然后上电后，系统软件读取这些BAR，分别为其分配对应的系统内存空间，并把相应的内存基地址写回到BAR. BAR的地址其实是PCI总线域的地址，CPU访问的是存储器域的地址，CPU访问PCIe设备时，需要把总线域地址转换成存储器域的地址.

> 当设备注册到总线时会扫描总线查找匹配该设备的驱动; 当驱动注册到总线时也会扫描总线, 查找匹配该驱动的设备.

进入PCIe时代，PCIe能耐更大. 整个配置空间由256 Bytes扩展成4KB，前面256 Bytes保持不变.