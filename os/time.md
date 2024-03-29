# time
秒的2种定义：
1. 以铯 133 的振荡频率来定义秒

	协调世界时 (Coordinated Universal Time, UTC) 基于国际原子钟 (International Atomic Time, TAI)，并通过不规则地加入闰秒来抵消地球自转变慢的影响.
1. 依据地球自转和公转来定义秒

	格林尼治时间 (Greenwich Mean Time, GMT) 是指位于英国伦敦郊区的皇家格林尼治天文台当地的平太阳时，因为本初子午线被定义为通过那里的经线.

	由于地球每天的自转是不规则的，而且正在缓慢减速，因此天文观测本身具有缺陷. **它后来被修正为 UTC 时间**. 

闰秒 (Leap Second) 是在协调世界时中增加或减少一秒，使它与符合人类习惯的时间贴近所做的调整. Linux 不关心闰秒的问题，它交给那些时间同步服务器处理.

对一个系统而言，需要定义一个 epoch，所有的时间表示是基于这个基准点的.

UNIX time(also POSIX time, UNIX Epoch time) 是从 UTC 1970 年 1 月 1 日 0 时 0 分 0 秒起至现在的秒数.

一个符合 POSIX 标准的系统必须提供系统时钟，**以不小于秒的精度来记录到 epoch 的时间值**.

## HZ
系统定时器能以可编程的频率中断处理器. 频率对应着HZ, HZ越大, 进程调度更准确, 但开销和电源消耗更多, 因为更多的处理器周期消耗在定时器中断上下文.

jiffies记录系统启动以来, 系统定时器触发的次数.

## NTP (Network Time Protocol)
参考:
- [chrony与其他NTP实现的比较](https://chrony.tuxfamily.org/comparison.html)

目前实现有2中:
1. ntp, **淘汰中**
1. [chrony](https://chrony.tuxfamily.org/index.html), **推荐**

时间同步的调整方式有两种：
1. 直接设定当前的时间值，这种会导致时间轴上的时间会向前或向后的跳跃，无法保证时间的连续性和单调性
1. 对本地振荡器的输出进行矫正，对时间轴缓慢地调整，从而保证了时间的连续性和单调性, **推荐**

## os time
参考:
- [Linux世界里的时间](http://notes.laoqinren.net/blog/2013/05/26/linuxshi-jie-li-de-shi-jian/)
- [Linux 下的时钟](https://vvl.me/2019/04/linux-clock/)

pc时钟源:
- 实时时钟RTC, Real Time Clock()也叫CMOS时钟, 它靠电池供电，即使系统断电，也可以维持日期和时间. 兼具时钟源和时钟中断两种功能, 但一般用作时钟源. 它只能产生周期性信号, 频率是2~8192Hz且必须是2的倍数(频率太低, 精度不够). 目前绝大多数计算机上都有它.

	更新RTC时间的hwclock是通过读写`/dev/rtc*`文件来实现的.
- PIT(programmable interval timer) : 可编程间隔计时器, 时钟中断设备, 频率固定为1.193182MHz. 老式系统中它也作为时钟源. pit可满足周期性和单触发两种时钟中断.
- TSC(Time Stamp Counter) : 时间戳计数器 是一个64位的寄存器, 从奔腾开始就在x86中. tsc记录处理器的时钟周期数, 程序可通过RDTSC指令读取它. tsc以其高精度, 低开销的优势成为x86_64机器的首选.
- HPET(high precision event timer) : 高精度定时器, 兼具时钟源和时钟中断两种功能, 它有一组(3~256个不等)定时器可以同时工作. 每个定时器都可以满足周期性和单触发两种时钟中断要求, 用于替代PIT和RTC.
- APIC(advanced programmable interrupt controller) : 高级可编程中断控制器, 作为时钟中断设备. 每个cpu都有一个local APIC. 它的精度较高, 可以满足周期性和单触发两种时钟中断要求.

RTC和OS时钟之间的关系通常也被称作操作系统的时钟运作机制.

> 就cs而言, PIT, TSC, HPET的clocksource->rating默认值分别是110, 300, 250.
> 就evt而言, PIT, APIC, HPET的clock_event_device->rating默认值分别是0, 100, 50.

OS时钟产生于PC主板上的定时/计数芯片，由操作系统控制这个芯片的工作，OS时钟的基本单位就是该芯片的计数周期，开机时操作系统取得RTC中的时间数据来初始化OS时钟，所以它只是在开机有效，由操作系统控制，已被称为软时钟或系统时钟. 操作系统通过OS时钟提供给应用程序和时间有关的服务.

ps: OS时钟其本质是一个计数器，计数器从计数初值开始，每收到一次脉冲信号，计数器减1，当减至0时，就会输出高电平或低电平，然后获取重载值重新从初值开始计数，不断循环，这样就得到一个输出脉冲，这个脉冲作用中断控制器上，产生中断信号，触发时钟中断.

linux时间精度: 
最初的 Linux 只支持 10ms 级别的时间精度，想要取得 us 甚至 ns 级别的精度是不可能的，因此后来发展出了高精度时钟.
虽然高精度时钟出现了，但并没有取代低精度时钟，俩者目前处于共存状态.

> 低精度 timer 是基于 tick 的，实际上，高精度 timer 不是 base on tick 的，它是基于 clock event 和 clock source 的. 假设一个晶振是 13M，那么每个 clock 就是 1/13 个 us，通过对 clock 的计数，当然很容易达到 us 的精度了.

> clock source是 clock-source device (时钟源设备)，它是可以提供一定精度的计时设备，产生 clock event. Clock-event device (时钟事件设备) 指的是系统中可以触发 one-shot (单次) 或周期性中断的设备. Tick device 是 clock-event device 一个 wrapper.

Linux 定义了以下的时钟:
```c
// https://elixir.bootlin.com/linux/v5.10.2/source/include/uapi/linux/time.h#L49
/*
 * The IDs of the various system clocks (for POSIX.1b interval timers):
 */
#define CLOCK_REALTIME			0
#define CLOCK_MONOTONIC			1
#define CLOCK_PROCESS_CPUTIME_ID	2
#define CLOCK_THREAD_CPUTIME_ID		3
#define CLOCK_MONOTONIC_RAW		4
#define CLOCK_REALTIME_COARSE		5
#define CLOCK_MONOTONIC_COARSE		6
#define CLOCK_BOOTTIME			7
#define CLOCK_REALTIME_ALARM		8
#define CLOCK_BOOTTIME_ALARM		9
/*
 * The driver implementing this got removed. The clock ID is kept as a
 * place holder. Do not reuse!
 */
#define CLOCK_SGI_CYCLE			10
#define CLOCK_TAI			11

#define MAX_CLOCKS			16
#define CLOCKS_MASK			(CLOCK_REALTIME | CLOCK_MONOTONIC)
#define CLOCKS_MONO			CLOCK_MONOTONIC
```

- CLOCK_REALTIME 描述真实世界的时钟。Realtime clock 也称为 wall time clock。
- CLOCK_MONOTONIC 单调递增，无法设置，但可通过 NTP 调整，基准点不一定是 Linux epoch
- CLOCK_MONOTONIC_RAW 类似 CLOCK_MONOTONIC，是一个完全基于本地晶振的时钟
- CLOCK_PROCESS_CPUTIME_ID 与 CLOCK_THREAD_CPUTIME_ID 这两个时钟属于 CPU-time clock，专门用于计算进程 / 线程的执行时间
- 后缀_COARSE : 这是低精度时钟
- CLOCK_BOOTTIME 类似 CLOCK_MONOTONIC，不同点在于它包含计算机的睡眠时间
- 后缀_ALARM : 在休眠状态下该时钟仍然递增
- CLOCK_TAI : 原子钟

在 timeline 上以 Linux epoch 为参考点，方便了计算机却不方便人类. 人类更习惯 broken-down time，也就是年月日时分秒的表示形式, 相关的数据结构是 struct tm.

传统的 unix 使用基于秒的时间定义，相关的数据结构是 time_t，它是 POSIX 标准定义的一个以秒计的时间值; 微妙精度表示的时间，相关的数据结构是 struct timeval; 而纳秒，数据结构是 struct timespec.

### 与系统时间相关的函数
1. 秒级别的时间函数 time() / stime()
1. 微秒级别的时间函数 gettimeofday() / settimeofday()
1. 纳秒级别的时间函数 clock_gettime() / clock_settime() / clock_getres
1. 渐进式的时间调整函数 adjtime() / adjtimex() / clock_adjtime()

秒级、微妙级的时间函数都与 timekeeper 模块有关，获取、设置是与其相关的操作.

纳秒级别的时间函数是 POSIX 定义的那套，调用了 clock source 模块的 read 函数来获取更高的精确度, 参数 clk_id 代表的是时钟的种类.

与强制设定系统时间的 set 系列函数不同，linux 提供渐进式的时间调整函数: adjtimex() 是根据 RFC 5905 实现的时间调整函数，比 adjtime() 强大，调整的是 timekeeper, clock_adjtime() 是支持时钟源调整的函数.

### 与进程睡眠相关的服务
- 秒级别的睡眠函数 sleep()
- 微妙级别的睡眠函数 usleep()
- 纳秒级别的睡眠函数 nanosleep()
- 参数更多的纳秒级别睡眠函数 clock_nanosleep()

### 与 timer 相关的服务
- alarm() 函数
- Interval timer 函数 getitimer() / setitimer()
- POSIX 定义的更高级的 timer 函数
        
		- timer 创建与删除函数 timer_create() / timer_delete()
		- timer 管理函数 timer_settime() / timer_gettime() / timer_getoverrun()
- 使用文件描述符通知的 timer 函数 timerfd_create() / timerfd_settime() / timerfd_gettime()

## OS时钟中断
- OS时钟是由可编程定时/计数器产生的输出脉冲触发中断而产生的,而输出脉冲的周期叫做一个`时钟节拍`（Tick，又称滴答）, 中断触发时会进入中断处理函数，使jiffies+1.
- 操作系统的“时间基准” 由设计者决定，Linux的时间基准是utc纪元: 1970年1月1日凌晨0点.
- OS时钟记录的时间就是系统时间. 系统时间以"时钟节拍"为单位
- 时钟中断触发的频率，由内核HZ来确定，系统启动时会按照定义的HZ值对硬件进行设置

## 时间时间
实际时间就是现实中钟表上显示的时间，其实内核中并不常用这个时间，主要是用户空间的程序有时需要获取当前时间，所以内核中也管理着这个时间.

实际时间的获取是在开机后，内核初始化时从RTC读取的.

内核读取这个时间后就将其放入内核中的 xtime 变量中，并且在系统的运行中不断更新这个值.

## os时间
通常，操作系统可以使用三种方法来表示系统的当前时间与日期：
1. 直接用一个64位的计数器来对时钟滴答进行计数
1. 用一个32位计数器来对秒进行计数，同时还用一个32位的辅助计数器对时钟滴答计数，直至累积到一秒为止. 因为2^32超过136年，因此这种方法直至22世纪都可以让系统工作得很好
1. 按时钟滴答进行计数，但是是相对于系统启动以来的滴答次数，而不是相对于某个确定的外部时刻；当读外部后备时钟（如RTC）或用户输入实际时间时，根据当前的滴答次数计算系统当前时间

## 概念
时钟周期（clock cycle）的频率：8253/8254 PIT的本质就是对由晶体振荡器产生的时钟周期进行计数，晶体振荡器在1秒时间内产生的时钟脉冲个数就是时钟周期的频率.

Linux用宏CLOCK_TICK_RATE来表示8254 PIT的输入时钟脉冲的频率,该宏定义在arch/x86/include/asm/timex.h头文件中
```c
// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/linux/timex.h#L163
/* The clock frequency of the i8253/i8254 PIT */
#define PIT_TICK_RATE 1193182ul

// https://elixir.bootlin.com/linux/v5.9-rc2/source/arch/x86/include/asm/timex.h#L9
/* Assume we use the PIT time source for the clock tick */
#define CLOCK_TICK_RATE		PIT_TICK_RATE

// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/linux/jiffies.h#L59
/* LATCH is used in the interval timer and ftape setup. */
#define LATCH ((CLOCK_TICK_RATE + HZ/2) / HZ)	/* For divider */
```

时钟滴答（clock tick）：当PIT通道0的计数器减到0值时，它就在IRQ0上产生一次时钟中断，也即一次时钟滴答. PIT通道0的计数器的初始值决定了要过多少时钟周期才产生一次时钟中断，因此也就决定了一次时钟滴答的时间间隔长度.

时钟滴答的频率（HZ）：即1秒时间内PIT所产生的时钟滴答次数. 类似地，这个值也是由PIT通道0的计数器初值决定的（反过来说，确定了时钟滴答的频率值后也就可以确定8254 PIT通道0的计数器初值）. 由`CONFIG_HZ/HZ`配置的. 现代kernel并不一直采用周期性的滴答, 在硬件运行的情况下, 更倾向于动态(非周期性)时钟中断, HZ作为很多模块衡量时间的基准而被保留下来.

Linux内核用宏HZ来表示时钟滴答的频率，而且在不同的平台上HZ有不同的定义值. 对于ALPHA和IA64平台HZ的值是1024，对于SPARC、MIPS、ARM和i386/amd64等平台HZ的值都是100. 该宏在amd64平台上的定义如下:
```c
// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/uapi/asm-generic/param.h#L6
#ifndef HZ
#define HZ 100
#endif
```

HZ表示每秒钟触发100次时钟中断，即每10ms触发一次. 每次中断jiffies+1，,则每秒jiffies增加了100.

时钟滴答的时间间隔：Linux用全局变量tick来表示时钟滴答的时间间隔长度，该变量定义在kernel/timer.c文件中，如下：


另外，Linux还用宏TICK_SIZE来作为tick变量的引用别名（alias），其定义如下：
```c
// https://elixir.bootlin.com/linux/v5.9-rc2/source/arch/x86/include/asm/timer.h#L9
#define TICK_SIZE (tick_nsec / 1000)
```

宏LATCH：Linux用宏LATCH来定义要写到PIT通道0的计数器中的值，它表示PIT将每隔多少个时钟周期产生一次时钟中断. 显然LATCH应该由下列公式计算：

LATCH＝（1秒之内的时钟周期个数）÷（1秒之内的时钟中断次数）＝（CLOCK_TICK_RATE）÷（HZ） 

类似地，上述公式的结果可能会是个小数，应该对其进行四舍五入. 所以，Linux将LATCH定义为
```c
// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/linux/jiffies.h#L59
/* LATCH is used in the interval timer and ftape setup. */
#define LATCH ((CLOCK_TICK_RATE + HZ/2) / HZ)	/* For divider */
```

类似地，被除数表达式中的`HZ/2`也是用来将LATCH向上圆整成一个整数.

## 时间的内核数据结构
作为一种UNIX类操作系统，Linux内核显然采用本文开始所述的第三种方法来表示系统的当前时间. Linux内核在表示系统当前时间时用到了三个重要的数据结构：
### 全局变量jiffies
全局变量jiffies：这是一个32位的无符号整数，用来表示自内核上一次启动以来的时钟滴答次数. 每发生一次时钟滴答，内核的时钟中断处理函数timer_interrupt()都要将该全局变量jiffies加1.

kernel主要存在jiffy和ktime_t两个时间单位, ktime_t本质上是纳秒, 可使用ktime_get获得当前以ktime_t为单位的时间, 可使用ktime_to_ns或ns_to_ktime等完成转换.

该变量定义如下所示：
```c
// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/linux/jiffies.h#L78
/*
 * The 64-bit value is not atomic - you MUST NOT read it
 * without sampling the sequence number in jiffies_lock.
 * get_jiffies_64() will do this for you as appropriate.
 */
extern u64 __cacheline_aligned_in_smp jiffies_64;
extern unsigned long volatile __cacheline_aligned_in_smp __jiffy_arch_data jiffies;
```

因此系统运行的时间以s为单位计数时就等于`jiffies/HZ`. 
内核启动时将该变量初始化为0，此后，每次时钟中断处理程序都会增加该变量的值，每秒钟触发中断的次数为Hz.

C语言限定符volatile表示jiffies是一个易改变的变量，因此编译器将使对该变量的访问从不通过CPU内部cache来进行.

> jiffies_64：为了解决jiffies溢出问题.

> __cacheline_aligned_in_smp是gcc的扩展函数.


只要在内核代码中看到jiffies,就等于此刻为当前时间.

### 全局变量xtime
全局变量xtime：它是一个timeval结构类型的变量，用来表示当前时间距UNIX时间基准`1970-01-01 00:00:00`的相对秒数值. 结构timeval是Linux内核表示时间的一种格式 （Linux内核对时间的表示有多种格式，每种格式都有不同的时间精度），其时间精度是微秒. 该结构是内核表示时间时最常用的一种格式，它定义在如下所示：
```c
// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/uapi/linux/time.h#L17
struct timeval {
	__kernel_old_time_t	tv_sec;		/* seconds */
	__kernel_suseconds_t	tv_usec;	/* microseconds */
};

// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/uapi/asm-generic/posix_types.h#L89
typedef __kernel_long_t	__kernel_old_time_t;

// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/uapi/asm-generic/posix_types.h#L15
#ifndef __kernel_long_t
typedef long		__kernel_long_t;
typedef unsigned long	__kernel_ulong_t;
#endif

// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/uapi/asm-generic/posix_types.h#L41
#ifndef __kernel_suseconds_t
typedef __kernel_long_t		__kernel_suseconds_t;
#endif
```

其中，成员tv_sec表示当前时间距UNIX时间基准的秒数值(自1970年7月1日即UTC纪元以来经过的时间)，而成员tv_usec则表示一秒之内的微秒值，且1000000>tv_usec>＝0. Linux内核通过timeval结构类型的全局变量xtime来维持当前时间.

xtime是os启动时, kernel从cmos电路中取得的时间.

当前kernel5.9已用[timekeeper](https://elixir.bootlin.com/linux/v5.9-rc2/source/include/linux/timekeeper_internal.h#L92)代替xtime.

### 全局变量sys_tz
全局变量sys_tz：它是一个timezone结构类型的全局变量，表示系统当前的时区信息. 结构类型timezone定义如下所示:
```c
// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/uapi/linux/time.h#L33
struct timezone {
	int	tz_minuteswest;	/* minutes west of Greenwich */
	int	tz_dsttime;	/* type of dst correction */
};

// https://elixir.bootlin.com/linux/v5.9-rc2/source/kernel/time/time.c#L50
/*
 * The timezone where the local system is located.  Used as a default by some
 * programs who obtain this value by using gettimeofday.
 */
struct timezone sys_tz;
```

### tick_period
ktime_t类型的全局变量, 周期性时钟中断的时间间隔. 即使采用动态时钟中断的系统, 它在运行的某些阶段依然可能周期性地触发中断. 该变量是为了计算周期性时钟中断的条件下, 下一次中断的触发时间.

### tick_cpu_device
每cpu变量, 类型是tick_device, 简称td. tick_cpu_device->evtdev指向该cpu的evt, mode字段是其工作模式, 分为TICKDEV_MODE_PERIODIC和TICKDEV_MODE_ONESHOT两种.

### tick_cpu_sched
每cpu变量, 类型是tick_sched, 简称ts, 于系统调度和更新系统时间有关.

## linux time
参考:
- [计时原理－timekeeper与clocksource](https://rootw.github.io/2018/01/%E8%AE%A1%E6%97%B6/)
- [Linux系统时钟变慢的思考和解决方案](http://www.51testing.com/html/01/276101-840957.html)

时间概述中示例程序使用的gettimeofday将返回实时间(real time，或叫墙上时间wall time)，代表现实生活中使用的时间。除了墙上时间，内核也提供了线性时间(monotonic time，它不可调整，随系统运行线性增加，但不包括休眠时间)、启动时间(boot time，它也不可调整，并包括了休眠时间)等多种时间类型，以使应用在不同场景(获取不同类型时间的用户态方法是clock_gettime)，下表汇总了各类时间的要素点:
时间类别 	精度 	可手动调整 	受NTP调整影响 	时间起点 	受闰秒影响 	系统暂停时是否可工作
REALTIME 	ns 	YES 	YES 	Linux epoch 	YES 	NO
MONOTONIC 	ns 	NO 	YES 	Linux epoch 	YES 	NO
MONOTONIC_RAW 	ns 	NO 	NO 	Linux epoch 	YES 	NO
REALTIME_COARSE 	tick 	YES 	YES 	Linux epoch 	YES 	NO
MONOTONIC_COARSE 	tick 	NO 	YES 	Linux epoch 	YES 	NO
BOOTTIME 	ns 	NO 	YES 	machine start 	YES 	NO
REALTIME_ALARM 	ns 	YES 	YES 	Linux epoch 	YES 	YES
BOOTTIME_ALARM 	ns 	NO 	YES 	machine start 	YES 	YES
TAI 	ns 	NO 	NO 	Linux epoch 	NO 	NO

### gettimeofday
用户态gettimeofday接口在内核中是通过vDSO/__NR_gettimeofday实现的，从调用层次上看，它可以分为timekeeper和clocksource两层.

timekeeper是内核中负责计时功能的核心对象，它通过使用当前系统中最优的clocksource来提供时间服务：
```c
// https://elixir.bootlin.com/linux/v5.10.2/source/include/linux/timekeeper_internal.h#L14
/**
 * struct tk_read_base - base structure for timekeeping readout
 * @clock:	Current clocksource used for timekeeping. 当前使用的时钟源
 * @mask:	Bitmask for two's complement subtraction of non 64bit clocks
 * @cycle_last: @clock cycle value at last update
 * @mult:	(NTP adjusted) multiplier for scaled math conversion
 * @shift:	Shift value for scaled math conversion
 * @xtime_nsec: Shifted (fractional) nano seconds offset for readout
 * @base:	ktime_t (nanoseconds) base time for readout
 * @base_real:	Nanoseconds base value for clock REALTIME readout
 *
 * This struct has size 56 byte on 64 bit. Together with a seqcount it
 * occupies a single 64byte cache line.
 *
 * The struct is separate from struct timekeeper as it is also used
 * for a fast NMI safe accessors.
 *
 * @base_real is for the fast NMI safe accessor to allow reading clock
 * realtime from any context.
 */
struct tk_read_base {
	struct clocksource	*clock;
	u64			mask;
	u64			cycle_last;
	u32			mult;
	u32			shift;
	u64			xtime_nsec;
	ktime_t			base;
	u64			base_real;
};

/**
 * struct timekeeper - Structure holding internal timekeeping values.
 * @tkr_mono:		The readout base structure for CLOCK_MONOTONIC
 * @tkr_raw:		The readout base structure for CLOCK_MONOTONIC_RAW
 * @xtime_sec:		Current CLOCK_REALTIME time in seconds
 * @ktime_sec:		Current CLOCK_MONOTONIC time in seconds
 * @wall_to_monotonic:	CLOCK_REALTIME to CLOCK_MONOTONIC offset
 * @offs_real:		Offset clock monotonic -> clock realtime
 * @offs_boot:		Offset clock monotonic -> clock boottime
 * @offs_tai:		Offset clock monotonic -> clock tai
 * @tai_offset:		The current UTC to TAI offset in seconds
 * @clock_was_set_seq:	The sequence number of clock was set events
 * @cs_was_changed_seq:	The sequence number of clocksource change events
 * @next_leap_ktime:	CLOCK_MONOTONIC time value of a pending leap-second
 * @raw_sec:		CLOCK_MONOTONIC_RAW  time in seconds
 * @monotonic_to_boot:	CLOCK_MONOTONIC to CLOCK_BOOTTIME offset
 * @cycle_interval:	Number of clock cycles in one NTP interval
 * @xtime_interval:	Number of clock shifted nano seconds in one NTP
 *			interval.
 * @xtime_remainder:	Shifted nano seconds left over when rounding
 *			@cycle_interval
 * @raw_interval:	Shifted raw nano seconds accumulated per NTP interval.
 * @ntp_error:		Difference between accumulated time and NTP time in ntp
 *			shifted nano seconds.
 * @ntp_error_shift:	Shift conversion between clock shifted nano seconds and
 *			ntp shifted nano seconds.
 * @last_warning:	Warning ratelimiter (DEBUG_TIMEKEEPING)
 * @underflow_seen:	Underflow warning flag (DEBUG_TIMEKEEPING)
 * @overflow_seen:	Overflow warning flag (DEBUG_TIMEKEEPING)
 *
 * Note: For timespec(64) based interfaces wall_to_monotonic is what
 * we need to add to xtime (or xtime corrected for sub jiffie times)
 * to get to monotonic time.  Monotonic is pegged at zero at system
 * boot time, so wall_to_monotonic will be negative, however, we will
 * ALWAYS keep the tv_nsec part positive so we can use the usual
 * normalization.
 *
 * wall_to_monotonic is moved after resume from suspend for the
 * monotonic time not to jump. We need to add total_sleep_time to
 * wall_to_monotonic to get the real boot based time offset.
 *
 * wall_to_monotonic is no longer the boot time, getboottime must be
 * used instead.
 *
 * @monotonic_to_boottime is a timespec64 representation of @offs_boot to
 * accelerate the VDSO update for CLOCK_BOOTTIME.
 */
struct timekeeper {
	struct tk_read_base	tkr_mono;
	struct tk_read_base	tkr_raw;
	u64			xtime_sec;
	unsigned long		ktime_sec;
	struct timespec64	wall_to_monotonic;
	ktime_t			offs_real;
	ktime_t			offs_boot;
	ktime_t			offs_tai;
	s32			tai_offset;
	unsigned int		clock_was_set_seq;
	u8			cs_was_changed_seq;
	ktime_t			next_leap_ktime;
	u64			raw_sec;
	struct timespec64	monotonic_to_boot;

	/* The following members are for timekeeping internal use */
	u64			cycle_interval;
	u64			xtime_interval;
	s64			xtime_remainder;
	u64			raw_interval;
	/* The ntp_tick_length() value currently being used.
	 * This cached copy ensures we consistently apply the tick
	 * length for an entire tick, as ntp_tick_length may change
	 * mid-tick, and we don't want to apply that new value to
	 * the tick in progress.
	 */
	u64			ntp_tick;
	/* Difference between accumulated time and NTP time in ntp
	 * shifted nano seconds. */
	s64			ntp_error;
	u32			ntp_error_shift;
	u32			ntp_err_mult;
	/* Flag used to avoid updating NTP twice with same second */
	u32			skip_second_overflow;
#ifdef CONFIG_DEBUG_TIMEKEEPING
	long			last_warning;
	/*
	 * These simple flag variables are managed
	 * without locks, which is racy, but they are
	 * ok since we don't really care about being
	 * super precise about how many events were
	 * seen, just that a problem was observed.
	 */
	int			underflow_seen;
	int			overflow_seen;
#endif
};
```

内核通过clocksource对象来描述物理计时设备，x86架构下最常见的计时设备是tsc，tsc对应的clocksource:
```c
// https://elixir.bootlin.com/linux/v5.9-rc2/source/arch/x86/kernel/tsc.c#L1147
/*
 * Must mark VALID_FOR_HRES early such that when we unregister tsc_early
 * this one will immediately take over. We will only register if TSC has
 * been found good. tsc的频率是开机时计算得到, 小误差在运行中也会被放大, 因此不能作为系统的watchdog, 而是属于被watchdog监控的时钟源.
 */
static struct clocksource clocksource_tsc = {
	.name			= "tsc",
	.rating			= 300,
	.read			= read_tsc,
	.mask			= CLOCKSOURCE_MASK(64),
	.flags			= CLOCK_SOURCE_IS_CONTINUOUS |
				  CLOCK_SOURCE_VALID_FOR_HRES |
				  CLOCK_SOURCE_MUST_VERIFY,
	.vdso_clock_mode	= VDSO_CLOCKMODE_TSC,
	.enable			= tsc_cs_enable,
	.resume			= tsc_resume,
	.mark_unstable		= tsc_cs_mark_unstable,
	.tick_stable		= tsc_cs_tick_stable,
	.list			= LIST_HEAD_INIT(clocksource_tsc.list),
};


// https://elixir.bootlin.com/linux/v5.10.2/source/include/linux/clocksource.h#L33
/**
 * struct clocksource - hardware abstraction for a free running counter
 *	Provides mostly state-free accessors to the underlying hardware.
 *	This is the structure used for system time.
 *
 * @read:		Returns a cycle value, passes clocksource as argument 读取时钟源的当前时钟数
 * @mask:		Bitmask for two's complement
 *			subtraction of non 64 bit counters
 * @mult:		Cycle to nanosecond multiplier = timekeeper->mult
 * @shift:		Cycle to nanosecond divisor (power of two) = timekeeper->mult
 * @max_idle_ns:	Maximum idle time permitted by the clocksource (nsecs)
 * @maxadj:		Maximum adjustment value to mult (~11%)
 * @archdata:		Optional arch-specific data
 * @max_cycles:		Maximum safe cycle value which won't overflow on
 *			multiplication
 * @name:		Pointer to clocksource name
 * @list:		List head for registration (internal)
 * @rating:		Rating value for selection (higher is better) 等级. kernel只会选择一个时钟源作为watchdog, 也只会选择一个时钟源作为system clocksource(与全局变量tk_core.timekeeper对应), 同等条件下, rating高优先
 *			To avoid rating inflation the following
 *			list should give you a guide as to how
 *			to assign your clocksource a rating
 *			1-99: Unfit for real use
 *				Only available for bootup and testing purposes.
 *			100-199: Base level usability.
 *				Functional for real use, but not desired.
 *			200-299: Good.
 *				A correct and usable clocksource.
 *			300-399: Desired.
 *				A reasonably fast and accurate clocksource.
 *			400-499: Perfect
 *				The ideal clocksource. A must-use where
 *				available.
 * @flags:		Flags describing special properties
 * @enable:		Optional function to enable the clocksource
 * @disable:		Optional function to disable the clocksource
 * @suspend:		Optional suspend function for the clocksource
 * @resume:		Optional resume function for the clocksource
 * @mark_unstable:	Optional function to inform the clocksource driver that
 *			the watchdog marked the clocksource unstable
 * @tick_stable:        Optional function called periodically from the watchdog
 *			code to provide stable syncrhonization points
 * @wd_list:		List head to enqueue into the watchdog list (internal)
 * @cs_last:		Last clocksource value for clocksource watchdog
 * @wd_last:		Last watchdog value corresponding to @cs_last
 * @owner:		Module reference, must be set by clocksource in modules
 *
 * Note: This struct is not used in hotpathes of the timekeeping code
 * because the timekeeper caches the hot path fields in its own data
 * structure, so no cache line alignment is required,
 *
 * The pointer to the clocksource itself is handed to the read
 * callback. If you need extra information there you can wrap struct
 * clocksource into your own struct. Depending on the amount of
 * information you need you should consider to cache line align that
 * structure.
 */
struct clocksource {
	u64			(*read)(struct clocksource *cs);
	u64			mask;
	u32			mult;
	u32			shift;
	u64			max_idle_ns;
	u32			maxadj;
#ifdef CONFIG_ARCH_CLOCKSOURCE_DATA
	struct arch_clocksource_data archdata;
#endif
	u64			max_cycles;
	const char		*name;
	struct list_head	list;
	int			rating;
	enum vdso_clock_mode	vdso_clock_mode;
	unsigned long		flags;

	int			(*enable)(struct clocksource *cs);
	void			(*disable)(struct clocksource *cs);
	void			(*suspend)(struct clocksource *cs);
	void			(*resume)(struct clocksource *cs);
	void			(*mark_unstable)(struct clocksource *cs);
	void			(*tick_stable)(struct clocksource *cs);

	/* private: */
#ifdef CONFIG_CLOCKSOURCE_WATCHDOG
	/* Watchdog related data, used by the framework */
	struct list_head	wd_list;
	u64			cs_last;
	u64			wd_last;
#endif
	struct module		*owner;
};

/*
 * Clock source flags bits:: CLOCK_SOURCE_(IS_CONTINUOUS|MUST_VERIFY|SUSPEND_NONSTOP)由时钟设备的driver设置, 其他一般由kernel设置
 */
#define CLOCK_SOURCE_IS_CONTINUOUS		0x01 // 是连续时钟
#define CLOCK_SOURCE_MUST_VERIFY		0x02 // 需要被监控

#define CLOCK_SOURCE_WATCHDOG			0x10 // 可作为watchdog来监控其他时钟设备
#define CLOCK_SOURCE_VALID_FOR_HRES		0x20 // 高精度模式
#define CLOCK_SOURCE_UNSTABLE			0x40 // 不稳定
#define CLOCK_SOURCE_SUSPEND_NONSTOP		0x80 // 系统suspend时, 该时钟不会停止
#define CLOCK_SOURCE_RESELECT			0x100 // 被选做system clocksource

// https://elixir.bootlin.com/linux/v5.10.2/source/include/linux/clockchips.h#L70
/**
 * struct clock_event_device - clock event device descriptor
 * @event_handler:	Assigned by the framework to be called by the low
 *			level handler of the event source 时钟终端到来时, 处理中断的回调函数
 * @set_next_event:	set next event function using a clocksource delta 设置下一个时钟中断
 * @set_next_ktime:	set next event function using a direct ktime value
 * @next_event:		local storage for the next event in oneshot mode
 * @max_delta_ns:	maximum delta value in ns
 * @min_delta_ns:	minimum delta value in ns
 * @mult:		nanosecond to cycles multiplier = timekeeper->mult
 * @shift:		nanoseconds to cycles divisor (power of two) = timekeeper->shift
 * @state_use_accessors:current state of the device, assigned by the core code
 * @features:		features 设备的特性, CLOCK_EVT_FEAT_XXX. 最常用的设备特性有PERIODIC, ONESHOT, C3STOP 三种, 其中C3STOP表示系统处于C3状态的时候设备停止
 * @retries:		number of forced programming retries
 * @set_state_periodic:	switch state to periodic set_state_xxx, 切换当前时钟中断设备状态的回调函数, 共计4种状态: periodic, 周期性触发中断; oneshot, 单触发.
 * @set_state_oneshot:	switch state to oneshot
 * @set_state_oneshot_stopped: switch state to oneshot_stopped
 * @set_state_shutdown:	switch state to shutdown
 * @tick_resume:	resume clkevt device
 * @broadcast:		function to broadcast events
 * @min_delta_ticks:	minimum delta value in ticks stored for reconfiguration
 * @max_delta_ticks:	maximum delta value in ticks stored for reconfiguration
 * @name:		ptr to clock event name
 * @rating:		variable to rate clock event devices 设备的等级, rating高优先
 * @irq:		IRQ number (only for non CPU local devices)
 * @bound_on:		Bound on CPU
 * @cpumask:		cpumask to indicate for which CPUs this device works 设备支持的cpu集
 * @list:		list head for the management code
 * @owner:		module reference
 */
struct clock_event_device {
	void			(*event_handler)(struct clock_event_device *);
	int			(*set_next_event)(unsigned long evt, struct clock_event_device *);
	int			(*set_next_ktime)(ktime_t expires, struct clock_event_device *);
	ktime_t			next_event;
	u64			max_delta_ns;
	u64			min_delta_ns;
	u32			mult;
	u32			shift;
	enum clock_event_state	state_use_accessors;
	unsigned int		features;
	unsigned long		retries;

	int			(*set_state_periodic)(struct clock_event_device *);
	int			(*set_state_oneshot)(struct clock_event_device *);
	int			(*set_state_oneshot_stopped)(struct clock_event_device *);
	int			(*set_state_shutdown)(struct clock_event_device *);
	int			(*tick_resume)(struct clock_event_device *);

	void			(*broadcast)(const struct cpumask *mask);
	void			(*suspend)(struct clock_event_device *);
	void			(*resume)(struct clock_event_device *);
	unsigned long		min_delta_ticks;
	unsigned long		max_delta_ticks;

	const char		*name;
	int			rating;
	int			irq;
	int			bound_on;
	const struct cpumask	*cpumask;
	struct list_head	list;
	struct module		*owner;
} ____cacheline_aligned;
```

kernel会选择一个不需要被监控的连续时钟源作为watchdog, 负责在程序运行时监控其他时钟源, 如果某一个时钟源的误差超过了可接受的范围, 那么, 将其设置为CLOCK_SOURCE_UNSTABLE, 并将其rate字段设为0.

时钟源设备在初始化后可调用`clocksource_register_hz`等函数完成注册, kernel会在所有的时钟源中选择rating最大的作为保持时间的时钟源, 用户调用gettimeofday获取当前时间.

> 有的设备即可保持时间也可提供定时器功能.

当前使用的时钟源: `cat /sys/devices/system/clocksource/*/current_clocksource`, 支持时钟源有:`cat /sys/devices/system/clocksource/*/available_clocksource`.

## 从内核的角度看时间
内核维护了多种时间: 比如最常用的REALTIME, MONOTONIC和BOOTTIME:

- REALTIME : 又成为WALLTIME/xtime/系统时间, 是根据cs的时钟周期计算得出的时间.

	REALTIME与RTC时间不是同一概念: REALTIME是kernel维护的当前时间; RTC时间是RTC对应芯片维护的硬件时间. 两者的联系是系统启动时, kernel会读取RTC时间作为REALTIME时间, 之后才独立维护.

	settimeofday系统调用会更新REALTIME时间, 但不会更新RTC时间, 需要`hwclock -w`同步到RTC时间.

- MONOTONIC : 不是绝对时间, 是系统启动到当前所经历的**非休眠**时间, 单独递增, 系统启动时由0(timekeeping_init)开始, 不受REALTIME更新的影响.

	MONOTONIC非完全单调, 会受到NTP的影响, 真正不受影响的是RAWMONOTONIC.

- BOOTTIME : 不是绝对时间, 系统启动到现在的时间, 与MONOTONIC不同, 它包括系统休眠时间.


tk的xtime_sec表示REALTIME, wall_to_monotinic是由xtime计算得到MONOTONIC时间时需要加上的时间段.

从5.5开始, kt移除了total_sleep_time, 以offs_real表示MONOTONIC和REALTIME的时间差, 以offs_boot表示MONOTONIC与BOOTTIME的时间差, getboottime/getboottime64可直接使用offs_real-offs_boot得到.

相关函数:
- getboottime : 获取系统启动时的REALTIME
- ktime_get_boottime : 获取BOOTTIME
- ktime_get : 获取MONOTONIC
- ktime_get_ts/ktime_get_ts64 : 获取MONOTONIC

## 周期性和单触发的时钟中断
现代计算机都会包含多种时钟中断设备, 以支持周期性和单触发的时钟中断, 这里以PIT和APICTime组合为例.

tsc的频率是运行时计算得到的, kernel启动时会优先利用PIT得到该值(PIT的频率固定, 利用PIT设定时间段, 除该时间段内tsc的周期数): PIT调用clockevents_register_device函数注册它的evt(即clock_event_device). 注册过程中会执行tick_check_new_device设置该cpu对应的td, 将evt赋值给td的evtdev字段. td的默认状态(mode字段)是TICKDEV_MODE_PERIODIC, 这样pit的evt的event_handler被赋值为tick_handle_periodic. 这是第一阶段, 此时系统处于周期性时钟中断模式, 即tick阶段.

APICTimer进入工作(set_APIC_timer)后, 由于它的rating比pit高, td的evtdev会换成apic的evt, 此时尝试切到nohz模式. kernel切到nohz后进入第二阶段.

> nohz模式(也叫tickless/oneshot/动态时钟模式)是指时钟中断不完全周期性的工作模式, 在该模式下, 时钟中断的间隔可以动态调整. 它主要要求系统当前的cs要有CLOCK_SOURCE_VALID_FOR_HRES标志(timekeeping_valid_for_hres)和evt可以满足单触发模式(tick_is_oneshot_available)这两个条件. nohz分两种: 高精度模式和低精度模式, 选择模式由hrtimer_hres_enabled变量决定, 该值可通过kernel boot args设置. hrtimer_hres_enabled=0即调用tick_nohz_switch_to_nohz切换到nohz低精度模式, 否则调用hrtimer_switch_to_hres切换到高精度模式.

低精度模式的最高频率是hz, evt->event_handler=tick_nohz_handler; 高精度模式的最高频率由时钟中断设备决定, 该特性与hrtimer完美配合, 可满足对时间间隔要求比较高的场景, 此时evt->event_handler=hrtimer_interrupt.

时钟中断发生后, 低精度的tick_nohz_handler会在函数体内完成进程时间计算, 进程调度等操作. 而hrtimer_interrupt只负责去处理到期的hrtimer, 进程时间计算, 进程调度等操作通过[tick_sched](https://elixir.bootlin.com/linux/v5.10.2/source/kernel/time/tick-sched.h#L23)的sched_timer字段表示的hrtimer完成的, tick_sched_timer函数最终负责这些操作.

> hrtimer不等同于timer, 它们虽然有类似的函数, 但hrtimer可以触发对时钟中断设备编程的动作.

## 创建定时器
- `setitimer` : 精度微妙, 对于一个进程而言, 同一时刻, 同一种setitimer定时器只能存在一个, 否则会覆盖原有定时器, 即进程全局.
- `timer_create` : 精度纳秒, 同一时刻, 同一种定时器可创建多个, 由id区分. 内核会维护定时器的链表, 有效的id是使用该定时器的唯一方法.

	time_getoverrun: 获得定时器到期提醒的信号丢失的次数. 因为它上次到期时产生的信号还处于挂起状态, 那么本次信号可能会丢失.


> `sleep()`是让出cpu~重新可执行间的时间, 重新可执行~继续执行间的时间不确定, 涉及进程调度, 比如高优先级插入, 因此sleep有误差.