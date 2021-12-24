# spdk
ref:
- [理解NVMe的内部实现原理，这一篇就够了](https://zhuanlan.zhihu.com/p/71932654)
- [NVMe/FC Host Configuration for SUSE Linux Enterprise Server 15 SP1 with ONTAP](https://docs.netapp.com/us-en/ontap-sanhost/nvme_sles15_sp3.html)

**spdk v21.10启用io_uring后iscsi_tgt和nvmf_tgt都存在问题无法使用见FAQ**

## 背景
目前这一代的闪存存储，比起传统的磁盘设备，在性能（performance）、功耗（power consumption）和机架密度（rack density）上具有显著的优势. 而且这些优势将会继续增大，使闪存存储作为下一代设备进入市场.

因此面临一个主要的挑战：随着存储设备继续发展，它将面临远远超过正在使用的软件体系结构的风险（即存储设备受制于相关软件的不足而不能发挥全部性能）. 因为吞吐量和延迟性能比传统的磁盘好太多，现在总的处理时间中，存储软件占用了更大的比例. 换句话说，存储软件栈的性能和效率在整个存储系统中越来越重要. 

Intel为了解决该矛盾, 帮助存储OEM（设备代工厂）和ISV（独立软件开发商）整合硬件而构造了一系列驱动，以及一个完善的、端对端的参考存储体系结构，被命名为Storage Performance Development Kit（SPDK）. SPDK的目标是通过同时使用Intel的网络技术，处理技术和存储技术来突出显著地提高效率和性能.

## 部分概念
- OCF : Open CAS(Cache Acceleration Software) Framework
- I/OAT devices : 即Intel I/O Acceleration Technology设备

## 架构
ref:
- [spdk探秘—–基本框架及bdev范例分析](https://blog.csdn.net/cyq6239075/article/details/106732499)

![SPDK Architecture 19.01](/misc/img/io/spdk/spdk_arch_1901.jpeg)

核心原理: 运行于用户态和轮询模式
- 避免内核上下文切换和中断将会节省大量的处理开销，允许更多的时钟周期被用来做实际的数据存储
- 轮询模式驱动（Polled Mode Drivers, PMDs）改变了I/O的基本模型

    在旋转设备时代（磁带和机械硬盘），中断开销只占整个I/O时间的一个很小的百分比. 然而，在固态设备的时代，持续引入更低延迟的持久化设备，中断开销成为了整个I/O时间中不能被忽视的部分. 这个问题在更低延迟的设备上只会越来越严重.


SPDK的指导原则是通过消除每一处额外的软件开销来提供最少的延迟和最高的效率. SPDK的目标是能够把硬件平台的计算、网络、存储的最新性能进展充分发挥出来.

spdk层次:
1. drivers

    - NVMe Devices：SPDK的基础组件，这个高优化无锁的驱动提供了高扩展性，高效性和高性能

        - NVMe over Fabrics（NVMe-oF）initiator：从程序员的角度来看，本地SPDK NVMe驱动和NVMe-oF启动器共享一套共同的API命令。这意味着，比如本地/远程复制非常容易实现.
    - Inter QuickData Technology：也称为Intel I/O Acceleration Technology（Inter IOAT，英特尔I/O加速技术），这是一种基于Xeon处理器平台上的copy offload引擎。通过提供用户空间访问，减少了DMA数据移动的阈值，允许对小尺寸I/O或NTB的更好利用。
1. storage services
    - Block device abstration layer（bdev）：这种通用的块设备抽象是连接到各种不同设备驱动和块设备的存储协议的粘合剂。还在块层中提供灵活的API用于额外的用户功能（磁盘阵列，压缩，去冗等等）
    - Ceph RADOS Block Device（RBD）：使Ceph成为SPDK的后端设备，比如这可能允许Ceph用作另一个存储层
    - Blobstore：为SPDK实现一个高精简的文件式语义（非POSIX）。这可以为数据库，容器，虚拟机或其他不依赖于大部分POSIX文件系统功能集（比如用户访问控制）的工作负载提供高性能基础
    - Linux Asynchrounous I/O（AIO）：允许SPDK与内核设备（比如机械硬盘）交互
    - Logical Volume：类似于内核软件栈中的逻辑卷管理，SPDK通过Blobstore的支持，同样带来了用户态逻辑卷的支持，包括更高级的按需分配、快照、克隆等功能
- storage protocols

    - iSCSI target

        建立了通过以太网的块流量规范，大约是内核LIO效率的两倍。现在的版本默认使用内核TCP/IP协议栈, 后期会加入对用户态TCP/IP协议栈的集成.
    - NVMe-oF target

        实现了新NVMe-oF规范。将本地的高速设备通过网络暴露出来，结合SPDK通用块层和高效用户态驱动，实现跨网络环境下的丰富特性和高性能。支持的网络不限于RDMA一种，FC，TCP等作为Fabrics的不同实现，会陆续得到支持.

        虽然这取决于RDMA硬件，NVMe-oF的目标可以为每个CPU核提供高达40Gbps的流量

        NVMe-oF协议在某种程度上希望替换掉iSCSI 协议, 屏蔽transport的差异.

        **SPDK支持的核心业务主要就是NVMe Over Fabrics即NVMe-oF.**


        spdk nvmf概念:
        - subsystem：spdk创建的nvme controller, 相当于nvme controller，拥有了subsystem就可以被nvme host识别到并挂载了, 就算它名下没有namespace也可以.

    - vhost-scsi target

        KVM/QEMU的功能利用了SPDK NVMe驱动，使得访客虚拟机访问存储设备时延迟更低，使得I/O密集型工作负载的整体CPU负载减低,支持不同的设备类型供虚拟机访问，比如SCSI, Block, NVMe块设备.

 
从流程上来看，spdk有数个子构件组成，包括网络前端、处理框架和存储后端:
- 前端由DPDK网卡驱动、用户态网络服务UNS（这是一个Linux内核TCP/IP协议栈的替代品，能够突破通用TCP/IP协议栈的种种性能限制瓶颈）组成. 

    DPDK给网卡提供一个高性能的包处理框架, 为数据从网卡到用户态空间提供一个数据快速通道；用户态网络服务则破解TCP/IP包并生成iSCSI命令
- 处理框架得到包的内容，并将iSCSI命令翻译为SCSI块级命令. 不过，在将这些命令送给后端驱动之前，SPDK提供一个API框架以加入用户指定的功能，即spcial sauce. 例如缓存，去冗，数据压缩，加密，RAID和纠删码计算等，诸如这些功能都包含在SPDK中。不过这些功能仅仅是为了帮助我们模拟应用场景，需要经过严格的测试优化才可使用。
- 数据到达后端驱动，在这一层中与物理块设备发生交互，即读与写

SPDK包括了几种存储介质的用户态轮询模式驱动：
- NVMe设备
- linux异步IO设备如传统磁盘
- 基于块地址的内存应用的内存驱动（如RAMDISKS）
- 使用Intel I/O加速技术设备

    Intel QuickData Technology：也称为Intel I/O Acceleration Technology（Inter IOAT，英特尔I/O加速技术）, 这是一种基于Xeon处理器平台上的copy offload引擎. 它通过提供用户空间访问，减少了DMA数据移动的阈值，允许对小尺寸I/O或NTB的更好利用.

spkd代码模块化分层, 以实现很强的适应性和扩展性:
1. 基础环境层

    主要是调用dpdk接口进行底层运行环境的初始化, 同时构建spdk基于cpu核进行串行, 轮询的调度机制
2. 诸如bdev, nvmf, nvme等分别归集的核心功能模块

    实现具体的业务逻辑
3. 特定应用示例程序和测试框架代码, 集成和运行各个功能的组件

## 其他
- [SPDK用户态hotplug处理](https://blog.csdn.net/weixin_37097605/article/details/101514668)
- [打造用户态存储利器，基于SPDK的存储引擎Blobstore & BlobFS](https://cloud.tencent.com/developer/article/1442627)

## 构建并执行
### 1. 构建
ref:
- [安装SPDK](https://pancrasl.gitee.io/2021/03/04/spdk-install/)

```bash
git clone --depth 1 -b v21.10 https://github.com/spdk/spdk
cd spdk
git submodule update --init
vim scripts/pkgdep.sh # `for id in $ID $ID_LIKE; do`上方添加`ID_LIKE="debian"`和`VERSION_ID="10.8"`用于指定os env # for [deepin 20.3](https://www.deepin.org/zh/2021/03/31/deepin-20-2-beautiful-and-wonderful/)
scripts/pkgdep.sh -u # 这里启用io_uring, 最简单的方法是使用`-a`(安装全部依赖)
./configure --with-uring --enable-debug
make
./test/unit/unittest.sh # 执行单元测试，查看是否安装成功
```

### 2. pre
在运行SPDK应用程序之前, 必须分配一些大页面, 并且必须从本机内核驱动程序中取消绑定任何NVMe和I/OAT设备.

SPDK包含一个脚本，可以在Linux和FreeBSD上自动执行此过程, 该脚本应该以root身份运行, 它只需要在系统上运行一次, 此后nvme设备将从`/dev`下消失.

```bash
$ sudo NRHUGE=512 scripts/setup.sh # `NRHUGE=N`设置HugePages_Total后按个数用于分配; `HUGEMEM=N`设置HugePages_Total后按MB用于分配, **不能申请太小**
$ sudo ./build/examples/identify # 查看系统中所有NVMe设备的示例程序
```

> **HugePages需要连续的空闲内存, 碎片化的内存可能导致分配失败, 此时建议reboot后再分配**.

要将设备重新绑定回内核，您可以运行:
```bash
# 解除绑定
$ sudo scripts/setup.sh reset # 也可释放已用的HugePages
# 查看所有可用参数
$ sudo scripts/setup.sh help
$ sudo scripts/setup.sh status # 查看设备状态
```

查看kernel Hugepage信息:
```bash
# cat /proc/meminfo | grep Huge
HugePages_Total:      512 # 总个数, 即`cat /proc/sys/vm/nr_hugepages`=`/sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages`
Hugepagesize:       2048 kB # 每个大小, 当前(默认)是2M
HugePages_Rsvd:        0
...
# grep -i huge /proc/mounts # 检查是否使用了hugepages
...
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,pagesize=2M) # 即挂载在/dev/hugepages
# ls /dev/hugepages # hugepages映射情况(名称中会包含使用该hugepage的pid, 比如`spdk_pid25051map_203`), 实测发现iscsi_tgt正常停止后, 有一个hugepage(`spdk_pid<pid>map_0)`不会释放.
# grep -B 11 'KernelPageSize:     2048 kB' /proc/<PID>/smaps | grep "^Size:" | awk 'BEGIN{sum=0}{sum+=$2}END{print sum/1024}'# 计算指定进程使用的huge page大小, 假设HugePage大小为2048 kB，输出单位为MB
```

### 3. 使用SPDK部署iSCSI
ref:
- [iSCSI Target Getting Started Guide](https://spdk.io/doc/iscsi.html)
- [使用SPDK部署iSCSI - (含有spdk iscsi相关概念)](https://pancrasl.gitee.io/2021/03/04/spdk-iscsi/)

server:
```bash
# build/bin/iscsi_tgt # iscsi_tgt中的tgt是target的意思
# --- other terminal
# scripts/rpc.py bdev_malloc_create -b Malloc0 64 512 # Construct one 64MB Malloc block devices with 512B sector size "Malloc0"
# scripts/rpc.py iscsi_create_portal_group 1 0.0.0.0:3260 # 创建ID为1的新的端口组, 地址为 0.0.0.0:3111
# scripts/rpc.py iscsi_create_initiator_group 2 ANY ANY # 创建一个ID为2的启动器组以接受任意initiator(第一个ANY)的任意连接([第二个ANY不能使用`0.0.0.0`](https://github.com/spdk/spdk/issues/2291)).
# scripts/rpc.py iscsi_create_target_node disk1 "Data Disk1" "Malloc0:0" 1:2 64 -d # 使用先前创建的bdev构建一个目标，如LUN0（Malloc0），名称为“disk1”，别名为“Data Disk1”，使用portal group 1和Initiator group 2, 64是queue_depth, `-d`禁用CHAP
# scripts/rpc.py save_config > iscsi.json # 此时建议保存配置, 因为之前使用`iscsiadm -m discovery -t sendtargets -p 127.0.0.1`时出现过iscsi_tgt进入死循环的情况
```

server添加nvme设备作为bdev:
```bash
# --- 获取NVMe设备的地址
# scripts/setup.sh status
Hugepages
node     hugesize     free /  total
node0   1048576kB        0 /      0
node0      2048kB      655 /   1024

Type     BDF             Vendor Device NUMA    Driver           Device     Block devices
NVMe     0000:03:00.0    15ad   07f0   0       uio_pci_generic  -          -

# --- 连接本地PCIe驱动器，在系统中创建物理设备的NVMe bdev
# scripts/rpc.py bdev_nvme_attach_controller -b NVMe1 -t PCIe -a 0000:03:00.0
NVMe1n1

# --- 查看块设备
# scripts/rpc.py bdev_get_bdevs
...
    "name": "NVMe1n1",
    "aliases": [],
...
```

也可以基于其他设备创建bdev，详见[Block Device User Guide](https://spdk.io/doc/bdev.html)

client:
```bash
# iscsiadm -m discovery -t sendtargets -p 127.0.0.1
127.0.0.1:3260,1 iqn.2016-06.io.spdk:disk1
# iscsiadm -m node -T iqn.2016-06.io.spdk:disk1 --login
```

其他命令:
```bash
# scripts/rpc.py -h |grep -e "bdev_.*_create" # 查看[bdev支持创建的设备](https://github.com/spdk/spdk/tree/master/module/bdev)
# scripts/rpc.py bdev_get_bdevs # 查看块设备
# scripts/rpc.py save_config > iscsi_tgt.cfg.json # dump iscsi target config
# scripts/rpc.py load_config < iscsi_tgt.cfg.json # 导入iscsi target config
# scripts/rpc.py bdev_get_iostat # 获取设备IO流量和负载信息
```

### 3. **使用SPDK部署NVMe over TCP**
ref:
- [使用SPDK部署NVMe over TCP](https://pancrasl.gitee.io/2021/03/08/spdk-nvme/)
- [NVMe-oF Target Getting Started Guide](https://spdk.io/doc/nvmf.html)
- [CentOS7 + SPDKでNVMe-oF (NVMe over Fabric) target構築](https://metonymical.hatenablog.com/entry/2018/07/22/094004)

server:
```bash
# build/bin/nvmf_tgt
# --- other terminal
# scripts/rpc.py nvmf_create_transport -t TCP -u 16384 -m 8 -c 8192
# scripts/rpc.py bdev_malloc_create -b Malloc0 64 512
# scripts/rpc.py nvmf_create_subsystem nqn.2016-06.io.spdk:cnode1 -a -s SPDK00000000000001 -d SPDK_Controller1 # 创建子系统, 没有`-a`(allow any host)即需要nvmf_subsystem_add_host添加allow host
# scripts/rpc.py nvmf_subsystem_add_ns nqn.2016-06.io.spdk:cnode1 Malloc0 # 给nvm subsystem添加一个namespace, 即  将Malloc0块设备分配给刚刚创建的子系统
# scripts/rpc.py nvmf_subsystem_add_listener nqn.2016-06.io.spdk:cnode1 -t TCP -a 192.168.88.236 -s 4420 # 即`NVMe/TCP Target Listening on 192.168.88.236 port 4420`, [这里`-a`必须是明确的ip(比如127.0.0.1), 否则`nvme connect`会报错](https://github.com/spdk/spdk/issues/2294).
```

client:
```bash
# apt install nvme-cli
# modprobe nvme-tcp
# nvme discover -t tcp -a 192.168.88.236 -s 4420

Discovery Log Number of Records 1, Generation counter 6
=====Discovery Log Entry 0======
trtype:  tcp
adrfam:  ipv4
subtype: nvme subsystem
treq:    not required
portid:  0
trsvcid: 4420
subnqn:  nqn.2016-06.io.spdk:cnode1
traddr:  192.168.88.236
sectype: none
# nvme connect -t tcp -n "nqn.2016-06.io.spdk:cnode1" -a 192.168.88.236 -s 4420
# nvme disconnect -n "nqn.2016-06.io.spdk:cnode1" # 取消挂载 by subnqn, 也可使用`nvme disconnect-all`取消全部
# nvme disconnect -d /dev/nvme0 # 通过指定盘符disconnect
# build/examples/perf -c [3] -q 128 -o 4096 -w randread -s 64 -t 10 -r 'trtype:TCP adrfam:IPv4 traddr:192.168.88.236 trsvcid:4420' # 使用tcp模式运行perf测试, 测试前要`nvme connect`否则target易提示`Disconnecting host  from subsystem nqn.2016-06.io.spdk:cnode1 due to keep alive timeout`之后perf卡死无法`ctrl+c`退出
```

### 4. **使用SPDK部署NVMe over rdma(使用SoftRoCE)**
ref:
- [用SoftRoCE测试SPDK NVMe-oF target](https://www.sdnlab.com/21248.html)
- [**使用SPDK实现NVMe over Fabrics Target**](https://cloud.tencent.com/developer/article/1694515)
- [RDMA之RoCE & Soft-RoCE](https://zhuanlan.zhihu.com/p/361740115)
- [Configuring Soft-RoCE](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_infiniband_and_rdma_networks/configuring-roce_configuring-infiniband-and-rdma-networks)

#### pre
```bash
# scripts/pkgdep.sh -r
# ./configure --with-rdma --enable-debug # 此时构建的nvmf_tgt也支持NVMe over TCP
# make
# --- 以下在server和client都执行 ---
# modprobe rdma_rxe nvme_rdma nvme_fabrics # 服务端和客户端都要load softRoCE的kernel模块
# --- 添加rxe网卡, **推荐使用iproute2**
# apt install ibverbs-utils rdma-core # for rxe_cfg, [rxe functionality is all in rdma-core now](https://github.com/SoftRoCE/librxe-dev/issues/13), 因此不需要构建https://github.com/SoftRoCE/librxe-dev.git, 且[Upstream will replace rxe_cfg with iproute2](https://bugzilla.redhat.com/show_bug.cgi?id=1784983)
# rxe_cfg start
  Name        Link  Driver  Speed  NMTU  IPv4_addr       RDEV  RMTU  
  enp2s0      yes   r8169          1500  192.168.88.236              
  uengine0    yes   bridge         1500  192.168.250.1               
  veth2D8JJR  yes   veth           1500
# rxe_cfg add enp2s0 # 添加一张合适的nic
# xe_cfg status # 检查enp2s0是否出现`RDEV=rxe0`
# apt install infiniband-diags # for ibstat
# ibv_devices/ibv_devinfo/ibstat rxe0 # 查看rdma设备,ibv_devinfo支持查看active_mtu
# rxe_cfg remove enp2s0 # 移除SoftRoCE设备
```

rxe网卡配置也可使用iproute2, 它提供rdma工具，支持`rdma link add`命令, **推荐使用**:
```bash
# rdma link add rxe0 type rxe netdev enp2s0 # rxe0是期望的RDMA的设备名, 可任意取名. enp2s0为Soft-RoCE设备所绑定的网络设备名
# rdma link # 查看是否添加成功
```

rdma网卡测试:
```bash
# --- 用rping测试, 是否可正常通信
# apt install rdmacm-utils
# rping -s -a 192.168.80.100 -v -C 10 # on server to listen类似于tcp server, 因此`rping -s`要有`rping -c`同时执行, `-a`是server ip
# rping -c -a 192.168.80.100 -v -C 10 # on client类似于tcp client, `-a`是server ip
# ---- 用ibv_rc_pingpong
# ibv_rc_pingpong -g 2 -d rxe0 # on server, `-g`是设置网卡的GID值必须正确(即client和server的GID类型要相同)否则不能通信
# ibv_rc_pingpong -g 2 -d rxe0 <server_ip> # on client
# --- perftest测试: 用来测试SEND操作的带宽
# apt install perftest
# ib_send_bw -d rxe0 # on server, 同rping是C/S模式
# ib_send_bw -d rxe0 <server_ip> # on client
```

#### 测试
server:
```bash
# build/bin/nvmf_tgt
# --- other terminal
# scripts/rpc.py nvmf_create_transport -t RDMA -u 8192 -m 4 -c 0
# scripts/rpc.py bdev_malloc_create -b Malloc0 64 512
# scripts/rpc.py nvmf_create_subsystem nqn.2016-06.io.spdk:cnode1 -a -s SPDK00000000000001 -d SPDK_Controller1
# scripts/rpc.py nvmf_subsystem_add_ns nqn.2016-06.io.spdk:cnode1 Malloc0
# scripts/rpc.py nvmf_subsystem_add_listener nqn.2016-06.io.spdk:cnode1 -t rdma -a 192.168.88.236 -s 4420 # 即`NVMe/RDMA Target Listening on 192.168.88.236 port 4420`
```

client:
```bash
# nvme discover -t rdma -a 192.168.88.236 -s 4420

Discovery Log Number of Records 1, Generation counter 6
=====Discovery Log Entry 0======
trtype:  rdma
adrfam:  ipv4
subtype: nvme subsystem
treq:    not required
portid:  0
trsvcid: 4420
subnqn:  nqn.2016-06.io.spdk:cnode1
traddr:  192.168.88.236
rdma_prtype: not specified
rdma_qptype: connected
rdma_cms:    rdma-cm
rdma_pkey: 0x0000
# nvme connect -t rdma -n "nqn.2016-06.io.spdk:cnode1" -a 192.168.88.236 -s 4420
# nvme disconnect -n "nqn.2016-06.io.spdk:cnode1" # 取消target, 也可使用`nvme disconnect-all`取消全部
```

#### perf
ref:
- [SPDK中常用的性能测试工具](https://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653337823&idx=1&sn=815db74da9e0d5ca0309c0f1a12514f1)
- [SPDK NVMe-oF RDMA (Target & Initiator) Performance Report Release 21.10](https://ci.spdk.io/download/performance-reports/SPDK_rdma_perf_report_2110.pdf)

perf options:
- -c <core mask for I/O submission/completion>
- -q <io depth>
- -t <time in seconds>
- -w <io pattern type: write|read|randread|randwrite>
- -s <DPDK huge memory size in MB>
- -o <io size in bytes>
- -r <transport ID for local PCIe NVMe or NVMeoF>

```bash
# cat bdev.json
{
    "subsystems":[
        {
            "subsystem":"bdev",
            "config":[
                {
                    "method":"bdev_malloc_create",
                    "params":{
                        "name":"Malloc0",
                        "num_blocks":131072,
                        "block_size":512
                    }
                }
            ]
        }
    ]
}
# test/bdev/bdevperf/bdevperf -c [3] -q 128 -s 64 -w randread -t 10 -o 4096 --json bdev.json # Malloc0 64M
# head:
# 1. 64M  : Total                       :  442041.60 IOPS    1726.72 MiB/s
# 2. 128M : Total                       :  440208.00 IOPS    1719.56 MiB/s
# build/examples/perf -c [3] -q 128 -o 4096 -w randread -s 64 -t 10 -r 'trtype:RDMA adrfam:IPv4 traddr:192.168.88.236 trsvcid:4420' # on client/server, 使用RDMA模式运行perf测试, 因为连入server测试, 因此此时不需要挂载盘. nvmf_tgt除transport外其他参数均相同(`-c [3] -q 128 -s 64 -w randread -t 10 -o 4096`, 且target,initiator是同机)的情况下, 本机测试结果:
# head:  Device Information                                                              :       IOPS      MiB/s    Average(us)        min        max
# 0. Malloc0(64M, 仅bdevperf)                                 Total                       :  333056.00     1301.00
# 1. Malloc0(64M, nvmf_tgt(one cpu0, 没有设入任何参数)+bdevperf(on cpu2)一起跑): Total       :  142502.40      556.65 #  bdevperf时也在cpu 0上生成了一个reactor_0与nvmf_tgt共用cpu core而影响了测试
# 2. tcp:       Total                                                                     :   45610.50     178.17    2806.46    1354.09    5352.58 ??? 性能不高
# 3. SoftRoCE:  Total                                                                     :   21293.68      83.18    6012.37    1565.29    7380.50
# 用ibv_devinfo查看rxe0, 发现active_mtu是max_mtu的1/4.
# build/examples/perf -c [3] -q 128 -o 4096 -w randread -s 64 -t 10 -r 'trtype:PCIe traddr:0000:06:00.0' # on server for local NVMe over PCIe, traddr from 'scripts/setup.sh status'
```

## dev
ref:
- [SPDK官方技术文章](https://spdk.io/cn/articles/)
- [SPDK 应用编程框架](https://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653334735&idx=1&sn=b81c263cffc74cf42338d2edda371d2c)
- [使用SPDK lib搭建自己的NVMe-oF Target应用](https://blog.csdn.net/weixin_37097605/article/details/108114450)
- [搭建远端存储，深度解读SPDK NVMe-oF target](https://mp.weixin.qq.com/s/ohPaxAwmhGtuQQWz--J6WA)
- [SPDK NVMe Reservation使用简介](https://mp.weixin.qq.com/s?__biz=MzI3NDA4ODY4MA==&mid=2653335852&idx=1&sn=5e08566473a1e2f14b9d1f697c4995cc)
- [SPDK块设备bdev简介](https://www.cnblogs.com/whl320124/articles/10064186.html)
- [Writing a Custom Block Device Module](https://spdk.io/doc/bdev_module.html)

> SPDK和DPDK的关系: SPDK 提供了一套环境抽象化库 (位于lib/env目录)，主要用于管理SPDK存储应用所使用的CPU资源、内存和PCIe等设备资源，其中DPDK是SPDK缺省的环境库.

> 使用SPDK application framework的应用程序, 都需要用CPU的亲和性（CPU Affinity）绑定某几个CPU, 对于SPDK的线程和应用线程之间的竞争可使用[SPDK RBD bdev的解决方案即spdk_call_unaffinitized](https://www.sdnlab.com/25330.html).

SPDK提供了一套编程框架 (SPDK Application Framework)，用于指导软件开发人员基于SPDK的用户态NVMe驱动以及用户态块设备层 (User space Bdev) 构造高效的存储应用. 用户有两种选择:
1. 直接使用SPDK应用编程框架实现应用的逻辑
2. 使用SPDK编程框架的思想，改造应用的编程逻辑，以更好的适配SPDK的用户态NVMe驱动

总体而言，SPDK的应用框架可以分为以下几部分:
1. 对CPU core和线程的管理
1. 线程间的高效通信
1. I/O的的处理模型以及数据路径(data path)的无锁化机制

## FAQ
### spdk config
结合源码: spdk_subsystem_init_from_json_config()和`SPDK_RPC_REGISTER("bdev_set_options", rpc_bdev_set_options, SPDK_RPC_STARTUP)`和`[spdk/test/json_config/extra_key.json](https://github.com/spdk/spdk/blob/master/test/json_config/extra_key.json)`

解决方法:
```bash
# cat iscsi.json
{
  "subsystems": [
    {
      "subsystem": "bdev",
      "config": [
        {
          "params": {
            "large_buf_pool_size": 128
          },
          "method": "bdev_set_options"
        }
      ]
    }
  ]
}
# build/bin/iscsi_tgt --json iscsi.json
```

### 执行`sudo ./build/examples/identify`报`Cannot get hugepage information`
未执行`scripts/setup.sh`去设置hugepage

### 执行`sudo ./build/bin/iscsi_tgt`报`bdev.c:1413:spdk_bdev_initialize: *ERROR*: create rbuf large pool failed`或`iscsi_subsystem.c:  96:iscsi_initialize_pdu_pool: *ERROR*: create PDU immediate data pool failed`
之前用`sudo NRHUGE=128 scripts/setup.sh`申请HugePages, 128M太小, 不够用.

> g_bdev_mgr.buf_small_pool(最小:8191*8960近70M) + g_bdev_mgr.buf_small_pool(最小:1023*68096近67M) > 128

解决方法: 多申请HugePages

经验证:
- `sudo NRHUGE=256 scripts/setup.sh`: 不行
- `sudo NRHUGE=512 scripts/setup.sh`: 可行

> 通过结合`ls /dev/hugepages`的hugepages映射情况, 我的测试环境iscsi_tgt默认需要`356*2`MB.

### spdk崩溃或`kill -9`后HugePages不释放
解决方法:`scripts/setup.sh reset && scripts/setup.sh`

### [`iscsiadm discover failed with "iscsi_pdu_hdr_handle: *ERROR*: before Full Feature"`](https://github.com/spdk/spdk/issues/2291)
去除io_uring仅使用`./configure --enable-debug`是正常的.

### [`nvme discover failed`](https://github.com/spdk/spdk/issues/2294)
去除io_uring仅使用`./configure --enable-debug`和`nvmf_subsystem_add_listener nqn.2016-06.io.spdk:cnode1 -t TCP -a 127.0.0.1 -s 4420`是正常的.

### `rxe_cfg status`报`Can't exec "ibv_devinfo": No such file or directory at /usr/bin/rxe_cfg line 215.`
`apt install ibverbs-utils`

### `nvme connect -t tcp -n "nqn.2016-06.io.spdk:cnode1" -a 192.168.88.236 -s 4420`报`Failed to write to /dev/nvme-fabrics: Operation already in progress`
情况:
1. 因为nqn.2016-06.io.spdk:cnode1已connected.
2. 因为某些原因nqn.2016-06.io.spdk:cnode1 connected但lsblk没显示出来. `nvme disconnect-all`后重新`nvme connect`即可