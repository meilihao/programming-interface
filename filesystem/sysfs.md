# sysfs
sysfs是为设备服务的内存文件系统, 主要作用是在用户态展示设备信息.

> sys下不能创建和删除文件, 因为它本身不提供这些功能.

> `/dev`目录下的设备文件只是一个代表符号, 不包括设备相关的信息.

sysfs入口是[int __init sysfs_init(void)](https://elixir.bootlin.com/linux/v5.12.9/source/fs/sysfs/mount.c#L97).

sysfs使用[sysfs_create_dir](https://elixir.bootlin.com/linux/v5.12.9/source/fs/sysfs/dir.c#L40)创建目录文件. 它的入参包括一个[kobject](https://elixir.bootlin.com/linux/v5.12.9/source/include/linux/kobject.h#L64)指针, kobject与sysfs是紧密结合的.