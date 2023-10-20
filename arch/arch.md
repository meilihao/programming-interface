# arch
## 指令集
指令集是具体的一套指令编码, 微架构是指令集的物理实现方式.

## 体系结构(architecture)
体系结构是指令集体系结构(instruction set architecture)的简称, 是硬件和底层软件间的抽象接口, 包含了需要编写正确执行的机器语言程序所需的全部信息, 包括指令, 寄存器, 存储访问和I/O等.

当前体系结构和体系结构的实现是分开演进的.

## 特权级别
计算机有3种主要资源受到保护: 内存, I/O端口, 执行特殊指令的能力

## 应用二进制接口(application binary interface, ABI)
ABI是用户部分的指令加上应用调用的操作系统接口的合称, 定义了二进制层次可移植的计算机标准.

## 并发操作中，解决数据同步的四种方法
1. 原子操作 拿下单体变量

	由硬件实现

	```c
	//定义一个原子类型
	typedef struct s_ATOMIC{
	    volatile s32_t a_count; //在变量前加上volatile，是为了禁止编译器优化，使其每次都从内存中加载变量
	}atomic_t;
	//原子读
	static inline s32_t atomic_read(const atomic_t *v)
	{        
	        //x86平台取地址处是原子
	        return (*(volatile u32_t*)&(v)->a_count);
	}
	//原子写
	static inline void atomic_write(atomic_t *v, int i)
	{
	        //x86平台把一个值写入一个地址处也是原子的 
	        v->a_count = i;
	}
	//原子加上一个整数
	static inline void atomic_add(int i, atomic_t *v)
	{
	        __asm__ __volatile__("lock;" "addl %1,%0"
	                     : "+m" (v->a_count)
	                     : "ir" (i));
	}
	//原子减去一个整数
	static inline void atomic_sub(int i, atomic_t *v)
	{
	        __asm__ __volatile__("lock;" "subl %1,%0"
	                     : "+m" (v->a_count)
	                     : "ir" (i));
	}
	//原子加1
	static inline void atomic_inc(atomic_t *v)
	{
	        __asm__ __volatile__("lock;" "incl %0"
	                       : "+m" (v->a_count));
	}
	//原子减1
	static inline void atomic_dec(atomic_t *v)
	{
	       __asm__ __volatile__("lock;" "decl %0"
	                     : "+m" (v->a_count));
	}
	```

	[linux x86 原子变量](https://elixir.bootlin.com/linux/v6.5/source/arch/x86/include/asm/atomic.h#L17), 代码中的LOCK_PREFIX 是一个宏，根据需要展开成“lock;”或者空串: 单核心 CPU 是不需要 lock 前缀的，只要在多核心 CPU 下才需要加上 lock 前缀. Linux 定义了`__READ_ONCE,__WRITE_ONCE` 这两个宏，是对代码封装并利用 GCC 的特性对代码进行检查，把让错误显现在编译阶段, 其中的`volatile int *`是为了提醒编译器：这是对内存地址读写，不要有优化动作，每次都必须强制写入内存或从内存读取
1. 中断控制  搞定复杂变量

	关中断只能保证单核心机器上的原子性 原子操作;（lock锁总线）只能保证多核心机器上基础数据类型的原子性.

	linux中断相关代码在[](https://elixir.bootlin.com/linux/v6.5/source/include/linux/irqflags.h), 分arch层代码和具体arch层代码.

	> 编译 Linux 代码时，编译器自动对宏进行展开。其中，do{}while(0)是 Linux 代码中一种常用的技巧，do{}while(0) 表达式会保证{}中的代码片段执行一次，保证宏展开时这个代码片段是一个整体

	> 带 native_ 前缀之类的函数则跟 hal_ 前缀对应，而 Linux 为了支持不同的硬件平台，做了多层封装.
1. 自旋锁 协调多核心 CPU

	由于多核心的 CPU 出现, 控制中断已经失效了, 因为系统中同时有多个代码执行流，为了解决这个问题，需要自旋锁，自旋锁要么一下子获取锁，要么循环等待最终获取锁.

	Linux 有多种[自旋锁](https://elixir.bootlin.com/linux/v6.5/source/include/linux/spinlock.h#L349)，比如原始自旋锁和排队自旋锁. 它们底层原理大同小异，但多了一些优化和改进.


	Linux 中的读写锁本质上是自旋锁的变种, Linux 读写锁的原理本质是基于计数器. 实际操作的时候，我们不是直接使用上面的函数和数据结构，而是应该使用 Linux 提供的标准接口，如 read_lock、write_lock 等.
1. 信号量  CPU 时间管理大师

	无论是原子操作，还是自旋锁，都不适合长时间等待的情况, 这种情况，用自旋锁来同步访问这种资源，是对 CPU 时间的巨大浪费.

	信号量核心是等待、互斥、唤醒（即重新激活等待的代码执行流）.

	```c
	#define SEM_FLG_MUTEX 0
	#define SEM_FLG_MULTI 1
	#define SEM_MUTEX_ONE_LOCK 1
	#define SEM_MULTI_LOCK 0
	//等待链数据结构，用于挂载等待代码执行流（线程）的结构，里面有用于挂载代码执行流的链表和计数器变量，这里我们先不深入研究这个数据结构。
	typedef struct s_KWLST
	{   
	    spinlock_t wl_lock;
	    uint_t   wl_tdnr;
	    list_h_t wl_list;
	}kwlst_t;
	//信号量数据结构
	typedef struct s_SEM
	{
	    spinlock_t sem_lock;//维护sem_t自身数据的自旋锁
	    uint_t sem_flg;//信号量相关的标志
	    sint_t sem_count;//信号量计数值
	    kwlst_t sem_waitlst;//用于挂载等待代码执行流（线程）结构
	}sem_t;

	//获取信号量
	void krlsem_down(sem_t* sem)
	{
	    cpuflg_t cpufg;
	start_step:    
	    krlspinlock_cli(&sem->sem_lock,&cpufg);
	    if(sem->sem_count<1)
	    {//如果信号量值小于1,则让代码执行流（线程）睡眠
	        krlwlst_wait(&sem->sem_waitlst);
	        krlspinunlock_sti(&sem->sem_lock,&cpufg);
	        krlschedul();//切换代码执行流，下次恢复执行时依然从下一行开始执行，所以要goto开始处重新获取信号量
	        goto start_step; 
	    }
	    sem->sem_count--;//信号量值减1,表示成功获取信号量
	    krlspinunlock_sti(&sem->sem_lock,&cpufg);
	    return;
	}
	//释放信号量
	void krlsem_up(sem_t* sem)
	{
	    cpuflg_t cpufg;
	    krlspinlock_cli(&sem->sem_lock,&cpufg);
	    sem->sem_count++;//释放信号量
	    if(sem->sem_count<1)
	    {//如果小于1,则说数据结构出错了，挂起系统
	        krlspinunlock_sti(&sem->sem_lock,&cpufg);
	        hal_sysdie("sem up err");
	    }
	    //唤醒该信号量上所有等待的代码执行流（线程）
	    krlwlst_allup(&sem->sem_waitlst);
	    krlspinunlock_sti(&sem->sem_lock,&cpufg);
	    krlsched_set_schedflgs();
	    return;
	}
	// krlspinlock_cli，krlspinunlock_sti 两个函数，只是对自旋锁函数的一个封装
	````

	信号量在使用之前需要先进行初始化. 这里假定信号量数据结构中的 sem_count 初始化为 1，sem_waitlst 等待链初始化为空, 使用信号量的步骤是:
	1. 获取信号量

		1. 首先对用于保护信号量自身的自旋锁 sem_lock 进行加锁
		2. 对信号值 sem_count 执行“减 1”操作，并检查其值是否小于 0
		3. 上步中检查 sem_count 如果小于 0，就让进程进入等待状态并且将其挂入 sem_waitlst 中，然后调度其它进程运行。否则表示获取信号量成功。当然最后别忘了对自旋锁 sem_lock 进行解锁
	1. 代码执行流开始执行相关操作
	1. 释放信号量

		1. 首先对用于保护信号量自身的自旋锁 sem_lock 进行加锁
		2. 对信号值 sem_count 执行“加 1”操作，并检查其值是否大于 0
		3. 上步中检查 sem_count 值如果大于 0，就执行唤醒 sem_waitlst 中进程的操作，并且需要调度进程时就执行进程调度操作，不管 sem_count 是否大于 0（通常会大于 0）都标记信号量释放成功。当然最后别忘了对自旋锁 sem_lock 进行解锁

	linux信号量定义是[struct semaphore](https://elixir.bootlin.com/linux/v6.5/source/include/linux/semaphore.h#L15). 在 Linux 源代码的 kernel/printk.c 中，使用宏 DEFINE_SEMAPHORE 声明了一个单值信号量 console_sem，也可以说是互斥锁，它用于保护 console 驱动列表 console_drivers 以及同步对整个 console 驱动的访问.

	信号量最大的优势是既可以使申请失败的进程睡眠，还可以作为资源计数器使用.

## 自旋锁
原理:
1. 首先读取锁变量，判断其值是否已经加锁，如果未加锁则执行加锁，然后返回，表示加锁成功
1. 如果已经加锁了，就要返回第一步继续执行后续步骤，因而得名自旋锁

关键: 必须保证读取锁变量和判断并加锁的操作是原子执行的. x86 CPU 提供了一个原子交换指令，xchg，它可以让寄存器里的一个值跟内存空间中的一个值做交换, 这个动作是原子的，不受其它 CPU 干扰.

实现:
```c
//自旋锁结构
typedef struct
{
     volatile u32_t lock;//volatile可以防止编译器优化，保证其它代码始终从内存加载lock变量的值 
} spinlock_t;
//锁初始化函数
static inline void x86_spin_lock_init(spinlock_t * lock)
{
     lock->lock = 0;//锁值初始化为0是未加锁状态
}
//加锁函数
// %0 对应 “r”(1)，表示由编译器自动分配一个通用寄存器, 并填入值 1
// %1 对应"m"(*lock)，表示 lock 是内存地址
static inline void x86_spin_lock(spinlock_t * lock)
{
    __asm__ __volatile__ (
    "1: \n"
    "lock; xchg  %0, %1 \n"//把值为1的寄存器和lock内存中的值进行交换
    "cmpl   $0, %0 \n" //用0和交换回来的值进行比较
    "jnz    2f \n"  //不等于0则跳转后面2标号处运行
    "jmp 3f \n"     //若等于0则跳转后面3标号处返回
    "2:         \n" 
    "cmpl   $0, %1  \n"//用0和lock内存中的值进行比较
    "jne    2b      \n"//若不等于0则跳转到前面2标号处运行继续比较  
    "jmp    1b      \n"//若等于0则跳转到前面1标号处运行，交换并加锁
    "3:  \n"     :
    : "r"(1), "m"(*lock));
}
//解锁函数
static inline void x86_spin_unlock(spinlock_t * lock)
{
    __asm__ __volatile__(
    "movl   $0, %0\n"//解锁把lock内存中的值设为0就行
    :
    : "m"(*lock));
}

// --- 自旋锁函数适应中断环境
// 下面代码实现了关中断下获取自旋锁，以及恢复中断状态释放自旋锁. 自旋锁解决多CPU冲突问题，但是单CPU中断抢占无法解决，所以使用自旋锁时要先关中断
static inline void x86_spin_lock_disable_irq(spinlock_t * lock,cpuflg_t* flags)
{
    __asm__ __volatile__(
    "pushfq                 \n\t"
    "cli                    \n\t"
    "popq %0                \n\t"
    "1:         \n\t"
    "lock; xchg  %1, %2 \n\t"
    "cmpl   $0,%1       \n\t"
    "jnz    2f      \n\t"
    "jmp    3f      \n"  
    "2:         \n\t"
    "cmpl   $0,%2       \n\t" 
    "jne    2b      \n\t"
    "jmp    1b      \n\t"
    "3:     \n"     
     :"=m"(*flags)
    : "r"(1), "m"(*lock));
}
static inline void x86_spin_unlock_enabled_irq(spinlock_t* lock,cpuflg_t* flags)
{
    __asm__ __volatile__(
    "movl   $0, %0\n\t"
    "pushq %1 \n\t"
    "popfq \n\t"
    :
    : "m"(*lock), "m"(*flags));
}
```