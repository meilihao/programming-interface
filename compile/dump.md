# dump

## coredump
保留进程现场.

方法:
1. kill -11 pid，11代表信号SIGSEGV，在kill这个进程的同时产生coredump，这样就可以迅速重启程序，然后慢慢分析
1. 在gdb中使用gcore（generate-core-file）, 一些bug我们是可以通过gdb线上修复的，但问题还是需要时候继续查看，这个时候就可以用这个命令先产生coredump，然后退出gdb，让修复后的程序继续执行