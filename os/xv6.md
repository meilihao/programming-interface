# [xv6](https://pdos.csail.mit.edu/6.828/2022/xv6.html)
ref:
- [asnr/ostep](https://github.com/asnr/ostep)
- [xv6中文文档](http://staff.ustc.edu.cn/~chizhang/OS/Labs/MIT-XV6-%D6%D0%CE%C4%B7%AD%D2%EB%B0%E6.pdf)
- [jserv/xv6-x86_64](https://github.com/jserv/xv6-x86_64)
- [MIT 6.828 学习笔记](https://github.com/GreyZhang/g_unix)

2020 8.11开始不再维护x86, 而是转向riscv

xv6不支持swap.

## x86
```bash
# git clone --depth 1 git://github.com/mit-pdos/xv6-public.git
# make # gcc 7.5.0执行成功; gcc 12.1.0报`-Werror=array-bounds`, `-Werror=infinite-recursion`, 见FAQ
# make qemu # 带qemu gui; make qemu-nox, 不带qemu gui. 不知道为什么`-smp 2`没生效, console log只提示了cpu0
```

调试:
```bash
# --- terminal1
make qemu-nox-gdb
# --- terminal1
gdb -x .gdbinit
```

## riscv
ref:
- [构建工具链](https://github.com/aQuaYi/Learning-MIT-6.828/blob/master/LAB/tools.md)
- [Tools Used in 6.1810](https://pdos.csail.mit.edu/6.828/2022/tools.html)
- [xv6-riscv-book](https://pdos.csail.mit.edu/6.828/2022/xv6/book-riscv-rev3.pdf)
- [xv6-riscv环境搭建](https://groverzhu.github.io/2021/08/17/xv6-riscv%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)

```bash
# git clone --depth 1 -b 2023.06.09 https://github.com/riscv-collab/riscv-gnu-toolchain.git
# cd riscv-gnu-toolchain
# git submodule update --init --recursive --depth 1 # 获取git https musl失败用: git clone --depth 1 -b v1.2.2 git://git.musl-libc.org/musl, -b用具体版本代替; 其他报错见FAQ, 比如`git获取boringssl失败`
# dnf install autoconf automake python3 libmpc-devel mpfr-devel gmp-devel gawk  bison flex texinfo patchutils gcc gcc-c++ zlib-devel expat-devel # form riscv-gnu-toolchain's README. texinfo for makeinfo
# make [linux]# 无需make install. 已使用`make`测试, 可行; `make linux`未测试
# riscv64-unknown-elf-gcc --version
# --- 参考 tour_book构建qemu 8.0
# qemu-system-riscv64 --version
QEMU emulator version 8.0.2
Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers
# git clone --depth 1 https://github.com/mit-pdos/xv6-riscv.git
# cd xv6-riscv
# make qemu # 按`ctrl+a`+`x`退出terminal
```

调试:
```bash
# --- terminal1
make CPUS=1 qemu-gdb
# --- terminal1
gdb-multiarch -q kernel/kernel
```

## FAQ
### 构建xv6
#### make时gcc 12.1.0报`-Werror=array-bounds`
ref:
- [Array subscripts out of bounds](https://www.ibm.com/docs/en/ztpf/1.1.0.15?topic=warnings-array-subscripts-out-bounds)

解决方法: 编辑`Makefile`, 在`CFLAGS`后追加` -Wno-array-bounds `即可.

#### make qemu时gcc 12.1.0构建xv6报`-Werror=infinite-recursion`
ref:
- [error: infinite recursion detected in function runcmd](https://github.com/mit-pdos/xv6-riscv/issues/125)

解决方法: 编辑`Makefile`, 在`CFLAGS`后追加` -Wno-error=infinite-recursion `即可.

#### 构建riscv-gnu-toolchain时, git获取boringssl失败
`vim qemu/roms/edk2/CryptoPkg/Library/OpensslLib/openssl/.gitmodules` 将repo换成boringssl github mirror: `https://github.com/google/boringssl.git`在执行`git submodule sync --recursive`即可重新`git submodule update`