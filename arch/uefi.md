# uefi (nified Extensible Firmware Interface)
参考:
- [<<UEFI原理与编程>>]
- [*使用Rust开发操作系统(UEFI基本介绍)](https://blog.csdn.net/qq_41698827/article/details/104081406)

uefi定义了os与平台固件之间的接口, 是一种标准. tianocore是它的一种开源实现.

UEFI实现一般可分为两部分：
1. 平台初始化（遵循Platform Initialization标准，同样由UEFI Forum发布）
1. 固件-操作系统接口

UEFI能迅速取代BIOS，得益于UEFI相对BIOS的几大优势:
1. UEFI的开发效率

	BIOS开发一般采用汇编语言，代码多是硬件相关的代码. 而在UEFI中，绝大部分代码采用C语言编写，UEFI应用程序和驱动甚至可以使用C++编写. UEFI通过固件-操作系统接口（BS和RT服务）为OS和OS加载器屏蔽了底层硬件细节，使得UEFI上层应用可以方便重用.

1. UEFI系统的可扩展性

	UEFI系统的可扩展性体现在两个方面：一是驱动的模块化设计；二是软硬件升级的兼容性. 大部分硬件的初始化通过UEFI驱动实现. 每个驱动是一个独立的模块，可以包含在固件中，也可以放在设备上，运行时根据需要动态加载. UEFI中每个表、每个Protocol（包括驱动）都有版本号，这使得系统的平滑升级变得简单.
1. UEFI系统的性能

	相比BIOS，UEFI有了很大的性能提升，从启动到进入操作系统的时间大大缩短. 性能的提高源于以下几个方面：
	1. UEFI提供了异步操作. 基于事件的异步操作，提高了CPU利用率，减少了总的等待时间
	1. UEFI舍弃了中断这种比较耗时的操作外部设备的方式，仅仅保留了时钟中断. 外部设备的操作采用“事件＋异步操作”完成
	1. 可伸缩的遍历设备的方式，启动时可以仅仅遍历启动所需的设备，从而加速系统启动

1. UEFI系统的安全性

	UEFI的一个重要突破就是其安全方面的考虑. 当系统的安全启动功能被打开后，UEFI在执行应用程序和驱动前会先检测程序和驱动的证书，仅当证书被信任时才会执行这个应用程序或驱动. UEFI应用程序和驱动采用PE/COFF格式，其签名放在签名块中.

## 趋势
systemd-boot是为现代硬件设计的，Fedora也正在计划迁移到Systemd-boot放弃grub2. 以及英特尔计划在 2020 年结束终止支持 Legacy BIOS.

20年之后的现在，UEFI已经变得越来越传统，曾经的屠龙骑士变成了恶龙，业界呼唤新的方案，Intel也不失时机的提出了[ModernFW, **实验性的**](https://github.com/intel/ModernFW). ModernFW基于TianoCore, 是一种从TianoCore派生的轻量级实现, 是一种为云服务器平台之类的机器构建最低可行平台固件的实验方法.

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
1. 事件服务：事件是异步操作的基础. 有了事件的支持，才可以在UEFI系统内执行并发操作
1. 内存管理：主要提供内存的分配与释放服务，管理系统内存映射
1. Protocol管理：提供了安装Protocol与卸载Protocol的服务，以及注册Protocol通知函数（该函数在Protocol安装时调用）的服务
1. Protocol使用类服务：包括Protocol的打开与关闭，查找支持Protocol的控制器。例如要读写某个PCI设备的寄存器，可以通过OpenProtocol服务打开这个设备上的PciIo Protocol，用PciIo->Io.Read()服务可以读取这个设备上的寄存器
1. 驱动管理：包括用于将驱动安装到控制器的connect服务，以及将驱动从控制器上卸载的disconnect服务. 例如，启动时，如果需要网络支持，则可以通过loadImage将驱动加载到内存，然后通过connect服务将驱动安装到设备。6）Image管理：此类服务包括加载、卸载、启动和退出UEFI应用程序或驱动
1. ExitBootServices：用于结束启动服务

RT提供的服务主要包括如下几个方面:
1. 时间服务：读取/设定系统时间. 读取/设定系统从睡眠中唤醒的时间
1. 读写UEFI系统变量：读取/设置系统变量，例如BootOrder用于指定启动项顺序. 通过这些系统变量可以保存系统配置
1. 虚拟内存服务：将物理地址转换为虚拟地址
1. 其他服务：包括重启系统的ResetSystem，获取系统提供的下一个单调单增值等

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
1. 作为可信系统的根：作为取得对系统控制权的第一部分，SEC阶段是整个可信系统的根. SEC能被系统信任，以后的各个阶段才有被信任的基础. 通常，SEC在将控制权转移给PEI之前，可以验证PEI.
1. 传递系统参数给下一阶段（即PEI）：SEC阶段的一切工作都是为PEI阶段做准备，最终SEC要把控制权转交给PEI，同时要将现阶段的成果汇报给PEI. 汇报的手段就是将如
下信息作为参数传递给PEI的入口函数:

	1. 系统当前状态，PEI可以根据这些状态判断系统的健康状况
	1. 可启动固件（Boot Firmware Volume）的地址和大小
	1. 临时RAM区域的地址和大小
	1. 栈的地址和大小

![SEC的执行流程](/misc/img/arch/20160728202857796.png)

SEC阶段执行流程以临时RAM初始化为界，SEC的执行又分为两大部分：临时RAM生效之前称为Reset Vector阶段，临时RAM生效后调用SEC入口函数从而进入SEC功能区.

其中Reset Vector的执行流程如下:
1. 进入固件入口
1. 从实模式转换到32位平坦模式（包含模式）
1. 定位固件中的BFV（Boot Firmware Volume）
1. 定位BFV中的SEC映像
1. 若是64位系统，从32位模式转换到64位模式
1. 调用SEC入口函数

### 2. PEI阶段
PEI（Pre-EFI  Initialization）阶段资源仍然十分有限，内存到了PEI后期才被初始化，其主要功能是为DXE准备执行环境，将需要传递到DXE的信息组成HOB（Handoff Block）列表，最终将控制权转交到DXE手中.

![PEI执行流程](/misc/img/arch/20160728203103656.jpeg)

从功能上讲，PEI可分为以下两部分:
1. PEI 内核（PEI Foundation）：负责PEI基础服务和流程
1. PEIM（PEI  Module）派遣器：主要功能是找出系统中的所有PEIM，并根据PEIM之间的依赖关系按顺序执行PEIM. PEI阶段对系统的初始化主要是由PEIM完成的.

	每个PEIM是一个独立的模块

通过PeiServices，PEIM可以使用PEI阶段提供的系统服务，通过这些系统服务，PEIM可以访问PEI内核. PEIM之间的通信通过PPI（PEIM-to-PEIM Interfaces）完成.

PPI与DXE阶段的Protocol类似，每个PPI是一个结构体，包含了函数指针和变量.

每个PPI都有一个GUID. 根据GUID，通过PeiServices的LocatePpi服务可以得到GUID对应的PPI实例.

UEFI的一个重要特点是其模块化的设计. 模块载入内存后生成Image. Image的入口函数为`_ModuleEntryPoint`. PEI也是一个模块，PEI  Image的入口函数`_ModuleEntryPoint`，位于`MdePkg/Library/PeimEntryPoint/PeimEntryPoint.c`. `_ModuleEntryPoint`最终调用PEI模块的入口函数`PeiCore`，位于`MdeModulePkg/Core/Pei/PeiMain/PeiMain.c`. 进入PeiCore后，首先根据从SEC阶段传入的信息设置Pei  Core  Services，然后调用PeiDispatcher执行系统中的PEIM，当内存初始化后，系统会发生栈切换并重新进入PeiCore. 重新进入PeiCore后使用的内存为我们所熟悉的内存. 所有PEIM都执行完毕后，调用PeiServices的LocatePpi服务得到DXE  IPL  PPI，并调用DXE  IPL  PPI的Entry服务，这个Entry服务实际上是DxeLoadCore，它找出DXE  Image的入口函数，执行DXE  Image的入口函数并将HOB列表传递给DXE.

### 3. DXE阶段
DXE（Driver  Execution  Environment）阶段执行大部分系统初始化工作，进入此阶段时，内存已经可以被完全使用，因而此阶段可以进行大量的复杂工作. 从程序设计的角度讲，DXE阶段与PEI阶段相似.

![执行流程](/misc/img/arch/20160728203131610.jpeg)

与PEI类似，从功能上讲，DXE可分为以下两部分:
1. DXE内核：负责DXE基础服务和执行流程
1. DXE派遣器：负责调度执行DXE 驱动，初始化系统设备.

DXE提供的基础服务包括系统表、启动服务、Run Time Services. 每个DXE驱动是一个独立的模块. DXE驱动之间通过Protocol通信. Protocol是一种特殊的结构体，每个Protocol对应一个GUID，利用系统BootServices的OpenProtocol，并根据GUID来打开对应的Protocol，进而使用这个Protocol提供的服务.

当所有的Driver都执行完毕后，系统完成初始化，DXE通过EFI_BDS_ARCH_PROTOCOL找到BDS并调用BDS的入口函数，从而进入BDS阶段. 从本质上讲，BDS是一种特殊的DXE阶段的应用程序.

### 4. BDS阶段
BDS（Boot Device Selection）的主要功能是执行启动策略，其主要功能包括:
1. 初始化控制台设备
1. 加载必要的设备驱动
1. 根据系统设置加载和执行启动项.

	如果加载启动项失败，系统将重新执行DXE  dispatcher以加载更多的驱动，然后重新尝试加载启动项

BDS策略通过全局NVRAM变量配置. 这些变量可以通过运行时服务的GetVariable()读取，通过SetVariable()设置. 例如，变量BootOrder定义了启动顺序，变量Boot####定义了各个启动项（####为4个十六进制大写符号）.
用户选中某个启动项（或系统进入默认的启动项）后，OS Loader启动，系统进入TSL阶段.

### 5. TSL阶段
TSL（Transient System Load）是操作系统加载器（OS Loader）执行的第一阶段，在这一阶段OS Loader作为一个UEFI应用程序运行，系统资源仍然由UEFI内核控制. 当启动服务的ExitBootServices()服务被调用后，系统进入Run Time阶段.

TSL阶段之所以称为临时系统，在于它存在的目的就是为操作系统加载器准备执行环境. 虽然是临时系统，但其功能已经很强大，已经具备了操作系统的雏形，UEFI  Shell是这个临时系统的人机交互界面. 正常情况下，系统不会进入UEFI  Shell，而是直接执行操作系统加载器，只有在用户干预下或操作系统加载器遇到严重错误时才会进入UEFI Shell.

### 6. RT阶段
系统进入RT（Run Time）阶段后，系统的控制权从UEFI内核转交到OS  Loader手中，UEFI占用的各种资源被回收到OS  Loader，仅有UEFI运行时服务保留给OS  Loader和OS使用. 随着OS Loader的执行，OS最终取得对系统的控制权.

### 7. AL阶段
在RT阶段，如果系统（硬件或软件）遇到灾难性错误，系统固件需要提供错误处理和灾难恢复机制，这种机制运行在AL（After  Life）阶段. UEFI和UEFI  PI标准都没有定义此阶段的行为和规范.

## tool
- efivar : 操作 UEFI 变量的库和工具 (被 efibootmgr 用到)
- efibootmgr — 操作 UEFI 固件启动管理器设置的工具

## FAQ
### 是否以uefi启动
使用`ls /sys/firmware/efi/efivars`, 如果目录存在，则系统是以 UEFI 模式启动的.

# coreboot
参考:
- [Mainboards supported by coreboot](https://coreboot.org/status/board-status.html)

由于coreboot要初始化裸硬件，所以必须为所要支持的每个芯片组和主板移植. 因此而言，coreboot只适用于有限的硬件平台和主板型号.