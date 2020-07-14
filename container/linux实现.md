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
    - `–memory-swap`：容器能够使用的 swap 大小
    - `–memory-swappiness`：默认情况下，主机可以把容器使用的匿名页 swap 出来，可以设置一个 0-100 之间的值，代表允许 swap 出来的比例
    - `–memory-reservation`：设置一个内存使用的 soft limit，如果 docker 发现主机内存不足，会执行 OOM (Out of Memory) 操作. 这个值必须小于 --memory 设置的值- `–kernel-memory`：容器能够使用的 kernel memory 大小
    - `–oom-kill-disable`：是否运行 OOM (Out of Memory) 的时候杀死容器. 只有设置了 -m，才可以把这个选项设置为 false，否则容器会耗尽主机内存，而且导致主机应用被杀死

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

在内核里面，clone 会调用 _do_fork->copy_process->copy_namespaces，也就是说，在创建子进程的时候，有一个机会可以复制和设置 namespace.

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
cgroup 全称是 control group，顾名思义，它是用来做“控制”的, 即控制资源的使用. 当前最新版本是cgroup v2(`grep cgroup /proc/filesystems`时会看到cgroup2).

首先，cgroup 定义了下面的一系列子系统，每个子系统用于控制某一类资源:
- CPU 子系统，主要限制进程的 CPU 使用率
- cpuacct 子系统，可以统计 cgroup 中的进程的 CPU 使用报告
- cpuset 子系统，可以为 cgroup 中的进程分配单独的 CPU 节点或者内存节点
- memory 子系统，可以限制进程的 Memory 使用量
- blkio 子系统，可以限制进程的块设备 IO
- devices 子系统，可以控制进程能够访问某些设备
- net_cls 子系统，可以标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块（traffic control）对数据包进行控制
- freezer 子系统，可以挂起或者恢复 cgroup 中的进程

这里面最常用的是对于 CPU 和内存的控制, 所以下面就详细来说它.

在 Linux 上，为了操作 cgroup，有一个专门的 cgroup 文件系统，运行 `mount -t cgroup` 命令可以查看. 可以看到cgroup 文件系统均挂载到 /sys/fs/cgroup 下，通过该命令可以看到可以用 cgroup 控制哪些资源.

```bash
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
```

![cgroup 对于 Docker 资源的控制，在用户态的表现](/misc/img/container/1c762a6283429ff3587a7fc370fc090f.png)

## cgroup的内核实现
参考:
- [云计算时代，容器底层 cgroup 的代码实现分析](https://www.xujun.org/note-113352.html)

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

在 for_each_subsys 的循环里面，cgroup_subsys[]数组中的每一个 cgroup_subsys，都会调用 [cgroup_init_subsys](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/cgroup/cgroup.c#L5574)，对于 cgroup_subsys 对于初始化.

cgroup_init_subsys 里面会做两件事情，一个是调用 cgroup_subsys 的 css_alloc 函数创建一个 cgroup_subsys_state；另外就是调用 online_css，也即调用 cgroup_subsys 的 css_online 函数，激活这个 cgroup.

对于 CPU 来讲，css_alloc 函数就是 [cpu_cgroup_css_alloc](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/core.c#L8015). 这里面会调用 sched_create_group 创建一个 [struct task_group](https://elixir.bootlin.com/linux/v5.8-rc4/source/kernel/sched/sched.h#L365). 在这个结构中，第一项就是 cgroup_subsys_state，也就是说，task_group 是 cgroup_subsys_state 的一个扩展，最终返回的是指向 cgroup_subsys_state 结构的指针，可以通过强制类型转换变为 task_group.

在 task_group 结构中，有一个成员是 sched_entity，即调度的实体，也即这一个 task_group 也是一个调度实体.

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

在 cgroup_init 函数中，cgroup 的初始化还做了一件很重要的事情，它会调用 cgroup_init_cftypes(NULL, cgroup_base_files)，来初始化对于 cgroup 文件类型 cftype 的操作函数，也就是将 struct kernfs_ops *kf_ops 设置为 cgroup_kf_ops.
