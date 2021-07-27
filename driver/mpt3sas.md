# mpt3sas
mpt3sas是SAS卡驱动, 源码在[drivers/scsi/mpt3sas](https://elixir.bootlin.com/linux/v5.10.53/source/drivers/scsi/mpt3sas)

## FAQ
### [debug mpt3sas](https://bugzilla.kernel.org/show_bug.cgi?id=60644)
> 未验证, 文章针对的是mpt2sas, 但预计[与mpt3sas相同或类似](https://elixir.bootlin.com/linux/v5.10.53/source/drivers/scsi/mpt3sas/mpt3sas_debug.h#L48).

参考:
- [kernel-drivers/mpt3sas-11.00.00.00/load.sh](https://github.com/vinsonlee/kernel-drivers/blob/master/mpt3sas-11.00.00.00/load.sh)

setting the driver logging level to 0x3f8, Here are the steps to set the mpt3sas driver logging level:

- While loading the driver 
        modprobe mpt3sas logging_level=0x3f8
    
- If driver is in ramdisk, then in RHEL5/SLES/OEL5 OS, following line has to be added in /etc/modprobe.conf and reboot the system
    options mpt3sas logging_level=0x3f8

- Add below word at the end of kernel module parameters line in /boot/grub/menu.lst or /boot/grub/grub.conf file and reboot the system
    mpt3sas.logging_level=0x3f8
 
- During driver run time
         echo 0x3f8 > /sys/module/mpt3sas/parameters/logging_level

> debug everything: insmod mpt3sas.ko logging_level=0xFFFFFFF. [类似的scsi debug可用](https://github.com/vinsonlee/kernel-drivers/blob/master/mpt3sas-11.00.00.00/load.sh): sysctl -w dev.scsi.logging_level=0x11C0.

> 拓展: 可用`modinfo mpt3sas | grep parm`查看所需参数.