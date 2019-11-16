# linux编译

如何替换linux内核:
```sh
# sudo apt-get install libncurses5-dev libssl-dev build-essential openssl bison flex bc
# wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.3.11.tar.xz
# xz -d linux-5.3.11.tar.xz
# tar -xf linux-5.3.11.tar
# cd linux-5.3.11
# cp /boot/config-4.15.0-30deepin-generic .config # 基于现有内核的配置来编译kernel
# make -j4
# sudo make modules_install # 编译并安装kernel的模块
# sudo make install # 编译并安装新kernel
# sudo reboot
```