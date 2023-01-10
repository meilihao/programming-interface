# pci设备
ref:
- [pcie基础概念](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzU4MTczMDg1Nw==&action=getalbum&album_id=1337043626001661952&scene=173&from_msgid=2247483660&from_itemidx=1&count=3&nolastread=1#wechat_redirect)
- [PCIE Configuration Space – Class Code即pci code and id assignment specification](https://blog.ladsai.com/pci-configuration-space-class-code.html)
- [pcie Configuration Space访问方法](https://blog.csdn.net/weixin_45279063/article/details/116988334)
- [浅谈PCI Express体系结构(一 ~ 三)](https://cloud.tencent.com/developer/article/1848375)

PCIe tree就是PCIe specification定义的了PCIe系统的拓扑结构, 定义了这个拓扑，所有的设备内部的PCIe设备都可以抽象对应到一颗倒立的“树”上来.

![](/misc/img/arch/device/pci/640-0.jpeg)

> spec里面这个拓扑图的RC和CPU是两个独立的部分，事实上，我们常用的处理器目前大部分都是把CPU和RC集成到了一起.

> PCI-SIG = PCI Special Interest  Group，是PCI/PCIe的行业组织，负责制定PCIe规范和标准.

树的根即是RC(Root Complex), 也叫做根节点或者根联合体, 是pcie tree的开始. 有几颗树（PCIe域）取决于有几个RC.

> 一般情况下RC间不能通讯, 如果RC支持P2P, 或者利用高级的NT、Fabric技术则是可以的.

从根节点开始向下生长出树干和树枝（PCIe链路），某些树枝还扩展生长出旁路的分支，这些可以扩展生长的地方，对应拓扑中的Switch. 最末端的叶/果实，对应拓扑中的EP（End Point）.

用`lscpi -t`即可看到pcie tree, 不过它是横向的, 这便于展示.
![](/misc/img/arch/device/pci/KGyuWH98IvV8sXdntaffyycDvl0PDSVtjupHg8DqKd3ZXKiaibJzAiauCdnK0siaArv4eXITNBLnSlTDpKUQobhk5Q.jpeg)

### 其他
- pci vs pcie

    PCIe是基于PCI的基础上演进而来的, 从软件角度看，基于PCI的驱动和软件几乎可以无缝移植到PCIe系统上来而不需要做任何改变.

    而从硬件角度看，差异就非常大了:
    - 从pci的并行传输改成了pcie的串行
    - PCI是共享型总线，多个设备共享一条总线，这种情况下必然存在总线总裁. PCIe则是点对点连接，一个设备直接连接到另一个设备，不存在总线竞争和仲裁.
    - PCI总线上是单向传输，任意时刻只有一个方向的传输，PCIe则是任意时刻都可以双向传输.
    - PCI有很多的边带控制信号，如`FRAME#, IRDY#, TRDY, STOP#`等. PCIe总线上传输的都是基于包（packet），控制和其他处理都嵌入在包里.

        PCIe链路是没有边带信号的，当然也没有单独的时钟线. PCIe的时钟是编码嵌入在link里的。接收端依靠同样的逻辑解析出时钟和数据.
- 为什么要并改串

    随着总线频率的提高，并行传输在高速传输的时候，并行的连线直接干扰异常严重。而串行总线使用差分信号传输（differential transmission），有更强的抗干扰能力，从而可以将传输频率大幅提升。另外，串行总线布线简单，节约PCB面积，并可以多条lane灵活组合扩展性更强。

- 为什么要120/130编码

    PCIe 3.0之前采用8b/10b编码，简单讲就是每10个bit中只有8个bit是有效数据，有效数据只占80%，严重浪费了带宽。而128b/130b编码中98%以上都是有效数据，提高了链路利用率.
- pcie switch
    
    从逻辑上看, Swith可以看作是多个PCI-PCI bridge的组合, 内部有虚拟PCI总线. Switch内部远比单纯的virtual bus复杂，一般会有内部的buffer和调度转发逻辑.

    ![](/misc/img/arch/device/pci/bridge.jpeg)
- [PCIe 6.0](https://zhuanlan.zhihu.com/p/500585186)

    从技术上来说，PCIe 6.0是PCIe问世近20年来，变化最大的一次, 主要有三大变化：数据传输速率从32GT/s翻倍至64GT/s；编码方式从NRZ 信令模式转向PAM4信令模式；从传输可变大小TLP到固定大小FLIT.
- pcie速率

    ![](/misc/img/arch/device/pci/v2-e9aa7a8d4d30a913b915bc6ff14a69d0_720w.jpg)

    PCIe Spec里面的GT/s是giga transfers persecond的缩写. Gbps和GT/s的差异是: GT/s描述的是链路上传输的原始数据，Gbps描述的是链路上传输的有效数据. 原始数据和有效数据的差异是编码里的有效数据占比: 比如PCIe 3.0采用的128b/130b编码, 原始数据是130b，其中有效数据是128b.

## pci桥接
ref:
- [基于PCIe总线的多路复用DMA高速传输系统的设计](http://www.eeskill.com/article/id/43067)

PCIe桥连接方式有2种：透明桥(Transparent Bridge)和非透明桥(Non-Transparent Bridge).

总系统中存在两个独立的处理器，此时使用透明桥方式不利于整个系统的配置与管理，可能出现PCIe总线地址的映射冲突，此外不能使用PCIe透明桥连接两个PCIe Agent设备，如x86处理器。采用非透明桥可有效的解决这些问题，提高PCIe传输系统的可移植性。非透明桥不是PCIe总线定义的标准桥片，但是这类桥片在连接两个处理器系统中得到了广泛的应用.

> 透明是指这个桥对于经过它的报文或者数据，不做任何的处理和表更，直接往下游或者上游传递.

## 上行(Upstream)和下行(Downstream)
在一个PCIe系统中, 传输方向的定义是以RC为准的: 面往RC方向称之为上行, 反之称之为下行.

由于PCIe是点对点连接的，每个连接的地方，我们称之为Port。对于Root Complex而言，它仅有一个下行端口。对于PCI-Express switch，它有一个上行端口（upstream port）和多个下行端口（downstream ports）。而PCIe设备（EP）仅有一个上行端口.

一般而言，上行端口是和一个下行端口连接在一起。注意，是一般而言，某些特殊的情况下，两个下行端口也是可以连接在一起的，称之为crosslink.

Switch不一定只能有一个上行口, 支持Virtual switch功能的情况下，可以有多个上行口. 通常Switch的端口划分是按一定规则可以灵活配置的.

## Lane和Link
![](/misc/img/arch/device/pci/lane-link.jpeg)

Lane，是指一组差分信号的组合，包括发送和接收。一个发送方向的差分信号包括TX+和TX-两条线，接收亦然. 所以一条lane有四条物理连线。发送和接收是同时进行的，故为全双工.

Link，是指两个PCIe部件的链接，通常是由端口和lane组成。（通常有多条lane）比如**有一个X2的链路，意思是指这条链路是两条lane组成，一共8条物理连线**. 链路上传送的是编码之后的数据.比如Gen1/Gen2所采用的8b/10b编码，Gen3之后改成了128b/130b编码。

多条lane组成的link，有效的扩展了link的带宽。Lane的初始化和多条lane的组合优化，是在link的初始化训练过程（Link Training）中实现的. Link的速率在初始化以后确定，所有lane的信号速率一样.

> Link初始化以及link建立过程（或者称之为链路训练，Link Training）是在设备上电或者链路重新建立链接时发生的.

>  Link的初始化和协商，确定lane宽度以及速率，是不需要操作系统或者软件参与的，纯粹是芯片的硬件行为.

> 协议规定了只能是x1, x2, x4, x8, x12, x16, x32, 因此不是任意条lane都可组成一个link.

> lane0和lane1可以交换顺序交叉连接, 协议上这种特性叫Lane Reversal, 不过需要注意的是，这个是可选特性，不是每个设备都支持的.

> 两个不同速率的device在协商阶段，首先按照**最低的Gen1**速率协商，均采用同样的8/10b编码。协商完成后，按照协商后的速率建立链接.

在PCIe的链路初始化和协商的过程中（Link Training），其中一个重要的步骤就是确定双方的Link链路需要几条Lane。给Lane做了编号，称之为Lane Number，从0开始。默认设备A的Lane需要和设备B的Lane对应序号连接.

为了降低PCB布线工程师的布线难度，PCIe spec定义了Lane Reversal和Polarity Inversion，即Lane反转和极性倒置（或称之为极性反转）. PCIe spec规定了Polarity Inversion是PCIe设备必须要实现的，而Lane Reversal则是可选支持特性. 另外，对于Lane Reversal，很容易看出，只需要链路一端的设备支持就可以实现了.

对于连续的多个字节，在链路上的发送是这样的:
- 链路是x1的话，即仅有一条Lane0，多个字节按照顺序依次编码后在Lane0发送即可
- 链路有多条Lane, 假设链路是x4的，即有四条Lane. 连续的字节按照如下的规则发送：Byte0从Lane0发送，Byte1从Lane1发送，Byte2从Lane2发送，Byte3从Lane4发送，以此类推

## pci数据路由
[PCIe的设备有三种资源（ID、Memory、IO）](https://mp.weixin.qq.com/s?__biz=MzU4MTczMDg1Nw==&mid=2247483766&idx=1&sn=3107750e77c1d93740e867b064d2f6c8), 具有了资源，PCIe设备才具有了被访问、被使用的基本能力:
- ID Resource

    ID资源: 即设备在系统中的定位和身份证. 用总线号（Bus）、设备号(Device)和功能号(Function)三个变量定义

    ![](/misc/img/arch/device/pci/function_device.jpeg)

    对于一个PCIe设备，如果它只具有一个功能，称之为Single FunctionDevice；如果它有多个功能，则称之为Multi Function Device. 例如, 有这么一个AMD的显卡，位于01号总线，00号设备，它具有两个功能，功能0是显示用的Radeon HD 6750，功能1是音频用的Radeon HD 5700.

     ID是由总线号、设备号和功能号共同组成的。因为有多功能设备，所以ID实际是一个功能的名字和地址. 在Linux系统中，表示ID的方式是[Bus:Device.Function]，如上的显示用的6750的ID号是01:00.0。音频用的5700的ID号是01:00.1

    PCIe EP的设备号（Device Number）都是0，PCIe Switch则不大一样. 在PCIe Switch内部，实际是由多个PCIe-To-PCIe的桥组成，对于每一个桥的设备号可能是0、1、2...
    ![](/misc/img/arch/device/pci/switch.jpeg)

    这些桥的Function可能也有多个，上图仅示意了Function0.

    关于ID号, PCIe Spec定义了如下规则:
    - 总线号由8bit表示，因此，最大总线号为256
    - 设备号由5bit表示，因此，最大设备号为32
    - 功能号由3bit表示，因此，最大功能号为8

- Memory Resource

    内存资源: 即设备具备哪些可以提供给外部或内部使用的内存.

    每个外设都是通过读写其寄存器来控制使用，寄存器也称为I/O端口. 外设上也同样拥有自己的存储空间，称为I/O内存. 在X86的系统中，对于内存访问和IO访问是独立编址的，而其他架构的CPU系统，如RISC指令系统的ARM、PowerPC等，采用统一编址.

    在x86系统中，I/O空间大小为64K，要求单个PCIe设备的I/O空间大小最大为256B；事实上，这个I/O空间的存在，只是在PCI规范上继承，为了兼容传统的一些设备, 在实际工作中几乎不会用到。在驱动程序中，访问I/O空间需要特殊的操作指令，如in、out.

    相对I/O，Memory就要大的多了. 最大大小取决于系统的地址空间，在32位系统中，就可以到4GB. Memroy空间通常用于数据传输. 大部分的PCIe设备也有一部分Memory空间用于寄存器的访问.

    显卡PCIe设备通常有一个内存资源, 比如大小是256M，这就是常说的某显卡有xxx M显存的由来.

- IO Resource

    IO资源: 即设备具有哪些可以使用的IO空间

**对于一个PCIe系统中的每一个PCIe设备, 如上的三种资源分配都是唯一的, 不会有两个设备具备相同的资源**. 在系统启动时，BIOS会给各个PCIe设备分配如上三种资源，这个过程就是常说的枚举（Enumeration）. 注意，在进入Linux系统后，内核会重新扫描总线并check资源.

整个枚举过程分为两个阶段，首先扫描设备, 其次再对每个设备进行资源分配.

一切从根开始，也就是扫描会从RootComplex开始，并且按照深度优先的方式遍历完所有的设备. 在扫描过程中，如果扫描到EP，则返回, 如果是桥，则继续往桥后扫描，总线号增加. 扫描完成后，就得到了整个PCIe大树的整体形状，也就是所说的整个PCIe拓扑. 资源的分配也是从RC开始，同样按照深度优先方式遍历，为每个设备分配必要的资源.

![](/misc/img/arch/device/pci/pcie_enum.jpeg)

Spec规定, 每个PCIe设备都有6个寄存器来向系统声明它需要什么样的资源以及大小. 这6个寄存器称之为BAR（Base  AddressRegister, 4B）, 从BAR0到BAR5. 当向这些寄存器写全1，并再读取这些寄存器的值. 第一个的非0的bit位即是此设备需要的资源大小. 比如写0xFFFF_FFFF到BAR0, 回读的值为0xFFFF_0000，低16bit为零，意味着整个BAR的大小需要2^16 = 64K bytes. 确认PCIe设备资源大小后，BIOS就把系统分配给这个设备的地址写入到这个BAR里面, 驱动程序只需要读取这个BAR寄存器，获取地址后，就可以操作对应的额数据了.

> 比如`lspci -s 1:0.0 -v -xxx`输出的`10`地址开始的6个寄存器就是bar寄存器.

lspci查看它的id资源:
![](/misc/img/arch/device/pci/640.jpeg)
上图这个设备的ID资源是：总线号3，设备号0，功能号0.

lspci查看它的memory和IO资源:
![](/misc/img/arch/device/pci/640-2.jpeg)
上图它的IO资源位于d00处, 大小256. 它的Memroy资源有两个，一个是位于f7a00000大小为4K, 一个是位于f0000000大小为16K.

路由是指一个数据包（也叫报文）从源端经过各种路径，最终到达目的端的过程.

PCIe系统里面从RC到EP, PCIe Spec定义了三种路由方法，分别是
- Routed by Device ID : 基于ID的ID路由

    它是依靠目标设备的ID来作为目标地址的. 即用Bus Number、Device Number和Function Number进行路由寻址。基于ID的路由方式主要用于配置读写请求，以及完成报文.

    带有目标ID（BDF）的报文首先到达目标总线上，当EP收到这样的报文时，它会对比报文里面的BDF是否和自己的相同，如果相同则接收并处理，否则就拒绝接收.

    ![](/misc/img/arch/device/pci/route_id.jpeg)

    当Switch收到时，首先对比报文里面的的BDF是否和自己的相同，如果匹配，则接收并处理。其次，会检查这个报文的Bus Number是否落在自己之下的总线范围内, 依据是Secondary Bus Number和Subordinate Bus Number, 如果在，则转发到对应的下游端口；否则则拒绝接收.

    ![](/misc/img/arch/device/pci/route_id_switch.jpeg)
- Routed by memory or IO Address : 基于地址的地址路由

    报文头里面包含着目的IO或者目的Memory的地址. 报文在总线上路由，就是寻找系统里的某个PCIe设备，而这个设备的资源范围包含这个地址. 以Memory为例来看看这个过程:

    ![](/misc/img/arch/device/pci/route_addr.jpeg)

    当报文到达Switch时，情况稍微复杂一点。当下行的报文到达Switch某个port时，这个报文的目的地址必须是在这个port的资源窗口内，Switch才会把这个报文向下一级转发.

    ![](/misc/img/arch/device/pci/route_addr_switch.jpeg)

    当上行的报文到达port时，情况相反，报文的目的地址必须是在这个port的资源窗口之外Switch才会把这个报文向上一级转发.

    ![](/misc/img/arch/device/pci/route_addr_switch_up.jpeg) 

    > PCIe Spec定义了三类窗口：Memory Base and Limit、Prefetchable Memory Base and Limit、IO Base and Limit.
- Implicit Routing : 隐式路由

    原因:
    1. 是因为PCI总线有一些边带信号来传送信息和控制，在PCIe中采用消息（Message）机制来实现。PCIe Spec规定消息的路由方式为隐式路由
    2. 是在系统中，有一些报文是由EP发给RC的或者RC发出的广播报文，这些广播报文可以传递到系统中每一个设备，这时候就不需要按照ID和地址路由来区分设备了

    EP收到隐式路由的报文，如果是RC发出的广播报文（通常是RC发出的）或者本地报文（Local - Terminate at Receiver，通常是INTx消息），EP接收此报文。
    Switch的处理则分上下行。如果上行端口收到RC的广播报文，则讲此报文发给所有下行端口。如果下行口收到发向RC的消息报文，则将此报文直接转发到上行端口，送给RC；如果收到本地消息报文，则接收此报文，不再向上或向下传播.

    隐式路由的消息有:
    - INTx Interrupt Signaling        

          中断信号。PCI里面有INTx的信号

    - Power Management           

           电源管理用

    - Error Signaling                        

          错误相关的信号

    - Locked Transaction Support          

          支持Lock事务

    - Slot Power Limit Support       

          功率限制

    - Vendor-Defined Messages            

          厂商自定义

    - LTR Messages                          

          Latency Tolerance Reporting

    - OBFF Messages                       

          Optimized Buffer Flush/Fill
    - etc

以[Address Routing](https://mp.weixin.qq.com/s?__biz=MzU4MTczMDg1Nw==&mid=2247483803&idx=1&sn=8b2bc3370d7aef6519fa8dd0976f3838)为例.


## NTB(Non-Transparent Bridge)
ref:
- [Non-Transparent Bridging and PCIe Interface Communication](https://docs.nvidia.com/drive/drive-os-5.2.6.0L/drive-os/index.html#page/DRIVE_OS_Linux_SDK_NGC_Development_Guide/Interfaces/sys_components_non_transparent_bridging.html)
- [3.2.2.9. PCIe Backplane](https://software-dl.ti.com/jacinto7/esd/processor-sdk-linux-jacinto7/07_00_01_01/exports/docs/linux/Foundational_Components/Kernel/Kernel_Drivers/PCIe/PCIe_Backplane.html)
- [Linux NTB](https://events.static.linuxfound.org/sites/events/files/slides/Linux%20NTB_0.pdf)
- [NTRDMA v0.1](https://ostconf.com/system/attachments/files/000/001/112/original/LinuxPiter-NTRDMA-v0.1-slides.pdf)
- [Non-Transparent Bridging Simplified](https://docs.broadcom.com/doc/12353427)
- [多功能PCIE交换机之四：非透明桥NTB](https://developer.aliyun.com/article/506858)
- [PEX87XX非透明桥(NTB)翻译-1](https://zhuanlan.zhihu.com/p/463029355)
- [NTB调试常见问题指南](https://blog.51cto.com/xiamachao/1794555)
- [NTB的地址映射和地址转换](https://blog.csdn.net/linjiasen/article/details/110563838)
- [linux NTB 的测试工具](https://blog.csdn.net/linjiasen/article/details/104532342)
- [Switchtec NTB Demo](https://asciinema.org/a/125573)

非透明桥技术是随着多处理器技术发展的.

### 原理

两个系统的间接通信有: 利用串口、USB通信，利用网卡通信、利用FC通信等, 直接通信有NTB(PCIe Non-Transparent Bridging).

两个系统不能通过透明交换机连接的原因: PCIe数据路由是基于地址的，两个系统可能资源分配冲突，这意味着两个设备具有相同的资源分配，因此具有该地址的数据包无法正确路由. 解决方案是当数据包通过结构从一个系统传输到另一个系统时进行地址转换，这是通过非透明桥（NT）完成的.

透明桥路由:
1. ①某个访问地址1500h的数据报文到达上行口P-P桥，P-P桥一看，这个地址在我的窗口范围内，向下行端口转发
1. ②下图中高亮的下行端口P-P透明桥，一看，这个地址在我的桥下窗口范围内，继续向下转发
1. ③穿过透明P-P桥的数据报文，地址仍然为1500h
![透明桥路由](/misc/img/arch/device/pci/tb.jpeg) 

非透明桥的路由:
1. ①某个访问地址Δ + 1500h的数据报文到达上行口P-P桥，P-P桥一看，这个地址在我的窗口范围内，向下行端口转发
1. ②下图中高亮的下行端口P-P透明桥，一看，这个地址在我的桥下窗口范围内，继续向下转发
1. ③穿过非透明P-P桥的数据报文，地址进行了翻译！从Δ +1500h变成1500h
1. ④⑤是数据报文在系统B里的路由
![非透明桥的路由](/misc/img/arch/device/pci/ntb.jpeg) 

> Δ + 1500h所在的资源范围是非透明桥NT的BAR资源，向系统申请的.

透明桥是进来什么地址，出去就是什么地址，对于桥上下两侧是“透明”的。非透明桥是有翻译功能的，可以把一个地址翻译成另一个地址（当然，也会有ID翻译功能）. 所谓非透明的部分意义也是在于此.

NT桥的内部，其实是两个Endpoint设备。NT桥将一个透明端口分成两个非透明端口（端点设备/功能）。透明桥的配置空间为Type1，非透明桥的配置空间则为Type0。同时，NT桥中的两个NT EP都有各自的Type0配置空间.

既然是EP，就知道每个EP都有6个BAR空间。BAR 0 到BAR 5。所谓BAR（Base Address Register）就是每个EP设备的一个寄存器，这个寄存器会向系统申请一段一定大小的空间地址，系统所有访问这个空间地址的报文，都会被路由到这个EP来处理。

![非透明桥的内部](/misc/img/arch/device/pci/ntd_inner.png)

通常，BAR0是用作映射到EP设备的配置空间，访问BAR0可以映射访问所有的寄存器。而NT的BAR2 到 BAR5通常都是用于NT桥接地址转换用的. 以BAR 2为例, NT EP向系统申请了个两端空间，一个是BAR0，所有访问BAR0地址范围的报文都将落入到EP的内存映射寄存器里面去。另一个就是BAR2，这个BAR2的地址空间，我们称为NT窗口。所有访问BAR2地址范围的报文都将进入NT，然后被地址转换

![非透明桥的内部2](/misc/img/arch/device/pci/ntd_inner2.png)

 进入NT Window的报文，会根据我们自己设置的NT桥转换基址（Translated Base Addres）做运算，运算之后的地址刚好要等于我们想访问的B系统里面的目标地址（Target Adress）。这样，即使加上一定的偏移（Offset），也能顺利转化为B系统里对应偏移的地址

![非透明桥的内部3](/misc/img/arch/device/pci/ntd_inner3.png)

NT地址转换的基本原理就是这样。需要注意的是，如果只是写请求，那么只需要地址翻译、转换就可以了。但如果是Non-Post的事务，这个时候需要的是ID 路由（ID Routing）, 此时需要进行id转换.

从PCIe域以及地址归属来看，NT桥两边的两个EP设备，其实是分别归属于两个系统的（蓝色属于系统A，绿色属于系统B）。至于两个NT EP中间的NT Bridge，那是厂商芯片自己内部的实现, 对双方系统来说都是不可见的.
![非透明桥的概览](/misc/img/arch/device/pci/ntd_system.png)

ntb的3种典型场景:
1. 单NT模式

    正常情况下，Host1访问、管理下面的设备EP。Host2不参与，处于Standby状态。当Host1出现异常时，Host2接管系统，Host2重新配置Switch，把原来的NT口配置成Upstream Port，并且重新分配、枚举PCIe设备资源。然后Host2接管下面EP设备的访问、管理.

    ![](/misc/img/arch/device/pci/ntd_use1.png)

1. 双NT模式

    Host1、2访问、管理自己下面的设备EP。Host1和Host2之间通过NTB互联。两个Host的CPU之间可以通过NT桥互相通信，因此，两个Host都可以访问到对端的所有资源

    ![](/misc/img/arch/device/pci/ntd_use2.png)

1. 多NT交换模式

    多个Host通过NT，与一个中央交换机（Switch）互联。透过NT桥，每个主机都能够互相两两访问。值得注意的是，中央交换机是需要一个外部的管理CPU来初始化、配置以及做异常处理的。这个管理CPU不一定非要使用PCIe和Switch连接，也可以使用带外通道，如I2C等做配置管理即可.

    ![](/misc/img/arch/device/pci/ntd_use3.png)


linux NTB driver软件栈: hardware目录是各个厂商具体的实现方式，NTB抽象出一个概念：ntb_dev，所有的hardware厂商要注册对应的OPS到ntb_dev->ops，这样可以提供统一的API，上层driver是看不到底层硬件具体操作.
![](/misc/img/arch/device/pci/20200227111435491.png)

### 实践
比如`07:00.0 Network controller: PLX Technology, Inc. PEX 8732 32-lane, 8-Port PCI Express Gen 3 (8.0 GT/s) Switch (rev ca)`, 但kernel 5.18没有PEX 8732相关驱动支持, [论坛RFC](https://groups.google.com/g/linux-ntb/c/kO6IAj4dB5k)也没有下文, 因此ntb设备应该选择kernel支持的硬件.

**kernel 5.19里没有plx ntb驱动, plx官方可提供相关SDK包含Driver但是适配的是2.26, 要求pci class=0x088000, 实际07:00.0的pci class=0x028000, 幸运的是[truenas维护的linux kernel已整合plx ntb驱动](https://github.com/truenas/linux/blob/SCALE-v5.10-stable/drivers/ntb/hw/plx/ntb_hw_plx.c)**.

> bios可能需要设置plx enable.

加载ntb驱动:
```bash
modprobe ntb
modprobe switchtec
modprobe ntb_hw_switchtec # ntb_hw_switchtec依赖switchtec, ntb_hw_intel, ntb_hw_amd等不需要
modprobe ntb_transport
modprobe ntb_netdev
```

当前使用ntb的方式是通过ntb_netdev创建网卡设备(需配置ip)来使用. ntb_netdev将公开虚拟以太网接口, 该接口流量都将通过 PCIe传输.

# device
root可使用 mknod 命令创建设备文件.

当前系统支持的设备见`/proc/devices`, 由驱动程序生成.它可产生一个major供mknod作为参数. /dev下的设备是通过mknod加上去的, 用户可通过此设备名来访问驱动.

常见的硬件设备及其文件名称:
- IDE 设备 /dev/hd[a-d]
- SCSI/SATA/U 盘 /dev/sd[a-p]
- virtio设备: /dev/vd[a-z]
- 软驱 /dev/fd[0-1]
- 打印机 /dev/lp[0-15]
- 光驱 /dev/cdrom
- 鼠标 /dev/mouse
- 磁带机 /dev/st0 或/dev/ht0

> /dev 目录中 sda 设备之所以是 a，并**不是由插槽决定的，而是由系统内核的识别顺序来决定的**，而恰巧很多主板的插槽顺序就是系统内核的识别顺序. 因此在使用 iSCSI 网络存储设备时就会发现，明明主板上第二个插槽是空着的，但系统却能识别到/dev/sdb 这个设备就是原因.
> 分区的数字编码不一定是强制顺延下来的，也有可能是**手工指定**的. 因此sda3 只能表示是编号为 3 的分区，而不能判断 sda 设备上已经存在了 3 个分区.

> linux网络设备是独立的类型, 即不是块设备也不是字符设备.

## blocksize
参考:
- [Advanced Format](https://zh.wikipedia.org/wiki/%E5%85%88%E9%80%B2%E6%A0%BC%E5%BC%8F%E5%8C%96)
- [磁盘格式的512，512e，4kn是什么意思](https://www.reneelab.net/what-is-4kn-disk.html)
- [过渡到高级格式化 4K 扇区硬盘](https://www.seagate.com/cn/zh/tech-insights/advanced-format-4k-sector-hard-drives-master-ti/)
- [Advanced Sector Format of Block Devices](https://www.thomas-krenn.com/en/wiki/Advanced_Sector_Format_of_Block_Devices)
- [512e and 4Kn Disk Formats](https://i.dell.com/sites/csdocuments/Shared-Content_data-Sheets_Documents/en/512e_4Kn_Disk_Formats_120413.pdf)
- [机械硬盘避坑大法：一文搞懂 PMR 和 SMR 有什么区别](https://www.ithome.com/0/436/608.htm)

对于大多数现代磁盘，逻辑扇区大小小于物理扇区大小是正常的(通过`fdisk -l /dev/sd<N>`获取), 这就是最常用的[高级格式磁盘512e](https://en.wikipedia.org/wiki/Advanced_Format)的实现方式.

> Linux内核可在`/sys/block/sdX/queue/physical_block_size`获取有关物理扇区大小的信息，并在`/sys/block/sdX/queue/logical_block_size`获取有关逻辑扇区大小的信息.

> The old format gave a format efficiency of 88.7%, whereas Advanced Format results in a format efficiency of 97.3%.

> `fdisk -l`的`I/O size`: optimal I/O size(最佳io的大小) 默认是 minimal I/O size(磁盘最小io的大小).

disk Advanced Format(Logical Sector Size/Physical Sector Size):
- 512n : 512/512
- 512e : 512/4,096

    512e是因为当时部分操作系统识别不了4k物理扇区，而做的兼容方案. 作为过渡时期的产物，512e(e的意思是Emulation)表示由disk firmware将4K的磁盘模拟成512，让系统以为看见的是512格式.

    在disk支持的情况下, `hdparm --set-sector-size 4096`可修改Logical Sector Size.
- 4Kn : 4,096/4,096

    带"Advanced 4Kn Format" (4K native). 没有512-byte emulation layer, 性能更佳.

磁盘大小度量:
1. IEEE1541-2002（以下简称IEEE1541）, 这个新系统针对二进制单位采用新的名称和后缀缩写 kibibytes（KiB，210字节）、mebibytes（MiB，220字节）、gibibytes（GiB，230字节）、tebibytes（TiB，240字节），依此类推, **推荐**
2. SI 单位使用十进制(基数为 10)乘数. SI 单位在二进制计量中的这种不一致的应用，可能会导致混乱. 传统硬盘制造商和某些磁盘实用工具历来以基于 10 的方式使用 SI 单位.

## 设备文件
在linux中, 硬件设备都以文件形式存在, 不同设备有不同的文件类型, 这些文件被称为设备文件. 设备文件让程序能够同系统的硬件和外围设备进行通信. 在Linux中,所有的设备以文件的形式出现在目录 /dev 中, 由`systemd-udevd`管理(随着kernel报告的硬件变化创建或删除设备文件).

linux通过设备号来区分不同的设备.设备号由两部分组成:[主设备号](https://elixir.bootlin.com/linux/v5.12.10/source/include/uapi/linux/major.h#L10)和次设备号.

主设备号用来查找一个特定的驱动程序即指明设备类型, 次设备号用来表示使用该驱动程序的各设备即指具体哪个设备.

> 设备文件的 i 节点中记录了设备文件的主、辅 ID. 每个设备驱动程序都会将自己与特定主设备号的关联关系向内核注册,藉此建立设备专用文件和设备
驱动程序之间的关系.

`crw-rw-rw-  1 root tty       4,     3 6月  26 21:34 tty3`, `4`是主设备号, `3`是次设备号.

实际上设备文件也分块设备文件(block special file) 和 字符设备文件(character special file).  块设备每次读取或写入一块数据(一组字节, 通常是512的倍数), 字符设备每次读取或写入一个字节.

还有一种有设备驱动但没有实际设备的虚拟设备称为伪设备, 比如pty(伪tty), /dev/null, /dev/random等.

常见设备:
- `/dev/sd${xxx}` : scsi磁盘
- `/dev/srx` : scsi光驱
- `/dev/stx` : scsi磁带
- `/dev/cdrom` : 指向光驱的符号连接.

## 块设备(block device) / 字符设备(character device)
- 块设备(**可寻址**) : 提供对设备(比如磁盘,蓝光光盘)带缓冲的访问, 每次访问以固定长度(块)为单位进行.
- 字符设备(**不可寻址**) : 接收或输出字符流的设备, 需**按顺序操作, 不支持随机操作**, 用于键盘, 串口, 打印机, 调制解调器等.

> Linux 专有文件/proc/partitions 记录了系统中每个磁盘分区的主辅设备编号、大小和名称.

## udev
参考:
- [在 Linux 使用 systemd-udevd 管理你的接入硬件](https://linux.cn/article-13691-1.html)
udev 负责监听 Linux 内核发出的改变设备状态的事件.

在用户空间实现的一种设备管理fs(devtmpfs), 它会将设备文件自动放在/dev下, 即/dev完全由udev管理. 它依赖sysfs(/sys下的虚拟文件系统)来了解系统的设备变化, 并试图将它收到的每个系统事件匹配一系列udev特有的规则集来指定其对设备的管理以及命令, 默认规则在`/lib/udev/rules.d`下, 自定义规则在`/etc/udev/rules.d`下.

```bash
# grep -ri '/dev/disk' /lib/udev/rules.d # 查找生成/dev/disk下信息的规则
/lib/udev/rules.d/60-persistent-storage.rules
```

> 部分相关代码: [udev-builtin-path_id.c](https://cgit.freedesktop.org/systemd/systemd/tree/src/udev/udev-builtin-path_id.c)

使用 systemd 的机器上，udev 操作由 systemd-udevd 守护进程管理，可以通过`systemctl status systemd-udevd` 检查 udev 守护进程的状态.

规则集文件包括匹配键和分配键，可用的匹配键包括 action、name 和 subsystem, 这意味着如果探测到一个属于某个子系统的、带有特定名称的设备，就会给设备指定一个预设的配置.

接着，“分配”键值对被拿来应用想要的配置。例如，可以给设备分配一个新名称、将其关联到文件系统中的一个符号链接、或者限制为只能由特定的所有者或组访问.

比如:
```conf
$ cat /lib/udev/rules.d/73-usb-net-by-mac.rules
# Use MAC based names for network interfaces which are directly or indirectly
# on USB and have an universally administered (stable) MAC address (second bit
# is 0). Don't do this when ifnames is disabled via kernel command line or
# customizing/disabling 99-default.link (or previously 80-net-setup-link.rules).
IMPORT{cmdline}="net.ifnames"
ENV{net.ifnames}=="0", GOTO="usb_net_by_mac_end"
ACTION=="add", SUBSYSTEM=="net", SUBSYSTEMS=="usb", NAME=="", \
    ATTR{address}=="?[014589cd]:*", \
    TEST!="/etc/udev/rules.d/80-net-setup-link.rules", \
    TEST!="/etc/systemd/network/99-default.link", \
    IMPORT{builtin}="net_id", NAME="$env{ID_NET_NAME_MAC}"
```
add 动作告诉 udev，只要新插入的设备属于网络子系统，并且是一个 USB 设备，就执行操作. 此外，只有设备的 MAC 地址由特定范围内的字符组成，并且 80-net-setup-link.rules 和 99-default.link 文件不存在时，规则才会生效.

## `/dev/loopN`
一种伪设备，使得文件可以如同块设备一般被访问.

可通过`losetup -a`查看.

snapd的`loopN`可通过`sudo apt autoremove --purge snapd`解决.

## [块设备持久化命名(Persistent block device naming)和多路径](https://www.zybuluo.com/tony-yin/note/1214135)
持久化命名，顾名思义区别于一次性或者是短暂的命名，它是一种长久的并且稳定靠谱的命名方案. 与之形成鲜明对比的就是/dev/sda这种非持久化命名，这两种命名方案各有各的用处，持久化命名方案有四种：by-label、by-uuid、by-id和by-path. 对于那些使用GUID分区表（GPT）的磁盘，还有额外的两种方案：by-partlabel和by-partuuid.

by-label和by-uuid都和文件系统相关，by-label是通过读取设备中的内容获取，by-uuid则是随着每次文件系统的创建而创建，所以by-uuid的持久化程度更高一些；持久化程度最高的要属by-path和by-id了，因为它们都是根据物理设备的位置或者信息而和链接做对应的，by-path会因为路径的变化而变化；而by-id则不会因为路径或者系统的改变而改变，它只会在多路径的情况下发生改变. 这两个在通过虚拟设备名称寻找物理设备的场景下都十分有用.

多路径设备则帮助我们在SAN等场景下提高了数据传输的可用性，目前由于网络带宽的发展，它在iscsi场景下也频繁亮相.

### by-label
label表示标签的意思，几乎每一个文件系统都有一个标签. 所有有标签的分区都在/dev/disk/by-label目录中列出.

label是通过从设备中的内容（即数据）获取，所以如果将该内容拷贝至另一个设备中，我们也可以通过blkid来获取磁盘的label.

### by-uuid
参考:
- [Using the New GUID Partition Table in Linux (Goodbye Ancient MBR)](https://www.linux.com/training-tutorials/using-new-guid-partition-table-linux-goodbye-ancient-mbr/)

UUID是给每个**文件系统**唯一标识的一种机制，这个标识是在**分区格式化时通过文件系统工具生成**，比如mkfs，这个唯一标识可以起到解决冲突的作用. 所有GNU/Linux文件系统（包括swap和原始加密设备的LUKS头）都支持UUID. FAT和NTFS文件系统并不支持UUID，但是在/dev/disk/by-uuid目录下还是存在着一个更为简单的UID（唯一标识）.

    $ ls -l /dev/disk/by-uuid/
    total 0
    lrwxrwxrwx 1 root root 10 May 27 23:31 0a3407de-014b-458b-b5c1-848e92a327a3 -> ../../sda2
    lrwxrwxrwx 1 root root 10 May 27 23:31 b411dc99-f0a0-4c87-9e05-184977be8539 -> ../../sda3
    lrwxrwxrwx 1 root root 10 May 27 23:31 CBB6-24F2 -> ../../sda1
    lrwxrwxrwx 1 root root 10 May 27 23:31 f9fe0b69-a280-415d-a03a-a32752370dee -> ../../sda4

使用UUID方法的优点是，名称冲突发生的可能性大大低于使用Label的方式. 更深层次地讲，它是在创建文件系统时自动生成的. 例如，即使设备插入到另一个系统(可能有一个标签相同的设备)，它仍然是唯一的.

缺点是uuid使得许多配置文件(例如fstab或crypttab)中的长代码行难以读取和破坏格式. 此外，每当一个分区被调整大小或重新格式化时，都会生成一个新的UUID，并且必须(手动)调整配置.


gpt-uuid:
- PTUUID : 分区表id, 可作为磁盘签名
- [PART_ENTRY_UUID](https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/v2.37/libblkid-docs/libblkid-Partitions-probing.html)：分区id

### by-path
该目录中的条目提供一个符号名称，该符号名称通过用于访问设备的硬件路径引用存储设备，首先引用PCI hierachy中的存储控制器，并包括SCSI host、channel、target和LUN号，以及可选的分区号. 虽然这些名字比使用major和minor号或sd名字更容易，但必须使用谨慎以确保target号不改变在光纤通道SAN环境中(例如，通过使用持久绑定)，如果一个主机适配器切换到到一个不同的PCI插槽的话这个路径也会随之改变. 此外，如果HBA无法探测，或者如果驱动程序以不同的顺序加载，或者系统上安装了新的HBA，那么SCSI主机号都有可能会发生变化.

上面说了很多种情况都会导致by-path的值可能发生变化，但是在同一时间来说，by-path的值是和物理设备是唯一对应的，也就是说不管怎么说by-path是对应物理机器上面的某个位置的，根据by-path可以获取对应物理位置的设备（此前megaraid通过逻辑磁盘获取物理磁盘位置就是根据这个原理）

对于iSCSI设备，路径/名称映射从目标名称和门户信息映射到sd名称.
应用程序通常不适合使用这些基于路径的名称. 这是因为这些路径引用可能会更改存储设备，从而可能导致将不正确的数据写入设备. 基于路径的名称也不适用于多路径设备，因为基于路径的名称可能被误认为是单独的存储设备，导致不协调的访问和数据的意外修改.

此外，基于路径的名称是特定于系统的. 当设备被多个系统访问时，例如在集群中，这会导致意外的数据更改.

### by-id

此目录中的条目提供一个符号名称，该符号名称通过唯一标识符(与所有其他存储设备不同)引用存储设备. 标识符是设备的属性，但不存储在设备的内容(即数据)中.

该id从设备的全局ID（WWID）或设备序列号中获取. /dev/disk/by-id条目也可能包含一个分区号.

World Wide Identifier（WWID）可用于可靠的识别设备, SCSI标准要求所有SCSI设备提供一个持久的、系统无关的ID. WWID标识符保证对每个存储设备都是唯一的，并且独立于用于访问设备的路径.

这个标识符可以通过发出SCSI查询来获取设备标识重要厂商数据(第0x83页)或单位序列号(第0x80页). 从这些wwid到当前/dev/sd名称的映射可以在/dev/disk/by-id/目录中维护的符号链接中看到.
例如，具有页0x83标识符的设备将具有:

    scsi-3600508b400105e210000900000490000 -> ../../sda

或者，具有页0x80标识符的设备将具有:

    scsi-SSEAGATE_ST373453LW_3HW1RHM6 -> ../../sda

Red Hat Enterprise Linux 5自动维护从基于wwid的设备名称到系统上当前/dev/sd名称的正确映射. 应用程序可以使用/dev/disk/by-id/的链接引用磁盘上的数据，即使设备的路径改变，甚至当从不同系统访问该设备时都是如此.

但是当设备被插入到硬件控制器的端口时，而这个端口又受另一个子系统控制（即多路径），by-id的值也会改变.

> 如果两个target设置相同的vpd_unit_serial(`/sys/kernel/config/target/core/iblock_xxx/${iblock_name}/wwwn/vpd_unit_serial`), 那么initiator挂载时因为/dev/disk/by-id下的路径可能相同会导致出现覆盖(即只能找到一个盘), 但/dev/disk/by-path下的路径因为生成规则不同而正常, 我在验证drbd+fc问题时遇到过该情况.

### by-partlabel && by-partuuid
这两个和上面提到的by-label和by-uuid类似，只不过是在GPT磁盘上.

查询lable和partuuid: `blkid /dev/<分区device>`; 单个属性查询: `blkid -s PARTUUID -o value /dev/<分区device>`; 或直接执行`blkid`查找全部分区的信息; 也可使用`blkid -o udev -p /dev/sdb1`(**推荐**).

### 多路径设备
多路径设备指的是从一个系统到一个设备存在多个路径，这种现象主要出现在光纤网络的SAN下，主要是做数据链路冗余以达到高可用的效果，即对应底层一个物理设备，可能存在多个路径表示它.

如果从一个系统到一个设备有多个路径，那么device-mapper-multipath使用WWID来检测它, 然后在/dev/mapper/wwid中显示一个“伪设备”，例如/dev/ mapper/3600508b400105df70000000ac0000.

Device-mapper-multipath显示映射到非持久标识符：`Host:Channel:Target:LUN`， `/dev/sd名称`，以及`major:minor`号.

    3600508b400105df70000e00000ac0000 dm-2 vendor,product 
    [size=20G][features=1 queue_if_no_path][hwhandler=0][rw] 
    \_ round-robin 0 [prio=0][active] 
     \_ 5:0:1:1 sdc 8:32  [active][undef] 
     \_ 6:0:1:1 sdg 8:96  [active][undef]
    \_ round-robin 0 [prio=0][enabled] 
     \_ 5:0:0:1 sdb 8:16  [active][undef] 
     \_ 6:0:0:1 sdf 8:80  [active][undef]

Device-mapper-multipath在系统上自动维护每个基于wwid的设备名称和其对应的/dev/sd名称的正确映射. 这些名称即使是在路径发生改变时也是持久的，并且当从不同的系统访问设备时它们仍然是一致的.

## tty
参考:
- [Linux TTY/PTS概述](https://segmentfault.com/a/1190000009082089)

```bash
# toe -a # 查看支持的tty
# tty # 查看当前终端绑定的tty
/dev/pts/1
# echo aaa > /dev/pts/1 # 往tty直接写入数据跟写标准输出是一样的效果
aaa
# stty -a # 查看当前tty的配置
```

当pts/1收到input的输入后，会检查当前前端进程组是哪一个，然后将输入放到进程组的leader的输入缓存中，这样相应的leader进程就可以通过read函数得到用户的输入. 因此这是有时看到`/proc/<pid>/fd/0 -> /dev/pts/1`, 但`echo aaa > /dev/pts/1`并不起作用的原因, 目前也没法方法跨pgid写pts.

## SAS
参考:
- [SAS学习笔记](https://www.jianshu.com/p/0f4333a5f1af)
- [SAS (Serial Attached SCSI) 技术详解](https://tech.hqew.com/fangan_127764)

 SAS（Serial Attached SCSI）即串行SCSI技术，是一种磁盘连接技术，它综合了并行SCSI和串行连接技术（如FC、SSA、IEEE1394等）的优势，以串行通讯协议为协议基础架构，采用SCSI-3扩展指令集，并兼容SATA设备，是多层次的存储设备连接协议栈.

SAS Phy：一个phy即是一个transceiver，每个phy都有一个SAS addresss，和一个唯一的identifier；
SAS Port：一个port包含一个或一组phy，每个SAS PORT有一个唯一的SAS地址，同一个Port中的所有phy共用一个address，即一个port只有一个SAS address；
SAS device：一个SAS device可以包括一个或多个SAS port，device里的每个phy有一个独立的identifier

> SAS addresss是**无法修改**的.

![SAS device,SAS port,SAS phy关系](/misc/img/arch/device/1201105091707051216.gif)

End device：是一种SAS device，SAS物理连接的末端设备，例如HBA卡、Disk driver都是end device.
Expander device：包括Edge expander device和Fanout expander device:
    - Fanout expander device：起中心交换作用，既可以直接连接到end device，也可以连接到edge expander device
    - Edge expander device：一般用于连接fanout expander device和end device，也可以连接其它的edge expander device，一个edge expander set中只能包含128个SAS address

![SAS Expander拓扑构图](/misc/img/arch/device/2201105091707051217.gif)

Domain：即整个SAS交换构架，由SAS device和SAS expander device组成，其中Device又区分为Initiator和Target，它们可以直接对接起来，也可以经过Expander进行连接，Expander起到通道交换或者端口扩展的作用.

> 一个SAS域理论上可以连接16384 - 256 = 16128个SAS End Device。对比光纤环路126 个device的上限，16128 这个数字仍然是非常可观

SAS协议共有6层，从上到下依次为: 应用层(application layer), 传输层(transport layer),端口层(port layer), 链路层(link layer), phy层(phy layer), 物理层(physical layer).

## ssd
ssd有page和block概念, page是最基本的组成，大小一般是4KB，，每个block通常包含64个page，容量是256KB，也有128个page的，容量就是512KB，不过目前主流的25nm工艺闪存普遍都是8KB page容量，128个page配置.

多个block再组成plane，而plane就是就是闪存中的一颗核心（die）了，而我们看到的闪存片其实是多颗die封装在一起的，一般是2-8颗，而整个SSD上则会由多片闪存组成.

> 实际上，如果SSD内部是以die颗粒的RAID 0模式组建的，那么block层级之上还有一个band之分，它是RAID 0模式中所有芯片的同一块block区块的总和.

ssd写一次的单位是page, 主要原因是为了防止电子干扰, 保证数据的文档和准确.

ssd write时只能写到空page上, 之前写个的page, 必须先进行一次erase, erase的单位是block, 如果一个page的数据被删除后, 想再写这个page, 需要三步:
1. 将同一个block的其他page读出
1. 将block擦除
1. 将block的数据写下去

上述过程是ssd的写放大.

写放大的解决方案:
1. 预留空间

    一般ssd都会有保留空间, 消费级ssd是7~10%, 企业级是20%以上, 甚至100%. 保留空间好处是随时能保证有未使用的空间, 减少写放大, ssd主控可在空闲时再对需要删除的block进行清除. 因此写入频繁的时候, 保留空间越多的ssd性能越好.
1. 使用trim

    trim在os层, os通过TRIM命令来通知ssd某个page的数据不需要了, 可以回收.

ssd读性能好于写, 且容量越大, 性能越好.

ssd监控是基于其SMART信息. windows不能有RAID卡, 因为RAID卡会屏蔽硬盘的SMART信息, 而linux不会.

```bash
smartctl -a -i /dev/sg0 # 低端板载的LSI MPT RAID卡, 读取第一块盘
smartctl -a -i /dev/sg1 # 读取第二块盘
smartctl -a -d megaRAID,0 /dev/sda # LSI MegaRAID卡
smartctl -a -d megaRAID,1 /dev/sda
smartctl -a -d sat_cciss,0 /dev/sda
smartctl -a -d sat_cciss,1 /dev/sda
```

ssd smart解读:
- Media_Wearout_indicator : 使用耗费, 100为没有任何耗费, 表示ssd上nand的擦写次数的程度, 初始值是100, 随着擦写次数的增加, 开始线性递减, 递减速度按照擦写次数从0到最大的比例. 一旦该值降为1, 表示不能再降了, 同时表示ssd上面已经有nand的擦写次数到达了最大次数. 此时建议备份数据, 更换ssd.
- Reallocated_Sector_Ct : 出厂后产生的坏块个数, 初始为100, 如果有坏块, 从1开始增加, 每4个坏块增1
- Host_Writes_32MiB: 已写入了N个`*32`M, 每写入65536个扇区(512B) raw value加1.
- Available_Reservd_Space: ssd上剩余的保留空间. 初始值是100, 表示100%; 阈值为10, 递减到10表示保留空间已经不能再减少.

## 块设备cache
flashcache是一个针对linux系统的块设备缓存工具, 综合了高/低设备的优势, 由device mapper实现, 原理是用高速设备做cache, 把数据先缓存到高速设备上, 到一定阈值, 再写到低速设备上.

DM-cache是一个device-mapper级别的， 用高速设备做低速设备cache的解决方案, 主要是用ssd做hdd的cache, 达到性能和容量的平衡.

lvm cache是在DM-cache上基于lvm做的配置.

写性能: flashcache>DM-cache(从kernel 3.9开始)>lvm cache

# linux实现
设备配置表, 总线, 驱动是linux设备架构的三大层次:
1. 设备配置表描述了设备本身物理特性, 包括设备的寄存器信息和内存信息
1. 总线(pcie物理总线, platform虚拟总线)是一个软件框架, 作用是作为一个容器, 把设备和驱动容纳在其中. 通过总线, 可以发现设备, 为设备发现驱动, 配置设备信息.

## 文件变设备
[init_special_inode](https://elixir.bootlin.com/linux/v5.12.10/source/fs/inode.c#L2115), 其参数rdev就是由主设备号和从设备号生成的设备号. 调用该函数后, 该inode变成了代表字符/块设备, fifo, socket的特殊inode.

linux提供mknod命令, 通过它用户可根据主从设备号创建特殊文件, 比如字符/块设备文件. 它的原理是:
1. 为特殊文件创建一个inode和dentry. inode的成员包含主从设备号和设备类型.
1. 调用init_special_inode为该inode设置不同的函数指针.

## 字符设备
字符设备的file_operations是[def_chr_fops](https://elixir.bootlin.com/linux/v5.12.10/source/fs/char_dev.c#L452), 重点是它的open函数: [chrdev_open](https://elixir.bootlin.com/linux/v5.12.10/source/fs/char_dev.c#L373).

### chrdev_open
根据设备号调用kobj_lookup搜索注册的字符设备对象, 如果找到就执行字符设备的open函数, 否则返回错误.

chrdev_open会调用设备驱动本身的open函数, 前提是已注册设备驱动.

### 以INPUT_MAJOR(13)设备号举例
> 一个键盘设备的驱动可分为input层, 虚拟键盘驱动(input_handler层), 真实键盘驱动层.

[input_init](https://elixir.bootlin.com/linux/v5.12.10/source/drivers/input/input.c#L2598)把input设备注册到系统. input_init最终会调用register_chrdev_region(MKDEV(INPUT_MAJOR, 0), INPUT_MAX_CHAR_DEVICES, "input")来注册.

区间是主设备号和从设备号共同占用的一段空间, register_chrdev_region要登记0~256的从设备号区间, 这个区间之前不能被占用. 登记区间由__register_chrdev_region实现.

__register_chrdev_region先会创建一个__register_chrdev_region结构, 然后要考虑输入的主设备号为0的情况, 为0时需要为字符设备分配一个主设备号.

[分配主设备号的算法](https://elixir.bootlin.com/linux/v5.12.10/source/fs/char_dev.c#L65)是从高到低遍历全局数组chrdevs, 如果某个主设备号为空, 则分配给字符设备. chrdevs是一个含255个元素的指针数组, 对应设备的主设备号. 如果输入的主设备号大于255, 则取其余数. chrdevs保存了所有的主设备号和从设备号.

__register_chrdev_region第二部分从chrdevs找到未占用的区间:
1. 通过主设备号索引获得char_device_struct
1. 遍历char_device_struct结构的单向链表, 依次比较从设备号, 找到一个合适的区间
1. 将创建的字符设备结构cd链接到单向链表, 完成字符设备区间的登记

### input设备架构
input设备是设备和驱动的封装. 由[input_register_handler](https://elixir.bootlin.com/linux/v5.12.10/source/drivers/input/input.c#L2393)注册input设备的驱动, [input_register_device](https://elixir.bootlin.com/linux/v5.12.10/source/drivers/input/input.c#L2259)注册input设备.

input框架的管理机制: input设备维护了两个链表, 一个是设备链表, 一个是handler链表, 注册一个驱动要和所有的设备一一匹配, 看是否适合.

input框架主要是为了复用代码, 简化其他层次的工作量. input框架提供事件处理函数input_event用于上报用户输入.

#### input_register_handler
它的最后一段是遍历所有注册的input设备, 检查能否和注册的handler匹配, 相关逻辑在[input_attach_handler](https://elixir.bootlin.com/linux/v5.12.10/source/drivers/input/input.c#L1026), 检查原理: 检查handler的id表是否和设备的id表相等.

如果handler和设备匹配, 调用handler的connect函数和设备建立连接.

#### 匹配input管理的设备和驱动
由[input_match_device](https://elixir.bootlin.com/linux/v5.12.10/source/drivers/input/input.c#L1011)实现. 它会逐个对比驱动的id表和设备的id表, 检查它们的总线类型, 制造商, 产品号和版本号, 以及事件类型是否相等.

#### input_register_device
1. 初始化设备, 将设备加入总的input设备链表. 这样, 通过链表就可以遍历所有的input设备. 初始化设备的timer, 定时时间到达时, 自动重复输入input设备的按键值.
1. 通过sysfs创建设备的属性文件 by [device_add](https://elixir.bootlin.com/linux/v5.12.10/source/drivers/base/core.c#L3130)
1. 这部分代码和input_register_handler最后的代码很像, 不过这次是遍历所有的驱动, 检查是否和设备匹配. 匹配算法: 检查驱动和设备的id表是否适合.