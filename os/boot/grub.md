# grub
ref:
- [Changes/BootLoaderSpecByDefault](https://fedoraproject.org/wiki/Changes/BootLoaderSpecByDefault)
- [Boot Loader Specification (BLS)](https://systemd.io/BOOT_LOADER_SPECIFICATION/)

## FAQ
### grub.cfg里search的uuid哪来的
bios: 所用的uuid是`/boot`分区的uuid
uefi: 所用的uuid是`/`分区的uuid

> 可用`blkid+lsblk`联合查询到

RHEL 8 和 Centos 8 中的 GRUB2 将 blscfg 文件和 /boot/loader 中的条目用作启动配置，**不再使用以前的 grub.cfg 格式**。建议使用 grubby 工具来管理 blscfg 文件和检索 /boot/loader/entries/ 中的信息.

### centos8禁用blscfg
```bash
sed -i "s/^GRUB_ENABLE_BLSCFG=.*/GRUB_ENABLE_BLSCFG=false/g" /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg
rm -rf /boot/grub2/grub.cfg.bak
rm -rf /boot/grub2/grub.cfg.old
```

### blscfg 更新 GRUB 引导程序中的默认内核
ref:
- [Red Hat Enterprise Linux 8 的新玩意 第4篇 通过`grub`配置Kernel启动参数](http://ackdo.com/2021/02/16/linux_ackdo_RHEL8_Kernel_cmdline/index.html)

```bash
--- 1.    运行 grubby --default-kernel 命令以查看当前默认内核：
# grubby --default-kernel
--- 2.    运行 grubby --info=ALL 命令以查看所有可用的内核及其索引：
# grubby --info=ALL
index=0
kernel="/boot/vmlinuz-4.18.0-305.el8.x86_64"
args="ro console=ttyS0,115200n8 console=tty0 net.ifnames=0 rd.blacklist=nouveau nvme_core.io_timeout=4294967295 crashkernel=auto $tuned_params"
root="UUID=d35fe619-1d06-4ace-9fe3-169baad3e421"
initrd="/boot/initramfs-4.18.0-305.el8.x86_64.img $tuned_initrd"
title="Red Hat Enterprise Linux (4.18.0-305.el8.x86_64) 8.4 (Ootpa)"
id="0c75beb2b6ca4d78b335e92f0002b619-4.18.0-305.el8.x86_64"
index=1
kernel="/boot/vmlinuz-0-rescue-0c75beb2b6ca4d78b335e92f0002b619"
args="ro console=ttyS0,115200n8 console=tty0 net.ifnames=0 rd.blacklist=nouveau nvme_core.io_timeout=4294967295 crashkernel=auto"
root="UUID=d35fe619-1d06-4ace-9fe3-169baad3e421"
initrd="/boot/initramfs-0-rescue-0c75beb2b6ca4d78b335e92f0002b619.img"
title="Red Hat Enterprise Linux (0-rescue-0c75beb2b6ca4d78b335e92f0002b619) 8.4 (Ootpa)"
id="0c75beb2b6ca4d78b335e92f0002b619-0-rescue"
index=2
kernel="/boot/vmlinuz-4.18.0-305.3.1.el8_4.x86_64"
args="ro console=ttyS0,115200n8 console=tty0 net.ifnames=0 rd.blacklist=nouveau nvme_core.io_timeout=4294967295 crashkernel=auto $tuned_params"
root="UUID=d35fe619-1d06-4ace-9fe3-169baad3e421"
initrd="/boot/initramfs-4.18.0-305.3.1.el8_4.x86_64.img $tuned_initrd"
title="Red Hat Enterprise Linux (4.18.0-305.3.1.el8_4.x86_64) 8.4 (Ootpa)"
id="ec2fa869f66b627b3c98f33dfa6bc44d-4.18.0-305.3.1.el8_4.x86_64"
--- 3.    运行 grubby --set-default 命令以更改实例的默认内核, **将 4.18.0-305.3.1.el8_4.x86_64 替换为需要的内核版本号**：
# grubby --set-default=/boot/vmlinuz-4.18.0-305.3.1.el8_4.x86_64
--- 4.    运行 grubby --default-kernel 命令以验证上述命令是否有效：
# grubby --default-kernel
```

`grep -n "kernelopts" -r /boot`发现`/boot/loader/entries/xxx.conf`里的kernelopts来源于`/boot/grub2/grubenv`
### grub2切换到blsc
`sudo grub2-switch-to-blscfg`