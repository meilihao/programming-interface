# concurrent
## 编译乱序/执行乱序
由于编译器的优化和CPU的乱序执行等特性，CPU访问内存的顺序并不一定与预期的顺序一致. CPU可能会重新排列、推迟或者采取集中访问内存等各种策略，这是无法预知的. 但某些情况下，程序需要按照先后顺序访问内存，内存屏障就派上用场了.

barrier，实际上不属于内存屏障，称之为优化屏障更为贴切. barrier是供编译器使用的，不直接产生CPU执行的代码。默认情况 下，编译器为了更高的效率会重新安排代码，barrier就是给编译器一些提示（限制）, 比如可能的实现`#define barrier() __asm__ _valatile__("":::"memory")`, memory的作用是告诉编译器内存会有变化，语句前后可能需要重新读写内存.

优化屏障只能保证编译器的行为，但影响不了CPU的行为，CPU并不一定会按照编译器产生的代码的顺序执行，而内存屏障解决了这个问题.

见[compile](compile/compile.md)的`优化`的`barrier`.

内存屏蔽指令: 解决多核间一个核的内存行为对另一个核可见的问题, 内存屏障，保证任何出现于屏障前的写操作在屏障后的写操作之前执行

ARM的内存屏蔽指令包括:
- DMB(数据内存屏障): 在DMB之后的显示内存访问执行前, 保证所有在DMB指令之前的内存访问完成
- DSB(数据同步屏障): 等待所有在DSB指令之前的指令完成(位于此指令前的所有显示内存访问均完成, 位于此指令前的所有缓存, 跳转预测和TLB维护操作全备完成)
- ISB(指令同步屏障): flush流水线, 使得所有ISB之后执行的指令都是从缓存或内存中获得的

x86:
- wmb
- smp_wmb

	smp_xxx适用于多处理器架构的环境下，在单处理器架构上它等同于barrier.

首先这几个内存屏障都隐含了barrier操作，所以mb之前不需要barrier操作。其次，它们对屏障之前的多条读写操作可 能没有影响，CPU依然可以自主决定它们的顺序.

举例:
```c
*ptr1=0
*ptr2=0
wmb(); // ptr3的写操作会在ptr1和ptr2之后执行，但ptr1和ptr2的写操作顺序是由CPU自身决定的
*ptr3=0
```

## 中断屏蔽
中断屏蔽会影响进程调度(包括抢占).

底层原理是让cpu不响应中断. 比如ARM是屏蔽CPSR的i位.

local_irq_disable/local_irq_enable仅操作本cpu内的中断, 并不能解决SMP引发的竞态.

local_irq_save(flags)与local_irq_restore(flags), 比上述的local_irq_xxx多了操作cpu的中断位, 对于ARM就是保存和恢复CPSR.

操作中断的底半部可使用local_bh_disable/local_bh_enable.

## 原子操作
分对位和整型变量的原子操作, 均是基于CPU的硬件即基于arch.

ARM使用LDREX和SETEX指令, 同时适用于核内/多核的并发.

## 自旋锁
自旋锁主要针对SMP或`单cpu但内核可抢占`的情况, 对于单cpu和内核不支持抢占的系统, 自旋锁退化为空操作. 在单cpu和内核可抢占的系统中, 自旋锁持有期间中内核的抢占被禁止. 在多核SMP的情况下, 任何一个核拿到自旋锁, 那么该核上的抢占被暂时禁止, 但不影响其他核的抢占.

持有自旋锁的进程一直处于旋转而不是睡眠, 因此它可用在中断等禁止睡眠的场景.

> 内核可抢占的单cpu的行为类似SMP, 因此自旋锁是有必要的.

自旋锁保证临界区不受别的cpu和本cpu内的抢占进程打扰, 但在临界区内执行时, 还可能受到中断和底半部(BH)的影响.

```
spin_lock_irq() = spin_lock() + local_irq_disable()
spin_unlock_irq() = spin_unlock() + local_irq_enable()
spin_lock_irqsave() = spin_lock() + local_irq_save()
spin_unlock_restore() = spin_unlock() + local_irq_restore()
spin_lock_bh() = spin_lock() + local_bh_disable()
spin_unlock_bh() = spin_unlock() + local_bh_enable()
```

注意:
1. 自旋锁是忙等锁. 但临界区很大, 或者共享设备的时候, 需要较长时间占用锁, 此时会降低系统性能
1. 递归使用自旋锁会导致系统死锁
1. 在自旋锁锁定期间不能调用可能引起进程调度的函数. 如果进程获取自旋锁后再阻塞, 比如调用copy_from_user(), copy_to_user(), kmalloc()和msleep()等函数, 则可能导致内核崩溃

ARM的自旋锁实现基于ldrex, strex, 内存屏蔽指令dmb和dsb, wfe和sev.

CONFIG_DEBUG_SPINLOCK可用于debug自旋锁.

### 读写自旋锁(rwlock)
读写自旋锁比自旋锁颗粒度更小, 允许多读至多1写, 类似读写锁.

### 顺序锁(seqlock)
顺序锁是读写自旋锁的一种优化. 写之间互斥, 但读写不互斥, 是一种写多于读的读写锁.

```
write_seqlock_irq() = write_seqlock() + local_irq_disable()
write_sequnlock_irq() = write_sequnlock() + local_irq_enable()
write_seqlock_irqsave() = write_seqlock() + local_irq_save()
write_sequnlock_restore() = write_sequnlock() + local_irq_restore()
write_seqlock_bh() = write_seqlock() + local_bh_disable()
write_sequnlock_bh() = write_sequnlock() + local_bh_enable()
```

### 读-复制-更新(RCU, Read-Copy-Update)
该机制用于提高读操作远多于写操作时的性能.

不同于自旋锁, RCU的读端没有锁, 内存屏障, 原子指令类的开销, 几乎可以认为是直接读; RCU的写在访问它的共享资源前先复制一个副本, 然后对副本进行修改, 最后使用一个回调机制在适当时间(所有引用该数据的cpu都退出对共享数据读操作的时候, 等待适当时机的时间被称为宽限期即Grace Period)把指向原数据的指针指向副本.

RCU可看作rwlock的高性能版本, 但不能代替rwlock, 因为写比较多时, 对读执行的性能提高不能弥补写执行单元同步导致的损失, 因为写执行间的同步开销比较大.

## 信号量(semaphore)
信号量与os概念中的PV操作对应, 其值是0, 1或n. 只有得到信号量的进程才能执行临界区代码.

自旋锁与信号量的区别:
1. 信号量不能用在中断处理和tasklet等不可睡眠场景

	linux以进程为单位调度, 如果在中断上下文睡眠, 中断将不能被正常处理

	获取不到信号量时, 进程不会在原地打转而是进入**睡眠**状态.
1. 可睡眠的场景即可使用信号量, 也可使用自旋锁. 自旋锁通常用于轻量级锁场景, 即持有锁的时间很短 

## 互斥锁
mutex_lock和mutex_lock_interruptible()的区别与down()和down_trylock()的区别完全一致. 前者引起的睡眠不能被信号打断, 而后者可以. mutex_trylock()用于尝试获取mutex, 获取不到时不会引起进程睡眠.

在严格意义上, 互斥锁和自旋锁是不同层次的互斥手段. 前者的实现依赖后者, 即自旋锁是更底层的手段. 在持有自旋锁后进行调度, 抢占以及在等待队列上睡眠都是禁止的.

互斥锁是进程级别的, 用于多个进程间对资源的互斥. 由于进程上下文切换的开销很大, 因此, 只有当进程占用资源时间较上时, 用互斥锁才是较好的选择. 当要保护的临界区访问时间较短时, 用自旋锁, 可以节省上下文切换的时间. 由此, 它俩的选择原则是:
1. 当锁不能被获取到时, 使用互斥锁会进入睡眠, 此时开销是进程上下文切换的时间, 使用自旋锁的开销是等待获取自旋锁. 若临界区较小, 使用自旋锁, 否则使用互斥锁.
1. 互斥锁所保护的临界区可能引起阻塞的代码, 而自旋锁则绝对要避免用来保护包含这样代码的临界区. 因为阻塞就意味着进程切换, 如果进程被切换出去后, 其他进程来获取本自旋锁会导致死锁.
1. 互斥锁存在于进程上下文. 因此, 如果被保护的共享资源需要在中断或者软中断情况下使用, 则只能选择自旋锁. 如果一定要使用互斥锁, 则只能通过 mutex_trylock() 进行, 不能获取就立即返回以避免阻塞.

## 完成量(completion)
用于一个执行单元等待另一个执行单元执行完某事.

## 每CPU变量
每CPU变量就是每个CPU都有一份对应存储的变量, 从逻辑角度也可以将它理解为数组, 定义一个每CPU变量和一个数组变量.

```c
// https://elixir.bootlin.com/linux/v6.6.25/source/include/linux/percpu-defs.h
#define DEFINE_PER_CPU(type, name)					\
	DEFINE_PER_CPU_SECTION(type, name, "")
```

在内核中使用DEFINE_PER_CPU定义的每CPU变量，在链接的过程中会被放置到指定的特殊section中，这些section`起始于__per_cpu_start, 结束于__per_cpu_end`.

> 当使用 DEFINE_PER_CPU 宏时，一个在 .data..percpu 段中的 per-cpu 变量就被创建了, 见vmlinux.

内核启动时，这些section中的内容会被复制到内存中，同时会预留一部分内存供动态申请每CPU变量的代码使用， 由[setup_percpu_segment](https://elixir.bootlin.com/linux/v6.6.25/source/arch/x86/kernel/setup_percpu.c#L106)函数实现. 即[调用 setup_per_cpu_areas 函数多次加载 .data..percpu 段，每个 CPU 一次](https://xinqiu.gitbooks.io/linux-insides-cn/content/Concepts/linux-cpu-1.html)

从 init/main.c 中调用 setup_per_cpu_areas 函数开始， 它输出在内核配置中以 CONFIG_NR_CPUS 配置项设置的最大 CPUs 数，实际的 CPU 个数，nr_cpumask_bits（对于新的 cpumask 操作来说和 NR_CPUS 是一样的），还有 NUMA 节点个数, 见`dmesg | grep percpu`.

每cpu变量函数表:
1. DEFINE_PER_CPU: 定义类型为type, 名字为name的每cpu变量
1. per_cpu_ptr(ptr, cpu): 得到cpu对应的元素的地址
1. per_cpu(var, cpu): 得到cpu对应的元素的值
1. this_cpu_ptr(ptr): 得到当前cpu对应的变量的地址
1. get_cpu_ptr(var): 禁内核抢占, 得到当前cpu对应的变量的地址
1. put_cpu_ptr(var): 使能内核抢占
1. alloc_percpu(type): 为每个cpu变量申请内存
1. alloc_percpu_gfp(type, gfp): 为每个cpu变量申请内存
1. free_percpu(volid `*pdata`) : 释放内存

每CPU变量的使用和数组也是不同的. 从本质上来讲，定义了每CPU变量后，把它当作数组来使用，在某些平台上可能是可行的，但这样的程序不具备可移植性.