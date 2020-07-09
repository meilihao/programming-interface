# ko
一个内核模块应该由以下几部分组成:
1. 头文件部分

	一般的内核模块，都需要 include 下面两个头文件:
	```c

	#include <linux/module.h>
	#include <linux/init.h>
	```

1. 定义一些函数，用于处理内核模块的主要逻辑

	例如打开、关闭、读取、写入设备的函数或者响应中断的函数.

1. 定义一个 file_operations 结构

	设备是可以通过文件系统的接口进行访问的，而对于某种文件系统的操作，都是放在 file_operations 里面的. 因此设备要想被文件系统的接口操作，也需要定义这样一个结构.

1. 定义整个模块的初始化函数和退出函数，用于加载和卸载这个 ko 的时候调用
1. 调用 module_init 和 module_exit，分别指向上面两个初始化函数和退出函数
1. 声明一下 lisense，调用 MODULE_LICENSE