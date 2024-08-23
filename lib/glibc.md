# glibc

## file
- fopen()/fdopen()
- fclose()
- fcloseall()

    关闭所有的和当前进程相关的流, 这个函数始终返回0, 是linux特有的
- fgetc() : 从流中读取一个字符, 并把该无符号字符强转为int返回, 强转是为了有足够的范围来表示文件结尾符和错误. 注意, 不能将该值转为char来保存.
- unget(): 将c转成无符号字符并放回流中
- fgets(): 读取一个字符串. 读取size-1个字节. 读到EOF或换行符(该换行符会被存入str)时读入结束; 当所有字节读入, 空字符被存入字符串末尾

## error
- `void perror(const char *string);`:向stderr输出错误信息
- `char *strerror(int errnum);`: 返回errnum描述的错误的字符串指针, 不能修改字符串内容
- `int strerror_r(int errnum, char buf[.buflen], size_t buflen)`: 向buf写入errnum描述的错误信息

注意事项:
1. 对于某些函数, 其返回类型的范围内的值都是合法的, 即无法通过其返回值判断函数是否出错了, 比如strtoul. 此时在该函数调用前errno必须被置为0, 且调用后需要判断errno(非0为有错误).
1. 库函数和系统调用也会修改errno, 因此跨函数调用前需保存errno, 再调用函数, 再用之前保存的临时值来处理errno, 以避免它被覆盖 

## glibc 系统调用 wrappers
对于很多系统调用，glibc 只用到了一个简单的 wrapper 程序：将参数放到合适的寄存器 ，然后执行 syscall 或 int $0x80 指令，或者调用 __kernel_vsyscall. 这个过程 用到了一系列的列表，这些列表的核心内容定义在几个文本文件里，然后被脚本文件处理之 后生成 C 代码.

> `sysdeps/unix/syscalls.list`文件描述了一些常规系统调用.
> 要了解每一列代表什么，请查看这个文件里的注释： [sysdeps/unix/make-syscalls.sh](https://github.molgen.mpg.de/git-mirror/glibc/blob/glibc-2.15/sysdeps/unix/make-syscalls.sh)