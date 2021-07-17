# lockup(转载)
参考:
- [NMI watchdog: BUG: soft lockup - CPU#2 stuck for 23s!](https://blog.csdn.net/Rong_Toa/article/details/109606560)
- [Softlockup detector and hardlockup detector (aka nmi_watchdog)](https://www.kernel.org/doc/Documentation/lockup-watchdogs.txt)

lockup是指某段内核代码占着CPU不放. Lockup严重的情况下会导致整个系统失去响应. Lockup有几个特点：
1. 首先只有内核代码才能引起lockup, 因为用户代码是可以被抢占的, 不可能形成lockup(只有一种情况例外, 就是SCHED_FIFO优先级为99的实时进程即使在用户态也可能使[watchdog/x]内核线程抢不到CPU而形成soft lock, 参见[<<Real-Time进程会导致系统Lockup吗？>>](http://linuxperf.com/?p=197))
1. 其次内核代码必须处于禁止内核抢占的状态(preemption disabled), 因为Linux是可抢占式的内核, 只在某些特定的代码区才禁止抢占, 在这些代码区才有可能形成lockup.

Lockup分为两种：soft lockup(内核软死锁) 和 hard lockup, 它们的区别是 hard lockup 发生在CPU屏蔽中断的情况下.
- Soft lockup是指CPU被内核代码占据, 以至于无法执行其它进程.

    检测soft lockup的原理是给每个CPU分配一个定时执行的内核线程[watchdog/x], 如果该线程在设定的期限内没有得到执行的话就意味着发生了soft lockup, [watchdog/x]是SCHED_FIFO实时进程, 优先级为最高的99, 拥有优先运行的特权.
- Hard lockup比soft lockup更加严重, CPU不仅无法执行其它进程, 而且不再响应中断.

    检测hard lockup的原理利用了PMU的NMI perf event, 因为NMI中断是不可屏蔽的, 在CPU不再响应中断的情况下仍然可以得到执行, 它再去检查时钟中断的计数器hrtimer_interrupts是否在保持递增, 如果停滞就意味着时钟中断未得到响应, 也就是发生了hard lockup.

可能导致lockup的原因, 见[这里](https://blog.csdn.net/Rong_Toa/article/details/109606560).

## NMI(非可屏蔽中断)
NMI, 即非可屏蔽中断. 即使在内核代码中设置了屏蔽所有中断的时候, NMI也是不可以被屏蔽的.

中断分为可屏蔽中断和非可屏蔽中断. 其中, 可屏蔽中断包含时钟中断, 外设中断(比如键盘中断, I/O设备中断, 等等), 当我们处理中断处理程序的时候, 在中断处理程序top half时候, 在不允许嵌套的情况下, 需要关闭中断.

但NMI就不一样了, 即便在关闭中断的情况下, 他也能被响应. 触发NMI的条件一般都是ECC error之类的硬件Error. 但NMI也给我们提供了一种机制, 在系统中断被误关闭的情况下, 依然能通过中断处理程序来执行一些紧急操作, 比如kernel panic.

这里涉及到了3个东西：kernel线程, 时钟中断, NMI中断(不可屏蔽中断). 它们具有不一样的优先级, 依次是kernel线程 < 时钟中断 < NMI中断, 其中, kernel 线程是可以被调度的, 同时也是可以被中断随时打断的.

## NMI Watchdog
Linux kernel设计了一个检测lockup的机制, 称为NMI Watchdog, 是利用NMI中断实现的, 用NMI是因为lockup有可能发生在中断被屏蔽的状态下, 这时唯一能把CPU抢下来的方法就是通过NMI, 因为NMI中断是不可屏蔽的. NMI Watchdog 中包含 soft lockup detector 和 hard lockup detector. 2.6之后的内核的实现方法如下.

NMI Watchdog 的触发机制包括两部分：
1. 一个高精度计时器(hrtimer), 对应的中断处理例程是kernel/watchdog.c: watchdog_timer_fn(), 在该例程中：

    - 要递增计数器hrtimer_interrupts, 这个计数器供hard lockup detector用于判断CPU是否响应中断
    - 还要唤醒[watchdog/x]内核线程, 该线程的任务是更新一个时间戳
    - soft lock detector检查时间戳, 如果超过soft lockup threshold一直未更新, 说明[watchdog/x]未得到运行机会, 意味着CPU被霸占, 也就是发生了soft lockup.
1. 基于PMU的NMI perf event, 当PMU的计数器溢出时会触发NMI中断, 对应的中断处理例程是 kernel/watchdog.c: watchdog_overflow_callback(), hard lockup detector就在其中, 它会检查上述hrtimer的中断次数(hrtimer_interrupts)是否在保持递增, 如果停滞则表明hrtimer中断未得到响应, 也就是发生了hard lockup.

> hrtimer的周期是：softlockup_thresh/5.

ps:
- 在2.6内核中：softlockup_thresh的值等于内核参数kernel.watchdog_thresh, 默认60秒
- 到3.10内核中：内核参数kernel.watchdog_thresh名称未变, 但含义变成了hard lockup threshold, 默认10秒; soft lockup threshold则等于(2*kernel.watchdog_thresh), 即默认20秒.


NMI perf event是基于PMU的, 触发周期(hard lockup threshold)在2.6内核里是固定的60秒, 不可手工调整；在3.10内核里可以手工调整, 因为直接对应着内核参数kernel.watchdog_thresh, 默认值10秒.

检测到 lockup 之后怎么办？可以自动panic, 也可输出条信息就算完了, 这是可以通过内核参数来定义的:
- kernel.softlockup_panic: 决定了检测到soft lockup时是否自动panic, 缺省值是0
- kernel.nmi_watchdog: 定义是否开启nmi watchdog、以及hard lockup是否导致panic, 该内核参数的格式是”=[panic,][nopanic,][num]”.

ps: 最新的kernel引入了新的内核参数kernel.hardlockup_panic, 可以通过检查是否存在 /proc/sys/kernel/hardlockup_panic来判断你的内核是否支持.

## 和lockup和watchdog有关的sysctl参数
kernel.nmi_watchdog = 1 开启或关闭nmi watchdog
kernel.softlockup_panic = 0 softlockup 不触发panic
kernel.watchdog = 1 同时开启或关闭soft lockup detector和nmi watchdog
kernel.watchdog_thresh = 10 hard lockup的阈值,soft lockup的阈值是 2*watchdog_thresh

## lockup 的解决
发生lockup通常是特定条件下触发了内核bug, 表现为负载高(因为cpu hang住了). 如果是偶然发生一次,可以忽略,如果频繁发生,则考虑两种方法:
- 看看是什么条件触发了内核bug,解决这个条件
- 向厂商/社区报告这个bug

## example
SoftLockup 示例代码:
```c

#include<linux/kernel.h>
#include<linux/module.h>
#include<linux/kthread.h>
 
struct task_struct *task0;
static spinlock_t spinlock;
int val;
 
int task(void *arg)
{
    printk(KERN_INFO "%s:%d\n",__func__,__LINE__);
    /* To generate panic uncomment following */
    /* panic("softlockup: hung tasks"); */
 
    while(!kthread_should_stop()) {
        printk(KERN_INFO "%s:%d\n",__func__,__LINE__);
        spin_lock(&spinlock);
        /* busy loop in critical section */
        while(1) {
            printk(KERN_INFO "%s:%d\n",__func__,__LINE__);
        }
 
        spin_unlock(&spinlock);
    }
 
    return val;
}
 
static int softlockup_init(void)
{
    printk(KERN_INFO "%s:%d\n",__func__,__LINE__);
 
    val = 1;
    spin_lock_init(&spinlock);
    task0 = kthread_run(&task,(void *)val,"softlockup_thread");
    set_cpus_allowed_ptr(task0, cpumask_of(0));
 
    return 0;
}
 
static void softlockup_exit(void)
{
    printk(KERN_INFO "%s:%d\n",__func__,__LINE__);
    kthread_stop(task0);
}
 
module_init(softlockup_init);
module_exit(softlockup_exit);
```

它通过spinlock()实现关抢占, 使得该CPU上的[watchdog/x]无法被调度. 另外, 通过set_cpus_allowed_ptr()将该线程绑定到特定的CPU上去.

Hardlockup 示例代码:
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kthread.h>
#include <linux/spinlock.h>
 
MODULE_LICENSE("GPL");
 
static int
hog_thread(void *data)
{
    static DEFINE_SPINLOCK(lock);
    unsigned long flags;
 
    printk(KERN_INFO "Hogging a CPU now\n");
    spin_lock_irqsave(&lock, flags);
    while (1);
 
    /* unreached */
    return 0;
}
 
static int __init
hog_init(void)
{
    kthread_run(&hog_thread, NULL, "hog");
    return 0;
}
 
module_init(hog_init);
```

上面代码重要一点就是关中断，这里，通过spin_lock_irqsave(). 中断被关闭, 普通中断无法被相应(包括时钟中断),线程无法被调度, 因此, 在这种情况下, 不仅仅[watchdog/x]线程也无法工作, hrtimer也无法被相应.

## FAQ
### `NMI watchdog: BUG: soft lockup - CPU#49 stuck for 22s! [systemd:1]`
env: ubuntu 16.04.6 arm64 + 飞腾主板

如果CPU太忙导致喂狗（watchdog）不及时，此时系统会打印CPU死锁信息:
```log
kernel:BUG: soft lockup - CPU#0 stuck for 38s! [kworker/0:1:25758]
kernel:BUG: soft lockup - CPU#7 stuck for 36s! [systemd:1]
```

可通过修改kernel.watchdog_thresh(/proc/sys/kernel/watchdog_thresh, 调整值时参数不能大于60, 通常是30), 延长喂狗等待时间. 虽不能彻底解决问题, 只能导致信息延迟打印.

可以打开panic, 将/proc/sys/kernel/panic的默认值0改为1, 便于定位, 找到根本问题.

### 安装系统时发生 soft lockup
在boot args追加`modprobe.blacklist=ast`