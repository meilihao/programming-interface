# misc

## time
时区信息位于目录`/usr/share/zoneinfo`中.

系统的本地时间由时区文件/etc/localtime 定义,通常链接到/usr/share/zoneinfo 下的一个文件.

函数 tzset()会首先检查环境变量 TZ. 如果尚未设置该变量,那么就采用/etc/localtime 中
定义的默认时区来初始化时区. 如果 TZ 环境变量的值为空,或无法与时区文件名相匹配,那
么就使用 UTC. 还可将 TZDIR 环境变量(非标准的 GNU 扩展)设置为搜寻时区信息的目录
名称,以替代默认的/usr/share/zoneinfo 目录.

## 地区定义
地区信息维护于`/usr/share/local`. 该目录下的每个子目录都包含一特定地区的信息.

目录内容:
- LC_CTYPE 该文件包含字符分类(参见 isalpha(3)手册页)以及大小写转换规则
- LC_COLLATE 该文件包含针对一字符集的排序规则
- LC_MONETARY 该文件包含对币值的格式化规则(见 localeconv(3)和`<locale.h>`)
- LC_NUMERIC 该文件包含对币值以外数字的格式化规则(见 localeconv(3)和`<locale.h>`)
- LC_TIME 该文件包含对日期和时间的格式化规则
- LC_MESSAGES 该目录下所含文件,针对肯定和否定(是/否)响应,就格式及数值做了规定

`locale`命令显示当前地区环境(本 shell 内)的相关信息. `locale – a`则将列出系统上定义的整套地区.