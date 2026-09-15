# boot
PC启动过程
1. 按下电源键, 主板发送信号给电源进行开机
1. 电源收到信号后, 会给PC组件提供合适的电量, 之后向主板发送[`Power good signal`](https://en.wikipedia.org/wiki/Power_good_signal)
1. 主板触发CPU启动
1. CPU执行复位操作

    清除寄存器中残留的数据, 并将预设值加载到寄存器中, 为执行第一条指令做好准备

    > 为了与早期处理器兼容, 从最初的 8086 到如今的 64 位 CPU, 所有 x86 兼容处理器都支持实模式.

## boot linux
有多种不同的引导加载程序（bootloader）可以引导 Linux 内核，例如 GRUB 2、syslinux、systemd-boot 等。Linux 内核定义了一套[引导协议](https://github.com/torvalds/linux/blob/master/Documentation/arch/x86/boot.rst)，规定了引导加载程序在实现 Linux 支持时必须满足的要求.

根据相关文档，引导加载程序必须将内核加载到内存中，填充内核设置头（kernel setup header）中的某些字段，并将控制权移交给内核代码.

内核设置头是一个嵌入在 Linux 早期引导代码中的特殊结构，其中包含描述内核应如何加载和启动的字段。设置头（setup header）位于内核映像起始处偏移量 [0x01F1 的位置](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L70). kernel setup header的定义于 [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L234).

内核设置代码的执行始于 `arch/x86/boot/header.S` 文件中的 `_start` 符号处.

# REF
ref:
- [UEFI 中的杂项知识总结-Protocol Handle 机制的详细介绍](https://www.cnblogs.com/ayuan01/p/19328857/uefi-Protocol-Handle)

## x86 cpu历史
- 8086

    - 位宽: 16位
    - 地址总线: 20位, 范围0x00000~0xFFFFF, 大小1MB
    - 寻址: 内存分段寻址, 每段64KB(2^16)
    
        Physical Address =`cs:ip`= Segment Selector << 4 + Offset

        如果段选择符和偏移量均取最大可能值即cs:ip=0xFFFF:0xFFFF，则得到的地址为: `0x10ffef`, 它位于 1 MB 边界之后 65520 字节处. 由于在 8086 CPU 的实模式下，CPU 只能访问前 1 MB 的内存，因此任何大于 0xFFFFF 的地址都会回绕到地址空间的开头即0x00ffef. 在现代 386 及更高版本的 CPU 中，即使在实模式下，物理总线也更宽，但地址计算仍然基于段:偏移量

## 查看cpu执行第一条指令前的寄存器状态
`qemu-system-x86_64 -M q35 -S [-monitor stdio] -s`, `-S` 让 CPU 在第一条指令前暂停; `-monitor stdio`将 QEMU 监视器重定向到当前终端; `-s`使用gdb远程调试

按Ctrl-Alt-2进入qemu monitor(Ctrl-Alt-1, 切回qemu screen), 拷贝monitor内容可用窗口`view`的`copy`:
```bash
QEMU 10.2.1 monitor - type 'help' for more information
(qemu) info registers

CPU#0
EAX=00000000 EBX=00000000 ECX=00000000 EDX=00060fb1
ESI=00000000 EDI=00000000 EBP=00000000 ESP=00000000
EIP=0000fff0 EFL=00000002 [-------] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0000 00000000 0000ffff 00009300
CS =f000 ffff0000 0000ffff 00009b00
SS =0000 00000000 0000ffff 00009300
DS =0000 00000000 0000ffff 00009300
FS =0000 00000000 0000ffff 00009300
GS =0000 00000000 0000ffff 00009300
LDT=0000 00000000 0000ffff 00008200
TR =0000 00000000 0000ffff 00008b00
GDT=     00000000 0000ffff
IDT=     00000000 0000ffff
CR0=60000010 CR2=00000000 CR3=00000000 CR4=00000000
DR0=0000000000000000 DR1=0000000000000000 DR2=0000000000000000 DR3=0000000000000000 
DR6=00000000ffff0ff0 DR7=0000000000000400
EFER=0000000000000000
FCW=037f FSW=0000 [ST=0] FTW=00 MXCSR=00001f80
FPR0=0000000000000000 0000 FPR1=0000000000000000 0000
FPR2=0000000000000000 0000 FPR3=0000000000000000 0000
FPR4=0000000000000000 0000 FPR5=0000000000000000 0000
FPR6=0000000000000000 0000 FPR7=0000000000000000 0000
XMM00=0000000000000000 0000000000000000 XMM01=0000000000000000 0000000000000000
XMM02=0000000000000000 0000000000000000 XMM03=0000000000000000 0000000000000000
XMM04=0000000000000000 0000000000000000 XMM05=0000000000000000 0000000000000000
XMM06=0000000000000000 0000000000000000 XMM07=0000000000000000 0000000000000000
(qemu) xp /16xb 0xfffffff0
00000000fffffff0: 0xea 0x5b 0xe0 0x00 0xf0 0x30 0x36 0x2f
00000000fffffff8: 0x32 0x33 0x2f 0x39 0x39 0x00 0xfc 0x00
# 打印0xffff0处的指令. i,格式是指令. 不知道为什么AMD cpu + QEMU 10.2.1这里会报错`0x000ffff0: Asm output not supported on this arch`, 这时可使用gdb(可能还是无法识别)/直接看内存内容
# intel cpu + QEMU 10.2.1 + fedora 44, 能正常反编译指令, 但使用gdb也无法识别
(qemu) xp/1i 0xffff0
0x000ffff0:  ea 5b e0 00 f0           ljmpw    $0xf000:$0xe05b
(qemu) xp/1i 0xfffffff0
0xfffffff0:  ea 5b e0 00 f0           ljmpw    $0xf000:$0xe05b
(qemu) c # 继续执行
```

80386 及更高版本的 CPU 在硬件复位后会设置以下寄存器值：
- ip 0xFFF0 指令指针；执行从当前代码段的此处开始
- cs（选择器） 0xF000 复位后可见的代码段选择器值
- cs（基址） 0xFFFF0000 复位期间加载到 cs 中的隐藏描述符基址

0xffff0000 + 0xfff0 = 0xfffffff0, 它比 4GB 低 16 个字节, 这是 CPU 复位后开始执行的第一个地址, 也叫复位向量. 通常，它包含一条跳转 (jmp) 指令，指向 BIOS 或 UEFI 的入口点.

`jmp f000:e05b` 的编码是：EA 5B E0 00 F0，首先是disas_insn()函数翻译指令，当碰到第1个字节EA，分析可知这是一条16位无条件跳转指令，因此依次从后续字节中得到 offset和selector.

这个地址属于主板上的 SPI Flash 里的 UEFI 固件镜像（通过芯片组如 Intel PCH 映射到 4GB 地址空间顶部，通常 4-16MB 大小）。0xFFFFFFF0 处仍是一条 jmp 指令（通常是 jmp far 或短跳转），跳到 UEFI 固件的真正入口（Reset Vector Code，位于固件镜像末尾附近）

地址 `0xFFFFFFF0` 远大于 `0xFFFFF`（即 1MB）. 较新的处理器虽然以实模式启动，但配备了 32 位或 64 位总线, CPU 启动后，会读取位于 `0xFFFFFFF0` 地址处的跳转指令并跳转到固件，从而开启漫长的引导过程.

gdb:
```bash
gdb -ex "target remote localhost:1234" -ex "set architecture i8086" -ex "hbreak *0xFFFFFFF0"
```

实际操作发现, 还是无法查看反汇编, 因为QEMU 上报的架构是 i386:x86-64，但 GDB 的 i8086 架构设置与它冲突. GDB 在连接后，会从 target（QEMU）获取架构信息，而set architecture i8086 尝试覆盖它，但被 QEMU 的 reported architecture 否决了. 可用在 GDB 中用 print 或 x 直接查看物理内存（绕过架构问题）.

## BIOS vs UEFI
BIOS：传统 x86 PC 固件，16 位实模式，功能有限
UEFI: 现代标准固件，支持 32/64 位，模块化、可扩展，有驱动模型和文件系统支持。

ARM 平台一般不用 BIOS，直接使用 BootROM + 自定义 Bootloader 或 ARM Trusted Firmware (ATF) + UEFI.

由于 BIOS 的概念深入人心，如今我们一般称传统的固件叫做 Legacy BIOS，现代固件叫做 UEFI BIOS

## OS Loader
Bootloader 是一段在操作系统内核运行前执行的代码，负责初始化硬件、加载内核、传递启动参数，在不同平台上名称和层级不同.

常见平台和系统使用的 OS Loader 示例：
平台类型	架构	操作系统	典型启动流程
嵌入式	ARM	Linux	BootROM → SPL → U-Boot → Kernel
PC	x86	Linux	BIOS/UEFI → OS Loader (GRUB/systemd-boot) → Kernel
PC	x86	Windows	UEFI → Windows Boot Manager → winload.efi → NTOSKRNL
嵌入式/PC	ARM	Windows（少见）	ARM64 UEFI → bootmgfw.efi → Windows Kernel

不同平台的启动流程对比
1. 嵌入式 Linux（ARM 架构，如 Raspberry Pi, i.MX6）

    BootROM（芯片内置） 
    → SPL（可选，如 U-Boot SPL） 
    → U-Boot（主 Bootloader） 
    → Linux Kernel（zImage/Image + dtb + initramfs） 
    → 用户空间（init）

    嵌入式设备启动的特点：

    - 无标准固件：不像 PC 有 BIOS/UEFI，依赖 SoC 厂商提供的 BootROM。
    - BootROM：固化在芯片内部ROM中的早期启动代码，上电后自动从预设介质（SD/eMMC/NAND/SPI Flash）加载第一段代码（通常是 SPL 或直接 U-Boot）。
    - U-Boot：最常用的嵌入式 Bootloader，支持命令行、脚本、设备树（DTB）传递。
    - 设备树（Device Tree）：ARM Linux 必须通过 Bootloader 传递 .dtb 文件描述硬件。
    - 无 ExitBootServices()：因为没有 UEFI，直接跳转到内核

1. PC 上的 Linux（x86/x86_64 架构，UEFI 模式）

    UEFI Firmware 
    → OS Loader（如 GRUB2 / systemd-boot / shim.efi） 
    → Linux Kernel（vmlinuz + initrd） 
    → 用户空间
    
    桌面端 Linux 启动特点

    - 标准化固件：UEFI 提供统一接口（如 EFI System Partition, ESP）。
    - ESP 分区：FAT32 格式，存放 .efi 可执行文件（如 grubx64.efi）。
    - OS Loader：负责加载内核和 initrd，解析 /etc/default/grub 等配置。
    - 无需设备树：x86 硬件信息通过 ACPI 表传递。
    - 调用 ExitBootServices()：OS Loader 在跳转内核前关闭 UEFI Boot Services。
    - BIOS 模式（Legacy）已逐渐淘汰，流程为：BIOS → MBR → GRUB Stage1/Stage2 → Kernel

- PC 上的 Windows（x86/x86_64，UEFI 模式）

    UEFI Firmware 
    → \EFI\Microsoft\Boot\bootmgfw.efi（Windows Boot Manager） 
    → winload.efi 
    → ntoskrnl.exe（Windows 内核） 
    → Session Manager (smss.exe) → 用户登录

    桌面端 Windows 启动特点:
    - 完全依赖 UEFI（现代 Windows 不再支持纯 Legacy BIOS 安装）。
    - Secure Boot：验证 bootmgfw.efi 和 winload.efi 的数字签名。
    - BCD（Boot Configuration Data）：替代旧的 boot.ini，存储在 ESP 或系统分区。
    - 同样调用 ExitBootServices()：由 winload.efi 完成。
    
    注意：Windows 的 Boot Manager 本身就是一个 UEFI 应用（.efi 文件）。

- ARM 架构的 Windows

    目前 ARM 架构与 Windows 的组合在市场上还不常见，但有，可通过转译实现，部分应用能够原生运行。目前较有前景的芯片例子：骁龙X Elite。

    ARM64 UEFI Firmware（由 SoC 提供，如 Qualcomm Snapdragon） 
    → bootmgfw.efi 
    → winload.efi 
    → ntoskrnl.exe
    
    与 x86 Windows 启动流程几乎一致，只是架构为 ARM64。
    需要 ARM64 版本的 UEFI 固件（通常由 OEM 集成在 SoC 中）。
    微软要求 Secure Boot 和 UEFI 支持。

- ARM64 服务器/开发板

    启动流程类似嵌入式 ARM Linux，但部分高端 ARM 服务器支持 UEFI + ACPI（而非设备树）。
    例如：EDK II UEFI → GRUB for ARM64 → vmlinuz（使用 ACPI 表）

    低端 ARM（嵌入式）：U-Boot + Device Tree
    高端 ARM（服务器/PC）：UEFI + ACPI（更接近 x86 PC 模式）

总结：

维度	嵌入式 Linux (ARM)	PC Linux (x86 UEFI)	PC Windows (x86 UEFI)	ARM PC (Windows/Linux)
固件	BootROM（厂商定制）	标准 UEFI	标准 UEFI	ARM64 UEFI（OEM 提供）
Bootloader	U-Boot / Barebox	GRUB / systemd-boot	bootmgfw.efi	U-Boot（嵌入式）或 UEFI + GRUB/bootmgfw
配置存储	环境变量（U-Boot）	grub.cfg / kernel cmdline	BCD	BCD（Win）或 U-Boot env（Linux）
硬件描述	Device Tree (.dtb)	ACPI	ACPI	DTB（嵌入式）或 ACPI（高端 ARM）
是否调用 ExitBootServices	❌（无 UEFI）	✅	✅	✅（若使用 UEFI）
Secure Boot	通常无（除非自实现）	可选（shim + signed kernel）	强制（Win 11 要求）	支持（Win on ARM 强制）
典型介质	eMMC / SPI Flash / SD	NVMe / SATA SSD（ESP 分区）	NVMe SSD（ESP）	eMMC / UFS / NVMe

## GRUB
GRUB 是 Linux 和类 Unix 系统中最著名、最常用的 Bootloader 之一, 全称 GRand Unified Bootloader，目前主流使用的是 GRUB 2（一般就读 GRUB），早期版本 GRUB Legacy 已基本淘汰.


GRUB 的工作流程：
1. 显示启动菜单
1. 从硬盘/SSD/U盘等设备读取操作系统内核文件（如 vmlinuz）
1. 加载初始内存盘（initrd/initramfs）
1. 传递启动参数给内核（如 root=/dev/sda2 quiet splash）
1. 将控制权交给操作系统内核，完成启动交接

在 UEFI 系统中，GRUB 本身是一个 .efi 可执行文件（如 grubx64.efi），由 UEFI 固件直接加载运行。

UEFI Firmware
    ↓
加载 ESP 分区中的 \EFI\ubuntu\grubx64.efi（即 GRUB）
    ↓
GRUB 读取 /boot/grub/grub.cfg（配置文件）
    ↓
显示启动菜单（Ubuntu / Advanced options / Windows Boot Manager 等）
    ↓
用户选择一项 → GRUB 加载对应的 vmlinuz + initrd
    ↓
GRUB 设置内核命令行参数（如 root=UUID=...）
    ↓
跳转到 Linux 内核入口，移交控制权

GRUB 的应用场景包括：

场景	说明
多系统启动	同一台电脑装了 Windows + Linux，GRUB 菜单让你选择进哪个系统
多内核选择	Linux 更新后保留旧内核，GRUB 允许你选择用哪个版本启动
救援模式	可通过 GRUB 编辑启动参数进入单用户模式或修复系统
网络启动（PXE）	高级用法，GRUB 支持从网络加载内核
在传统 BIOS 模式下，GRUB 会分阶段加载（Stage 1 → Stage 1.5 → Stage 2），但在 UEFI 模式下，这个过程被简化了，因为 UEFI 本身提供了文件系统访问能力。

GRUB 关键组件

组件	作用
grub-install	安装 GRUB 到磁盘或 ESP 分区的工具（如 grub-install /dev/sda）
grub-mkconfig	自动生成 /boot/grub/grub.cfg 的命令（通常通过 update-grub 调用）
grub.cfg	主配置文件，定义菜单项、内核路径、启动参数等（不要手动编辑！）
/etc/default/grub	用户可编辑的 GRUB 配置模板，运行 update-grub 后生效
grub-efi	UEFI 版本的 GRUB 包（如 Debian/Ubuntu 中的 grub-efi-amd64）
各种 Bootloader 总结

Bootloader	平台	特点
GRUB 2	x86/x86_64（PC/Linux）	功能强大，支持脚本、主题、多系统
systemd-boot	UEFI-only PC	轻量、简单，仅支持 EFI 分区内的内核
U-Boot	嵌入式 ARM/MIPS	用于开发板，支持设备树、网络启动
rEFInd	UEFI 多系统	图形化菜单，自动检测所有 OS
Windows Boot Manager	Windows	仅用于 Windows，通过 BCD 管理启动项
