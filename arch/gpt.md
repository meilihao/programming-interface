# gpt
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
# dd if=/dev/sda of=gpt-mbr bs=512 count=1 # 备份Protective MBR, 因为mbr是512个字节([正常mbr](https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95): 前446是程序代码,后64字节包含分区表信息,最后2字节是MBR的有效标志: 0x55AA)
# dd if=/dev/sda of=gpt-partition bs=512 skip=1 count=33 # 仅备份GPT头和GPT分区
# dd if=/dev/sda of=gpt-partition bs=512 count=34 备份完整的GPT分区表(含Protective MBR, GPT头，以及分区表)
# dd if=gpt--partition of=/dev/sda bs=512 count=34 # 恢复完整的GPT分区表信息
```
