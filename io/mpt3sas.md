# mpt3sas
mpt3sas是SAS卡驱动.

## FAQ
### [debug mpt3sas](https://bugzilla.kernel.org/show_bug.cgi?id=60644)
> 未验证, 且这是mpt2sas的, 预计与mpt3sas相同或类似.

setting the driver logging level to 0x3f8, Here are the steps to set the mpt2sas driver logging level:

- While loading the driver 
        modprobe mpt2sas logging_level=0x3f8
    
- If driver is in ramdisk, then in RHEL5/SLES/OEL5 OS, following line has to be added in /etc/modprobe.conf and reboot the system
    options mpt2sas logging_level=0x3f8

- Add below word at the end of kernel module parameters line in /boot/grub/menu.lst or /boot/grub/grub.conf file and reboot the system
    mpt2sas.logging_level=0x3f8
 
- During driver run time
         echo 0x3f8 > /sys/module/mpt2sas/parameters/logging_level