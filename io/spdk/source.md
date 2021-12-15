# source

## arch
- app

    - app/iscsi_tgt: iscsi target
    - app/nvmf_tgt: NVMe-oF target
    - app/iscsi_top：iscsi top工具, 类似于linux top, 用来监控iscsi
    - app/trace：iscsi target和nvme-of target trace工具
    - app/vhost：将virtio控制器呈现给基于qemu的虚拟机，并对IO进行处理
- build
- doc：spdk 下的doc文件
- dpdk：spdk调用了dpdk的很多基础库
- etc：各类型使用方式的基本配置
- examples：示例代码
- lib：开发库
- mk：makefile文件
- scripts：脚本及环境配置相关
- test：各模块功能性能测试