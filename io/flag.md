# flags
> 定义在/usr/include/asm-generic/fcntl.h.

## open
```
// /usr/include/fcntl.h
extern int open (const char *__file, int __oflag, ...) __nonnull ((1));
```
必须且互斥flag:
- O_RDONLY : 只读打开
- O_WRONLY : 只写打开
- O_RDWR : 读,写打开

可选flag:
- O_APPEND : 每次写时都追加到文件的尾端
- O_CREAT : 此文件不存在则创建
- O_EXCL : 以独占模式打开文件.如果同时设置了O_CREAT,而文件已经存在则打开操作出错
- O_SYNC : 对该文件的写操作会等到数据被写到磁盘上才算结束
- O_TRUNC : 如果文件存在,而且可写打开会截取文件长度为0
- O_DIRECT : 提供对直接 I/O 的支持
- O_DIRECTORY : 表明所打开的文件必须是目录，否则打开操作失败

## lseek
```
# include <unistd.h>
off_t lseek(int fd, off_t offset, int whence)
# 成功返回新的偏移量,否则-1
```

whence:
- SEEK_SET(0) : 相对于文件开头,非负整数
- SEEK_CUR(1) : 相对于当前偏移量,整数
- SEEK_END(2) : 相对于文件结尾,整数