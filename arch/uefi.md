# uefi (nified Extensible Firmware Interface)
参考:
- [<<UEFI原理与编程>>]
- [*使用Rust开发操作系统(UEFI基本介绍)](https://blog.csdn.net/qq_41698827/article/details/104081406)
- [UEFI入门必读的12本书](https://www.eet-china.com/mp/a150514.html)

uefi定义了os与平台固件之间的接口, 是一种标准. tianocore是它的一种开源实现.

UEFI实现一般可分为两部分：
1. 平台初始化（遵循Platform Initialization标准，同样由UEFI Forum发布）
1. 固件-操作系统接口

UEFI能迅速取代BIOS，得益于UEFI相对BIOS的几大优势:
1. UEFI的开发效率

	BIOS开发一般采用汇编语言，代码多是硬件相关的代码. 而在UEFI中，绝大部分代码采用C语言编写，UEFI应用程序和驱动甚至可以使用C++编写. UEFI通过固件-操作系统接口（BS和RT服务）为OS和OS加载器屏蔽了底层硬件细节，使得UEFI上层应用可以方便重用.

1. 突破rom的容量限制

	可以解析fs, 使用磁盘空间作为自己的存储空间

1. UEFI系统的可扩展性

	UEFI系统的可扩展性体现在两个方面：一是驱动的模块化设计；二是软硬件升级的兼容性. 大部分硬件的初始化通过UEFI驱动实现. 每个驱动是一个独立的模块，可以包含在固件中，也可以放在设备上，运行时根据需要动态加载. UEFI中每个表、每个Protocol（包括驱动）都有版本号，这使得系统的平滑升级变得简单.
1. UEFI系统的性能

	相比BIOS，UEFI有了很大的性能提升，从启动到进入操作系统的时间大大缩短. 性能的提高源于以下几个方面：
	1. UEFI提供了异步操作. 基于事件的异步操作，提高了CPU利用率，减少了总的等待时间
	1. UEFI舍弃了中断这种比较耗时的操作外部设备的方式，仅仅保留了时钟中断. 外部设备的操作采用“事件＋异步操作”完成
	1. 可伸缩的遍历设备的方式，启动时可以仅仅遍历启动所需的设备，从而加速系统启动

1. UEFI系统的安全性

	UEFI的一个重要突破就是其安全方面的考虑. 当系统的安全启动功能被打开后，UEFI在执行应用程序和驱动前会先检测程序和驱动的证书，仅当证书被信任时才会执行这个应用程序或驱动. UEFI应用程序和驱动采用PE/COFF格式，其签名放在签名块中.

1. 易用性好

	支持gui, 鼠标

## 趋势
ref:
- [BIOSer必读：Intel的下一代BIOS标准USF](https://zhuanlan.zhihu.com/p/443610480)
- [下一代BIOS标准探讨引子：之各种Bootloader大比拼](https://zhuanlan.zhihu.com/p/354914114)

systemd-boot是为现代硬件设计的，Fedora也正在计划迁移到Systemd-boot放弃grub2. 以及英特尔计划在 2020 年结束终止支持 Legacy BIOS.

20年之后的现在，UEFI已经变得越来越传统，曾经的屠龙骑士变成了恶龙，业界呼唤新的方案，Intel也不失时机的提出了[ModernFW, **实验性的(已废弃)**](https://github.com/intel/ModernFW). ModernFW基于TianoCore, 是一种从TianoCore派生的轻量级实现, 是一种为云服务器平台之类的机器构建最低可行平台固件的实验方法.

后来intel又提出了[Universal Scalable Firmware (USF)](https://zhuanlan.zhihu.com/p/443610480)作为下一代BIOS标准.

## 概念
1. image: uefi定义的包含可执行代码的二进制程序
1. service: 调用接口的集合, 运行uefi程序和os调用. 这些接口由uefi应用程序, 驱动和uefi os loader提供
1. protocol: 本质是一种数据结构, 它包含全局唯一标志符, 接口数据结构和服务3个部分.
1. 系统表: 用于定位uefi protocol

	所有uefi image都会接收到一个指向uefi system table的指针来访问固件提供的uefi protocol.

uefi环境是开启分页的, 每页是4k.

## 组成
UEFI提供给操作系统的接口分为两部分:
1. 启动服务（Boot Services，BS）以及隐藏在BS之后的丰富的Protocol

	启动服务的主要服务对象是操作系统加载器以及其他UEFI应用程序和UEFI驱动.

	操作系统加载器通过启动服务逐步取得对整个计算机系统资源的控制。当加载器完全控制计算机软硬件资源后，系统结束启动服务，进入运行时.

1. 运行时服务（Runtime Service，RT）

	运行时服务的主要服务对象是操作系统、操作系统加载器以及UEFI应用和UEFI驱动.

BS和RT以表的形式（C语言中的结构体）存在. UEFI驱动和服务以Protocol的形式通过BS提供给操作系统.

![](/misc/img/arch/20160728202700202.png)

从操作系统加载器（OS  Loader）被加载，到OS  Loader执行ExitBootServices()的这段时间，是从UEFI环境向操作系统过渡的过程. 在这个过程中，OS  Loader可以通过BS和RT使用UEFI提供的服务，将计算机系统资源逐渐转移到自己手中，这个过程称为TSL（Transient System Load）.

当OS  Loader完全掌握了计算机系统资源时，BS也就完成了它的使命. OS  Loader调用ExitBootServices()结束BS并回收BS占用的资源，之后计算机系统进入UEFI Runtime阶段. 在Runtime阶段只有运行时服务继续为OS提供服务，BS已经从计算机系统中销毁.

在TSL阶段，系统资源通过BS管理，BS提供的服务如下:
1. Event服务：事件是异步操作的基础. 有了事件的支持，才可以在UEFI系统内执行并发操作
1. Timer服务: 配合Event提供定时器
1. 内存管理：主要提供内存的分配与释放服务，管理系统内存映射
1. Protocol管理：提供了安装Protocol与卸载Protocol的服务，以及注册Protocol通知函数（该函数在Protocol安装时调用）的服务
1. Protocol使用类服务：包括Protocol的打开与关闭，查找支持Protocol的控制器。例如要读写某个PCI设备的寄存器，可以通过OpenProtocol服务打开这个设备上的PciIo Protocol，用PciIo->Io.Read()服务可以读取这个设备上的寄存器
1. 驱动管理：包括用于将驱动安装到控制器的connect服务，以及将驱动从控制器上卸载的disconnect服务. 例如，启动时，如果需要网络支持，则可以通过loadImage将驱动加载到内存，然后通过connect服务将驱动安装到设备
1. Image管理：此类服务包括加载、卸载、启动和退出UEFI应用程序或驱动
1. 其他服务: 设置看门狗定时器, 延时, 计算crc等
1. ExitBootServices：用于结束启动服务

RT提供的服务主要包括如下几个方面:
1. 时间服务：提供读取和设置系统时间的功能, 也提供了读取和设定定时唤醒系统的功能
1. 读写UEFI系统变量：读取/设置系统变量，比如设置启动项顺序, efibootmgr就是通过调用此服务来实现的
1. 虚拟内存服务：将物理地址转换为虚拟地址
1. 其他服务：包括重启系统, 更新BIOS(Capsules服务), 交互os与bios的数据等各种服务

## UEFI系统的启动过程
UEFI系统的启动遵循UEFI平台初始化（PlatformInitialization）标准. UEFI系统从加电到关机可分为7个阶段：`SEC（安全验证）→PEI（EFI前期初始化）→DXE（驱动执行环境）→BDS（启动设备选择）→TSL（操作系统加载前期）→RT（Run Time）→AL（系统灾难恢复期）`

![](/misc/img/arch/20160728202835390.png)

前三个阶段是UEFI初始化阶段，DXE阶段结束后，UEFI环境已经准备完毕.
BDS和TSL是OS Loader作为UEFI应用程序运行的阶段.
OS Loader调用ExitBootServices()服务后进入RT阶段，该阶段包括OS Loader后期和OS运行期.
当系统硬件或OS出现严重错误不能正常运行时，固件会尝试修复错误，这时系统进入AL期, 但PI规范和UEFI规范都没有规定AL期的行为. "?"表示其行为由系统供应商自行定义.

### 1. SEC阶段
SEC（Security  Phase）阶段是平台初始化的第一个阶段，计算机系统加电(开机/重启)后进入这个阶段.

SEC阶段它执行以下4种任务:
1. 接收并处理系统启动和重启信号：系统加电信号、系统重启信号、系统运行过程中的严重异常信号
1. 初始化临时存储区域：系统运行在SEC阶段时，仅CPU和CPU内部资源被初始化，各种外部设备和内存都没有被初始化，因而系统需要一些临时RAM区域，用于代码和数据的存取，我们将之称为临时RAM，以示与内存的区别. 这些临时RAM只能位于CPU内部. 最常用的临时RAM是Cache，当Cache被配置为no-eviction模式时，可以作为内存使用，读命中时返回Cache中的数据，读缺失时不会向主存发出缺失事件；写命中时将数据写入Cahce，写缺失时不会向主存发出缺失事件，这种技术称为CAR（Cache As Ram）

	需要临时RAM区域的原因: 除了最初的一些汇编代码, SEC还有C代码需要堆栈, 在当前物理内存没有初始化时就临时使用cpu cache作为临时内, 该技术叫CAR(Cache As Ram)

	此时程序还是在rom上执行, 但可使用CAR.
1. 作为可信系统的根：作为取得对系统控制权的第一部分，SEC阶段是整个可信系统的根. SEC能被系统信任，以后的各个阶段才有被信任的基础. 通常，SEC在将控制权转移给PEI之前，可以验证PEI.
1. 传递系统参数给下一阶段（即PEI）：SEC阶段的一切工作都是为PEI阶段做准备，最终SEC要把控制权转交给PEI，同时要将现阶段的成果汇报给PEI. 汇报的手段就是将如
下信息作为参数传递给PEI的入口函数:

	1. 系统当前状态，PEI可以根据这些状态判断系统的健康状况
	1. 可启动固件（BFV, Boot Firmware Volume, 可启动固件）的地址和大小
	1. 临时RAM区域的地址和大小
	1. 栈的地址和大小

![SEC的执行流程](/misc/img/arch/20160728202857796.png)

code: `UefiCpuPkg`

> OvmfPkg\ResetVector\Ia32\PageTables64.asm和Flat32ToFlat64.asm(32平坦到64平坦), 调用时机未知

SEC阶段执行流程以临时RAM初始化为界，SEC的执行又分为两大部分：临时RAM生效之前称为Reset Vector阶段，临时RAM生效后调用SEC入口函数从而进入SEC功能区.

其中Reset Vector的执行流程如下:
1. 刷新cache, 进入固件入口
1. 从实模式转换到32位平坦模式（是不分页的平坦模式）
1. 定位固件中的BFV（Boot Firmware Volume）
1. 定位BFV中的SEC映像
1. 若是64位环境，从32位模式转换到64位模式
1. 调用SEC入口函数

> SEC会设置IDT处理各种情况以避免崩溃. SEC可执行微码升级, 以解决cpu缺陷.

### 2. PEI阶段
PEI（Pre-EFI  Initialization, 早期EFI初始化）阶段资源仍然十分有限，内存到了PEI后期才被初始化，其主要功能是为DXE准备执行环境，将需要传递到DXE的信息组成HOB（Handoff Block）列表，最终将控制权转交到DXE手中.

> HOB是一种用于交换的块, 负责阶段间的数据传递

> PEI会会初始化一部分永久内存并设置页表位, 为DXE服务.

![PEI执行流程](/misc/img/arch/20160728203103656.jpeg)

从功能上讲，PEI可分为以下两部分:
1. PEI 内核（PEI Foundation）：负责PEI基础服务和流程

	接收SEC发送的交换数据, 并扮演模块分发的角色

	PEI存在与BFV中, 它在SEC阶段已被验证过是否收到破坏. 它会建立一个系统表即PEI服务表(PEI Services Table), 所有的PEI模块都可以访问它
1. PEIM（PEI  Module）派遣器：主要功能是找出系统中的所有PEIM，并根据PEIM之间的依赖关系按顺序执行PEIM.

	PEI阶段对系统的初始化主要是由PEIM完成的. 每个PEIM是一个独立的模块, 封装了关于处理器, 芯片组, 设备或平台相关的功能.

	PEIM间通过PPI(PEIM to PEIM Interfaces)来实现.

	PEIM可通过 PEI 服务 InstallPPI() 和 ReinstallPPI() 来发布一个有效的 PPI，而其他 的 PEIM可使用 PEI 服务 LocatePPI() 来找到需要的 PPI.

PEI具体执行流程:
1. 接收从SEC传递过来的参数, 使用Cache作为内存
1. PEIM派遣器执行循环, 依次执行各PEIM, 包括CPU PEIM, 平台相关PEIM, 内存初始化PEIM.

	其中平台PEIM对一系列硬件进行早期的初始化, 包括内存控制器, i/o控制器, 以及初始化各种平台接口, 比如stall(延时), 重启等
1. 检查是否处于S3启动模式, 如果是, 执行S3返回流程; 否则准备HOB列表, 进入DXE入口

通过PeiServices，PEIM可以使用PEI阶段提供的系统服务，通过这些系统服务，PEIM可以访问PEI内核. PEIM之间的通信通过PPI（PEIM-to-PEIM Interfaces）完成.

PPI与DXE阶段的Protocol类似，每个PPI是一个结构体，包含了函数指针和变量.

每个PPI都有一个GUID. 根据GUID，通过PeiServices的LocatePpi服务可以得到GUID对应的PPI实例.

UEFI的一个重要特点是其模块化的设计. 模块载入内存后生成Image. Image的入口函数为`_ModuleEntryPoint`. PEI也是一个模块，PEI  Image的入口函数`_ModuleEntryPoint`，位于`MdePkg/Library/PeimEntryPoint/PeimEntryPoint.c`. `_ModuleEntryPoint`最终调用PEI模块的入口函数`PeiCore`，位于`MdeModulePkg/Core/Pei/PeiMain/PeiMain.c`. 进入PeiCore后，首先根据从SEC阶段传入的信息设置Pei  Core  Services，然后调用PeiDispatcher执行系统中的PEIM，当内存初始化后，系统会发生栈切换并重新进入PeiCore. 重新进入PeiCore后使用的内存为我们所熟悉的内存. 所有PEIM都执行完毕后，调用PeiServices的LocatePpi服务得到DXE  IPL  PPI，并调用DXE  IPL  PPI的Entry服务，这个Entry服务实际上是DxeLoadCore，它找出DXE  Image的入口函数，执行DXE  Image的入口函数并将HOB列表传递给DXE.

ps: PEI到DXE转换的最后部分的`Library/PeilessStartupLib/DxeLoad.c:  HandOffToDxeCore`的`CreateIdentityMappingPageTables`有注释`虚拟地址和物理地址是一一对应`, 其实还有另一次设置页表在`OvmfPkg\ResetVector\Ia32\PageTables64.asm`.

### 3. DXE阶段
DXE（Driver  Execution  Environment, 驱动执行环境）阶段加载硬件的基本驱动, 并执行大部分系统初始化工作, 为后续的uefi和os提供了uefi的系统表, 启用服务和运行时服务. 进入此阶段时，内存已经可以被完全使用，因而此阶段可以进行大量的复杂工作. 从程序设计的角度讲，DXE阶段与PEI阶段相似.

![执行流程](/misc/img/arch/20160728203131610.jpeg)

与PEI类似，从功能上讲，DXE可分为以下两部分:
1. DXE内核：它抽象于系统硬件, 负责DXE基础服务和执行流程, 创建uefi启用服务,运行时服务和DXE服务
1. DXE派遣器：负责调度执行DXE 驱动，初始化系统设备.
1. DXE驱动: 负责初始化处理器, 芯片组, 系统组件，所有的DXE驱动都可以使用uefi启动服务, 运行时服务或者DXE服务实现所需功能, 比如产生protocol.
	为更上层的接口提供基础支持, 比如显卡驱动就支持`Print()`

	驱动分两种: 
	1. 早期DXE驱动

		通常包含基础服务, 处理器/芯片组/平台初始化代码, 它会主动寻找设备并初始化它
	2. 符合UEFI驱动模型的驱动

		系统服务根据设备来寻找驱动, 找到合适的驱动再执行初始化

DXE提供的基础服务包括系统表、启动服务、Run Time Services. 每个DXE驱动是一个独立的模块. DXE驱动之间通过Protocol通信. Protocol是一种特殊的结构体，每个Protocol对应一个GUID，利用系统BootServices的OpenProtocol，并根据GUID来打开对应的Protocol，进而使用这个Protocol提供的服务.

当所有的Driver都执行完毕后，系统完成初始化，DXE通过EFI_BDS_ARCH_PROTOCOL找到BDS并调用BDS的入口函数，从而进入BDS阶段. 从本质上讲，BDS是一种特殊的DXE阶段的应用程序.

DXE具体执行流程:
1. 进入DXE入口
2. 根据HOB列表初始化系统服务
3. DXE派遣器调度遍历系统中的驱动, 直到所有满足条件的驱动都被加载
4. 进入BDS服务

### 4. BDS阶段
BDS（Boot Device Selection）的主要功能是执行启动策略，其主要功能包括:
1. 初始化控制台设备
1. 加载必要的设备驱动
1. 根据系统设置加载和执行启动项.

	生成开机的启动菜单: 寻找具有FAT32分区的设备

	如果加载启动项失败，系统将重新执行DXE  dispatcher以加载更多的驱动，然后重新尝试加载启动项

在DXE阶段加载的各个驱动, 会在本阶段与系统中的硬件进行匹配连接, 使得各个启动设备可以进行读写.

BDS策略通过全局NVRAM变量配置, os loader的路径就在NVRAM中. 这些变量可以通过运行时服务的GetVariable()读取，通过SetVariable()设置. 例如，变量BootOrder定义了启动顺序，变量Boot####定义了各个启动项（####为4个十六进制大写符号）.
用户选中某个启动项（或系统进入默认的启动项）后，OS Loader启动，系统进入TSL阶段.

如果BDS阶段启动失败, 系统将重新调用DXE派遣器, 再次进入寻找启动设备的流程.

### 5. TSL阶段
TSL（Transient System Load）是操作系统加载器（OS Loader）执行的第一阶段，在这一阶段OS Loader作为一个UEFI应用程序运行, 需要loader找到并加载os，此时系统资源仍然由UEFI内核控制. 当os loader调用ExitBootServices()并回收启用服务的资源后，系统进入Run Time阶段.

TSL阶段之所以称为临时系统，在于它存在的目的就是为操作系统加载器准备执行环境. 虽然是临时系统，但其功能已经很强大，已经具备了操作系统的雏形，UEFI  Shell是这个临时系统的人机交互界面. 正常情况下，系统不会进入UEFI  Shell，而是直接执行操作系统加载器，只有在用户干预下或操作系统加载器遇到严重错误时才会进入UEFI Shell.

> loader可使用BootService和RuntimeService; os只能使用RuntimeService.

> loader可使用BootService的Gop显示字符串或图片

loader属于APPLICATION, 普通APPLICATION使用完成后会退出, 但loader会调用`EFI_BOOT_SERVICES.ExitBootServices()`后再结束.

### 6. RT阶段
系统进入RT（Run Time）阶段后，系统的控制权从UEFI内核转交到OS  Loader手中，UEFI占用的各种资源被回收到OS  Loader，仅有UEFI运行时服务保留给OS  Loader和OS使用. 随着OS Loader的执行，OS最终取得对系统的控制权.

### 7. AL阶段
在RT阶段，如果系统（硬件或软件）遇到灾难性错误，系统固件需要提供错误处理和灾难恢复机制，这种机制运行在AL（After  Life）阶段. UEFI没有定义此阶段的行为和规范, 由系统供应商自行提供.

	关机, 休眠, 睡眠, 重启后会进入该阶段. 在服务器和工作站上, 异步事件(SMI/NMI中断)也可触发进入该阶段.

## 驱动模型
Legacy BIOS受限于实模式1MB内存, 能用的内存资源很少. Option Rom会包含16位汇编代码. Option Rom自己扫描总线上的外部设备, 与硬件关联, 根本原因是bios没有统一标准.

因此UEFI是一启动就先切到平坦模式, 切换模式的汇编代码在`UefiCpuPkg/ResetVector`. 它将cpu从16位模式切换到32位保护模式或长模式(如果支持).

UEFI对硬件, 功能, 驱动分别进行抽象, 各自的职责也划分的层次分明.

不同的硬件设备统一抽象为控制器(ControllerHandler), 控制器的主要责任就是与CPU的I/O.

ControllerHandler分:
- 总线控制器

	总线控制器下还可有连接总线	
- 终端控制器(叶子节点)

	叶子节点设备由设备控制器连接到总线, 通过总线控制器与cpu交互

cpu和设备就抽象成了一颗设备树. uefi把总线驱动固化在主板固件里, 该驱动负责发现连接在自身上的叶子节点设备, 创建控制器句柄. 同时UEFI也定义了叶子节点设备的驱动程序标准, 比如pci设备的驱动. 总线驱动发现符号uefi标准的节点设备驱动, 并加载到内存里. 驱动程序在内存中被称为一个image, 通过ImageHandle访问, 具体的访问方式就是protocol.

protocol是一种接口的合集, 分:
- 驱动protocol
- 固件功能protocol, 比如EFI_DRIVER_BINDING_PROTOCOL(用于绑定驱动和设备)

一个驱动程序可支持多个procotol. 每个procotol都有唯一guid.

uefi提供了大部分总线控制器的驱动程序, 以及大量的功能性procotols.

## tool
- efivar : 操作 UEFI 变量的库和工具 (被 efibootmgr 用到)
- efibootmgr — 操作 UEFI 固件启动管理器设置的工具

	启动管理器通过全局的NVRAM变量来管理加载.

## FAQ
### 是否以uefi启动
使用`ls /sys/firmware/efi/efivars`, 如果目录存在，则系统是以 UEFI 模式启动的.

### bootx64.efi/boot.efi
uefi固件会发现所有fat(不区分大小写)分区并加入启动菜单, 选中某个菜单, 就是检查其是否存在`efi/boot/bootx64.efi(64 os)或boot.efi(32 os)`并执行它. 如果该efi存在, 某些情况下会自动执行它, 比如qemu环境仅有一个fat分区.

# coreboot
参考:
- [Mainboards supported by coreboot](https://coreboot.org/status/board-status.html)

由于coreboot要初始化裸硬件，所以必须为所要支持的每个芯片组和主板移植. 因此而言，coreboot只适用于有限的硬件平台和主板型号.

# edk2
## 环境搭建
ref:
- [「Coding Tools」 第3话 Ubuntu下EDK2开发环境搭建](https://www.bilibili.com/read/cv12197402/)
- [Linux UEFI 学习环境搭建](https://martins3.github.io/uefi/uefi-linux.html)
- [我的第一支 edk2 Application](https://damn99.com/2020-05-18-edk2-first-app/)
- [Using EDK II with Native GCC](https://github.com/tianocore/tianocore.github.io/wiki/Using-EDK-II-with-Native-GCC)

```bash
# git clone -b <release_version> --depth 1 https://github.com/tianocore/edk2.git
# cd edk2
# git submodule update --init
# apt build-essential uuid-dev iasl git gcc nasm python-is-python3 # `build-essential uuid-dev` from `/BaseTools/ReadMe.rst`; iasl for OvmfPkg
# ln -s /usr/bin/python3.8 /usr/bin/python # 如果安装python3-distutils而不是python-is-python3就需要这句, 因为EDK2还是用的python2.x版本，而其命令是python
# make -C BaseTools # BaseTools contains all the tools required for building EDK II
# source edksetup.sh # 执行两次原因: `_omb_alias_general_cp_init：未找到命令`部分命令依赖edksetup.sh先设置env
# source edksetup.sh
# vim Conf/target.txt # 参考Conf/tools_def.txt, 定义了支持的编译工具链
ACTIVE_PLATFORM       = MdeModulePkg/MdeModulePkg.dsc # 想要编译的内容
TARGET_ARCH           = X64 # 取决于你要运行的 Guest 机器的架构
TOOL_CHAIN_TAG        = GCC5 # 关于编译器，官方文档反复强调是 gcc5，但是参考 [stackoverflow](https://stackoverflow.com/questions/63725239/build-edk2-in-linux)实际上系统中的 gcc 是 gcc9 或者 gcc10 也是无所谓的
MAX_CONCURRENT_THREAD_NUMBER = 9 # 这个取决于你的机器 CPU 核心数量
# build
ls Build/MdeModule/DEBUG_*/*/HelloWorld.efi # 构建出HelloWorld.efi
```

> 当ACTIVE_PLATFORM=OvmfPkg/OvmfPkgX64.dsc, build时可构建出OVMF.fd(`Build/OvmfX64/DEBUG_GCC5/FV/OVMF.fd`)

如果想要在 x86 电脑上编译安装 ARM 版本的 edk2, 其 Conf/target.txt 对应的配置为:
```
ACTIVE_PLATFORM       = ArmVirtPkg/ArmVirtQemu.dsc
TARGET_ARCH           = AARCH64
TOOL_CHAIN_TAG        = GCC5
MAX_CONCURRENT_THREAD_NUMBER = 9
```

并设置env:
```
export GCC5_AARCH64_PREFIX=aarch64-linux-gnu-          # ubuntu 中
```

### 构建
命令行编译platform pkg:
```bash
# build默认使用`Conf/target.txt`参数, 也支持指定
build -p $WORKSPACE/EmulatorPkg/EmulatorPkg.dsc -a X64 -b DEBUG -t GCC5 [-D BUILD_64 -D UNIX_SEC_BUILD] -n 3

option说明：
-p PLATFORMFILE: 目标平台描述文件
-a TARGETARCH: 目标平台X64/IA32(32位x86)
-b BUILDTARGET: 可选项（DEBUG, RELEASE, NOOPT），将只编译dsc文件中特定的模块
-m MODULEFILE: 编译目标module. 不指定时编译dsc中所有的模块
-t TOOLCHAIN : 使用目标编译器编译
-n THREADNUMBER : 多线程编译
-D MACROS: Macro格式: "Name [= Value]"，传入宏定义
```

编译结果在`edk2/Build`下.

将自己的uefi project放在edk2外编译容易遇到错误, 因此推荐将其放入edk2, 具体方法:
1. 将自己项目加入现有的edk2项目进行构建, 比如`OvmfPkg`, 参考[UEFI 基础教程 （一） - 运行第一个APP HelloWorld](https://blog.csdn.net/weixin_41028621/article/details/112546820)

	步骤:
	1. OvmfPkg/HelloWorld/HelloWorld.c

		```
		#include <Uefi.h>
		#include <Library/UefiLib.h>
		#include <Library/BaseLib.h>
		#include <Library/DebugLib.h>
		#include <Library/BaseMemoryLib.h>
		#include <Library/UefiBootServicesTableLib.h>

		//ShellCEntryLib call user interface ShellAppMain
		EFI_STATUS
		EFIAPI
		HelloWorldEntry(
		  IN EFI_HANDLE        ImageHandle,
		  IN EFI_SYSTEM_TABLE  *SystemTable
		)
		{
		  EFI_STATUS  Status = EFI_SUCCESS;
		  Print (L"[Console]  HelloWorldEntry Start..\n");

		  Print (L"[Console]  HelloWorldEntry  End ... \n");
		  return Status;
		}
		```
	2. OvmfPkg/HelloWorld/HelloWorld.inf
		```
		[Defines]
		  INF_VERSION = 0x00010007
		  BASE_NAME = HelloWorld
		  FILE_GUID = 69A6DE6D-FA9F-485E-9A4E-EA70FDCFD82F
		  MODULE_TYPE = UEFI_APPLICATION
		  VERSION_STRING = 1.0
		  ENTRY_POINT = HelloWorldEntry

		[Sources]
		  HelloWorld.c

		[Packages]
		  MdePkg/MdePkg.dec

		[LibraryClasses]
		  UefiApplicationEntryPoint
		  UefiLib
		```
	3. OvmfPkg/OvmfPkgX64.dsc
		```
		diff --git a/OvmfPkg/OvmfPkgX64.dsc b/OvmfPkg/OvmfPkgX64.dsc
		index c9235e4..c42708a 100644
		--- a/OvmfPkg/OvmfPkgX64.dsc
		+++ b/OvmfPkg/OvmfPkgX64.dsc
		@@ -661,6 +661,7 @@
		 ################################################################################
		 [Components]
		   OvmfPkg/ResetVector/ResetVector.inf
		+  OvmfPkg/HelloWorld/HelloWorld.inf
		```
	4. `build -a X64 -p OvmfPkg/OvmfPkgX64.dsc`
	5. 获取`Build/OvmfX64/DEBUG_GCC5/X64/HelloWorld.efi`
	6. 测试见`https://github.com/emvivre/uefi_hello_world.git`

1. 将自己项目放在edk2根目录进行构建, 参考[UEFI QEMU虚拟机下运行第一个APP HelloWorld图解](https://blog.csdn.net/Wang20122013/article/details/108737763)

	为了能用git管理自己的Pkg, 我是在edk2外建立Pkg, 再软链接到edk2根目录下

### Stdlib
ref:
- [使用 Rust 编写 UEFI Application](https://martins3.github.io/uefi/uefi-linux.html)

UEFI 提供了 StdLib, 其尽可能提供和 glibc 相同的接口，这样，很多用户态程序几乎不需要做任何修改就可以 直接编译为 .efi 文件，在 UEFI shell 中运行.

```bash
git clone https://github.com/tianocore/edk2-libc
mv edk2-libc/* path/to/edk2
cd path/to/edk2
build -p AppPkg/AppPkg.dsc
```

其实 edk2-libc 主要就是两个文件夹:
- StdLib : 利用 UEFI native 的接口实现 glib 的接口
- AppPkg : 各种测试程序，甚至包括 lua 解释器

### 如何让自动运行 efi 程序
UEFI 启动之后会自动执行 startup.nsh

在 edk2 中搜索 startup.nsh 可以找到 OvmfPkg/PlatformCI/PlatformBuildLib.py, 了解到 QEMU 可以通过参数实现`-drive file=fat:rw:${VirtualDrive},format=raw,media=disk`

> 在 shell 会等待 5s 来等待程序的执行, 可在 ShellPkg/Application/Shell/Shell.c 中修改为等待时间 0s

### qemu
```
cp ~/edk2/Build/MdeModule/DEBUG_GCC5/X64/HelloWorld.efi ~/run-ovmf/hda-contents/
qemu-system-x86_64 -bios OVMF.fd -hda fat:rw:hda-contents -net none
```

- `-hda fat:rw:hda-contents`=`-drive format=raw,file=fat:rw:hda-contents`
- `-bios OVMF.fd`=`-drive if=pflash,format=raw,file=OVMF_CODE.fd,readonly=on`

	建议启用uefi bios时追加`-drive if=pflash,format=raw,file=OVMF_VARS.fd`(建议单独拷贝一份OVMF_VARS.fd), 否则会在fat分区生成其他名称的OVMF_VARS, 比如NvVars

### gdb
ref:
- [Debugging OVMF with GDB](https://retrage.github.io/2019/12/05/debugging-ovmf-en.html)
- [我的第一支 edk2 Application](https://damn99.com/2020-05-18-edk2-first-app/)+[uefi.sh](https://github.com/Martins3/Martins3.github.io/blob/master/docs/uefi/uefi/uefi.sh)

具体步骤:
1. 在项目中添加调试代码

	1. 添加头文件: `#include <Library/DebugLib.h>`
	2. 在主函数UefiMain中添加获取程序入口地址的代码: `DEBUG ((EFI_D_INFO, "Entry point is 0x%08x\n", (CHAR16*)UefiMain));`
1. 编译ovmf和程序

	需要DEBUG模式

	将编译好的程序和OVMF.fd拷贝到hda-contents
1. 执行UEFI shell

	`qemu-system-x86_64 -s -pflash OVMF.fd -hda fat:rw:hda-contents -net none -debugcon file:debug.log -global isa-debugcon.iobase=0x402`

	在debug.log中获取加载驱动的指针地址(Loading driver point) 0x6841000.
1. gdb

	```bash
	# cd hda-contents
	# gdb
	(gdb) file HelloWorld.efi
	(gdb) info files
	根据输出的信息(可能需要根据段顺序调整偏移量)计算代码段和数据段的偏移地址
	text = 0x6841000 + <text offset> = 0x6841240
	date =  0x6841000 + <text offset> + <data offset> = 0x6843400
	(gdb) file # 卸载gdb加载的efi
	(gdb) add-symbol-file HelloWorld.debug 0x6841240 -s .data 0x6843400
	(gdb) target remote :1234
	(gdb) c
	```

	网上也有基于Intel UDK Debugger Tool的调试案例.

## examples
- [从零开始的UEFI裸机编程](https://kagurazakakotori.github.io/ubmp-cn/index.html)
- [luobing/uefi-practical-programming](https://github.com/luobing/uefi-practical-programming)
- [UEFI](https://www.bilibili.com/video/BV1HL4y1W7dJ)

	[code](https://gitee.com/tanyugang/UEFI)
- [xv6_uefi](https://gitee.com/naoki9911/xv6_uefi)

## src
- `*Pkg`: edk2的主体, 每个Pkg都是一个解决方案

	某些Pkg比如MdePkg提供了很多的接口, 充当了lib的角色

	说明:
	- dsc: Platform Description File, 是对整个包的描述

		section:
		- Defines, 必须

			定义变量， 以供后续编译使用
		- Compenents, 必须

			定义模块
		- LibraryClasses： 提供模块使用的库入, 这些库会被Compenents的模块使用

	- dec: Package Declaration File, 定义了公开的数据和接口, 其他Pkg可以调用它们, 是UEFI接口的实现. 每个包只用一个dec文件
	- inf: 描述具体工程

		- Defines, 必须: 描述了这个工程的名称, guid, 类型, 版本, 执行入口等
		- Sources: 工程用到的源代码以及字符串资源等文件的列表
		- Packages: 本工程需要引用的接口来自哪些Pkg即其DEC文件
		- Protocols: 模块使用的协议
		- LibraryClasses: 用到了哪些Pkg里的具体哪些库的接口
		- Pcd: 工程用到的全局字符串等信息, 是引用的DEC定义的内容
	- fdf: 描述固件在flash中的布局和位置
- BaseTools: 编译pkg所需的基本工具

	edk2有自己的工具链, 不使用系统的
- Build: Pkg的编译结果, 按Pkg, TARGET_ARCH, TOOL_CHAIN_TAG, TARGET存放

	build命令通过分析DSC, INF, DEC等工程文件, 生成编译所需的Makefile和AutoGen文件, 在配合源码, 使用make生成UEFI驱动或应用
- Conf: 保存配置

模块是UEFI上最小的可单独编译的代码单元，或者是预编译的二进制文件比如efi执行文件.

包由模块(可以没有)、平台描述文件（DSC)和包声明文件（DEC)组成.

### 模块类型
- UEFI_APPLICATION

	- UefiMain入口的应用

		可运行于DXE和UEFI Shell
	- ShellAppMain入口的应用

		可运行于UEFI Shell
	- main入口的应用

		使用StdLib编写的应用, 可运行于UEFI Shell

### Pkg
- EmulatorPkg: uefi模拟器
- ShellPkg: 可用于制作uefi bios的启动盘

	用`build -p ShellPkg/ShellPkg.dsc -t GCC5 -b RELEASE -a X64`将编译出的程序重命名为bootx32.efi/bootx64.efi, 放到fat32分区的efi/boot下即可, 开机按F11选择启动项

	> /usr/lib/systemd/boot/efi/systemd-bootx64.efi是systemd-boot
- OvmfPkg : x64相关代码以及特定的虚拟化代码，如virtio驱动
- ArmVirtPkg : ARM特定代码
- MdePkg, MdeModulePkg ： 主要核心代码，如PCI支持，USB至此和，通用服务和驱动等等
- PcAtChipsetPkg : Intel架构的驱动和库
- ArmPkg, ArmPlatformPkg : ARM架构支持代码
- CryptoPkg, NetworkPkg, FatPkg, CpuPkg, ... : 加密支持(使用openssl，网络支持(包括网络启动),FAT文件系统驱动等等

### 头文件
- Uefi.h: 定义了UEFI中的基本数据类型和核心数据结构
- Library/UefiLib.h: 提供通用的库函数, 包括时间, 简单锁, 任务优先级, 驱动管理和字符,图形显示输出等
- Library/BaseLib.h: 提供字符串处理, 数学, 文件路径处理等函数
- Library/BaseMemory.h: 处理内存的库函数, 包括内存拷贝, 内存填充, 内存清空等
- Library/DebugLib.h: 提供调试输出功能的函数

## 编译执行过程
build编译HelloWorld.c的过程：
1. HelloWorld.c 首先被编译成目标文件 HelloWorld.obj
1. 连接器将目标文件HelloWorld.c 和其它库连接成HelloWorld.dll(看build时的log就可见该过程)

	连接器在生成HelloWorld.dll时使用了`/dll/entry:_ModuleEntryPoint`. efi是遵循了PE32格式的二进制文件，`_ModuleEntryPoint`便是这个二进制文件的入口函数.

	efi是pe32+的变种, 主要是程序头略有不同.

	PE32+ Subsystem type确定了Image的子类型, 有3种:
	- APPLICATION(10): 应用程序
	- BOOT_SERVICE_DRIVER(11): 启动时驱动
	- RUNTIME_DRIVER(12)： 运行时驱动

	PE32+ Machine type确定了指令集架构
1. GenFw 工具将HelloWorld.dll 转化成 HelloWorld.efi


标准应用程序加载过程:
1. 将HelloWorld.efi 文件加载到内存
1. 当shell中执行HelloWorld.efi时，shell首先用gBS->LoadImage()将HelloWorld.efi文件加载到内存生成Image对象，然后调用gBS->StartImag(Image)启动这个Image对象。gBS->StartImage()是一个函数指针，它实际指向的是CoreStartImage()
1. 进入映像入口函数

	CoreStartImage()的主要作用是调用映像入口函数，在gBS->StartImage 的核心是Image->EntryPoint(···),它就是程序映像的入口函数，对应程序来说就是`_ModuleEntryPoint` 函数. 进入 `_ModuleEntryPoint` 后，控制权才转交给应用程序(HelloWorld.efi).
　　
	`_ModuleEntryPoint`主要处理三件事:
　　1. 初始化：初始化函数ProcessLibraryConstructorList中调用一系列构造函数
　　2. 调用本模块的入口函数 : ProcessModuleEntryPointList 中调用的是工程模块定义的入口函数
　　3. 析构：ProcessLibraryDestructorList 中调用一系列析构函数

	> 这三个对应的函数AutoGen.h,AutoGen.c中.

1.进入模块入口函数

	在ProcessModuleEntryPointList函数中调用了工程模块的真正入口函数UefiMain

### 接口说明
```
// https://uefi.org/specs/UEFI/2.10/07_Services_Boot_Services.html?highlight=getmemorymap#efi-boot-services-getmemorymap
// MdePkg/Include/Uefi/UefiSpec.h
typedef
EFI_STATUS
(EFIAPI \*EFI_GET_MEMORY_MAP) (
   IN OUT UINTN                  *MemoryMapSize,
   OUT EFI_MEMORY_DESCRIPTOR     *MemoryMap,
   OUT UINTN                     *MapKey,
   OUT UINTN                     *DescriptorSize,
   OUT UINT32                    *DescriptorVersion
  );
```

EFIAPI,EFI_STATUS,IN,OUT都是宏定义(在`MdePkg/Include/Base.h`), 只是简单定义一下, 没有具体值, 纯粹是说明性的. UINTN是数据类型(在`MdePkg/Include/X64/ProcessorBind.h`).

```c
#include <Uefi.h>

EFI_STATUS EFIAPI UefiMain(
	IN EFI_HANDLE ImageHandle, // image句柄
	IN EFI_SYSTEM_TABLE *SystemTable) // 系统表指针
{
	SystemTable->ConOut->OutputString(SystemTable->ConOut,L"Hello World\n");
	return EFI_SUCCESS;
}
```
ImageHandle指向模块自身加载到内存的image对象.

SystemTable是系统表指针, SystemTable是全局实例, 且仅存在一个, 定义在`MdePkg/Libaray/UefiBootServicesTableLib/UefiBootServicesTableLib.c`. 它是uefi应用和uefi内核交互的桥梁.

> EFI_BOOT_SERVICES其实全局也只有一份实例.

> uefi字符串是双字节, 即有`L`前缀的原因

## uefi shell
ref:
- [UEFI Shell commands](https://techlibrary.hpe.com/docs/iss/proliant_uefi/UEFI_TM_030617/GUID-D7147C7F-2016-0901-0A6D-000000000E1B.html)

- memmap: 查看内存map

	`fs0:`+`memmap > memmap.out`即可在fs0得到输出

## FAQ
### 显卡
真机环境, uefi优先初始化并使用集显.

因为固件只加载必要的设备.

### ovmf
ref:
- [编译QEMU+OVMF(ARM架构)](https://cloud-atlas.readthedocs.io/zh_CN/latest/kvm/arm_kvm/build_qemu_ovmf.html)

	支持多个构建选项, 可见OvmfPkgX64.dsc的if逻辑

编译x64的UEFI镜像:
```conf
# build -t GCC5 -a X64 -p OvmfPkg/OvmfPkgX64.dsc
ACTIVE_PLATFORM       = OvmfPkg/OvmfPkgX64.dsc
TARGET_ARCH           = X64
TOOL_CHAIN_TAG        = GCC5
````

编译ARM 64位的UEFI镜像:
```conf
# build -t GCC5 -a AARCH64 -p ArmVirtPkg/ArmVirtQemu.dsc
ACTIVE_PLATFORM       = ArmVirtPkg/ArmVirtQemu.dsc
TARGET_ARCH           = AARCH64
TOOL_CHAIN_TAG        = GCC5
```

x86 firmware build会创建3各不同镜像:
- OVMF_VARS.fd : 这是持久化的UEFI变量的firmware卷, 即firmware存储所有配置(引导条目和引导顺序、安全引导密钥等).

	通常这个文件用作空变量存储的模板，每个VM都有自己的私有副本. 例如 Libvirt虚拟机管理器 将文件存储在 /var/lib/libvirt/qemu/nvram 中
- OVMF_CODE.fd : 带有代码的firmware卷, 将它和 VARS 分开可以:

	- 确保轻松更新固件
	- 允许将只读代码映射到guest操作系统

- OVMF.fd: 是包含 CODE 和 VARS 的一体化镜像, 这样就可以使用`-bios`直接作为ROM加载, 但是有2个缺点:
	
	- UEFI 变量不是持久的
	- 不适用于 SMM_REQUIRE=TRUE 构建

