# programming interface

记录编程所需的基础知识体系,侧重于Linux.

> 除os目录外, 其他md无特殊说明其中的os均为Linux.

## 进阶
1. linux命令行
    1. 鸟哥的 Linux 私房菜
    1. UNIX/Linux 系统管理技术手册
    1. linux就是这个范儿
1. 操作系统原理
    1. 现代操作系统v3
    1. 操作系统：精髓与设计原理
    1. 操作系统设计与实现v3
1. 通过系统调用进行程序开发
    1. 深入理解计算机系统
    1. UNIX 环境高级编程
    1. Linux UNIX系统编程手册
1. 了解linux kernel机制(不需要具体代码, 要流程)
    1. 深入理解 LINUX 内核
1. 阅读 Linux 内核代码(核心逻辑和场景)
    1. LINUX 内核源代码情景分析

## other
### 源码
文中引用的kernel/glibc repo来自[清华大学开源软件镜像站](https://mirror.tuna.tsinghua.edu.cn/help/linux-stable.git/)

#### linux kernel在线阅读
- [elixir.bootlin.com/linux **推荐**](https://elixir.bootlin.com/linux/latest/source)和[sourcegraph linux](https://sourcegraph.com/github.com/torvalds/linux@v5.8-rc3), 适合跳转
- [sourcegraph.com](https://sourcegraph.com/search?q=context:global+repo:%5Egithub%5C.com/torvalds/linux%24%40master&patternType=literal), 适合跳转+搜索
- [github.com/torvalds/linux](https://github.com/torvalds/linux), 适合搜索
- [轻松认识 Linux kernel](http://www.bricktou.com/)

## 其他书单
- `<<深入理解程序设计使用Linux汇编语言>>的Chapter13`

## todo
- [Linux 内核揭秘](https://github.com/MintCN/linux-insides-zh)
- [Why’s THE Design](https://juejin.cn/user/2418581313961677/posts)

## tool
### 内核预处理
- [linux 源码分析小技巧，查看某一文件经编译预处理后的内容](http://blog.chinaunix.net/uid-21419530-id-5820125.html)
