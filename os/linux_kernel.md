# kernel
init/main.c是kernel启动的起点, 但不像普通c程序那样入口是`main()`, 而是`start_kernel()`, 此时已处于保护模式. `start_kernel()`前的代码是`bios+bootloader`/`uefi`. 通常bios+bootloader是使用汇编语言来进行硬件初始化, 而uefi使用c.

> Linux下有3个特殊的进程，idle进程(PID = 0), init进程(PID = 1)和kthreadd(PID = 2), idle是唯一一个没有通过fork创建的进程. 在smp系统中，每个处理器单元有独立的一个运行队列，而每个运行队列上又有一个idle进程，即有多少处理器单元，就有多少idle进程.

start_kernel()分析:
- [init_task](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L840) : [0号进程的task_strcut](https://elixir.bootlin.com/linux/v5.9-rc5/source/include/linux/sched/task.h#L48), 具体实现在[init/init_task.c](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/init_task.c#L64)

	[init_task最终会变成idle进程](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L708).
- [trap_init()](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L890) : 中断向量的初始化

	其中的[cpu_init()](https://elixir.bootlin.com/linux/v5.9-rc6/source/arch/x86/kernel/traps.c#L1082) -> [syscall_init()](https://elixir.bootlin.com/linux/v5.9-rc6/source/arch/x86/kernel/cpu/common.c#L1902) -> [wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64)](https://elixir.bootlin.com/linux/v5.9-rc5/source/arch/x86/kernel/cpu/common.c#L1705)设置了syscall(即内嵌汇编中的syscall指令)的入口[entry_SYSCALL_64](https://elixir.bootlin.com/linux/v5.9-rc5/source/arch/x86/entry/entry_64.S#L95)
- [mm_init()](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L891) : 内存管理的初始化
- [sched_init()](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L903) : 调度模块的初始化
- [arch_call_rest_init()](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L1048) : 0号进程, 它创建了1号进程init和其他一些服务进程

	它会调到[`rest_init()`](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L663), 它会新建内核线程:
	- [kernel_init](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L1398) : 在内核空间完成初始化后, 加载init程序, 并最终转入用户空间

		Linux中的所有进程都是有init进程创建并运行的. 首先它在Linux内核中启动，然后转入用户空间启动init进程，再启动其他系统进程. 在系统启动完成完成后，init将变为守护进程监视系统其他进程.
	- [kthreadd](https://elixir.bootlin.com/linux/v5.9-rc5/source/kernel/kthread.c#L605) : 始终运行在内核空间, 负责所有内核线程的调度和管理

		它的任务就是管理和调度其他内核线程kernel_thread. 它里面的for循环作用就是如果发现kthread_create_list是一空链表，则调用schedule(), 否则进入while循环, 遍历kthread_create_list全局链表中维护的kthread, 对于该列表上的每一个entry，都会得到对应的类型为struct kthread_create_info的节点的指针create, 然后函数在kthread_create_list中删除create对应的列表entry, 接下来执行 create_kthread(create)会调用kernel_thread来生成一个新的进程. 因此所有的内核线程都是直接或者间接的以kthreadd为父进程.

	对比发现:
	1. [init_task是直接初始化的](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/init_task.c#L64). 因此init_task是唯一一个没有通过fork创建的进程

		>　原先初始化init_task的`INIT_TASK()`宏已被删除 : [Expand INIT_TASK() in init/init_task.c and remove　＠v4.16-rc1](https://github.com/torvalds/linux/commit/d11ed3ab3166a2bfad60681aebf3e13e1c3408a9)
	1. `rest_init()`中的[`kernel_thread()`](https://elixir.bootlin.com/linux/v5.9-rc5/source/kernel/fork.c#L2470)是通过fork的.

## 搭建MenuOS
1. 按照`https://github.com/meilihao/mykernel/blob/master/mykernel-2.0_for_linux-5.8.9.patch`编译出bzImage

	`.config`变更:
	- Kernel hacking -> Compile-time checks and compiler options -> Compile the kernel with debug info -> yes : 对kernel进行跟踪调试
1. 构建rootfs

	```bash
	$ mkdir rootfs
	$ git clone --depth 1 https://github.com/mengning/menu.git
	$ cd menu
	$ gcc -pthread -o init linktable.c menu.c test.c -m64 -static
	$ ./init 
	                                                            
	  *    *                                   ****       ****  
	 ***  ***     **        **      *    *    *    *     **     
	 * *  * *    *  *      *  *     *    *   *      *   **      
	 * *  * *   *    *    *    *    *    *   *      *    ***    
	 *  **  *   ******    *    *    *    *   *      *      **   
	 *      *   *         *    *    *    *   *      *       **  
	 *      *    *        *    *     *  **    *    *       **   
	 *      *     ***     *    *      **  *    ****     ****    
	                                                            
	MenuOS>>exit
	MenuOS>>^C
	$ cd ../rootfs
	$ cp ../menu/init .
	$ find .|cpio -o -Hnewc |gzip -9 > ../rootfs.img # 把当前rootfs下的所有文件打包成img
	```
1. 运行MenuOS

	```bash
	$ tree -L 1                          
	.                                                                             
	├── linux-5.8.9
	├── menu
	├── rootfs
	└── rootfs.img

	3 directories, 1 file
	$ qemu-system-x86_64 -kernel linux-5.8.9/arch/x86/boot/bzImage -initrd rootfs.img
	```
1. 调试MenuOS

```bash
$ qemu-system-x86_64 -kernel linux-5.8.9/arch/x86/boot/bzImage -initrd rootfs.img  -S -s # `-s`是创建gdb-server, 端口1234. `-S`是cpu初始化前冻结住
$ # --- 新开terminal
$ gdb -quiet
(gdb) file linux-5.8.9/vmlinux
Reading symbols from linux-5.8.9/vmlinux...done.
(gdb) target remote:1234
Remote debugging using :1234
0x000000000000fff0 in ?? ()
(gdb) break start_kernel
Breakpoint 1 at 0xffffffff8184fa39: file init/main.c, line 837.
(gdb) c
Continuing.

Breakpoint 1, start_kernel () at init/main.c:837
837             set_task_stack_end_magic(&init_task);
(gdb) b rest_init
Breakpoint 2 at 0xffffffff81173200: file init/main.c, line 667.                                                                                             
(gdb) c                                                                                                                                                     
Continuing.

Breakpoint 2, rest_init () at init/main.c:667
667             rcu_scheduler_starting();
(gdb)
```

# kernel架构
## Linux 内核上下层通信方式
定义一个结构体，包含各种函数指针. 下层通过生成一个这样的结构体，将自己的操作函数赋值给该结构体的对应域，然后调用上层的注册函数，将自己的信息注册到上层.  这样上层就可以用统一的函数调用不同的下层接口. 上层可以遍历，通过名称或设置哪个函数有效来确定调用哪个下层结构体对应的同名
操作. 这种方式使用了C语言, 同时融入了面向接口和面向过程的编程思路.

当上层调用下层的函数时, 一般有3种返回值的方式:
1. 通过函数的返回值（在大部分情况下）
1. 传递的参数是指向结果的指针
1. 使用回调函数（在内核中有广泛用处），通常让用户提供执行过程的某一处逻辑甚至是用户定义的返回结果. 回调函数通常是调用者提供.

内核首先按照功能分成了几个大型的子系统，再通过上下层通信的方式将内核各个子系统有效地分出了层级关系, 使得架构清晰, 解偶.

## 处理资源竞争
kernel对架构级别的资源竞争处理的主要思路是队列缓存, 比如将磁盘读写产生的 bio 结构体提交给设备时, 当其发现设备忙(队列不为空)时就将该请求加入队列，然后返回; 如果设备不忙(队列为空)就直接提交.

针对细节的资源竞争， kernel提供了很多的锁实现. 如果kernel出现了低效事件， 那么基本上都是由于这些锁导致的.

## kernel的strcut
kernel通过组合来实现继承.

总体来说, kernel有两种结构体模型:
1. 对对象的建模
1. 对操作的建模

两个体系互相有交叉关系, 又各有各的继承结构.

## 如果规避GPL
采用两个模块的方式来规避. 常见的是一个是 LGPL 模块，封装了内核的 GPL 调用，但是自己本
身是 LGPL 协议的，另一个模块则可以直接调用在第一个模块中封装过的接口.

这种方式严格的说也并不符 GPL 协议的要求，但是在 Linux kernel中却是被允许的. 因为 GPL 协议的传染性不能穿透二进制, 否则用 GCC 写的所有程序都得开源.

## module
模块是kernel支持动态功能扩展的最主要机制.

模块内部定义的钩子函数是 static 的，目的是只内部可见. `__init`和`__exit`是GCC 的特性，提供回收明确表示无用代码的能力.

标记为`__init`的函数被直接编译进kernel时会被放入`.init.text`段, 同时在`.initcall.init`保留一份函数指针, 在内核初始时会通过这些指针调用`__init`函数, 在完成初始化后会释放init段(包括`.init.text`,`.initcall.init`等)即回收节省内存, 因为不会再用到.

除了函数外, 数据也可以被定义为`__initdata`, 对于只是初始化阶段需要的数据, 内核在初始化完成后, 也会释放其占用的内存.

`__exit`被直接编译进内核时, 其修饰的函数会被编译器忽略, 因为内置了就无法卸载, 卸载函数也就没必要存在了. 此外退出阶段采用的数据也可用`__exitdata`修饰.

缺少`MODULE_LICENSE`时, kernel加载时就会告警:`... no license`

### 模块声明
- MOUDLE_DEVICE_TABLE: 表面所支持的设备. 对于USB, PCI等设备驱动, 通常存在该声明

#### vermagic
比如`modinfo zfs`输出的"vermagic: 5.4.0-58-generic SMP mod_unload". `5.4.0-58-generic`是内核版本信息, `SMP`即编译kernel时设置了`CONFIG_SMP=y`表示支持smp, 同理`mod_unload`即设置了`CONFIG_MODULE_UNLOAD=y`表示开启卸载模块相关的功能.

### 内核符号表
内核符号表是内核内部各个功能模块之间互相调用的纽带，各个模块之间依赖这些函数调用进行通信.  各个功能模块必须要导出符号表才能被模块使用. 还有动态加载的模块的链接需求, 在加载时符号表是对内核其他部分描述本模块的最好方
式. 加载的模块所导出的函数通过导出操作就可以被其他模块定位并调用.

kernel使用`EXPORT SYMBOL(<func_name>);`表示导出内核符号, 之后就可以被任何其他的模块和内核部分使用(仍然需要遵守GPL）. kernel其实还有其他类似的导出方式: `EXPORT_SYMBOL_GPL(<func_name>);`, 表示该函数只能被自己是遵守 GPL 协议的模块使用. 

两种导出的区别在于每个模块都可以声明本模块遵守的Lisence.

比如遵守 GPL 的模块，就可在自己的模块中添加`MODULE_LICENSE("GPL")`，或者是商业的模块可以使用`MODULE_LICENSE("xxx")`之后就可以用另外一个模块调用本模块封装的函数, 而另外的模块就不需要开源. 只有设置了遵守 GPL 的模块才可以调用`EXPORT_SYMBOL_GPL`导出的函数.

使用`cat /proc/kallsyms`可输出**包含了加载模块**的内核当前的符号表, 通过`less /boot/System.map*`可查看内核二 进制符号列表. 通过`nm vmlinux`也可查看内核符号表，但不包括module的. `nm <module_name>`可查看模块的符号表, 但是得到的是相对地址，只有加载后才会被分配绝对地址.

模块加载之后，可在用户空间通过`echo-n ${value} > /sys/module/${module_name}/parameters/${parm}`修改模块参数, 每个模块参数都有单独的文件存放.

每个模块在编译时都会从内核目录中获得版本号写入编译的模块，运行中的内核在插入新的模块时会检签名是否一致，若不一致就不会加载.

模块签名有两层含义: 一层是版本号; 另一层是哈希签名(可能没有). 使用`modinfo xxx`即可查看. 如果签名了，则会在 modinfo 的输出中多出`signer, sig_key, sig_hashalgo`这3个字段. 有的内核如果在编译的时候选择了`CONFIG_MODULE_SIG_FORCE`, 那么没有签名的模块都是拒绝加载的.

### module可用的内核组件
workqueue和tasklet都是延迟执行的机制, 不同的是.

#### 1. workqueue
Linux下的工作队列和tasklet都是一种将工作推后执行的方式. 但workqueue有自己的进程上下文即执行上下文是内核线程, 因此可以被**睡眠、调度**，与内核线程表现基本一致，但使用起来又比直接使用内核线程简单，一般用来处理任务内容比较动态的任务链. 每 workqueue 都可以添加多个 work (使用 queue_work函数); 但tasklet不可睡眠.

workqueue机制最小的调度单元是work_struct，即工作任务. 它由schedule_work()进行调度.

workqueue早期在每个cpu上创建一个worker内核线程, 所有在这个核上调度的工作都由该worker执行, 并发性不佳. 当前使用cmwq(Concurrency-managed workqueues), 它会自动维护工作队列的线程池以提高并发性, 同时保持了API的向后兼容.

kernel 提供了[create_workqueue](https://elixir.bootlin.com/linux/v5.11.1/source/include/linux/workqueue.h#L428)和[create_singlethread_workqueue](https://elixir.bootlin.com/linux/v5.11.1/source/include/linux/workqueue.h#L433)函数用于用户创建自己的工作队列和执行线程, 而不用kernel提供的工作队列.

> create_singlethread_workqueue和create_workqueue类似, 但create_singlethread_workqueue只创建一个kernel线程, 而不是为每个cpu创建一个内核线程.

[kblockd_workqueue](https://elixir.bootlin.com/linux/v5.11.1/source/block/blk-core.c#L73)是通用块层提供的工作队列, 由kblockd_schedule_work来添加work.

#### 2. 中断系统和 tasklet
Linux 中的中断分为3个层次. 最低的层次是在源代码 arch 目录下与各个平台相关的代码，一般位于平台代码下面的`irq.c`文件中，该部分代码直接与硬件相关，最后都要调用`do_IRQ(__do_IRQ)`进行执行.

`do_IRQ`是中断系统的中层，其根据下层传来的中断号找到对应的中断处理函数，处理多 CPU 访问和中断重入问题，然后调用真实的中断处理函数，也就是中断的上层.  但是这里内核做了区别，如果内核判断中断发生了嵌套（同时发生的中断很多)或者有其他的高时间成本的需求，则将中断处理函数以内核线程的形式（软中断)运行，否则直接运行.  中断的最上层则与各个中断的具体功能相关.

tasklet一般专用于中断，因为中断不能阻塞，所以耗时较长的操作都交给 tasklet在中断上下文之外调度执行, 执行时机通常是上半部返回的时候. 它基于软中断实现即它的执行上下文是软中断, 且tasklet**不可睡眠**.  软中断被内核直接使用，但是如果用户模块想要直接使用则会非常难，因为需要考虑在不同 CPU 上的调度问题，所以软中断是锁密集型的机制. 内核线程 ksoftirqd 专门用来调度软中断，而模块开发的时候希望使用这种软中断的延时执行机制，就可以调用内核封装好的 tasklet.

tasklet通过tasklet_schedule()在适当dd时候进行调度运行.

同一时刻一个tasklet只能有一个cpu执行, 不同的tasklet可以在不同的cpu上执行, 这和软中断不同, 软中断同一时刻可以在不同的cpu上并行执行, 因此软中断必须考虑重入问题.

每个cpu都有自己的tasklet链表, 由kernel根据情况决定何时执行tasklet.

中断系统是一个非常复杂的子系统, 除非深度的内核开发者, 例如比较细节的多CPU 中断、中断亲和度、中 断域等概念都是不太容易接触到的. 其中中断亲和度常被运维人员用于锁定应用性能.

`cat /proc/irq`的每个中断号下面的文件都可以进行中断亲和度的绑定, 将特定的进程绑定到特定的中断号上, 这样进程不容易被抢占.

#### 软中断(softirq)
软中断的执行时机通常是上半部返回的时候, tasklet是基于软中断实现的, 因此它也运行于软中断上下文, 不允许睡眠.

内核用softirq_action表示一个软中断, 用open_softirq注册软中断对应的处理函数, raise_softirq()可触发一个软中断.

local_bh_diable()和local_bh_enable()用于禁止和使能软中断及tasklet下半部机制的函数.

一般来说, 驱动不会也不宜直接使用softirq.

软中断或tasklet在某段时间内大量出现的话, 内核会将后续软中断放入ksoftirqd内核线程中执行. 总的来说, 中断优先级高于软中断, 软中断又高于任一线程. 软中断适度线程化, 可以缓解高负载下系统的响应.

#### 线程化irq(threaded_irq)
内核还可以使用request_threaded_irq()和devm_request_threaded_irq(), 它们比request_irq()和devm_request_irq()多了一个thread_fn, 即申请中断时, 为相应的中断号分配一个对应的内核线程, 该线程只针对这个中断号.

它们在发生中断时, 首先指向handler, 在该函数返回IRQ_WAKE_THREAD时, 内核会调度thread_fn对应的函数. 其irqflags设置IRQF_ONESHOT, 内核会自动在中断上下文屏蔽对应的中断号, 而在thread_fn执行后, 重新使能该中断号.

#### 3. 自旋锁
自旋锁用来在多处理器的环境下保护数据, 因此在单处理器且非抢占内核下, 它没有作用. 如果kernel发现数据未锁, 就会获取锁并执行; 否则就一直等待(即一直反复执行一条指令). 被自旋锁锁着的进程一直在等待而不是睡眠, 因此它可用在中断等禁止睡眠的场景.

使用方法:
```c
spin_lock();
...
spin_unlock();
```

#### 4. 内核信号量
与自旋锁类似, 作用也是保护数据. 不同在于, 进程获取内核信号量时如果不能获取则会进入睡眠状态.


内核信号量与自旋锁不同点:
1. 内核信号量不能用于中断处理和tasklet等不可睡眠场景

	本质是linux kernel以进程为单位调度, 如果在中断上下文睡眠, 中断将不能被正确处理.
1. 可睡眠创建即可用内核信号量也可用自旋锁, 但自旋锁常用于在轻量级的锁场景即锁的时间很短.

#### 5. 原子变量
原子变量提供了一种原子的, 不可中断的操作.
```c
atomic_t mapped;
```


## 特殊硬件框架
- fpga : 驱动是XillyBus, 在用户空间设备名是`/dev/xillybus_*`
- rtc : 时钟源, 设备是`/dev/rtc*`, rtc精度不高.

	kernel提供一个通用的时钟抽象层`timerkeeper`, 它里面有一个clocksource的时钟源抽象封装, 可提供更高精度. 通过`cat /sys/devices/system/clocksource/clocksource0/available_clocksource或cat /sys/devices/system/clocksource/clocksource0/current_clocksource`查看当前可用的和正在使用的时钟源.

## 特殊软件机制
- UIO : 一种在用户端实现内核驱动的机制. 其在内核中有 UIO 模块，目前
该模块只支持字符设备.

	设备名是"/dev/uioX". 支持通过`/sys/class/uio/uioX`访问其配置.
- VFIO : 用来取代UIO的框架, 允许用户端直接访问设备细节, 也就是说让用户端设备驱动成为可能.

	其主要的工作原理是用户端可以配置 IOMMU(访问设备内存的机制, 原本是为虚拟化而设计的, 因为vm在用户态, 没法高效使用设备io内存), 从而向用户暴露 DMA, 使用这个驱动需要设备与os原来的驱动解绑.

	因为新, 其目前还仅支持 PCI 设备的驱动访问(vfio-pci 模块). 另外对 CPU IOMMU实现了 x86, arm, PowerPC, 其他未知.

	其用户端的设备文件是`/dev/vfio/N`, ，用户可以使用这个设备文件实现完全的设备驱
	动程序，目前主要用途是虚拟机时的设备驱动透明访问.
- SysRq : 处理特殊问题非常有效.