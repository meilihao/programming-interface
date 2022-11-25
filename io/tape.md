# tape
磁带库也有磁带库扩展柜.

## 磁带
按用途划分:
1. 数据带: 保存数据
1. 清洗带: 清洗磁头

## FAQ
### WORM
一次写入多次读取, 写入后不可修改.

### LTFS(linear tape fs, 线性磁带文件系统)
支持:
- 分区

### 兼容性
ref:
- [LTO 磁带机的性能规格](https://www.ibm.com/docs/zh/ts4500-tape-library)

当前有: LTO(Linear Tape Open,线性磁带开发):
- 9
	cap: 18TB(未压缩)/45(压缩)TB
	speed: 400MB(未压缩)/s或1000MB(压缩)/s
- 8
	cap: 12TB/30TB 
	speed: 2.7(压缩)TB/h
- 7
	cap: 6TB/15TB
- 6
	cap: 2.5TB/6.25TB
- 5
	cap: 1.5TB/3TB
- 4

LTO联盟规定是向下写一代, 向下读两代.