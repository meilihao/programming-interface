# func
## FAQ
### [如何查看kernel函数的参数](https://zhuanlan.zhihu.com/p/463399962)
前提: 带debuginfo的vmlinux, 即开启`CONFIG_DEBUG_INFO`

1. systemtap

	`stap -L 'kernel.function("dentry_open")'`
1. perf-probe

	`perf probe -V dentry_open`
1. bpftrace

	需开启`CONFIG_DEBUG_INFO_BTF`

	`bpftrace -lv kfunc:dentry_open`