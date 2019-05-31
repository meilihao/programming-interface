# 文件描述符
一个非负整数,kernel用来标识一个特定进程正在使用的文件.

# 标准输入 标准输出 标准错误
每当运行一个程序时,其所在的shell会为它打开3个文件描述符,即:
- 标准输入(standard input,0)
- 标准输入(standard output,1)
- 标准输入(standard error,2)
它们默认都链接到终端,可使用重定向方式来修改.

## 常用数值
- I/O Buffer 大小 8192 : unix环境高级编程v3 图3-6 linux上用不同缓存长度进行读操作的时间结果

## 磁盘IO调度算法
```bash
$ cat /sys/block/sda/queue/scheduler
noop deadline [cfq] # `[]`表示选中
```

参考:
- [详解Linux 磁盘I/O优化（oracle RAC）]https://www.tuicool.com/articles/NFZZfqY