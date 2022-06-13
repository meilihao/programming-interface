# firmware
许多设备需要两个依赖才能正常运行: 驱动程序和固件.

驱动程序从`/lib/firmware`处请求固件. 这是硬件所需的特殊文件，不是二进制文件. 然后driver执行将固件加载到设备中所需的操作, 固件会对设备内部的硬件进行编程.

## uefi
- [百敖BIOS培训系列一：UEFI启动流程总览](https://www.bilibili.com/video/BV1Nr4y1w7dP/)

## FAQ
### 如何查找设备芯片型号
可到`/usr/share/hwdata/pci.ids`获取芯片型号， 根据pci.ids开头的说明， 对应的device字段即为芯片型号.

> [The PCI ID Repository](https://pci-ids.ucw.cz)

### 如何查找驱动支持的设备芯片型号
Linux内核driver模块中`MODULE_FIRMWARE(filepath)`是kernel module声明支持的固件(固件由`request_firmware()`加载， filepath是相对于`/lib/firmware`的路径), 其实就是将firmware="filepath"的字符串信息放到`.modinfo`段, 并在`modinfo <mod_name, like "i915">`时会以`firmware:       <filepath>`的形式输出.