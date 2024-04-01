# pcie
PCI Express，是计算机总线PCI的一种，它沿用现有的PCI编程概念及通信标准，但建基于更快的串行通信系统. 最新版本是6.0, 但7.0已在制定中.

任何厂商生产的设备只要符合PCIE规范, 都能通过PCI插槽和CPU进行通信. 每个PCI设备都有唯一的识别编号, 其内容包括:总线号（bus number）、设备号
（device number）、功能号（function number）. 识别编号可用于在PCIE设备枚举阶段快速识别设备. 

`lspci -tv`命令可以显示pcie的总线拓扑.  一个PCIe设备的ID由以下几个部分组成, 以`0000:00:00.0`为例，分别对应PCI域，总线号，设备号，功能号:
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

## CXL
ref:
- [关于CXL，你想知道的都在这里](https://xilinx.eetrend.com/content/2023/100568963.html)
- [从PCIe 5.0到CXL：共存 or 取代？](https://www.doit.com.cn/p/480458.html)

CXL是行业支持的处理器、内存扩展和加速器的Cache-Coherent互连，该技术保持CPU内存空间和附加设备上内存的一致性，允许资源共享，从而获得更高的性能，降低软件栈的复杂性，降低整体系统成本，用户也借此摆脱加速器中的冗余内存管理硬件带来的困扰，将更多精力转向目标工作负载.

让CPU与GPU、FPGA或其他加速器之间实现高速高效的互联，这就是英特尔推出CXL这种新的开放性互联协议的初衷. 英特尔原计划以CXL来取代PCIe, 但由于CXL构建于PCIe逻辑和物理层级之上. 因此, CXL作为PCIe物理层之上运行的一种可选协议仍将与PCIe共存一段时间, 英特尔也计划在今年初推出的PCIe 6.0规范上大力推进CXL的采用.

CXL 联盟已经确定了将采用新互连的三类主要设备：
1. 类型1设备：智能 NIC 等加速器通常缺少本地内存。通过 CXL，这些设备可以与主机处理器的 DDR 内存进行通信。
1. 类型2设备：GPU、ASIC 和 FPGA 都配备了 DDR 或 HBM 内存，并且可以使用 CXL 使主机处理器的内存在本地可供加速器使用，并使加速器的内存在本地可供 CPU 使用。它们还共同位于同一个缓存一致域中，有助于提升异构工作负载。
1. 类型 3 设备：内存设备可以通过 CXL 连接，为主机处理器提供额外的带宽和容量。内存的类型独立于主机的主内存。

CXL 标准通过三种协议支持各种用例：CXL.io、CXL.cache 和 CXL.memory。
- CXL.io：该协议在功能上等同于 PCIe 协议，并利用了 PCIe 的广泛行业采用和熟悉度。作为基础通信协议，CXL.io 用途广泛，适用于广泛的用例。
- CXL.cache：该协议专为更具体的应用程序而设计，使加速器能够有效地访问和缓存主机内存以优化性能。
- CXL.memory：该协议使主机（例如处理器）能够使用加载/存储命令访问设备连接的内存。


这三个协议共同促进了计算设备（例如 CPU 主机和 AI 加速器）之间内存资源的一致共享。从本质上讲，这通过共享内存实现通信简化了编程。用于设备和主机互连的协议如下：
1. 类型 1 设备：CXL.io + CXL.cache
1. 类型2设备：CXL.io + CXL.cache + CXL.memory
1. 类型 3 设备：CXL.io + CXL.memory


## 组成
ref:
- [【原创】Linux PCI驱动框架分析（一） ](https://www.cnblogs.com/LoyenWang/p/14165852.html)
- [PCIe学习笔记之pcie结构和配置空间](https://www.codenong.com/cs106676528/)

![](/misc/img/driver/pcie/1771657-20201220230203790-720754449.png)

PCIe采用的是树形拓扑结构， 它的体系架构一般由root complex，switch，endpoint等类型的PCIe设备组成:
- Root Complex：根桥设备, 主要负责PCIe报文的解析和生成. CPU和PCIe总线之间的接口可能会包含几个模块（处理器接口、DRAM接口等），甚至可能还会包含芯片，这个集合就称为Root Complex，它作为PCIe架构的根，代表CPU与系统其它部分进行交互. 广义来说，Root Complex可以认为是CPU和PCIe拓扑之间的接口，Root Complex会将CPU的request转换成PCIe的4种不同的请求（Configuration、Memory、I/O、Message）

	RC接受来自CPU的IO指令，生成对应的PCIe报文，或者接受来自设备的PCIe TLP报文，解析数据传输给CPU或者内存.

    如果CPU想读PCIe外设的数据，先叫RC通过TLP把数据从PCIe外设读到Host内存，然后CPU从Host内存读数据；如果CPU要往外设写数据，则先把数据在内存中准备好，然后叫RC通过TLP写入到PCIe设备.

    实现:
	- Root Complex通常会实现一个内部总线结构和多个桥，从而扇出到多个端口上
	- Root Complex的内部实现不需要遵循标准，因此都是厂家specific的

	> PCIe在软件上保持了后向兼容性，那么在PCIe的设计上，需要考虑在PCI总线上的软件视角，比如Root Complex的实现可能就是`Host Bridge+N*PCI-PCI Bridge`，从而看起来与PCI总线相差无异
- Switch： PCIe的转接器设备，目的是扩展PCIe总线. 从图中看出Swtich提供扇出能力，让更多的PCIe设备连接在PCIe端口上

	和PCI并行总线不同，PCIe的总线采用了高速差分总线，并采用端到端的连接方式, 因此在每一条PCIe链路中两端只能各连接一个设备， 如果需要挂载更多的PCIe设备，那就需要用到switch转接器. switch在linux下不可见，软件层面可以看到的是switch的上行口（upstream port， 靠近RC的那一侧)和下行口(downstream port).

    一般而言，一个switch 只有一个upstream port， 可以有多个downstream port.

    > 上图中白色的小方块代表Downstream端口，灰色的小方块代表Upstream端口.

    > switch看起来就是多个PCI-PCI Bridge的连接路由
- Bridge：桥接设备，用于去连接其他的总线，比如PCI总线或PCI-X总线，甚至另外的PCIe总线
- PCIe Endpoint：PCIe终端设备，是PCIe树形结构的叶子节点

	比如网卡，NVME卡，显卡都是PCIe ep设备.

## PCIe数据传输
- 与PCI总线不同（PCI设备共享总线），PCIe总线使用端到端的连接方式，互为接收端和发送端，全双工，基于数据包的传输
- 物理底层采用差分信号（PCI链路采用并行总线，而PCIe链路采用串行总线），一条Lane中有两组差分信号，共四根信号线，而PCIe Link可以由多条Lane组成，可以支持1、2、4、8、12、16、32条

![](/misc/img/driver/pcie/1771657-20201220230235825-1305673998.png)
PCIe规范定义了分层的架构设计，包含三层:
- Transaction层

	负责TLP包（Transaction Layer Packet）的封装与解封装，此外还负责QoS，流控、排序等功能
- Data Link层

	负责DLLP包（Data Link Layer Packet）的封装与解封装，此外还负责链接错误检测和校正，使用Ack/Nak协议来确保传输可靠
- Physical层

	负责Ordered-Set包的封装与解封装，物理层处理TLPs、DLLPs、Ordered-Set三种类型的包传输

数据包的封装与解封装，与网络包的创建与解析很类似:
![](/misc/img/driver/pcie/1771657-20201220230245485-837598636.png)

更详细的PCIe分层图:
![](/misc/img/driver/pcie/1771657-20201220230255134-36369532.png)

## PCIe设备的配置空间
为了兼容PCI软件，PCIe保留了256Byte的配置空间:
![](/misc/img/driver/pcie/1771657-20201220230302764-343994745.png)

此外，在这个基础上将配置空间扩展到了4KB，还进行了功能的扩展，比如Capability、Power Management、MSI中断等:
![](/misc/img/driver/pcie/1771657-20201220230309811-220551169.png)

扩展后的区域将使用MMIO的方式进行访问

## 代码
在`arch/x86/pci`和`drivers/pci`下

![](/misc/img/driver/pcie/1771657-20201229232444671-1127290866.png)
![](/misc/img/driver/pcie/1771657-20201229232452417-1541535796.png)

```c
// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/pci.h#L647
struct pci_bus {
	struct list_head node;		/* Node in list of buses */ // 对于根pci总线, 是链接到全局根总线链表pci_root_buses的连接件; 对于非根pci总线, 是链接到其父总线链表children的连接件 
	struct pci_bus	*parent;	/* Parent bus this bridge is on */ // 指向该pci总线的父总线
	struct list_head children;	/* List of child buses */ // 这条pci总线的子总线链表的表头
	struct list_head devices;	/* List of devices on this bus */ // 这条pci总线的pci设备链表的表头
	struct pci_dev	*self;		/* Bridge device as seen by parent */ // 对于根pci总线是NULL; 对于非根pci总线是指向引出这条pci总线的桥设备; 
	struct list_head slots;		/* List of slots on this bus;
					   protected by pci_slot_mutex */ // 这条pci总线的插槽链表的表头
	struct resource *resource[PCI_BRIDGE_RESOURCE_NUM]; // 对于根pci总线, 指向ioport_resource(代表I/O端口)或iomem_resource(I/O内存); 对于非根pci总线, 指向引出这条总线的桥设备的代表资源窗口的resource数组
	struct list_head resources;	/* Address space routed to this bus */ // 如果pci总线指向更多的资源, 则需链入以此字段为表头的链表
	struct resource busn_res;	/* Bus numbers routed to this bus */

	struct pci_ops	*ops;		/* Configuration access functions */ // 这条pci总线所使用的配置空间访问函数[pci_root_ops](https://elixir.bootlin.com/linux/v6.6.14/source/arch/x86/pci/common.c#L72). 在pci总线扫描时赋值给根总线, 再传递给非根总线
	void		*sysdata;	/* Hook for sys-specific extension */ // 系统级扩展数据钩子, 用来记录所在根总线特有的信息, 这些信息通常是与arch相关, 比如pci域, numa节点和iommu等
	struct proc_dir_entry *procdir;	/* Directory entry in /proc/bus/pci */ // 这条pci总线在/proc/bus/pui中的目录项

	unsigned char	number;		/* Bus number */ // 总线编号
	unsigned char	primary;	/* Number of primary bridge */ // 指向引出这条pci总线的桥设备的primary编号
	unsigned char	max_bus_speed;	/* enum pci_bus_speed */ // 最大总线速度
	unsigned char	cur_bus_speed;	/* enum pci_bus_speed */ // 当前总线速度
#ifdef CONFIG_PCI_DOMAINS_GENERIC
	int		domain_nr;
#endif

	char		name[48]; // 总线名称, 格式`PCI Bus #<%02x>`, `%02x`是总线号

	unsigned short	bridge_ctl;	/* Manage NO_ISA/FBB/et al behaviors */ // 引出这条pci总线的桥设备的桥控制寄存器, 即其配置寄存器的PCI_BRUDGE_CONTROL域
	pci_bus_flags_t bus_flags;	/* Inherited by child buses */ // 总线标志. 用途标记引出这条总线的主桥设备在设计上的一些缺陷, 且被各级子总线继承
	struct device		*bridge; // 对于根总线, 指向新创建的虚拟device; 对于非根总线, 指向引出这条pci总线的桥设备内嵌device
	struct device		dev; // 内嵌的类设备对象. pci总线通过这个字段链入pci总线类的设备链表
	struct bin_attribute	*legacy_io;	/* Legacy I/O for this bus */ // 用于在sysfs中为该总线生成legacy I/O属性文件
	struct bin_attribute	*legacy_mem;	/* Legacy mem */ // 用于在sysfs中为该总线生成legacy mem属性文件
	unsigned int		is_added:1; // 1, 表示设备已添加到sysfs
	unsigned int		unsafe_warn:1;	/* warned about RW1C config write */
};


// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/pci.h#L322
// pci_dev中很多字段是pci设备的配置空间内容的副本, 它们在pci子系统初始化过程中被读出, 如果发生修改, 需要将修改后的数据更新到pci设备的配置空间
/* The pci_dev structure describes PCI devices */
struct pci_dev {
	struct list_head bus_list;	/* Node in per-bus list */ // 链接到其所属pci总线的devices链表的连接件
	struct pci_bus	*bus;		/* Bus this device is on */ // 指向这个pci设备所在的pci总线
	struct pci_bus	*subordinate;	/* Bus this device bridges to */ // 指向这个pci设备所桥接的下级总线, 仅对桥设备有效

	void		*sysdata;	/* Hook for sys-specific extension */ // 系统级扩展数据钩子, 用来记录所在根总线特有的信息, 这些信息通常是与arch相关, 比如pci域, numa节点和iommu等
	struct proc_dir_entry *procent;	/* Device entry in /proc/bus/pci */ // 这个pci设备在/proc/bus/pci中的目录项
	struct pci_slot	*slot;		/* Physical slot this device is in */ // 指向这个设备所在的物理插槽

	unsigned int	devfn;		/* Encoded device & function index */ // pci设备的设备功能号, 即pci逻辑设备号, 是将高5位为插槽号, 低3位为功能号一起编码的结果, 它们是在pci总线扫描过程中配置的
	unsigned short	vendor; // 从pci设备配置寄存器中的vendor id
	unsigned short	device; // 从pci设备配置寄存器中的device id
	unsigned short	subsystem_vendor; // 对应标准pci设备配置寄存器中的子系统vendor id
	unsigned short	subsystem_device; // 对应标准pci设备配置寄存器中的子系统device id
	unsigned int	class;		/* 3 bytes: (base,sub,prog-if) */ // 从pci设备配置寄存器中的class code
	u8		revision;	/* PCI revision, low byte of class word */ // 从pci设备配置寄存器中的修正号id
	u8		hdr_type;	/* PCI header type (`multi' flag masked out) */ // 从pci设备配置寄存器中的头类型域的低7位, 第8位多功能标志已被屏蔽(以用multifunction表示): 00h, 标准的pci设备, 01h是pci-pci桥设备
#ifdef CONFIG_PCIEAER
	u16		aer_cap;	/* AER capability offset */
	struct aer_stats *aer_stats;	/* AER stats for this device */
#endif
#ifdef CONFIG_PCIEPORTBUS
	struct rcec_ea	*rcec_ea;	/* RCEC cached endpoint association */
	struct pci_dev  *rcec;          /* Associated RCEC device */
#endif
	u32		devcap;		/* PCIe Device Capabilities */
	u8		pcie_cap;	/* PCIe capability offset */ // pcie能力偏移
	u8		msi_cap;	/* MSI capability offset */
	u8		msix_cap;	/* MSI-X capability offset */
	u8		pcie_mpss:3;	/* PCIe Max Payload Size Supported */
	u8		rom_base_reg;	/* Config register controlling ROM */ // rom基地址寄存器在pce配置空间中的位置
	u8		pin;		/* Interrupt pin this device uses */ // 从pci设备配置寄存器中的读出的中断引脚
	u16		pcie_flags_reg;	/* Cached PCIe Capabilities Register */
	unsigned long	*dma_alias_mask;/* Mask of enabled devfn aliases */

	struct pci_driver *driver;	/* Driver bound to this device */ // 指向这个pci设备所关联的pci_driver
	u64		dma_mask;	/* Mask of the bits of bus address this
					   device implements.  Normally this is
					   0xffffffff.  You only need to change
					   this if your device has broken DMA
					   or supports 64-bit transfers.  */ // pci设备的DMA掩码, 即它实现的总线地址位数. 内核默认是32位, 其他需要调用dma_set_mask等函数修改

	struct device_dma_parameters dma_parms; // 这个pci设备的dma参数(包括最大segment长度, 以及segment边界掩码等)

	pci_power_t	current_state;	/* Current operating state. In ACPI,
					   this is D0-D3, D0 being fully
					   functional, and D3 being off. */ // 当前电源工作状态. 用ACPI的术语D0-D3: D0, 完全工作; D3, 断电
	u8		pm_cap;		/* PM capability offset */ // 电源管理能力在配置空间中的偏移
	unsigned int	imm_ready:1;	/* Supports Immediate Readiness */
	unsigned int	pme_support:5;	/* Bitmask of states from which PME#
					   can be generated */ // 电源管理事件(Power Management Event)状态掩码
	unsigned int	pme_poll:1;	/* Poll device's PME status bit */ // 1, 启用了电源管理事件中断
	unsigned int	d1_support:1;	/* Low power state D1 is supported */ // 1, 支持节能状态D1, 从pcie空间的电源管理能力寄存器中读取
	unsigned int	d2_support:1;	/* Low power state D2 is supported */ // 1, 支持节能状态D2, 从pcie空间的电源管理能力寄存器中读取
	unsigned int	no_d1d2:1;	/* D1 and D2 are forbidden */ // 1, 设备在电源管理方面只允许节能状态D0或D3
	unsigned int	no_d3cold:1;	/* D3cold is forbidden */
	unsigned int	bridge_d3:1;	/* Allow D3 for bridge */
	unsigned int	d3cold_allowed:1;	/* D3cold is allowed by user */
	unsigned int	mmio_always_on:1;	/* Disallow turning off io/mem
						   decoding during BAR sizing */
	unsigned int	wakeup_prepared:1; // 1, 可以将该pci设备作为唤醒事件源
	unsigned int	skip_bus_pm:1;	/* Internal: Skip bus-level PM */
	unsigned int	ignore_hotplug:1;	/* Ignore hotplug events */
	unsigned int	hotplug_user_indicators:1; /* SlotCtl indicators
						      controlled exclusively by
						      user sysfs */
	unsigned int	clear_retrain_link:1;	/* Need to clear Retrain Link
						   bit manually */
	unsigned int	d3hot_delay;	/* D3hot->D0 transition time in ms */
	unsigned int	d3cold_delay;	/* D3cold->D0 transition time in ms */

#ifdef CONFIG_PCIEASPM
	struct pcie_link_state	*link_state;	/* ASPM link state */ // 链接状态
	u16		l1ss;		/* L1SS Capability pointer */
	unsigned int	ltr_path:1;	/* Latency Tolerance Reporting
					   supported from root to here */
#endif
	unsigned int	pasid_no_tlp:1;		/* PASID works without TLP Prefix */
	unsigned int	eetlp_prefix_path:1;	/* End-to-End TLP Prefix */

	pci_channel_state_t error_state;	/* Current connectivity state */ // 当前连接状态
	struct device	dev;			/* Generic device interface */ // 内嵌的通用设备对象, pci设备通过它链入pci总线类型(pci_bus_type)的设备链表

	int		cfg_size;		/* Size of config space */ // 配置空间的长度, 常规pci设备是256, pci-x2和pcie是4096

	/*
	 * Instead of touching interrupt line and base address registers
	 * directly, use the values stored here. They might be different!
	 */
	unsigned int	irq; // 这个pci设备通过哪根irq输入线产生中断, 一般是0~15
	struct resource resource[DEVICE_COUNT_RESOURCE]; /* I/O and memory regions + expansion ROMs */ // 这个设备的资源数组. 对于pci设备: 0~5, I/O或内存区间; 6, 扩展ROM空间; 对于桥设备, 0~1, 资源区间; 2~5, I/O或内存区间; 6, 扩展ROM空间, 从PCI_BRIDGE_RESOURCES开始的PCI_BRIDGE_RESOURCES_END表示资源窗口
	struct resource driver_exclusive_resource;	 /* driver exclusive resource ranges */

	bool		match_driver;		/* Skip attaching driver */

	unsigned int	transparent:1;		/* Subtractive decode bridge */ // 1, 是透明pic桥
	unsigned int	io_window:1;		/* Bridge has I/O window */
	unsigned int	pref_window:1;		/* Bridge has pref mem window */
	unsigned int	pref_64_window:1;	/* Pref mem window is 64-bit */
	unsigned int	multifunction:1;	/* Multi-function device */ // 1, 是多功能设备的一部分

	unsigned int	is_busmaster:1;		/* Is busmaster */ // 1, pci设备是总线主控设备
	unsigned int	no_msi:1;		/* May not use MSI */ // 1, 不可以使用MSI
	unsigned int	no_64bit_msi:1;		/* May only use 32-bit MSIs */
	unsigned int	block_cfg_access:1;	/* Config space access blocked */ // 1, 对配置空间的访问被阻塞
	unsigned int	broken_parity_status:1;	/* Generates false positive parity */ // 1, pci设备有校验和问题
	unsigned int	irq_reroute_variant:2;	/* Needs IRQ rerouting variant */ // 某些芯片需要将linux kernel的原始中断重新映射到它的引导中断上
	unsigned int	msi_enabled:1; // 1, 启用MSI中断
	unsigned int	msix_enabled:1; // 1, 启用MSIX中断
	unsigned int	ari_enabled:1;		/* ARI forwarding */ // 1, 支持ARI(alternative routing-id interpretation)能力, 是pcie特性, 允许一个pcie部件支持对于8个功能
	unsigned int	ats_enabled:1;		/* Address Translation Svc */
	unsigned int	pasid_enabled:1;	/* Process Address Space ID */
	unsigned int	pri_enabled:1;		/* Page Request Interface */
	unsigned int	is_managed:1;		/* Managed via devres */ // 1, 该pci设备是可管理的, 即其设备资源释放将自动完成. 如果调用pcim_enable_device, 而不是pci_enable_device, 此标记会被设置. 设备资源链表被用于记录为设备申请过的资源, 在驱动卸载或设备禁用时统一释放
	unsigned int	is_msi_managed:1;	/* MSI release via devres installed */
	unsigned int	needs_freset:1;		/* Requires fundamental reset */ // 1, 设备需要基本复位(fundamental reset). pcie适配器的基本复位表示它的设备状态机, 硬件逻辑, 端口状态和配置寄存器等都恢复到初始化状态. 某些设备只能通过基本复位才能从错误中恢复
	unsigned int	state_saved:1; // 1, pci设备的状态已被保存
	unsigned int	is_physfn:1; // 1, 是pf设备
	unsigned int	is_virtfn:1; // 1, 是vf设备
	unsigned int	is_hotplug_bridge:1; // 1, 支持热插拔的桥设备
	unsigned int	shpc_managed:1;		/* SHPC owned by shpchp */
	unsigned int	is_thunderbolt:1;	/* Thunderbolt controller */
	/*
	 * Devices marked being untrusted are the ones that can potentially
	 * execute DMA attacks and similar. They are typically connected
	 * through external ports such as Thunderbolt but not limited to
	 * that. When an IOMMU is enabled they should be getting full
	 * mappings to make sure they cannot access arbitrary memory.
	 */
	unsigned int	untrusted:1;
	/*
	 * Info from the platform, e.g., ACPI or device tree, may mark a
	 * device as "external-facing".  An external-facing device is
	 * itself internal but devices downstream from it are external.
	 */
	unsigned int	external_facing:1;
	unsigned int	broken_intx_masking:1;	/* INTx masking can't be used */
	unsigned int	io_window_1k:1;		/* Intel bridge 1K I/O windows */
	unsigned int	irq_managed:1;
	unsigned int	non_compliant_bars:1;	/* Broken BARs; ignore them */
	unsigned int	is_probed:1;		/* Device probing in progress */
	unsigned int	link_active_reporting:1;/* Device capable of reporting link active */
	unsigned int	no_vf_scan:1;		/* Don't scan for VFs after IOV enablement */
	unsigned int	no_command_memory:1;	/* No PCI_COMMAND_MEMORY */
	unsigned int	rom_bar_overlap:1;	/* ROM BAR disable broken */
	unsigned int	rom_attr_enabled:1;	/* Display of ROM attribute enabled? */ // 1, 允许显示rom属性
	pci_dev_flags_t dev_flags; // 设备标志. 用来标记引出这个pci设备在设计上的一些缺陷. 在pci子系统初始化时记录以备用
	atomic_t	enable_cnt;	/* pci_enable_device has been called */ // pci_enable_device被调用的次数

	spinlock_t	pcie_cap_lock;		/* Protects RMW ops in capability accessors */
	u32		saved_config_space[16]; /* Config space saved at suspend time */ // 设备被挂起时, 保存其配置空间
	struct hlist_head saved_cap_space; // 设备被挂起时, 保存其能力列表
	struct bin_attribute *res_attr[DEVICE_COUNT_RESOURCE]; /* sysfs file for resources */ // 用于在sysfs中为该pci设备生成资源属性文件
	struct bin_attribute *res_attr_wc[DEVICE_COUNT_RESOURCE]; /* sysfs file for WC mapping of resources */ // 用于在sysfs中为该pci设备生成用于WC映射资源的sysfs文件

#ifdef CONFIG_HOTPLUG_PCI_PCIE
	unsigned int	broken_cmd_compl:1;	/* No compl for some cmds */
#endif
#ifdef CONFIG_PCIE_PTM
	u16		ptm_cap;		/* PTM Capability */
	unsigned int	ptm_root:1;
	unsigned int	ptm_enabled:1;
	u8		ptm_granularity;
#endif
#ifdef CONFIG_PCI_MSI
	void __iomem	*msix_base;
	raw_spinlock_t	msi_lock;
#endif
	struct pci_vpd	vpd; // 指向pci产商产品数据描述符
#ifdef CONFIG_PCIE_DPC
	u16		dpc_cap;
	unsigned int	dpc_rp_extensions:1;
	u8		dpc_rp_log_size;
#endif
#ifdef CONFIG_PCI_ATS
	union {
		struct pci_sriov	*sriov;		/* PF: SR-IOV info */
		struct pci_dev		*physfn;	/* VF: related PF */
	};
	u16		ats_cap;	/* ATS Capability offset */
	u8		ats_stu;	/* ATS Smallest Translation Unit */
#endif
#ifdef CONFIG_PCI_PRI
	u16		pri_cap;	/* PRI Capability offset */
	u32		pri_reqs_alloc; /* Number of PRI requests allocated */
	unsigned int	pasid_required:1; /* PRG Response PASID Required */
#endif
#ifdef CONFIG_PCI_PASID
	u16		pasid_cap;	/* PASID Capability offset */
	u16		pasid_features;
#endif
#ifdef CONFIG_PCI_P2PDMA
	struct pci_p2pdma __rcu *p2pdma;
#endif
#ifdef CONFIG_PCI_DOE
	struct xarray	doe_mbs;	/* Data Object Exchange mailboxes */
#endif
	u16		acs_cap;	/* ACS Capability offset */
	phys_addr_t	rom;		/* Physical address if not from BAR */
	size_t		romlen;		/* Length if not from BAR */
	/*
	 * Driver name to force a match.  Do not set directly, because core
	 * frees it.  Use driver_set_override() to set or clear it.
	 */
	const char	*driver_override;

	unsigned long	priv_flags;	/* Private flags for the PCI driver */

	/* These methods index pci_reset_fn_methods[] */
	u8 reset_methods[PCI_NUM_RESET_METHODS]; /* In priority order */
};

// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/pci.h#L76
/* pci_slot represents a physical slot */
struct pci_slot {
	struct pci_bus		*bus;		/* Bus this slot is on */ // 指向这个slot所在的pci总线
	struct list_head	list;		/* Node in list of slots */ // 链入所在pci总线的插槽链表的连接件
	struct hotplug_slot	*hotplug;	/* Hotplug info (move here) */ // 热插拔信息
	unsigned char		number;		/* PCI_SLOT(pci_dev->devfn) */ // 插槽编号, 即pci_dev的devfn的高5位. 判断pci设备和pci插槽是否有关联, 需要看它们的功能/插槽号的高5位是否一样
	struct kobject		kobj;
};

// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/pci.h#L918
/**
 * struct pci_driver - PCI driver structure
 * @node:	List of driver structures.
 * @name:	Driver name.
 * @id_table:	Pointer to table of device IDs the driver is
 *		interested in.  Most drivers should export this
 *		table using MODULE_DEVICE_TABLE(pci,...).
 * @probe:	This probing function gets called (during execution
 *		of pci_register_driver() for already existing
 *		devices or later if a new device gets inserted) for
 *		all PCI devices which match the ID table and are not
 *		"owned" by the other drivers yet. This function gets
 *		passed a "struct pci_dev \*" for each device whose
 *		entry in the ID table matches the device. The probe
 *		function returns zero when the driver chooses to
 *		take "ownership" of the device or an error code
 *		(negative number) otherwise.
 *		The probe function always gets called from process
 *		context, so it can sleep.
 * @remove:	The remove() function gets called whenever a device
 *		being handled by this driver is removed (either during
 *		deregistration of the driver or when it's manually
 *		pulled out of a hot-pluggable slot).
 *		The remove function always gets called from process
 *		context, so it can sleep.
 * @suspend:	Put device into low power state.
 * @resume:	Wake device from low power state.
 *		(Please see Documentation/power/pci.rst for descriptions
 *		of PCI Power Management and the related functions.)
 * @shutdown:	Hook into reboot_notifier_list (kernel/sys.c).
 *		Intended to stop any idling DMA operations.
 *		Useful for enabling wake-on-lan (NIC) or changing
 *		the power state of a device before reboot.
 *		e.g. drivers/net/e100.c.
 * @sriov_configure: Optional driver callback to allow configuration of
 *		number of VFs to enable via sysfs "sriov_numvfs" file.
 * @sriov_set_msix_vec_count: PF Driver callback to change number of MSI-X
 *              vectors on a VF. Triggered via sysfs "sriov_vf_msix_count".
 *              This will change MSI-X Table Size in the VF Message Control
 *              registers.
 * @sriov_get_vf_total_msix: PF driver callback to get the total number of
 *              MSI-X vectors available for distribution to the VFs.
 * @err_handler: See Documentation/PCI/pci-error-recovery.rst
 * @groups:	Sysfs attribute groups.
 * @dev_groups: Attributes attached to the device that will be
 *              created once it is bound to the driver.
 * @driver:	Driver model structure.
 * @dynids:	List of dynamically added device IDs.
 * @driver_managed_dma: Device driver doesn't use kernel DMA API for DMA.
 *		For most device drivers, no need to care about this flag
 *		as long as all DMAs are handled through the kernel DMA API.
 *		For some special ones, for example VFIO drivers, they know
 *		how to manage the DMA themselves and set this flag so that
 *		the IOMMU layer will allow them to setup and manage their
 *		own I/O address space.
 */
struct pci_driver {
	struct list_head	node; // 将pci驱动链入某个链表的连接件
	const char		*name; // 驱动名称
	const struct pci_device_id *id_table;	/* Must be non-NULL for probe to be called */ // 驱动支持的(静态) ID列表, 会被probe调用
	int  (*probe)(struct pci_dev *dev, const struct pci_device_id *id);	/* New device inserted */ // 探测回调函数, 在新设备插入时调用
	void (*remove)(struct pci_dev *dev);	/* Device removed (NULL if not a hot-plug capable driver) */ // 在设备移除时调用
	int  (*suspend)(struct pci_dev *dev, pm_message_t state);	/* Device suspended */ // pci电源管理相关函数
	int  (*resume)(struct pci_dev *dev);	/* Device woken up */ // pci电源管理相关函数
	void (*shutdown)(struct pci_dev *dev); // 在设备被断电时调用
	int  (*sriov_configure)(struct pci_dev *dev, int num_vfs); /* On PF */
	int  (*sriov_set_msix_vec_count)(struct pci_dev *vf, int msix_vec_count); /* On PF */
	u32  (*sriov_get_vf_total_msix)(struct pci_dev *pf);
	const struct pci_error_handlers *err_handler; // 实现pci错误恢复的回调函数
	const struct attribute_group **groups;
	const struct attribute_group **dev_groups;
	struct device_driver	driver;
	struct pci_dynids	dynids; // 驱动支持的(动态) ID链表, 是通过sysfs动态添加的
	bool driver_managed_dma;
};
```


pci工作模式: pci核心注册pci总线类型, 扫描pci总线得到所有的pci设备. pci hba驱动注册pci设备驱动, 逐个和pci设备根据设备ID匹配.


- 顶层的结构为pci_host_bridge，这个结构一般由**Host驱动**负责来初始化创建. pci_host_bridge指向root bus，也就是编号为0的总线，在该总线下，可以挂接各种外设或物理slot，也可以通过PCI桥去扩展总线
- struct pci_dev描述PCI设备，以及PCI-to-PCI桥设备

	pci_dev描述的是pci逻辑设备, 因此一个pci主机适配器, 可能存在多个pci_dev
- struct pci_bus用于描述PCI总线
- struct pci_slot用于描述总线上的物理插槽

根据根目录下makefile, `arch/x86/pci`在`drivers/pci`前指向; 通过`drivers/pci`的makefile, drivers/pci/probe.c在drivers/pci/pci-driver.c前执行, pci子系统的初始化顺序是:
1. [pcibus_class_init](https://elixir.bootlin.com/linux/v6.6.14/source/drivers/pci/probe.c#L104):初始化总线类

	`class_register(&pcibus_class)`即在`/sys/class`下创建pci_bus目录, 后续的pci核心代码将在该目录下为每条pci总线创建一个子目录(格式:`<%04d总线域编号>:<%02d:总线编号>`)
1. [pci_driver_init](https://elixir.bootlin.com/linux/v6.6.14/source/drivers/pci/pci-driver.c#L1723):初始化总线类型

	在内核里边创建一个PCI总线，用于挂接PCI设备和PCI驱动.
1. [pci_arch_init](https://elixir.bootlin.com/linux/v6.6.14/source/arch/x86/pci/init.c#L10):配置对访问空间的访问方法

	主要是为raw_pci_ops/raw_pci_ext_ops赋值

	kernel提供了多种访问配置空间的方式, 由kernel编译选项和启动命令指定
	- PCI_BIOS: 支持pci bios访问方式的二进制代码

		pci bios方式即通过主板bios提供的服务来完成
	- PCI_DIRECT: 支持机制`#1`访问方式的二进制代码

		[pci_direct_conf1](https://elixir.bootlin.com/linux/v6.6.14/source/arch/x86/pci/direct.c#L83)

	逻辑:
	1. pci_direct_probe:检查机制`#1/#2`

		返回:
		- 0: 都不可用
		- 1: 机制#1可用
		- 2: 机制#2可用
	1. pci_pcbios_init: 检查pci bios方式
	1. pci_direct_init: 决定是否要使用机制`#1/#2`

	PC-AT兼容系统cpu只有内存和I/O空间, 没有专用的配置空间. PCI协议规定利用特定的I/O空间操作驱动主桥转换为配置空间的操作. 目前存在2种转换机制: 机制#1和机制#2, 机制#2在新硬件架构中已淘汰.

	访问指定pci总线给定设备/功能编号的pci设备的配置空间接口: drivers/pci/access.c
1. [pci_subsys_init](https://elixir.bootlin.com/linux/v6.6.14/source/arch/x86/pci/legacy.c#L58): pci总线扫描

	总线编号过程采用深度优先算法.

	x86_init是一个全局变量, 是基于x86 kernel初始化过程中使用的操作表的一个实例, 它的大多数回调函数取决于编译kernel的选项.
	x86_init.pci.init指向x86_default_pci_init, x86_default_pci_init由编译选项决定.

	pci_legacy_init->pcibios_scan_root(扫描0号总线)

	> pci_legacy_init是多种pci扫描方式中的传统扫描模式
1. PCI中断路由: 建立pci硬件中断引脚和kernel中断号的关联

	对x86, pci中断路由的入口是x86_init.pci.init_irq指向x86_default_pci_init_irq(比如 pcibios_irq_init)
1. PCI资源(I/O空间和内存空间)分配: 将PCI设备的I/O地址空间和内存地址空间映射到kernel的总线空间, 这样对总线空间范围内的访问会被自动`重定向`到pci设备

	两种分配方法:
	1. pci_resource_survey: pci设备的I/O空间和内存空间已经配置好, 此时只需要验证配置的正确性

		pci_subsys_init->pcibios_init->pci_resource_survey
	1. pci_assign_resources: ci设备的I/O空间和内存空间还未配置好, 需要分配资源
1. pci_proc_init: 初始化procfs中与pci有关的目录项
1. pci_sysfs_init: pci扫描过程中会为pci总线和pci设备在sysfs创建目录项和一些基本的属性文件, 此时会为扫描到的pci设备创建更多的属性文件

从pci_bus_type的函数操作接口也能看出来，pci_bus_match用来检查设备与驱动是否匹配，一旦匹配了就会调用pci_device_probe函数.

设备或者驱动注册后，触发pci_bus_match函数的调用，实际会去比对vendor和device等信息，这个都是厂家固化的，在驱动中设置成PCI_ANY_ID就能支持所有设备.

pci_bus_match:
1. to_pci_dev: 获取到要匹配的设备
1. to_pci_driver： 获取到要匹配的驱动
1. pci_match_device: 匹配操作

	1. pci_match_one_device

一旦匹配成功后, 就会去触发pci_device_probe:
1. to_pci_dev: 获取到要匹配的设备
1. to_pci_driver: 获取到要匹配的驱动
1. pci_assign_irq: 通过读取PCI配置信息来分配PCI设备使用的中断

	1. pci_find_host_bridge

		1. hbrg->map_irq: of_irq_parese_and_map_pci
1. __pci_device_probe -> pci_match_device

	match上是触发: pci_call_probe->local_pci_probe(回调实际驱动中的probe探测函数)->`pci_drv->probe`

pci设备枚举:
1. pci_host_probe-> pci_scan_root_bus_bridge

	1. pci_register_host_bridge: 注册host bridge设备. 在注册的过程中需要创建一个root bus，也就是bus 0. 在pci_register_host_bridge函数中，主要是一系列的初始化和注册工作，此外还为总线分配资源，包括地址空间等.

		1. pci_alloc_bus
		1. pci_bus_add_resource
	1. pci_scan_child_bus: 从root bus开始扫描并添加设备

		1. pci_scan_child_bus_extend: 扫描总线上的各类设备(PCI设备与桥设备)

			1. pci_scan_slot: 扫描pci设备, 从循环也能看出来，每条总线支持32个设备，每个设备支持8个功能，扫描完设备后将设备注册进系统，pci_scan_device(扫描设备)的过程中会去读取PCI设备的配置空间，获取到BAR的相关信息等

				扫描单个设备用pci_scan_single_device

				pci_scan_device只读取了制造商ID和HEADER_TYPE, pci_setup_device用于进一步读取并设置pci设备
			1. pci_scan_bridge_extend: PCI桥设备扫描，PCI桥是用于连接上一级PCI总线和下一级PCI总线的，当发现有下一级总线时，创建子结构，并再次调用pci_scan_child_bus_extend的函数来扫描下一级的总线，从这个过程看，就是一个递归过程
### 配置空间
pci_raw_ops控制配置空间的读写, 通常读函数是pci_conf1_read, 写函数是pci_conf1_write.

pci_read->raw_pci_read->`raw_pci_ops->read`(访问原生pci配置空间)或`raw_pci_ext_ops->read`(访问扩展pci配置)

### platform_device
![](/misc/img/driver/pcie/1771657-20210109185820984-571538081.png)
![](/misc/img/driver/pcie/1771657-20210109185829006-646052988.png)

根据device_node节点，创建platform_device结构，并最终注册进系统，这个也就是PCIe Host设备的创建过程.

以driver.md中的`"xlnx,nwl-pcie-2.11"`对应的nwl_pcie_probe举例:
1. 通常probe函数都是进行一些初始化操作和注册操作：

	- 初始化包括：数据结构的初始化以及设备的初始化等，设备的初始化则需要获取硬件的信息（比如寄存器基地址，长度，中断号等），这些信息都从DTS而来；
	- 注册操作主要是包含中断处理函数的注册，以及通常的设备文件注册等;
 

1. 针对PCI控制器的驱动，核心的流程是需要分配并初始化一个pci_host_bridge结构，最终通过这个bridge去枚举PCI总线上的所有设备；
1. devm_pci_alloc_host_bridge：分配并初始化一个基础的pci_hsot_bridge结构；
1. nwl_pcie_parse_dt：获取DTS中的寄存器信息及中断信息，并通过irq_set_chained_handler_and_data设置intx中断号对应的中断处理函数，该处理函数用于中断的级联；
1. nwl_pcie_bridge_init：硬件的Controller一堆设置，这部分需要去查阅Spec，了解硬件工作的细节。此外，通过devm_request_irq注册misc中断号对应的中断处理函数，该处理函数用于控制器自身状态的处理；
1. pci_parse_request_of_pci_ranges：用于解析PCI总线的总线范围和总线上的地址范围，也就是CPU能看到的地址区域；
1. nwl_pcie_init_irq_domain和mwl_pcie_enable_msi与中断级联相关
1. pci_scan_root_bus_bridge：对总线上的设备进行扫描枚举. brdige结构体中的pci_ops字段，用于指向PCI的读写操作函数集，当具体扫描到设备要读写配置空间时，调用的就是这个函数，由具体的Controller驱动实现

### 中断处理
ref:
- [PCIe扫盲——两种中断传递方式/三种中断机制（INTx/MSI/MSI-X）](https://aijishu.com/a/1060000000289702)

PCIe控制器，通过PCIe总线连接各种设备，因此它本身充当一个中断控制器，级联到上一层的中断控制器（比如GIC）
![](/misc/img/driver/pcie/1771657-20210109185902554-698495300.png)

PCIe总线支持两种中断的处理方式:
- Legacy Interrupt：总线提供INTA#, INTB#, INTC#, INTD#四根中断信号，PCI设备借助这四根信号使用电平触发方式提交中断请求

	提供Legacy中断方式的主要原因: 在PCIe体系结构中，存在许多PCI设备，而这些设备通过PCIe桥连接到PCIe总线中
- MSI(Message Signaled Interrupt) Interrupt：基于消息机制的中断，也就是往一个指定地址写入特定消息，从而触发一个中断

	与Legacy中断方式相比，PCIe设备使用MSI或者MSI-X中断机制，可以消除INTx这个边带信号，而且可以更加合理地处理PCIe总线的`序`. 目前绝大多数PCIe设备使用MSI或者MSI-X中断机制提交中断请求.

	MSI和MSI-X机制的基本原理相同，其中MSI中断机制最多只能支持32个中断请求，而且要求中断向量连续，而MSI-X中断机制可以支持更多的中断请求，而并不要求中断向量连续。**与MSI中断机制相比，MSI-X中断机制更为合理**

针对两种处理方式，NWL PCIe驱动中，实现了两个irq_chip，也就是两种方式的中断控制器:
1. nwl_leg_irq_chip
1. nwl_irq_chip

> irq_domain对应一个中断控制器（irq_chip），irq_domain负责将硬件中断号映射到虚拟中断号上.

作为两种不同的中断处理方式，套路都是一样的，都是创建irq_chip中断控制器，为该中断控制器添加irq_domain，具体设备的中断响应流程如下:
1. 设备连接在PCI总线上，触发中断时，通过PCIe控制器充当的中断控制器路由到上一级控制器，最终路由到CPU
1. CPU在处理PCIe控制器的中断时，调用它的中断处理函数，也就是上文中提到过的nwl_pcie_leg_handler，nwl_pcie_msi_handler_high，和nwl_pcie_leg_handler_low
1. 在级联的中断处理函数中，调用chained_irq_enter进入中断级联处理
1. 调用irq_find_mapping找到具体的PCIe设备的中断号
1. 调用generic_handle_irq触发具体的PCIe设备的中断处理函数执行
1. 调用chained_irq_exit退出中断级联的处理

### pci设备驱动编程模式
采用pci接口的主机适配器(scsi, 以太网卡, 显卡等)需要的驱动称为PCI设备驱动. 例如e1000驱动.

pci设备驱动需要实现pci_driver. 驱动指定所支持的PCI设备ID列表由静态或动态两种.

需要实现的方法:
1. probe

	对于scsi hba: 创建scsi适配器的scsi_host

	逻辑:
	1. 启用PCI设备
		1. pci_enable_device:

			在访问任何设备寄存器前, 驱动需要启用PCI设备
		1. pci_set_master: 启用DMA. 关闭DMA是pci_clear_master
	1. 请求MMIO/IOP资源
	
		1. pci_request_region: 验证没有其他的设备正在使用相同的地址资源.
	1. 设置DMA掩码长度
	
		1. pci_set_dma_mask: 表面设备的DMA能力
	1. 设置相干DMA缓冲区
	1. 初始化设备寄存器
	1. 注册IRQ处理函数
	1. 注册到其他子系统

		比如scsi, usb等
1. remove: 关闭pci设备

	逻辑:
	1. 禁止设备生成IRQ
	1. 释放IRQ
	1. 停止所有DMA活动
	1. 释放DMA缓冲区
	1. 从其他子系统注销
	1. 禁止设备响应MMIO/IO端口地址

		io_unmap + pci_disable_device
	1. 释放MMIO/IO端口资源

		pci_release_region


pci_register_driver: 注册pci驱动, 作用:
1. pci驱动如何与它支持的pci设备绑定
1. 如何通过sysfs为pci驱动添加和删除动态ID

	by sysfs pci驱动目录下的new_id和remove_id