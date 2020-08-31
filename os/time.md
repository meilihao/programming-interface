# time
参考:
- [Linux世界里的时间](http://notes.laoqinren.net/blog/2013/05/26/linuxshi-jie-li-de-shi-jian/)

大部分PC机中有两个时钟源，分别是实时时钟（RTC, Real Time Clock）和 操作系统（OS）时钟:
- 实时时钟也叫CMOS时钟或墙上时钟，它靠电池供电，即使系统断电，也可以维持日期和时间
- RTC和OS时钟之间的关系通常也被称作操作系统的时钟运作机制

OS时钟产生于PC主板上的定时/计数芯片，由操作系统控制这个芯片的工作，OS时钟的基本单位就是该芯片的计数周期，开机时操作系统取得RTC中的时间数据来初始化OS时钟，所以它只是在开机有效，由操作系统控制，已被称为软时钟或系统时钟. 操作系统通过OS时钟提供给应用程序和时间有关的服务.

ps: OS时钟其本质是一个计数器，计数器从计数初值开始，每收到一次脉冲信号，计数器减1，当减至0时，就会输出高电平或低电平，然后获取重载值重新从初值开始计数，不断循环，这样就得到一个输出脉冲，这个脉冲作用中断控制器上，产生中断信号，触发时钟中断.

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

时钟滴答的频率（HZ）：即1秒时间内PIT所产生的时钟滴答次数. 类似地，这个值也是由PIT通道0的计数器初值决定的（反过来说，确定了时钟滴答的频率值后也就可以确定8254 PIT通道0的计数器初值）. 由`CONFIG_HZ/HZ`配置的

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

### do_gettimeofday
用户态gettimeofday接口在内核中是通过do_gettimeofday实现的，从调用层次上看，它可以分为timekeeper和clocksource两层.

timekeeper是内核中负责计时功能的核心对象，它通过使用当前系统中最优的clocksource来提供时间服务：
```c
// https://elixir.bootlin.com/linux/v5.9-rc2/source/include/linux/timekeeper_internal.h#L92
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
 * been found good.
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
```