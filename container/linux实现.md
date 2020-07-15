# container
容器实现封闭的环境主要要靠两种技术:
1. namespace（命名空间, 资源隔离）

    在每个 namespace 中的应用看到的，都是不同的 IP 地址、用户空间、进程 ID 等
    
1. cgroup（资源限制）

    明明整台机器有很多的 CPU、内存，但是一个应用只能用其中的一部分

> 无论是容器，还是虚拟机，都依赖于内核中的技术，虚拟机依赖的是 KVM，容器依赖的是 namespace 和 cgroup 对进程进行隔离. 容器更加轻量.

> 容器里面不包含内核，是共享宿主机的内核的. 对比虚拟机，虚拟机在 qemu 进程里面是有客户机内核的，应用运行在客户机的用户态.

目前最主流的容器技术的实现 Docker. 下一代容器架构可以约等于`Podman+Skopeo+Buildah`, 可参考[为什么要和 Docker 告别？](https://www.infoq.cn/article/OLMbGbRDw-IKekPDqkDc)

启动一个容器需要一个叫 entrypoint 的东西，也就是入口. 一个容器启动起来之后，会从这个指令开始运行，并且只有这个指令在运行，容器才启动着. 如果这个指令退出，整个容器就退出了.

```c
# docker run -it --entrypoint bash ubuntu:20.04
```

人们通常会基于docker实现其他功能, 比如:
1. 持续集成
1. 弹性伸缩
1. 跨云迁移

Docker 本身提供了限制cpu和内存的使用.
- cpu
    
    - Docker 允许用户为每个容器设置一个数字，代表容器的 CPU share，默认情况下每个容器的 share 是 1024. 这个数值是相对的，本身并不能代表任何确定的意义. 当主机上有多个容器运行时，每个容器占用的 CPU 时间比例为它的 share 在总额中的比例. Docker 为容器设置 CPU share 的参数是`-c --cpu-shares`.
    - Docker 提供了 --cpus 参数可以限定容器能使用的 CPU 核数. Docker 可以通过 --cpuset 参数让容器只运行在某些核上
- memory

    - `-m --memory`：容器能使用的最大内存大小
    - `--memory-swap`：容器能够使用的 swap 大小
    - `--memory-swappiness`：默认情况下，主机可以把容器使用的匿名页 swap 出来，可以设置一个 0-100 之间的值，代表允许 swap 出来的比例
    - `--memory-reservation`：设置一个内存使用的 soft limit，如果 docker 发现主机内存不足，会执行 OOM (Out of Memory) 操作. 这个值必须小于 --memory 设置的值- `--kernel-memory`：容器能够使用的 kernel memory 大小
    - `--oom-kill-disable`：是否运行 OOM (Out of Memory) 的时候杀死容器. 只有设置了 -m，才可以把这个选项设置为 false，否则容器会耗尽主机内存，而且导致主机应用被杀死

## [namespace](https://man7.org/linux/man-pages/man7/namespaces.7.html)
参考:
- [Separation Anxiety: A Tutorial for Isolating Your System with Linux Namespaces](https://www.toptal.com/linux/separation-anxiety-isolating-your-system-with-linux-namespaces)

对应到容器技术，为了隔离不同类型的资源，Linux 内核里面实现了以下几种不同类型的 namespace
- UTS，对应的宏为 CLONE_NEWUTS，表示不同的 namespace 可以配置不同的 hostname
- User，对应的宏为 CLONE_NEWUSER，表示不同的 namespace 可以配置不同的用户和组
- Mount，对应的宏为 CLONE_NEWNS，表示不同的 namespace 的文件系统挂载点是隔离的
- PID，对应的宏为 CLONE_NEWPID，表示不同的 namespace 有完全独立的 pid，也即一个 namespace 的进程和另一个 namespace 的进程，pid 可以是一样的，但是代表不同的进程
- Network，对应的宏为 CLONE_NEWNET，表示不同的 namespace 有独立的网络协议栈
- IPC, 对应的宏为 CLONE_NEWIPC
- Time, 对应的宏为 CLONE_NEWTIME

`docker inspect ${container id}`可在`State`中找到其运行的pid, 在对应的`/proc/${pid}/ns`下能看到该进程所属的6中namespace.

### 操作namespace的命令
- 操作 namespace 的常用指令 nsenter，可以用来运行一个进程，进入指定的 namespace, 比如`nsenter --target 58212 --mount --uts --ipc --net --pid -- env --ignore-environment -- /bin/bash`
- unshare，它会离开当前的 namespace，创建且加入新的 namespace，然后执行参数中指定的命令, 比如`unshare --mount --ipc --pid --net --mount-proc=/proc --fork /bin/bash`
- 通过函数操作 namespace

    - `int clone(int (*fn)(void *), void *child_stack, int flags, void *arg)`，也就是创建一个新的进程，并把它放到新的 namespace 中
    - `int setns(int fd, int nstype)`，用于将当前进程加入到已有的 namespace 中. 其中，fd 指向 /proc/[pid]/ns/ 目录里相应 namespace 对应的文件，表示要加入哪个 namespace; nstype 用来指定 namespace 的类型
    - `int unshare(int flags)`，它可以使当前进程退出当前的 namespace，并加入到新创建的 namespace. flags 用于指定一个或者多个上面的 CLONE_NEWXXX flag.

    > clone 和 unshare 的区别是，unshare 是使当前进程加入新的 namespace；clone 是创建一个新的子进程，然后让子进程加入新的 namespace，而当前进程保持不变

通过 clone 函数来进入一个 namespace:
```c

#define _GNU_SOURCE
#include <sys/wait.h>
#include <sys/utsname.h>
#include <sched.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#define STACK_SIZE (1024 * 1024)

static int childFunc(void *arg)
{
    printf("In child process.\n");
    execlp("bash", "bash", (char *) NULL);
    return 0;
}

int main(int argc, char *argv[])
{
    char *stack;
    char *stackTop;
    pid_t pid;

    stack = malloc(STACK_SIZE);
    if (stack == NULL)
    {
        perror("malloc"); 
        exit(1);
    }
    stackTop = stack + STACK_SIZE;

    pid = clone(childFunc, stackTop, CLONE_NEWNS|CLONE_NEWPID|CLONE_NEWNET|SIGCHLD, NULL);
    if (pid == -1)
    {
        perror("clone"); 
        exit(1);
    }
    printf("clone() returned %ld\n", (long) pid);

    sleep(1);

    if (waitpid(pid, NULL, 0) == -1)
    {
        perror("waitpid"); 
        exit(1);
    }
    printf("child has terminated\n");
    exit(0);
}
```

```bash

# echo $$       # 查看当前 bash 的进程号
64267

# ps aux | grep bash | grep -v grep
root     64267  0.0  0.0 115572  2176 pts/0    Ss   16:53   0:00 -bash

# ./a.out           
clone() returned 64360
In child process.

# echo $$
1

# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

# exit
exit
child has terminated

# echo $$           
64267
```

在内核里面，clone 会调用 [_do_fork](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/fork.c#L2416)->[copy_process](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/fork.c#L1841)->[copy_namespaces](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/nsproxy.c#L151)，也就是说，在创建子进程的时候，有一个机会可以复制和设置 namespace.

namespace 在每一个进程的 task_struct 里面，有一个指向 namespace 结构体的指针 nsproxy.
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/nsproxy.h#L31
/*
 * A structure to contain pointers to all per-process
 * namespaces - fs (mount), uts, network, sysvipc, etc.
 *
 * The pid namespace is an exception -- it's accessed using
 * task_active_pid_ns.  The pid namespace here is the
 * namespace that children will use.
 *
 * 'count' is the number of tasks holding a reference.
 * The count for each namespace, then, will be the number
 * of nsproxies pointing to it, not the number of tasks.
 *
 * The nsproxy is shared by tasks which share all namespaces.
 * As soon as a single namespace is cloned or unshared, the
 * nsproxy is copied.
 */
struct nsproxy {
	atomic_t count;
	struct uts_namespace *uts_ns;
	struct ipc_namespace *ipc_ns;
	struct mnt_namespace *mnt_ns;
	struct pid_namespace *pid_ns_for_children;
	struct net 	     *net_ns;
	struct time_namespace *time_ns;
	struct time_namespace *time_ns_for_children;
	struct cgroup_namespace *cgroup_ns;
};
```

在系统初始化的时候，有一个默认的 init_nsproxy:
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/nsproxy.c#L32
struct nsproxy init_nsproxy = {
	.count			= ATOMIC_INIT(1),
	.uts_ns			= &init_uts_ns,
#if defined(CONFIG_POSIX_MQUEUE) || defined(CONFIG_SYSVIPC)
	.ipc_ns			= &init_ipc_ns,
#endif
	.mnt_ns			= NULL,
	.pid_ns_for_children	= &init_pid_ns,
#ifdef CONFIG_NET
	.net_ns			= &init_net,
#endif
#ifdef CONFIG_CGROUPS
	.cgroup_ns		= &init_cgroup_ns,
#endif
#ifdef CONFIG_TIME_NS
	.time_ns		= &init_time_ns,
	.time_ns_for_children	= &init_time_ns,
#endif
};
```

接下来看 copy_namespaces 的实现:
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/nsproxy.c#L151
/*
 * called from clone.  This now handles copy for nsproxy and all
 * namespaces therein.
 */
int copy_namespaces(unsigned long flags, struct task_struct *tsk)
{
	struct nsproxy *old_ns = tsk->nsproxy;
	struct user_namespace *user_ns = task_cred_xxx(tsk, user_ns);
	struct nsproxy *new_ns;
	int ret;

	if (likely(!(flags & (CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC |
			      CLONE_NEWPID | CLONE_NEWNET |
			      CLONE_NEWCGROUP | CLONE_NEWTIME)))) {
		if (likely(old_ns->time_ns_for_children == old_ns->time_ns)) {
			get_nsproxy(old_ns);
			return 0;
		}
	} else if (!ns_capable(user_ns, CAP_SYS_ADMIN))
		return -EPERM;

	/*
	 * CLONE_NEWIPC must detach from the undolist: after switching
	 * to a new ipc namespace, the semaphore arrays from the old
	 * namespace are unreachable.  In clone parlance, CLONE_SYSVSEM
	 * means share undolist with parent, so we must forbid using
	 * it along with CLONE_NEWIPC.
	 */
	if ((flags & (CLONE_NEWIPC | CLONE_SYSVSEM)) ==
		(CLONE_NEWIPC | CLONE_SYSVSEM)) 
		return -EINVAL;

	new_ns = create_new_namespaces(flags, tsk, user_ns, tsk->fs);
	if (IS_ERR(new_ns))
		return  PTR_ERR(new_ns);

	ret = timens_on_fork(new_ns, tsk);
	if (ret) {
		free_nsproxy(new_ns);
		return ret;
	}

	tsk->nsproxy = new_ns;
	return 0;
}
```

如果 clone 的参数里面没有 CLONE_NEWNS | CLONE_NEWUTS | CLONE_NEWIPC | CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWCGROUP，就调用 get_nsproxy 返回原来的 namespace.

接着，调用 [create_new_namespaces](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/nsproxy.c#L67).

在 create_new_namespaces 中，可以看到对于各种 namespace 的复制. 来看 [copy_pid_ns](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/pid_namespace.c#L141) 对于 pid namespace 的复制.

在 copy_pid_ns 中，如果没有设置 CLONE_NEWPID，则返回老的 pid namespace；如果设置了，就调用 create_pid_namespace，创建新的 pid namespace.

再来看 [copy_net_ns](https://elixir.bootlin.com/linux/v5.8-rc4/source/net/core/net_namespace.c#L455) 对于 network namespace 的复制.

在copy_net_ns里面，需要判断，如果 flags 中不包含 CLONE_NEWNET，也就是不会创建一个新的 network namespace，则返回 old_net；否则需要新建一个 network namespace. 然后，copy_net_ns 会调用 net = net_alloc()，分配一个新的 struct net 结构，然后调用 setup_net 对新分配的 net 结构进行初始化，之后调用 list_add_tail_rcu，将新建的 network namespace，添加到全局的 network namespace 列表 net_namespace_list 中.

来看一下 [setup_net](https://elixir.bootlin.com/linux/v5.8-rc4/source/net/core/net_namespace.c#L324) 的实现.

在 setup_net 中有一个循环 list_for_each_entry，对于 pernet_list 的每一项 struct pernet_operations，运行 ops_init，也就是调用 pernet_operations 的 init 函数. 这个 pernet_list 是怎么来的呢？在网络设备初始化的时候，要调用 net_dev_init 函数，这里面有下面的代码.
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/net/core/net_namespace.c#L1300

/**
 *      register_pernet_device - register a network namespace device
 *	@ops:  pernet operations structure for the subsystem
 *
 *	Register a device which has init and exit functions
 *	that are called when network namespaces are created and
 *	destroyed respectively.
 *
 *	When registered all network namespace init functions are
 *	called for every existing network namespace.  Allowing kernel
 *	modules to have a race free view of the set of network namespaces.
 *
 *	When a new network namespace is created all of the init
 *	methods are called in the order in which they were registered.
 *
 *	When a network namespace is destroyed all of the exit methods
 *	are called in the reverse of the order with which they were
 *	registered.
 */
int register_pernet_device(struct pernet_operations *ops)
{
	int error;
	down_write(&pernet_ops_rwsem);
	error = register_pernet_operations(&pernet_list, ops);
	if (!error && (first_device == &pernet_list))
		first_device = &ops->list;
	up_write(&pernet_ops_rwsem);
	return error;
}
EXPORT_SYMBOL_GPL(register_pernet_device);

// https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/loopback.c#L233
/* Registered in net/core/dev.c */
struct pernet_operations __net_initdata loopback_net_ops = {
	.init = loopback_net_init,
};

/*
 *	Initialize the DEV module. At boot time this walks the device list and
 *	unhooks any devices that fail to initialise (normally hardware not
 *	present) and leaves us with a valid list of present and active devices.
 *
 */

// https://elixir.bootlin.com/linux/v5.8-rc4/source/net/core/dev.c#L10616
/*
 *       This is called single threaded during boot, so no need
 *       to take the rtnl semaphore.
 */
static int __init net_dev_init(void)
{
	...
	if (register_pernet_device(&loopback_net_ops))
		goto out;

	...
}
```

register_pernet_device 函数注册了一个 loopback_net_ops，在这里面，把 init 函数设置为 [loopback_net_init](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/loopback.c#L205).

在 loopback_net_init 函数中，会创建并且注册一个名字为"lo"的 struct net_device. 注册完之后，在这个 namespace 里面就会出现一个这样的网络设备，称为 loopback 网络设备.

这就是为什么上面的实验中，创建出的新的 network namespace 里面有一个 lo 网络设备的原因.

![/misc/img/container/56bb9502b58628ff3d1bee83b6f53cd7.png]

## cgroup
参考:
- [docker cgroup 技术之memory（首篇）](https://www.cnblogs.com/charlieroro/p/10180827.html)

cgroup 全称是 control group，顾名思义，它是用来做“控制”的, 即控制资源的使用. 当前最新版本是cgroup v2(`grep cgroup /proc/filesystems`时会看到cgroup2).

首先，cgroup 定义了下面的[一系列子系统(subsystem也称为resource controller)](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/cgroup_subsys.h)，每个子系统用于控制某一类资源:
- cpuset，可以为 cgroup 中的进程分配单独的 CPU 节点或者NUMA节点
- cpu，主要限制进程的 CPU 使用率
- cpuacct，可以统计 cgroup 中的进程的 CPU 使用报告
- io，可以限制进程的块设备 IO速度
- memory，可以限制进程的 Memory 使用量, 包括process memory, kernel memory, 和swap
- devices，可以控制进程能够访问某些设备创建(mknod)
- freezer，可以挂起或者恢复 cgroup 中的进程
- net_cls，可以标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块（traffic control）/nftables对数据包进行控制. 只对发出去的网络包生效，对收到的网络包不起作用
- net_prio, 针对cgroup中的每个网络接口提供一种动态修改网络流量优先级的方法
- perf_event, 支持访问cgroup中的性能事件
- hugetlb, 为cgroup开启对大页内存的支持
- pids, 限制一个cgroup及其子孙cgroup中的总进程数
- rdma, cgroup中使用的rdma

这里面最常用的是对于 CPU 和内存的控制, 所以下面就详细来说它.

在 Linux 上，为了操作 cgroup，有一个专门的 cgroup 文件系统，运行 `mount -t cgroup` 命令可以查看. 可以看到cgroup 文件系统均挂载到 /sys/fs/cgroup 下，通过该命令可以看到可以用 cgroup 控制哪些资源.

```bash
# --- ubuntu 20.04默认系统boot时，systemd默认使用cgroup v1, 且cgroup v2挂载在/sys/fs/cgroup/unified
$ mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,name=systemd)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
cgroup on /sys/fs/cgroup/rdma type cgroup (rw,nosuid,nodev,noexec,relatime,rdma)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
$ mount -t cgroup2
cgroup2 on /sys/fs/cgroup/unified type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
# --- 用`cgroup_no_v1=all`仅启用cgroup v2的效果
$ ll /sys/fs/cgroup
总用量 0
-r--r--r--  1 root root 0 7月  15 14:26 cgroup.controllers
-rw-r--r--  1 root root 0 7月  15 14:28 cgroup.max.depth
-rw-r--r--  1 root root 0 7月  15 14:28 cgroup.max.descendants
-rw-r--r--  1 root root 0 7月  15 14:26 cgroup.procs
-r--r--r--  1 root root 0 7月  15 14:28 cgroup.stat
-rw-r--r--  1 root root 0 7月  15 14:26 cgroup.subtree_control
-rw-r--r--  1 root root 0 7月  15 14:28 cgroup.threads
-rw-r--r--  1 root root 0 7月  15 14:28 cpu.pressure
-r--r--r--  1 root root 0 7月  15 14:28 cpuset.cpus.effective
-r--r--r--  1 root root 0 7月  15 14:28 cpuset.mems.effective
drwxr-xr-x  2 root root 0 7月  15 14:26 init.scope/
-rw-r--r--  1 root root 0 7月  15 14:28 io.cost.model
-rw-r--r--  1 root root 0 7月  15 14:28 io.cost.qos
-rw-r--r--  1 root root 0 7月  15 14:28 io.pressure
-rw-r--r--  1 root root 0 7月  15 14:28 memory.pressure
drwxr-xr-x 54 root root 0 7月  15 14:28 system.slice/
drwxr-xr-x  3 root root 0 7月  15 14:27 user.slice/
```

![cgroup 对于 Docker 资源的控制，在用户态的表现](/misc/img/container/1c762a6283429ff3587a7fc370fc090f.png)

> [禁用cgroup v1的方法: 在/boot/grub/grub.cfg添加内核启动参数`cgroup_no_v1=all(Available from Linux 4.6 onwards)`或`systemd.unified_cgroup_hierarchy=1(Available from systemd v226 onwards)`](https://sourcegraph.com/github.com/torvalds/linux@v5.8-rc4/-/blob/Documentation/admin-guide/kernel-parameters.txt#L507:1). 因为cgroup v1/2默认都是开启的, 但默认使用v1, 禁用v1后就等于默认使用了v2.

	```bash
	# vim /etc/default/grub
	GRUB_CMDLINE_LINUX="cgroup_no_v1=all"
	# update-grub2
	```

## cgroup v1/2
![](/misc/img/container/20200714222559.png) from [cgroupv2-fosdem.pdf](https://chrisdown.name/talks/cgroupv2/cgroupv2-fosdem.pdf)

## cgroup v2
CGroup V2 在 Linux Kernel 4.5中被引入，并且考虑到其它已有程序的依赖，V2 会和 V1 并存几年. 针对于 CGroup V1 中 Subsystem, Herarchy, CGroup 的关系混乱，CGroup V2 中，引入 unified hierarchy 的概念，即只有一个 Hierarchy, 即将所有的controller挂载到unified hierarchy，仍然通过 mount 来挂载 CGroup V2:
```bash
mount -t cgroup2 none $MOUNT_POINT # 会将所有可用的controller自动被挂载进去
```

> cgroup v2可以使用的controller必须没有被cgroup v1使用. 如果想在cgroup v2使用已经被cgroup v1使用的controller，则需要将cgroup v1已经挂载的controller umount掉之后，才能在v2中使用.

挂载完成之后，目录下会有三个 CGroup 核心文件:
- cgroup.controllers: 一个read-only文件, 列出当前 CGroup 支持的所有 Controller，如: cpu io memory
- cgroup.procs: 在刚挂载时，Root CGroup 目录下的 cgroup.procs 文件中会包含系统当前所有的Proc PID(除了僵尸进程)。同样，可以通过将 Proc PID 写入 cgroup.procs 来将 Proc 加入到 CGroup
- cgroup.subtree_control: 用于控制该 CGroup 下 Controller 开关，只有列在 cgroup.controllers 中的 Controller 才可以被开启，默认情况下所有的 Controller 都是关闭的

	cgroup.subtree_control文件内容格式: controller之间使用空格间隔，前面用”+”表示启用,使用”-“表示停用, 比如`echo '+pids -memory' > x/y/cgroup.subtree_control`

	![](/misc/img/container/t01ba8513f0d6685aa0.png)

> 站在进程的角度来说，在挂载 CGroup V2时，所有已有Live Proc PID 都会加入到 Root CGroup，之后所有新创建的进程都会自动加入到父进程所属的 CGroup，由于 V2 只有一个 Hierarchy，因此一个进程同一时间只会属于一个 CGroup, 此时`cat /proc/${pid}/cgroup`仅返回一行内容.

## cgroup的内核实现
参考:
- [云计算时代，容器底层 cgroup 的代码实现分析](https://www.xujun.org/note-113352.html)
- [源码解析容器底层cgroup的实现](https://www.sohu.com/a/402302235_827544)
- [云计算时代，容器底层cgroup如何实现资源分组？](https://mp.weixin.qq.com/s?__biz=MjM5NzM0MjcyMQ==&mid=2650091443&idx=2&sn=2b53d6b5dac5d87a92b50afceabe0379&chksm=bedaf4dd89ad7dcbd4bc12fd92dc1645a2003614e4295d5be0ca7ec9e29c79ad3df9829db47a&scene=21#wechat_redirect)

在系统初始化的时候，cgroup 也会进行初始化: 在 [start_kernel](https://elixir.bootlin.com/linux/v5.8-rc4/source/init/main.c#L830) 中，[cgroup_init_early](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L5633) 和 [cgroup_init](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L5672) 都会进行初始化.

在 cgroup_init_early 和 cgroup_init 中均会有for_each_subsys的循环:
```c
	for_each_subsys(ss, i) {
		...
	}


// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup-internal.h#L164
/**
 * for_each_subsys - iterate all enabled cgroup subsystems
 * @ss: the iteration cursor
 * @ssid: the index of @ss, CGROUP_SUBSYS_COUNT after reaching the end
 */
#define for_each_subsys(ss, ssid)					\
	for ((ssid) = 0; (ssid) < CGROUP_SUBSYS_COUNT &&		\
	     (((ss) = cgroup_subsys[ssid]) || true); (ssid)++)
```

for_each_subsys 会在 cgroup_subsys 数组中进行循环. 这个 cgroup_subsys 数组是如何形成的呢？

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L121
/* generate an array of cgroup subsystem pointers */
#define SUBSYS(_x) [_x ## _cgrp_id] = &_x ## _cgrp_subsys,
struct cgroup_subsys *cgroup_subsys[] = {
#include <linux/cgroup_subsys.h>
};
#undef SUBSYS
```

SUBSYS 这个宏定义了这个 cgroup_subsys 数组，数组中的项定义在 [cgroup_subsys.h](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/cgroup_subsys.h) 头文件中

根据 SUBSYS 的定义，SUBSYS(cpu) 其实是[cpu_cgrp_id] = &cpu_cgrp_subsys，而 SUBSYS(memory) 其实是[memory_cgrp_id] = &memory_cgrp_subsys.

因此能够找到 cpu_cgrp_subsys 和 memory_cgrp_subsys 的定义.
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/cgroup-defs.h#L621
/*
 * Control Group subsystem type.
 * See Documentation/admin-guide/cgroup-v1/cgroups.rst for details
 */
struct cgroup_subsys {
	struct cgroup_subsys_state *(*css_alloc)(struct cgroup_subsys_state *parent_css);
	int (*css_online)(struct cgroup_subsys_state *css);
	void (*css_offline)(struct cgroup_subsys_state *css);
	void (*css_released)(struct cgroup_subsys_state *css);
	void (*css_free)(struct cgroup_subsys_state *css);
	void (*css_reset)(struct cgroup_subsys_state *css);
	void (*css_rstat_flush)(struct cgroup_subsys_state *css, int cpu);
	int (*css_extra_stat_show)(struct seq_file *seq,
				   struct cgroup_subsys_state *css);

	int (*can_attach)(struct cgroup_taskset *tset);
	void (*cancel_attach)(struct cgroup_taskset *tset);
	void (*attach)(struct cgroup_taskset *tset);
	void (*post_attach)(void);
	int (*can_fork)(struct task_struct *task,
			struct css_set *cset);
	void (*cancel_fork)(struct task_struct *task, struct css_set *cset);
	void (*fork)(struct task_struct *task);
	void (*exit)(struct task_struct *task);
	void (*release)(struct task_struct *task);
	void (*bind)(struct cgroup_subsys_state *root_css);

	bool early_init:1;

	/*
	 * If %true, the controller, on the default hierarchy, doesn't show
	 * up in "cgroup.controllers" or "cgroup.subtree_control", is
	 * implicitly enabled on all cgroups on the default hierarchy, and
	 * bypasses the "no internal process" constraint.  This is for
	 * utility type controllers which is transparent to userland.
	 *
	 * An implicit controller can be stolen from the default hierarchy
	 * anytime and thus must be okay with offline csses from previous
	 * hierarchies coexisting with csses for the current one.
	 */
	bool implicit_on_dfl:1;

	/*
	 * If %true, the controller, supports threaded mode on the default
	 * hierarchy.  In a threaded subtree, both process granularity and
	 * no-internal-process constraint are ignored and a threaded
	 * controllers should be able to handle that.
	 *
	 * Note that as an implicit controller is automatically enabled on
	 * all cgroups on the default hierarchy, it should also be
	 * threaded.  implicit && !threaded is not supported.
	 */
	bool threaded:1;

	/*
	 * If %false, this subsystem is properly hierarchical -
	 * configuration, resource accounting and restriction on a parent
	 * cgroup cover those of its children.  If %true, hierarchy support
	 * is broken in some ways - some subsystems ignore hierarchy
	 * completely while others are only implemented half-way.
	 *
	 * It's now disallowed to create nested cgroups if the subsystem is
	 * broken and cgroup core will emit a warning message on such
	 * cases.  Eventually, all subsystems will be made properly
	 * hierarchical and this will go away.
	 */
	bool broken_hierarchy:1;
	bool warned_broken_hierarchy:1;

	/* the following two fields are initialized automtically during boot */
	int id;
	const char *name;

	/* optional, initialized automatically during boot if not set */
	const char *legacy_name;

	/* link to parent, protected by cgroup_lock() */
	struct cgroup_root *root;

	/* idr for css->id */
	struct idr css_idr;

	/*
	 * List of cftypes.  Each entry is the first entry of an array
	 * terminated by zero length name.
	 */
	struct list_head cfts;

	/*
	 * Base cftypes which are automatically registered.  The two can
	 * point to the same array.
	 */
	struct cftype *dfl_cftypes;	/* for the default hierarchy */
	struct cftype *legacy_cftypes;	/* for the legacy hierarchies */

	/*
	 * A subsystem may depend on other subsystems.  When such subsystem
	 * is enabled on a cgroup, the depended-upon subsystems are enabled
	 * together if available.  Subsystems enabled due to dependency are
	 * not visible to userland until explicitly enabled.  The following
	 * specifies the mask of subsystems that this one depends on.
	 */
	unsigned int depends_on;
};

// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/core.c#L8015
struct cgroup_subsys cpu_cgrp_subsys = {
	.css_alloc	= cpu_cgroup_css_alloc,
	.css_online	= cpu_cgroup_css_online,
	.css_released	= cpu_cgroup_css_released,
	.css_free	= cpu_cgroup_css_free,
	.css_extra_stat_show = cpu_extra_stat_show,
	.fork		= cpu_cgroup_fork,
	.can_attach	= cpu_cgroup_can_attach,
	.attach		= cpu_cgroup_attach,
	.legacy_cftypes	= cpu_legacy_files,
	.dfl_cftypes	= cpu_files,
	.early_init	= true,
	.threaded	= true,
};

// https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/memcontrol.c#L6255
struct cgroup_subsys memory_cgrp_subsys = {
	.css_alloc = mem_cgroup_css_alloc,
	.css_online = mem_cgroup_css_online,
	.css_offline = mem_cgroup_css_offline,
	.css_released = mem_cgroup_css_released,
	.css_free = mem_cgroup_css_free,
	.css_reset = mem_cgroup_css_reset,
	.can_attach = mem_cgroup_can_attach,
	.cancel_attach = mem_cgroup_cancel_attach,
	.post_attach = mem_cgroup_move_task,
	.bind = mem_cgroup_bind,
	.dfl_cftypes = memory_files,
	.legacy_cftypes = mem_cgroup_legacy_files,
	.early_init = 0,
};
```

在 for_each_subsys 的循环里面，cgroup_subsys[]数组中的每一个 [cgroup_subsys](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/cgroup-defs.h#L621)，都会调用 [cgroup_init_subsys](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L5574)，s 和 cgroup 是多对多的关系，通过 css 实现, 由cgroup_init_subsys完成.

cgroup_init_subsys 里面会做两件事情，一个是调用 cgroup_subsys 的 css_alloc 函数创建一个 cgroup_subsys_state；另外就是调用 online_css，也即调用 cgroup_subsys 的 css_online 函数，激活这个 cgroup.

对于初始化 cgroup_subsys, 其中最重要的是它关联的 cftype 文件, mount 和 mkdir 的时候创建的一系列文件实际上就是它们, 由cgroup_init -> `cgroup_init_cftypes(NULL, cgroup_base_files)`完成

对于 CPU 来讲，css_alloc 函数就是 [cpu_cgroup_css_alloc](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/core.c#L8015). 这里面会调用 sched_create_group 创建一个 [struct task_group](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/sched.h#L365). 在这个结构中，第一项就是 [cgroup_subsys_state](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/cgroup-defs.h#L138)，也就是说，task_group 是 cgroup_subsys_state 的一个扩展，最终返回的是指向 cgroup_subsys_state 结构的指针，可以通过强制类型转换变为 task_group.

在 task_group 结构中，有一个成员是 sched_entity，即调度的实体，也即这一个 task_group 也是一个调度实体.

>  task_struct 的 cgroups 字段就是 css_set 指针类型，也就是进程指向一个 css_set 对象（某一组 css）

接下来，online_css 会被调用. 对于 CPU 来讲，online_css 调用的是 [cpu_cgroup_css_online](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/core.c#L7202). 它会调用 [sched_online_group](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/core.c#L7070)->[online_fair_sched_group](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/fair.c#L10964).

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/fair.c#L10964
void online_fair_sched_group(struct task_group *tg)
{
	struct sched_entity *se;
	struct rq_flags rf;
	struct rq *rq;
	int i;

	for_each_possible_cpu(i) {
		rq = cpu_rq(i);
		se = tg->se[i];
		rq_lock_irq(rq, &rf);
		update_rq_clock(rq);
		attach_entity_cfs_rq(se);
		sync_throttle(tg, i);
		rq_unlock_irq(rq, &rf);
	}
}
```

在这里面，对于每一个 CPU，取出每个 CPU 的运行队列 rq，也取出 task_group 的 sched_entity，然后通过 attach_entity_cfs_rq 将 sched_entity 添加到运行队列中.

对于内存来讲，css_alloc 函数就是 mem_cgroup_css_alloc. 这里面会调用 mem_cgroup_alloc，创建一个 [struct mem_cgroup](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/memcontrol.h#L201). 在这个结构中，第一项就是 cgroup_subsys_state，也就是说，mem_cgroup 是 cgroup_subsys_state 的一个扩展，最终返回的是指向 cgroup_subsys_state 结构的指针，因此可以通过强制类型转换变为 mem_cgroup.

在 cgroup_init 函数中，cgroup 的初始化还做了一件很重要的事情，它会调用 cgroup_init_cftypes(NULL, cgroup_base_files)，来初始化对于 cgroup 文件类型 cftype 的操作函数，也就是将 struct kernfs_ops *kf_ops 设置为 [cgroup_kf_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L3778).

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L4812
// cgroup_base_files is for cgroup v2, cgroup1_base_files is for cgroup v1.
/* cgroup core interface files for the default hierarchy */
static struct cftype cgroup_base_files[] = {
	{
		.name = "cgroup.type",
		.flags = CFTYPE_NOT_ON_ROOT,
		.seq_show = cgroup_type_show,
		.write = cgroup_type_write,
	},
	{
		.name = "cgroup.procs",
		.flags = CFTYPE_NS_DELEGATABLE,
		.file_offset = offsetof(struct cgroup, procs_file),
		.release = cgroup_procs_release,
		.seq_start = cgroup_procs_start,
		.seq_next = cgroup_procs_next,
		.seq_show = cgroup_procs_show,
		.write = cgroup_procs_write,
	},
	{
		.name = "cgroup.threads",
		.flags = CFTYPE_NS_DELEGATABLE,
		.release = cgroup_procs_release,
		.seq_start = cgroup_threads_start,
		.seq_next = cgroup_procs_next,
		.seq_show = cgroup_procs_show,
		.write = cgroup_threads_write,
	},
	{
		.name = "cgroup.controllers",
		.seq_show = cgroup_controllers_show,
	},
	{
		.name = "cgroup.subtree_control",
		.flags = CFTYPE_NS_DELEGATABLE,
		.seq_show = cgroup_subtree_control_show,
		.write = cgroup_subtree_control_write,
	},
	{
		.name = "cgroup.events",
		.flags = CFTYPE_NOT_ON_ROOT,
		.file_offset = offsetof(struct cgroup, events_file),
		.seq_show = cgroup_events_show,
	},
	{
		.name = "cgroup.max.descendants",
		.seq_show = cgroup_max_descendants_show,
		.write = cgroup_max_descendants_write,
	},
	{
		.name = "cgroup.max.depth",
		.seq_show = cgroup_max_depth_show,
		.write = cgroup_max_depth_write,
	},
	{
		.name = "cgroup.stat",
		.seq_show = cgroup_stat_show,
	},
	{
		.name = "cgroup.freeze",
		.flags = CFTYPE_NOT_ON_ROOT,
		.seq_show = cgroup_freeze_show,
		.write = cgroup_freeze_write,
	},
	{
		.name = "cpu.stat",
		.seq_show = cpu_stat_show,
	},
#ifdef CONFIG_PSI
	{
		.name = "io.pressure",
		.seq_show = cgroup_io_pressure_show,
		.write = cgroup_io_pressure_write,
		.poll = cgroup_pressure_poll,
		.release = cgroup_pressure_release,
	},
	{
		.name = "memory.pressure",
		.seq_show = cgroup_memory_pressure_show,
		.write = cgroup_memory_pressure_write,
		.poll = cgroup_pressure_poll,
		.release = cgroup_pressure_release,
	},
	{
		.name = "cpu.pressure",
		.seq_show = cgroup_cpu_pressure_show,
		.write = cgroup_cpu_pressure_write,
		.poll = cgroup_pressure_poll,
		.release = cgroup_pressure_release,
	},
#endif /* CONFIG_PSI */
	{ }	/* terminate */
};

// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L3778
static struct kernfs_ops cgroup_kf_ops = {
	.atomic_write_len	= PAGE_SIZE,
	.open			= cgroup_file_open,
	.release		= cgroup_file_release,
	.write			= cgroup_file_write,
	.poll			= cgroup_file_poll,
	.seq_start		= cgroup_seqfile_start,
	.seq_next		= cgroup_seqfile_next,
	.seq_stop		= cgroup_seqfile_stop,
	.seq_show		= cgroup_seqfile_show,
};
```

在 cgroup 初始化完毕之后，接下来就是创建一个 cgroup 的文件系统，用于配置和操作 cgroup. cgroup 是一种特殊的文件系统. 它的定义如下：
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L2157
struct file_system_type cgroup_fs_type = {
	.name			= "cgroup",
	.init_fs_context	= cgroup_init_fs_context,
	.parameters		= cgroup1_fs_parameters,
	.kill_sb		= cgroup_kill_sb,
	.fs_flags		= FS_USERNS_MOUNT,
};

static struct file_system_type cgroup2_fs_type = {
	.name			= "cgroup2",
	.init_fs_context	= cgroup_init_fs_context,
	.parameters		= cgroup2_fs_parameters,
	.kill_sb		= cgroup_kill_sb,
	.fs_flags		= FS_USERNS_MOUNT,
};
```

> cgroup2_fs_type.mount deleted on 90129625d9203a917fc1d3e4768976ba90d71b44 for "cgroup: start switching to fs_context "

cgroup 被组织成为树形结构，因而有 [cgroup_root](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L157)即cgrp_dfl_root, 看名字就知道，default cgroup_root，默认的 cgroup 层级结构，它在 cgroup v1 中戏份有限，在 v2 中是 c 位. cgroup_init_early -> [init_cgroup_root](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L1908) 会初始化这个 [cgroup_root](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L157).

它有一个成员 kf_root，是 cgroup 文件系统的根 [struct kernfs_root](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/kernfs.h#L180). kernfs_create_root 就是用来创建这个 kernfs_root 结构的 by `cgroup_init -> [cgroup_setup_root](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L1927)`.

就像在普通文件系统上，每一个文件都对应一个 inode，在 cgroup 文件系统上，每个文件都对应一个 struct kernfs_node 结构，当然 kernfs_root 作为文件系的根也对应一个 kernfs_node 结构.

接下来，cgroup_setup_root中的[css_populate_dir](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L1927) 会调用 [cgroup_addrm_files](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L3858)->[cgroup_add_file](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L3810)，来创建整棵文件树，并且为树中的每个文件创建对应的 kernfs_node 结构，并将这个文件的操作函数设置为 kf_ops，也即指向 [cgroup_kf_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L3778) 

从 cgroup_setup_root 返回后，接下来是创建 sysfs 的 fs/cgroup（也就是/sys/fs/cgroup 目录），注册文件系统cgroup2_fs_type.

初始化完毕，接下来就可以 mount cgroup 文件系统了.

当 mount 这个 cgroup 文件系统的时候，会调用 [cgroup_init_fs_context](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L2117)???. ，要做的一件事情是 [cgroup_get_tree](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L2084)->[cgroup_do_get_tree](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L2025)，调用 kernfs_get_tree 真的去 mount 这个文件系统. 这种特殊的文件系统对应的文件操作函数为 [kernfs_file_fops](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/kernfs/file.c#L961) by kernfs_get_tree -> kernfs_fill_super -> kernfs_get_inode -> kernfs_init_inode 的 `inode->i_fop = &kernfs_file_fops`

> cgroup_do_mount deleted on cca8f32714d3a8bb4d109c9d7d790fd705b734e5 for "cgroup: store a reference to cgroup_ns into cgroup_fs_context".

> kernfs_mount deleted on 23bf1b6be9c291a7130118dcc7384f72ac04d813 for "kernfs, sysfs, cgroup, intel_rdt: Support fs_context".

当要写入一个 CGroup 文件来设置参数的时候，根据文件系统的操作，kernfs_fop_write 会被调用，在这里面会调用 kernfs_ops 的 write 函数，根据上面的定义为 cgroup_file_write，在这里会调用 cftype 的 write 函数. 对于 CPU 和内存的 write 函数，有以下不同的定义.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/core.c#L7975
static struct cftype cpu_files[] = {
#ifdef CONFIG_FAIR_GROUP_SCHED
	{
		.name = "weight",
		.flags = CFTYPE_NOT_ON_ROOT,
		.read_u64 = cpu_weight_read_u64,
		.write_u64 = cpu_weight_write_u64,
	},
	{
		.name = "weight.nice",
		.flags = CFTYPE_NOT_ON_ROOT,
		.read_s64 = cpu_weight_nice_read_s64,
		.write_s64 = cpu_weight_nice_write_s64,
	},
#endif
#ifdef CONFIG_CFS_BANDWIDTH
	{
		.name = "max",
		.flags = CFTYPE_NOT_ON_ROOT,
		.seq_show = cpu_max_show,
		.write = cpu_max_write,
	},
#endif
#ifdef CONFIG_UCLAMP_TASK_GROUP
	{
		.name = "uclamp.min",
		.flags = CFTYPE_NOT_ON_ROOT,
		.seq_show = cpu_uclamp_min_show,
		.write = cpu_uclamp_min_write,
	},
	{
		.name = "uclamp.max",
		.flags = CFTYPE_NOT_ON_ROOT,
		.seq_show = cpu_uclamp_max_show,
		.write = cpu_uclamp_max_write,
	},
#endif
	{ }	/* terminate */
};
```

如果设置的是 cpu.weight，则调用 cpu_weight_write_u64 -> [sched_group_set_shares](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/fair.c#L11040). 在sched_group_set_shares里面，task_group 的 shares 变量更新了，并且更新了 CPU 队列上的调度实体.