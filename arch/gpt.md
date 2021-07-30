# gpt
参考:
- [GPT分区数据格式分析](https://blog.csdn.net/diaoxuesong/article/details/9406015)


gpt特点:
- GPT 只使用 LBA
- GPT 数据结构在磁盘上存储两次：开始和结束各一次. 在因事故或坏扇区导致损坏的情况下，这种重复提高了成功恢复的几率.
循环冗余检验 (CRC) 值针对关键数据结构而计算，提高了数据崩溃的检测几率.
- GPT 将所有分区存储在单个分区表中（带有备份），因此扩展分区或逻辑分区没有存在的必要. GPT 默认支持 128 个分区，当然也可以更改分区表的大小(分区软件支持的话)
- GPT 使用一个 16 字节的全局唯一标识符 (GUID) 值来标识分区类型。这使分区类型更不容易冲突.
- GPT 支持存储人类可读的分区名称.

使用 GPT需有三类主要的软件的支持：内核、引导装载程序和低级别磁盘实用工具.

Linux 提供三种主要的分区工具系列，均不同程度支持 GPT：
- fdisk系列

    这些程序（fdisk、cfdisk和sfdisk）是文本模式的工具，可以处理 MBR 和一些更独特的分区表，但它们不能处理 GPT.

- GNU Parted (libparted)

    GNU Parted 项目提供一个库 (libparted) 和一个文本模式的实用工具 (parted) 进行分区. 若干个图形用户界面 (GUI) 实现工具也构建于libparted之上. libparted库可以处理 MBR、GPT 和几种其他分区表类型.

- GPTfdisk

    该系列（gdisk、cgdisk和sgdisk）根据fdisk系列进行建模，但可以在 GPT 磁盘上工作

作为一般性规则，基于 GNU Parted 的工具（尤其是 GParted 或 Palimpsest Disk Utility 等 GUI 工具）易于使用；不过，GPTfdisk（特别是gdisk）可以使用更多 GPT 特性. 因此，推荐使用 GParted 或其他 GUI 工具来设置磁盘，但使用 GPTfdisk来微调配置或修复 GPT 磁盘的损坏.

> 512e磁盘在分区与物理扇区边界不一致时会导致潜在的严重性能问题. 自 2010 年年底发布的分区工具一般能够很好地处理这个问题，但如果在使用较旧版本的工具，请务以创建正确匹配的分区.

在GPT分区中，每一个数据读写单元成为LBA（逻辑块地址, 即`gdisk -l /dev/sd<x>`中的logical sector size）, 一个“逻辑块”相当于传统MBR分区中的一个“扇区”，之所以会有区别，是因为GPT除了要支持传统硬盘，还需要支持以NAND FLASH为材料的SSD硬盘，这些硬盘的一个读写单元是2KB或4KB，所以GPT分区中干脆用LBA来表示一个基础读写块，当GPT分区用在传统硬盘上时，通常，LBA就等于扇区号，有些物理硬盘支持2KB对齐，此时LBA所表示的一个逻辑块就是2KB的空间. 换句话说，**对于 512 字节的扇区，Partition table header (LBA 1)从磁盘开头的第 512 个字节开始，而对于 4 KB 的扇区，它从第 4096 个字节开始**.

## GPT Partition Scheme
GPT分区, 全称为Globally Unique Identifier Partition Table，也叫做GUID分区表，它是UEFI 规范的一部分, 具体在[uefi官网](https://uefi.org/specifications)的[UEFI Specification Version 2.8 (Errata A) (released February 2020)](https://uefi.org/sites/default/files/resources/UEFI_Spec_2_8_A_Feb14.pdf)的`Chapter 5`.

![布局](/misc/img/arch/Guid-partition-table.png)
![](/misc/img/arch/1306gpt.jpg)

[GPT分区格式](https://www.cnblogs.com/cwcheng/p/11270774.html):
1. Protective MBR (LBA 0)

    一个保护性的MBR，以确保与不懂GPT格式的实用程序兼容. 在这个MBR中，只有一个标志为0xEE的分区，以此表示这块硬盘使用GPT分区格式. 不支持GPT分区格式的软件，会识别出未知类型的分区；支持GPT分区格式的软件，可正确识别GPT分区磁盘.
1. Primary Partition Table (LBA 1)
1. Partition entries (LBA 2–33)
1. Backup Partition Table : 在硬盘的末端还有一个备份分区表，保证了分区信息不容易丢失

> LBA: 逻辑块地址

查看gpt分区:
```bash
$ sudo parted /dev/nvme0n1
GNU Parted 3.2
Using /dev/nvme0n1
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) p
Model: Unknown (unknown)
Disk /dev/nvme0n1: 256GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt # Partition table: msdos 则是一块 MBR/MS-DOS 格式的磁盘
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  538MB   537MB   fat32              boot, esp
 2      538MB   2685MB  2147MB  ext4
 3      2685MB  45.4GB  42.7GB  ext4
 4      45.4GB  256GB   211GB   ext4
```

## FAQ
### ESP（EFI系统分区）
EFI System Partition，FAT格式, 通常是FAT32，分区类型ID是0xEF, 主要目录是EFI. EFI/boot/bootx64.efi是EFI默认的启动项, 由uefi标准规定. 安装的操作系统会建立相应的目录EFI/xxx，并将自己的启动项复制到EFI/boot/bootx64.efi作为缺省启动项.

deepin上esp的mount point是`/boot/efi`.

安装Ubuntu的时候，会在ESP分区中建立EFI/Ubuntu子目录，并将EFI/ubuntu/grubx64.efi（grub bootloader）复制为EFI/boot/bootx64.efi.

> 因为Grub本身会扫描磁盘上的分区并找到windows启动程序（bootmgr.efi），因此先装windows后装ubuntu仍能通过grub让windows启动.

### gpt备份
```
# dd if=/dev/sda of=gpt-mbr bs=512 count=1 # 备份Protective MBR, 因为mbr是512个字节([正常mbr](https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95): 前446是程序代码,后64字节包含分区表信息,最后2字节是MBR的有效标志: 0x55aa)
# dd if=/dev/sda of=gpt-partition bs=512 skip=1 count=33 # 仅备份GPT头和GPT分区
# dd if=/dev/sda of=gpt-partition bs=512 count=34 备份完整的GPT分区表(含Protective MBR, GPT头，以及分区表)
# dd if=gpt--partition of=/dev/sda bs=512 count=34 # 恢复完整的GPT分区表信息
```

### `GPT: Use GNU Parted to correct GPT errors.`
dmesg中的gpt错误. 因为某些原因比如自定义的磁盘签名(写在磁盘前面)导致系统启动时修正了gpt分区表, 最终会导致自定义签名被抹除.