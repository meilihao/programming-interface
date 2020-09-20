# kernel
init/main.c是kernel启动的起点, 但不像普通c程序那样入口是`main()`, 而是`start_kernel()`. `start_kernel()`前的代码是`bios+bootloader`/`uefi`. 通常bios+bootloader是使用汇编语言来进行硬件初始化, 而uefi使用c.

> Linux下有3个特殊的进程，idle进程(PID = 0), init进程(PID = 1)和kthreadd(PID = 2), idle是唯一一个没有通过fork创建的进程. 在smp系统中，每个处理器单元有独立的一个运行队列，而每个运行队列上又有一个idle进程，即有多少处理器单元，就有多少idle进程.

start_kernel()分析:
- [init_task](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L840) : [0号进程的task_strcut](https://elixir.bootlin.com/linux/v5.9-rc5/source/include/linux/sched/task.h#L48), 具体实现在[init/init_task.c](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/init_task.c#L64)

	[init_task最终会变成idle进程](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L708).
- [trap_init()](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/main.c#L890) : 中断向量的初始化
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