# android

## 架构
1. 应用层
1. Framework+类库
1. 硬件抽象层(HAL)

	属于用户空间

	作用:
	1. 保护硬件产商的知识产权, 避免GPL传染
	1. 屏蔽硬件差异

1. bionic

	类似glibc
1. kernel