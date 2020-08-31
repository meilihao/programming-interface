# uefi
systemd-boot是为现代硬件设计的，Fedora也正在计划迁移到Systemd-boot放弃grub2. 以及英特尔计划在 2020 年结束终止支持 Legacy BIOS.

20年之后的现在，UEFI已经变得越来越传统，曾经的屠龙骑士变成了恶龙，业界呼唤新的方案，Intel也不失时机的提出了[ModernFW, **实验性的**](https://github.com/intel/ModernFW).

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