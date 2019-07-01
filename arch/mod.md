# mod
通常可加载模块是放在`/lib/modules/$(uname -r)`里.

相关命令:
```sh
$ lsmod // 查看当前kernel已加载的模块
$ sudo insmod /path_to/xxx.ko [io=xxx] // 加载模块时顺便传递参数
$ sudo rmmod xxx // 删除模块, 仅该模块引用数为0时才有用(`lsmod`的`Used`列)
$ sudo modprobe -v xxx // 加载模块并自动处理依赖 
```