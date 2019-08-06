# 硬件支持
- `dmidecode` : 查看硬件信息
- [`lshw`](https://linux.cn/article-11194-1.html) : 提供硬件规格的详细信息

## cpu
### 指令集
```
$ cat /proc/cpuinfo | grep flags // 获得CPU所有指令集
```