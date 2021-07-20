# debug_linux
- [动态追踪技术(三) ：Tracing your kernel Functions!](https://riboseyim.github.io/2017/04/17/DTrace_FTrace/)
- [Tracing the Linux kernel with ftrace](https://embeddedbits.org/tracing-the-linux-kernel-with-ftrace/)
- ![Debugging Embedded Linux Systems: Dynamic Debug](/misc/pdf/os/Kernel-Debug-Series-Part4-dynamic-debug.pdf)

## FAQ
### 禁用pr_debug输出
参考:
- [内核调试——dyndbg特性](http://linux.laoqinren.net/kernel/kernel-dynamic-debug/)


1. 仅屏蔽某个文件的pr_debug

    ```bash
    echo 'file tcm_qla2xxx.c -p' > /sys/kernel/debug/dynamic_debug/control
    ```

1. 屏蔽NFS服务模块中所有的动态输出

    ```bash
    echo -n 'module nfsd -p' > /sys/kernel/debug/dynamic_debug/control
    ```

1. 屏蔽其他

    [control文件的语法格式](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/dynamic-debug-howto.rst): `command ::= match-spec* flags-spec`, 其中match-spec的关键词是：
    ```config
    match-spec ::= 'func' string |
                   'file' string |
                   'module' string |
                   'format' string |
                   'line' line-range
    ```

ps: `-p` 仅支持关闭`pr_debug`; 其他级别, 比如`pr_err`照常输出

其他方法:
#### 编译时处理
查看pr_debug定义:
```c
// https://elixir.bootlin.com/linux/v5.7-rc7/source/include/linux/printk.h#L324
/* If you are writing a driver, please use dev_dbg instead */
#if defined(CONFIG_DYNAMIC_DEBUG)
#include <linux/dynamic_debug.h>

/* dynamic_pr_debug() uses pr_fmt() internally so we don't need it here */
#define pr_debug(fmt, ...) \
  dynamic_pr_debug(fmt, ##__VA_ARGS__)
#elif defined(DEBUG)
#define pr_debug(fmt, ...) \
  printk(KERN_DEBUG pr_fmt(fmt), ##__VA_ARGS__)
#else
#define pr_debug(fmt, ...) \
  no_printk(KERN_DEBUG pr_fmt(fmt), ##__VA_ARGS__)
#endif
```

原来，三个宏作为判断条件决定了pr_debug到底采用哪种用法：
1. 第一种用法，如果定义了CONFIG_DYNAMIC_DEBUG，就使用动态debug机制dynamic_pr_debug()

  按照[Dynamic Debug](http://linux.laoqinren.net/kernel/kernel-dynamic-debug/)处理
1. 第二种用法，如果定义了DEBUG，就使用printk(KERN_DEBUG...)

  检查一下module的Makeflie里是否添加了`-DDEBUG`, 删除即可.
1. 第三种用法，默认情况下，不打印

  因为本身就不打印, 不做处理.