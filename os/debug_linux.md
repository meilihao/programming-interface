# debug_linux
- [动态追踪技术(三) ：Tracing your kernel Functions!](https://riboseyim.github.io/2017/04/17/DTrace_FTrace/)
- [Tracing the Linux kernel with ftrace](https://embeddedbits.org/tracing-the-linux-kernel-with-ftrace/)

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