# power
ref:
- [<<Linux设备驱动开发详解>>]()

Linux内核电源管理的整体架构。大体可以归纳为如下几类：
1. CPU在运行时根据系统负载进行动态电压和频率变换的CPUFreq
2. CPU在系统空闲时根据空闲的情况进行低功耗模式的CPUIdle
3. 多核系统下CPU的热插拔支持
4. 系统和设备对于延迟的特别需求而提出申请的PMQoS，它会作用于CPUIdle的具体策略
5. 设备驱动针对系统SuspendtoRAM/Disk的一系列入口函数
6. SoC进入suspend状态、SDRAM自刷新的入口
7. 设备的runtime(运行时)动态电源管理，根据使用情况动态开关设备
8. 底层的时钟、稳压器、频率/电压表(OPP模块完成)支撑，各驱动子系统都可能用到

## CPUFreq
位于`drivers/cpufreq`, 负责进行运行过程中cpu频率和电压的动态调整, 即DVFS(Dynamic Voltage Frequency Scaling, 动态电压频率调整)

`cpufreq_register_driver()`用于注册CPUFreq驱动

## Regulator驱动
Regulator是linux中电源管理的基础设施之一, 用于稳压电源的管理, 是各种驱动子系统中设置电压dd标准接口

## cpu热插拔
通过`/sys/devices/system/cpu/cpu<n>/online`可控制cpu的在离线

## Tools
- PowerTop: 用于进行电量消耗分析和电源管理诊断