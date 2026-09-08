# biso
ref:
- [Linux x86/x86_64现在将始终保留前1MB的内存](https://www.51cto.com/article/665648.html)

	自Linux 5.13开始将无条件地始终保留前1MB内存, 参考[[PATCH v2] x86/efi: unconditionally hold the whole low-1MB memory regions](https://lore.kernel.org/linux-kernel/20210531090023.16471-1-lijiang@redhat.com/)

BIOS全称为Base Input Output System(基本输入/输出系统)，它一组存储在主板ROM中的程序代码主要功能有:
1. 自检程序，用于开机时对硬件的检测
1. 系统初始化，包括对硬件，BIOS中断向量等初始化
1. 基本的IO处理
1. CMOS设置程序

BIOS运行在16位模式下，实模式下最大可寻址范围为1MB其中0x0C0000~0x0FFFFF留给BIOS使用.

开机后，CPU跳到0xFFFFFFF0处执行，一般这里是一条跳转指令，跳到真正的BIOS入口处执行. BIOS代码首先做的是“加电自检”（Power On Self Test，POST），主要是检测关机设备是否正常工作，设备设置是否与CMOS中的设置一致. 如果发现硬件错误，则通过喇叭报警. POST检测通过后初始化显示设备并显示显卡信息，接着初始化其他设备. 设备初始化完毕后开始检查CPU和内存并显示检测结果. 内存检测通过以后开始检测标准设备，例如硬盘、光驱、串口设备、并口设备等. 然后检测即插即用设备，并为这些设备分配中断号、I/O端口和DMA通道等资源/ 如果硬件配置发生变化，那么这些变化的配置将更新到CMOS中. 随后，根据配置的启动顺序从设备启动，将启动设备主引导记录的启动代码通过BIOS中断读入内存，然后控制权交到引导程序手中，最终引导进入操作系统.