# ipmi
参考:
- [IPMI 详解](https://bbs.huaweicloud.com/forum/thread-67657-1-1.html)

[IPMI(Intelligent Platform Management Interface，智能平台管理接口)](https://www.intel.com/content/www/us/en/servers/ipmi/ipmi-home.html), 是一种开放标准的硬件管理接口规格，定义了嵌入式管理子系统进行通信的特定方法.

IPMI 信息通过基板管理控制器 (BMC)（位于 IPMI 规格的硬件组件上）进行通信. 使用低级硬件智能管理而不使用操作系统进行管理，具有两个主要优点：
1. 此配置允许进行带外服务器管理
1. 操作系统不必负担传输系统状态数据的任务

用户可以利用IPMI监视服务器的物理健康特征，如温度、电压、风扇工作状态、电源状态等.

IPMI的核心是一个专用芯片/控制器(叫做服务器处理器或基板管理控制器(BMC))，其并不依赖于服务器的处理器、BIOS或操作系统来工作，可谓非常地独立，是一个单独在系统内运行的无代理管理子系统，只要有BMC与IPMI固件其便可开始工作，而BMC通常是一个安装在服务器主板上的独立的板卡. IPMI良好的自治特性便克服了以往基于操作系统的管理方式所受的限制，例如操作系统不响应或未加载的情况下其仍然可以进行开关机、信息提取等操作.

IPMI规定了很多的东西，BMC是其中最重要的一个部分，此外还有一些"卫星"控制器通过IPMB与BMC相连，这些"卫星"控制器一般控制特定的设备.

IPMB全称Intelligent Platform Management Bus，是一种基于I2C的串行总线，它用于BMC与"卫星"控制器的通信，其上传递的是IPMI命令.

![与IPMI有关的各个模块](/misc/img/arch/112941yxo4v6s7hsjnm6vv.jpeg)