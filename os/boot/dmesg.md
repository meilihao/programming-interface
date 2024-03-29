# dmesg

## `BIOS-provided physical RAM map`
```
[    0.000000] BIOS-provided physical RAM map
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000003efff] usable
[    0.000000] BIOS-e820: [mem 0x000000000003f000-0x000000000003ffff] ACPI NVS
```

BIOS-provided physical RAM map是kernel从bios中读取到的系统内存映射.

BIOS-e820是通过BIOS的int 0x15服务(实模式下才可用)执行0xe820号函数获得系统的内存映射信息.

> linux 内核通过调用 bios的 int 15h 中断来访问它, 方法是将EAX 寄存器设置为十六进制值 E820(eax=E820).

## `Command line`
Command line是指用户在启动操作系统时输入的命令行参数，这些参数可以通过启动管理器（如GRUB）或直接在BIOS中设置.

Command line只在内核启动时起作用，内核启动后，这些参数将被丢弃.

## `Kernel command line`
同`/proc/cmdline`, 引导程序给kernel传递的参数

Kernel command line则会在内核启动后一直保留，并可以被内核中的各种模块使用.

## `Calibrating delay loop (skipped), value calculated using timer frequency.. 4000.00 BogoMIPS (lpj=8000000)`
来自文件init/ calibrate.c中的函数calibrate_delay()，该函数主要作用根据不同的配置计算BogoMIPS的值.

在启动过程中, 内核会计算处理器在一个jiffy时间内运行一个内部的延迟循环的次数. jiffy是系统定时器2个连续节拍间的间隔. 该计算必须被校准到所用cpu的处理速度. 校准的结果保存在loops_per_jiffy中. 某些驱动程序希望进行小的微妙级延迟时会用.

BogoMIPS = loops_per_jiffy * 1s内的jiffy数*`延迟循环消耗的指令`

BogoMIPS 的值是 linux 内核通过在一个时钟节拍里不断的执行循环指令而估算出来, 因此它可作为衡量处理器运行速度的相对尺度.

## `NET: Registered xxx protocol family`
注册协议

## `Freeing initrd memory: 146224K`
initrd是一种由引导程序加载到内存的虚拟磁盘镜像, 内核启动后, 会将其挂载为初始根文件系统.

2.6开始initrd实际就是initramfs了, 它取代了早先的initrd, 在启动过程中, 内核会将文件解压为一个initramfs根文件系统.

## `io scheduler mq-deadline registered`
注册io调度器

## `[   12.276977] EXT4-fs (sda2): mounted filesystem 388da031-2b34-4266-b83b-25c8739e4726 r/w with ordered data mode. Quota mode: none.`
挂载fs