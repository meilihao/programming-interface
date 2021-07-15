# nbd

## FAQ
### 指定生成nbd设备数量
`modprobe nbd nbds_max=<N>`, 默认是1024, 即N in [0,1023].

> 必须提前生成nbd设备, nbd-client才可绑定, 否则会报`Error: Cannot open NBD: No such file or directory`