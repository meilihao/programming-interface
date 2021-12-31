# source(v21.10)

## arch
- app

    - iscsi_tgt: iscsi target
    - iscsi_top：iscsi top工具, 类似于linux top, 用来监控iscsi
    - nvmf_tgt: NVMe-oF target
    - spdk_dd: 用于在文件和SPDK bdevs之间高效地复制数据
    - spdk_lspci:
    - spdk_tgt: 类似于标准 top，它通过 SPDK 轻量级线程和轮询器提供对 CPU 内核的使用情况进行实时反馈. from [SPDK动态负载均衡](https://blog.csdn.net/weixin_37097605/article/details/120558970)
    - spdk_trace：spdk trace工具, 用于查看trace文件内容, 以便进行相关分析
    - spdk_trace_record: 用于转储share memory下trace文件到其他位置
    - vhost：将virtio控制器呈现给基于qemu的虚拟机，并对IO进行处理
- build
- doc：spdk 下的doc文件
- dpdk：spdk调用了dpdk的很多基础库
- etc：各类型使用方式的基本配置
- examples：示例代码
- lib：开发库
- mk：makefile文件
- scripts：脚本及环境配置相关
- test：各模块功能性能测试

## nvmf_tgt
ref:
- [SPDK NVMe-oF target代码分析（转载+改编）](https://www.cnblogs.com/whl320124/articles/10450250.html)

参照创建nvmf_tgt流程:
```bash
build/bin/nvmf_tgt
--- other terminal
scripts/rpc.py nvmf_create_transport -t TCP -u 16384 -m 8 -c 8192
scripts/rpc.py bdev_malloc_create -b Malloc0 64 512
scripts/rpc.py nvmf_create_subsystem nqn.2016-06.io.spdk:cnode1 -a -s SPDK00000000000001 -d SPDK_Controller1
scripts/rpc.py nvmf_subsystem_add_ns nqn.2016-06.io.spdk:cnode1 Malloc0
scripts/rpc.py nvmf_subsystem_allow_any_host -e nqn.2016-06.io.spdk:cnode1
scripts/rpc.py nvmf_subsystem_add_listener nqn.2016-06.io.spdk:cnode1 -t TCP -a 192.168.88.236 -s 4420
```

### 创建target
nvmf_tgt app位于`spdk/app/nvmf_tgt`, 入口是`nvmf_main.c`. 它的main函数和其他spdk app main函数类似很简洁, 只是调用了spdk_app_opts_init初始化了一下相应的参数, 然后调用spdk_app_parse_args解析命令行的相应参数, 接着调用spdk_app_start, 如果有错误或终止, 最终会执行spdk_app_fini退出.

spdk_app_start 函数会先调用spdk_subsystem_init 进行所有subsystem模块的初始化. 目前来讲, SPDK里面的subsystem有两个概念:
1. 第一个subsystem的概念指模块的subsystem

    主要位于代码目录module/event/subsystems中, 通过`SPDK_SUBSYSTEM_REGISTER(即module入口, 与kernel module有点类似)->spdk_add_subsystem`来引入subsystem, 比如现在SPDK之中有以下10个模块subsystem, 分别是`accel  bdev  iscsi  nbd  nvmf  scheduler  scsi  sock  vhost  vmd`. 这些模块subsystem有些有依赖关系, spdk在对这些模块初始化的时候会先根据依赖关系进行排序, 然后进行初始化, 代码在`lib/init/subsystem.c`.

    > [module/event/subsystems=lib/event/subsystems: lib/event: move rpc and subsystems dirs to module](https://github.com/spdk/spdk/commit/b8fd519584f5e2623139a5c21af14acf3bbb274c)
1. 第二个subsystem的概念指NVMe-oF中的NVM subsystem

    主要代码在`module/event/subsystems/nvmf`.

    > 没有module/event/subsystems/nvmf/conf.c的原因: [subsystem/nvmf: remove legacy config support](https://github.com/spdk/spdk/commit/dbc7e89f58d5dc33bad7115d1f5bd25714d58e33)

    另外一部分代码位于spdk/lib/nvmf, 主要是处理来自远端的NVMe-oF请求, 包括transport层的抽象, 以及实际基于RDMA transport的实现.


nvmf_tgt.c: 提供NVMe-oF subsystem的函数入口和出口, 以及相应的状态机跳转函数nvmf_tgt_advance_state. 状态包括:
```c
enum nvmf_tgt_state {
    NVMF_TGT_INIT_NONE = 0,
    NVMF_TGT_INIT_CREATE_TARGET,
    NVMF_TGT_INIT_CREATE_POLL_GROUPS,
    NVMF_TGT_INIT_START_SUBSYSTEMS,
    NVMF_TGT_RUNNING,
    NVMF_TGT_FINI_STOP_SUBSYSTEMS,
    NVMF_TGT_FINI_DESTROY_POLL_GROUPS,
    NVMF_TGT_FINI_FREE_RESOURCES,
    NVMF_TGT_STOPPED,
    NVMF_TGT_ERROR,
};
```
- NVMF_TGT_INIT_NONE

    初始状态, 之后直接转入NVMF_TGT_INIT_CREATE_TARGET
- NVMF_TGT_INIT_CREATE_TARGET

    调用nvmf_tgt_create_target开始创建target, 并进入NVMF_TGT_INIT_CREATE_POLL_GROUPS
- NVMF_TGT_INIT_CREATE_POLL_GROUPS： 

    在每个SPDK thread上创建polling group, nvmf_tgt_create_poll_groups->nvmf_tgt_create_poll_group-> spdk_nvmf_poll_group_create.

    在spdk_nvmf_poll_group_create中，传入的channel的io_device的地址实际是：g_spdk_nvmf_tgt. 那么在每个SPDK thread上运行的polling group (数据结构是：struct spdk_nvmf_poll_group)就会被创建, 实际上触发了nvmf_tgt_create_poll_group的调用，主要做了以下工作：
    
    1. 循环所有的transport, 通过nvmf_poll_group_add_transport, 对每一个transport创建一个polling group-> tgroup, 然后加入到这个poll group的中的tgroups数据结构中
    1. 通过nvmf_poll_group_add_subsystem把g_spdk_nvmf_tgt 中的所有NVM subsystem 加入到这个polling group中，存储在sgroups (位于struct spdk_nvmf_poll_group) 中. 

        每个subsystem都拥有一个struct spdk_nvmf_subsystem_poll_group的数据结构，在这个数据结构中，定义了每个命名空间的channel（channels, num_channels）, 以及这个subsystem在这个polling group的状态，并且包含了用于pending nvmf request的list（名字是queued）
    1. 通过SPDK_POLLER_REGISTER创建一个poller， 这个poller会调用spdk_nvmf_poll_group_poll， 这个函数用于在每个transport上polling
    1. 如果所有的polling group被创建完毕，那么进入状态NVMF_TGT_INIT_START_SUBSYSTEMS

- NVMF_TGT_INIT_START_SUBSYSTEMS：

    在每个poll group上把所有的NVMe-oF的subsystem 状态设置为ACTIVE, 然后进入状态NVMF_TGT_RUNNING
- NVMF_TGT_RUNNING：

    这个状态表明NVMe-oF 这个模块的subsystem已经初始化好了，可以初始化下一个subsystem
- NVMF_TGT_FINI_STOP_SUBSYSTEMS：

    这个状态只有在整个app退出的时候，实际上主要是主动退出，或者大部分状态是收到kill （比如ctrlr+c）命令的时候，才会触发NVMf 模块的subsystem的退出，即被调用spdk_nvmf_subsystem_stop函数，然后被调用到nvmf_tgt_subsystem_stopped函数. 进入这个状态后, 会按照顺序关闭每一个NVMe-oF 这个 subsystem, 接着进入NVMF_TGT_FINI_DESTROY_POLL_GROUPS状态
- NVMF_TGT_FINI_DESTROY_POLL_GROUPS：

    在每个SPDK thread上，调用nvmf_tgt_destroy_poll_group, 来销毁polling group. 在这个函数里面会调用spdk_nvmf_poll_group_destroy销毁这个polling group上的所有qpair. 当nvmf_tgt_destroy_poll_group_done被调用到的时候，就进入NVMF_TGT_FINI_FREE_RESOURCES

- NVMF_TGT_FINI_FREE_RESOURCES： 

    销毁g_spdk_nvmf_tgt 所拥有的资源. 最终调用函数nvmf_tgt_destroy_done, 然后进入NVMF_TGT_STOPPED状态
- NVMF_TGT_STOPPED： 

    NVMe-oF 这个模块的subsystem已经被销毁，可以处理下一个模块的subsystem
- NVMF_TGT_ERROR： 

    主要是有错误的时候进入这个状态，然后进行下一个模块subsystem的初始化

nvmf核心代码是nvmf_tgt_advance_state, 主要通过状态机的形式来初始化和运行整个NVMe-oF Target系统. 它的默认g_tgt_state是NVMF_TGT_INIT_NONE, 之后转为NVMF_TGT_INIT_CREATE_TARGET, 开始调用nvmf_tgt_create_target->[spdk_nvmf_tgt_create](https://github.com/spdk/spdk/blob/v21.10/lib/nvmf/nvmf.c#L260)创建全局的NVMe-oF target 对象g_spdk_nvmf_tgt.

spdk_nvmf_tgt_create会通过spdk_io_device_register为g_spdk_nvmf_tgt注册相应的I/O  device. 之后g_tgt_state变为NVMF_TGT_INIT_CREATE_POLL_GROUPS, 开始执行nvmf_tgt_create_poll_groups->nvmf_tgt_create_poll_group->spdk_nvmf_poll_group_create->spdk_get_io_channel, 此时在第一次I/O channel被创建的时候, dev.create_cb即spdk_io_device_register传入的nvmf_tgt_create_poll_group 这个当初被传入的I/O channel的call back 函数就会被触发.

> I/O channel本质上是thread 到一个io_device 的mapping，也就是说对于一组（thread, io_device）会产生唯一的一个I/O channel，直到这个I/O device 最终被调用spdk_io_device_unregister销毁掉.

spdk_nvmf_tgt_create还会调用nvmf_add_discovery_subsystem, 用于创建discovery NVM subsystem, 这个subsystem一般设置为给所有的host可见. 其主要用于实现相应的log discovery命令, 告诉host端有多少NVM subsystem在线. 通过nvmf_get_discovery_log_page可获取discovery_log_page. 如果新的subsystem 被加入, 或者有旧的subsystem 被删除, Log page的更新是通过nvmf_update_discovery_log.

### rpc
[nvmf_rpc.c](https://github.com/spdk/spdk/blob/v21.10/lib/nvmf/nvmf_rpc.c): 主要提供一些RPC的调用函数, 用于在NVMe-oF tgt启动以后, 动态地进行配置.

目前这个文件中主要包括以下的RPC 函数, 其中 括号内（A, B）中A代表scripts/rpc.py 中相应的函数, B 代表SPDK 代码库中相应的C函数, 它们都是通过SPDK_RPC_REGISTER进行相应的注册.

```
SPDK_RPC_REGISTER("nvmf_get_subsystems", rpc_nvmf_get_subsystems, SPDK_RPC_RUNTIME) // 得到所有的NVM subsystem
SPDK_RPC_REGISTER_ALIAS_DEPRECATED(nvmf_get_subsystems, get_nvmf_subsystems)
SPDK_RPC_REGISTER("nvmf_create_subsystem", rpc_nvmf_create_subsystem, SPDK_RPC_RUNTIME) // 创建一个NVM subsystem
SPDK_RPC_REGISTER_ALIAS_DEPRECATED(nvmf_create_subsystem, nvmf_subsystem_create)
SPDK_RPC_REGISTER("nvmf_delete_subsystem", rpc_nvmf_delete_subsystem, SPDK_RPC_RUNTIME) // 删除一个NVM subsystem
SPDK_RPC_REGISTER_ALIAS_DEPRECATED(nvmf_delete_subsystem, delete_nvmf_subsystem)
SPDK_RPC_REGISTER("nvmf_subsystem_add_listener", rpc_nvmf_subsystem_add_listener, // 给NVM subsystem增加一个listener
SPDK_RPC_REGISTER("nvmf_subsystem_remove_listener", rpc_nvmf_subsystem_remove_listener, // 给NVM subsystem 删除一个listener
SPDK_RPC_REGISTER("nvmf_subsystem_listener_set_ana_state", 
SPDK_RPC_REGISTER("nvmf_subsystem_add_ns", rpc_nvmf_subsystem_add_ns, SPDK_RPC_RUNTIME)        // 给NVM subsystem 增加一个namespace
SPDK_RPC_REGISTER("nvmf_subsystem_remove_ns", rpc_nvmf_subsystem_remove_ns, SPDK_RPC_RUNTIME)  // 给NVM subsystem删除一个name space
SPDK_RPC_REGISTER("nvmf_subsystem_add_host", rpc_nvmf_subsystem_add_host, SPDK_RPC_RUNTIME)    // 给NVM subsystem 增加一个可以访问的host，主要用于访问控制
SPDK_RPC_REGISTER("nvmf_subsystem_remove_host", rpc_nvmf_subsystem_remove_host,                // 给NVM subsystem 删除一个可访问的host, 主要用于访问控制
SPDK_RPC_REGISTER("nvmf_subsystem_allow_any_host", rpc_nvmf_subsystem_allow_any_host,          // 设置NVM subsystem是否可以给任何host 访问
/* private */ SPDK_RPC_REGISTER("nvmf_create_target", rpc_nvmf_create_target, SPDK_RPC_RUNTIME);
/* private */ SPDK_RPC_REGISTER("nvmf_delete_target", rpc_nvmf_delete_target, SPDK_RPC_RUNTIME);
/* private */ SPDK_RPC_REGISTER("nvmf_get_targets", rpc_nvmf_get_targets, SPDK_RPC_RUNTIME);
SPDK_RPC_REGISTER("nvmf_create_transport", rpc_nvmf_create_transport, SPDK_RPC_RUNTIME) // 创建相应的transport
SPDK_RPC_REGISTER("nvmf_get_transports", rpc_nvmf_get_transports, SPDK_RPC_RUNTIME)     // 得到所有的transport
SPDK_RPC_REGISTER_ALIAS_DEPRECATED(nvmf_get_transports, get_nvmf_transports)
SPDK_RPC_REGISTER("nvmf_get_stats", rpc_nvmf_get_stats, SPDK_RPC_RUNTIME)
SPDK_RPC_REGISTER("nvmf_subsystem_get_controllers", rpc_nvmf_subsystem_get_controllers,
SPDK_RPC_REGISTER("nvmf_subsystem_get_qpairs", rpc_nvmf_subsystem_get_qpairs, SPDK_RPC_RUNTIME);
SPDK_RPC_REGISTER("nvmf_subsystem_get_listeners", rpc_nvmf_subsystem_get_listeners,
```

#### 添加transport
每个transport 会解析以下的信息： 诸如Type， 这个类型是必须要的，主要指明需要用哪个transport， 这里用TCP，目前也可以指RDMA, FC. 另外也可以通过rpc传入以下各种参数配置， 诸如：MAX_QUEUE_DEPTH, IN_CAPSULE_DATA_SIZE, MAX_IO_SIZE, IO_UNIT_SIZE, MAX_AQ_DEPTH. 但是这些不是必需的，如果不指定，就使用默认值.

rpc nvmf_create_transport, 它实际调用了spdk_nvmf_transport_create, 用于创建相应的transport, 当然每个类型的transport 只会被创建一次.
rpc nvmf_subsystem_add_listener对相应的transport的端口进行监听. 它通过spdk_nvmf_subsystem_pause->nvmf_rpc_listen_paused->spdk_nvmf_subsystem_add_listener将transport加入到这个subsystem的listener list中.

### NVMe-oF qpair的处理流程
比如对于tcp这个transport, 会被以下函数调用: [spdk_nvmf_transport_tcp.nvmf_tcp_accept](https://github.com/spdk/spdk/blob/v21.10/lib/nvmf/tcp.c#L3004) -> nvmf_tcp_port_accept -> nvmf_tcp_handle_connect -> spdk_nvmf_tgt_new_qpair.

spdk_nvmf_tgt_new_qpair函数在正常情况下(这里忽略一些一些异常处理的case), 会选择一个CPU core, 算法目前采用的是round-robin, 使得这个core执行_nvmf_poll_group_add. 在这个函数中，会根据qpair属于的transport，然后调用spdk_nvmf_transport_poll_group_add加入到这个transport所在的thread的polling group中.

nvmf_tgt_create_poll_group->nvmf_poll_group_poll->nvmf_transport_poll_group_poll调用spdk_nvmf_transport_tcp.poll_group_poll(即tcp transport的nvmf_tcp_poll_group_poll, 这个函数会处理每个qpair过来的请求)->nvmf_tcp_req_process-> spdk_nvmf_request_exec, 在这个函数中，也有一些处理：
1. 如果req->cmd->nvmf_cmd.opcode == SPDK_NVME_OPC_FABRIC, 执行nvmf_ctrlr_process_fabrics_cmd
1. 判断该req是不是admin qpair, 如果是, 则执行nvmf_ctrlr_process_admin_cmd, 否则nvmf_ctrlr_process_io_cmd.

无论是哪种cmd, 无论命令是不是同步，最终总要执行spdk_nvmf_request_complete, 这个函数中会调用_nvmf_request_complete->nvmf_transport_req_complete函数, 如果这个transport是TCP, 则会调用nvmf_tcp_req_complete, 然后TCP transport 会进行后续的处理.

销毁:
在通用NVMe-oF层，主要是调用spdk_nvmf_qpair_disconnect进行对qpair的断链, 比如nvmf_tcp_qpair_disconnect调用它.