# sched_class

## stop_sched_class
stop_sched_class(以下简称stop类)代表优先级最高的进程,它实 际 上 只 定 义 了 pick_next_task_stop 和 put_prev_task_stop ,
enqueue_task_stop和dequeue_task_stop也只是操作了rq的nr_running字段而已,其余的操作要么没有定义,要么为空.

task_tick_stop为空,说明时钟中断对stop类代表的进程没有影响,理论上占用CPU无时间限制。check_preempt_curr_stop也为空,说明没
有进程可以抢占它.

enqueue_task_stop和dequeue_task_stop的策略说明进程不可能通过wakeup类函数变成stop类管理的进程。实际上,一般只能通过调用
sched_set_stop_task函数指定进程的sched_class为stop类,该函数会将进程的task_struct赋值给rq的stop字段,所以对一个rq而言,同一时刻
最多只能有一个stop进程。

pick_next_task_stop判断rq->stop是否存在,如果存在且stop->on_rq等于TASK_ON_RQ_QUEUED,返回stop,否则返回NULL。
总结,只要stop进程存在,且没有睡眠,它就是最高优先级,不受运行时间限制,也不会被抢占。鉴于它如此强大的“背景”,一般进程是不可能享受类似待遇的,少数场景如hotplug才有可能使用它.

## rt_sched_class
rt_sched_class(以下简称rt类)比stop_sched_class丰满许多,它管理实时进程.

sched_rt_entity结构体充当rq和task_struct的媒介.

enqueue_task_rt的主要任务就是要建立rq和task_struct的关系.

enqueue_task_rt中enqueue_rt_entity的任务是将rt_se插入到进程优先级(p->prio)对应的链表中,如果进程不是rq当前执行的进程,且它可以在其他rq上
执行,enqueue_pushable_task将进程(p->pushable_tasks)按照优先级顺序插入rq->rt.pushable_tasks链表中(优先级高的进程在前),并根
据情况更新rq->rt.highest_prio.next字段。

check_preempt_curr_rt判断进程p是否应该抢占目标rq当前正在执行的进程,如果p的优先级更高,则调用resched_curr(rq)请求抢占rq。
如果二者优先级相等(意味着都是实时进程),且当前进程的TIF_NEED_RESCHED标记没有置位, 调用check_preempt_equal_prio。如果rq->curr可以在其他CPU上执行(可迁移),而p不可以(不可迁移),check_preempt_equal_prio也会调用resched_curr(rq),这样p可以在目标CPU上执行,rq->curr换一个CPU继
续执行。

check_preempt_curr_rt可以抢占普通进程,也可以抢占实时进程。

task_woken_rt在满足一系列条件的情况下,调用push_rt_tasks.

注意,此时的rq指的并不是当前进程的rq,而是被唤醒的进程所属的rq,二者可能不同。满足以上条件的情况下,push_rt_tasks循环执
行push_rt_task(rq)直到它返回0。其中一个条件是dl_task(rq->curr)||rt_task(rq->curr),也就是说task_woken_rt只会在rq正在执行实时进程或者最后期限进程的时候才会考虑push_rt_tasks。

push_rt_task选择rq->rt.pushable_tasks链表上的进程,判断它的优先级是否比rq当前执行的进程优先级高。如果是,则调用resched_curr
请求进程切换,返回0;否则调用find_lock_lowest_rq尝试查找另一个合适的rq,记为lowest_rq,找到则切换被唤醒进程的rq至lowest_rq,
然后调用resched_curr(lowest_rq)请求抢占lowest_rq。

所以,task_woken_rt可能抢占进程所属的rq,有可能抢占其他rq。当然了,抢占其他rq并不是没有要求的,find_lock_lowest_rq查找
lowest_rq的最基本要求就是lowest_rq->rt.highest_prio.curr>p->prio,也
就是进程的优先级比rq上的进程的优先级都高。

不难发现,内核将可以在多个CPU上执行的(p->nr_cpus_allowed>1)实时进程划为一类,称作pushable tasks,并给它
们定制了特殊的机制。它们比较灵活,如果抢占不了目标rq,则尝试抢占其他可接受的rq,也可以在一定的条件下让出CPU,选择其他CPU继续执行,这样的策略无疑可以减少实时进程的等待时间.

task_tick_rt在时钟中断时被调用.

第1步,update_curr_rt更新时间.

update_curr_rt引入了实时进程带宽(rt bandwidth)的概念,为了不让普通进程“饿死”,默认情况下1秒(sysctl_sched_rt_period)时间
内,实时进程总共可占用CPU 0.95秒(sysctl_sched_rt_runtime)。

rt_rq->rt_time表示一个时间间隔(1s)内,实时进程累计执行的时间, sched_rt_runtime_exceeded判断该时间是否已经超过0.95s(rt_rq->rt_runtime),如果是,将rt_rq->rt_throttled置1,并返回1;这种情况下update_curr_rt会调用resched_curr申请切换进程。

update_curr_rt只是增加了rt_rq->rt_time,那么1s时间间隔的逻辑是如何实现的呢?答案是有一个hrtimer定时器(rt_bandwidth的
rt_period_timer字段),每隔1s执行一次sched_rt_period_timer函数,由它来配合调整rt_rq->rt_time,并在一定的条件下清除rt_rq-
>rt_throttled,让实时进程可以继续执行。

此处需要插入说明,函数中使用的curr->se是sched_entity类型的,也就是说sched_entity结构体的部分成员不仅用于CFS调度,也可以在其他调度中用来统计时间.

第2步,如果进程的调度策略不是SCHED_RR,函数直接返回。实时进程共有SCHED_FIFO(先到先服务)和
SCHED_RR(时间片轮转)两种调度策略,这说明采用SCHED_FIFO策略的进程除了受带宽影响之外,不受执行时间限制。

一个SCHED_RR策略的实时进程的时间片由task_struct的rt.time_slice字段表示,初始值为sched_rr_timeslice,等于100×HZ/1000,也就是100ms。每一次tick,rt.time_slice就会减1(第3步),如果减1后它仍大于0,说明进程的时间片还没有用完,继续执
行;如果等于0,说明分配给进程的时间片已经用完了。

第4步,将rt.time_slice恢复至sched_rr_timeslice,将进程移动到优先级链表(rt_rq->active.queue[prio])的尾部,这样下次调度优先执行链表中其他进程,然后调用resched_curr申请切换进程

pick_next_task_rt尝试选择一个实时进程.

查找优先级链表中优先级最高的非空链表,获取链表上第一个进程,更新它的执行时间,然后将它返回。同一个链表上的进程,按照先后顺序依次执行,呼应了SCHED_RR策略(用光了时间片的进程插入到链表尾部).

## 完全公平调度类
fair_sched_class管理普通进程的调度,它的调度方式又被称为完全公平调度(Completely Fair Scheduler,CFS)。完全公平是理论上
的,所有的进程同时执行,每个占用等份的一小部分CPU时间。实际的硬件环境中这种假设是不可能满足的,而且进程间由于优先级不
同,本身就不应该公平对待,所以CFS引入了虚拟运行时间(virtual runtime)的概念.

虚拟运行时间是CFS挑选进程的标准,由实际时间结合进程优先级转换而来,由task_struct的se.vruntime字段表示,vruntime小的进程优先执行。

该转换与task_struct的se.load字段有关,类型为load_weight,它有weight和inv_weight两个字段,由set_load_weight函数设置

数组中的每一个元素对应一个进程优先级,数组的下标与p->static_prio-MAX_RT_PRIO相等。普通进程的优先级(static_prio)范围为[100,140),优先级越高,weight越大,inv_weight越小,weight和inv_weight的乘积约等于2^32。

以优先级等于120的进程作为标准,它的实际时间和虚拟时间相等。其他进程需要在此基础上换算,由calc_delta_fair函数完成.

NICE_0_LOAD等于prio_to_weight[120-100],也就是1024,不考虑数据溢出等因素的情况下,__calc_delta可以粗略简化为delta*=NICE_0_LOAD/(se->load.weight)。同样的delta,weight越大的进程,转换后得到的时间越小,也就是说进程优先级越高,它的虚拟时间增加得越慢,而虚拟时间越小的进程越先得到执行。由此可见,CFS确实优待了优先级高的进程.

```c
// https://elixir.bootlin.com/linux/v6.6.29/source/include/linux/sched.h#L548
struct sched_entity {
	/* For load-balancing: */
	struct load_weight		load;
	struct rb_node			run_node; // 将它链接到cfs_rq的红黑树中
	u64				deadline;
	u64				min_deadline;

	struct list_head		group_node;
	unsigned int			on_rq; // 是否在rq上

	u64				exec_start; // 上次更新时的时间, 实际时间
	u64				sum_exec_runtime; // 总共执行的时间, 实际时间
	u64				prev_sum_exec_runtime; // 被调度执行时的总执行时间, 实际时间
	u64				vruntime; // virtual runtime, 虚拟时间
	s64				vlag;
	u64				slice;

	u64				nr_migrations;

#ifdef CONFIG_FAIR_GROUP_SCHED
	int				depth;
	struct sched_entity		*parent;
	/* rq on which this entity is (to be) queued: */
	struct cfs_rq			*cfs_rq;
	/* rq "owned" by this entity/group: */
	struct cfs_rq			*my_q;
	/* cached value of my_q->h_nr_running */
	unsigned long			runnable_weight;
#endif

#ifdef CONFIG_SMP
	/*
	 * Per entity load average tracking.
	 *
	 * Put into separate cache line so it does not
	 * collide with read-mostly values above.
	 */
	struct sched_avg		avg;
#endif
};
```

task_fork_fair在普通进程被创建时调用.

task_fork_fair本身并不复杂,但它涉及了理解CFS的两个重点.

第2步,update_curr更新cfs_rq和它当前执行进程的信息,是CFS中比较重要的函数.

clock_task 是 在 第 1 步 中 update_rq_clock 函 数 更 新 的 , curr->exec_start表示进程上次update_curr的时间,二者相减得到时间间隔
delta_exec 。 delta_exec 是 实 际 时 间 , 可 以 直 接 加 到 curr->sum_exec_runtime 上 , 增 加 进 程 的 累 计 运 行 时 间 ; 但 它 必 须 通 过calc_delta_fair函数转换为虚拟时间,才可以加到curr->vruntime上。

update_min_vruntime 会 更 新 cfs_rq->min_vruntime.

cfs_rq->min_vruntime是单调递增的,可以理解为cfs_rq上可运行状态的进程中,最小的vruntime。可运行状态的进程包含两部分,一
部分是进程时间轴红黑树上的进程,还有一个就是cfs_rq正在执行的进程(它并不在红黑树上)。update_min_vruntime就是比较当前进程和
红黑树最左边的进程的vruntime,然后取小。

进程时间轴红黑树是按照进程的vruntime来排序的,较小者居左。

第4步,place_entity计算se->vruntime

进程被创建的时候,initial等于1,se->vruntime等于cfs_rq->min_vruntime加上sched_vslice(cfs_rq,se),sched_vslice根据进程优先级和当前cfs_rq的情况计算得到进程期望得到的虚拟运行时间。

如果进程因为之前睡眠被唤醒,place_entity也会为它计算vruntime,initial等于0,se->vruntime等于cfs_rq->min_vruntime减去thresh,算是对睡眠进程的奖励。所以,是时候优化下那些while(true)
的程序了,不需要执行的时候适当睡眠,无论对程序本身还是对系统都有好处。

第5步,se->vruntime减去cfs_rq->min_vruntime,等于sched_vslice(cfs_rq,se)。task_fork_fair执行时,新进程还不是“完全体”,不能插入cfs_rq的进程时间轴红黑树中.

enqueue_task_fair调用enqueue_entity将进程插入进程优先级红黑树.

如果并不是之前睡眠被唤醒,执行enqueue_entity第1步,结果是vruntime+=cfs_rq->min_vruntime;如果进程之前睡眠,执行
enqueue_entity第3步,调用place_entity,结果是se->vruntime=cfs_rq->min_vruntime-thresh。

第4步,__enqueue_entity根据se->vruntime将se插入进程优先级红黑树,vrumtime值越小,越靠左。如果新插入的进程的vruntime最小,
更新cfs_rq->tasks_timeline.rb_leftmost使其指向se。

check_preempt_wakeup(fair_sched_class的check_preempt_curr操作,“操作名_class名”的特例)调用update_curr并在以下两种情况下会
调用resched_curr申请切换进程。

第一种,rq->curr->policy==SCHED_IDLE且p->policy!=SCHED_IDLE,采用SCHED_IDLE调度策略,就类似于告诉CFS:“我
并不着急”。

第二种,进程p的vruntime比rq->curr进程的vruntime小的“明显”,是否“明显”,由wakeup_preempt_entity函数决定,满足条件函数返回1,一般是判断差值是否小于sysctl_sched_wakeup_granularity转换为虚拟时间之后的值.

task_tick_fair在时钟中断时被调用,调用entity_tick实现。entity_tick先调用update_curr,如果cfs_rq->nr_running>1,调用check_preempt_tick检查是否应该抢占进程.

第1步,调用sched_slice计算进程被分配的时间ideal_runtime。

sched_slice先调用__sched_period,得到的进程可以执行的时间(记为slice),该时间与进程优先级无关,只与当前的进程数有关。得到了时间后,sched_slice再调用__calc_delta将进程优先级考虑进去,进程最终获得的时间等于slice*se->load.weight/cfs_rq->load.weight。

进程的优先级越高,weight越大,获得的执行时间越多,这是CFS给高优先级进程的又一个优待。总之,优先级越高的进程,执行的机会越多,可执行的时间越久。

需要说明的是,此处se->on_rq等于1,sched_slice在task_fork_fair的时候也会被调用,计算sched_vslice,那里se->on_rq等于0。另外,
sched_slice尽管考虑了进程优先级,但它仍然是实际时间。

第2步,计算进程获得了执行时间后累计执行的时间delta_exec,其中curr->sum_exec_runtime是在update_curr时更新的,curr->prev_sum_exec_runtime是在进程被挑选执行的时候更新的.

如果进程已经用完了分配给它的时间,接下来调用resched_curr申请切换进程。

如果进程并没有用完它的时间,第3步,找到进程优先级红黑树上最左的进程,计算当前进程的vruntime和它的vruntime的差值delta。如果delta小于0,当前进程的vruntime更小,函数返回,进程继续执行。如果delta大于0,说明当前进程已经不是CFS的第一选择,但是它并没有用完自己的时间,这种情况下只要delta不大于ideal_runtime,当前进程继续执行,否则调用resched_curr申请切换进程。这么做可以防止进程频繁切换,比如在前面两个进程的vruntime接近的情况下,如果不这么做会出现二者快速交替执行的情况。

切换的流程由__schedule函数定义,具体的调度类仅决定了回调函数部分.下面分别分析dequeue_task_fair、put_prev_task_fair和
pick_next_task_fair.

1.dequeue_task

dequeue_task_fair调用dequeue_entity实现,它完成如下几步.

(1)调用update_curr
(2)如果进程当前没有占用CPU(se!=cfs_rq->curr),调用__dequeue_entity从进程优先级红黑树将其删除(如果进程正在执行,它不在红黑树中,见pick_next)
(3)更改se->on_rq为0
(4)如果进程接下来不会睡眠(!(flags&DEQUEUE_SLEEP)),更新进程的vruntime,se->vruntime-=cfs_rq->min_vruntime.

一个从不睡眠的普通进程,被enqueue的时候,se->vruntime会加上cfs_rq->min_vruntime,被dequeue的时候,会减去cfs_rq-
>min_vruntime.

一加一减,看似回到原状实则不然,enqueue和dequeue的时候cfs_rq->min_vruntime可能不一样。试想一种情况,一个进程被enqueue
之后始终没有得到执行,它的vruntime没有变化。过了一段时间,cfs_rq->min_vruntime变大了,再被dequeue的时候,它的vruntime就变
小了,等的越久vruntime就越小。不同的cfs_rq的min_vruntime可能是不等的,如果进程从一个CPU迁移到另一个CPU上,采用这种方式可
以保证一定程度的公平,因为它可以衡量进程在前一个CPU上等待的时间的多少。

2.put_prev
put_prev_task_fair调用put_prev_entity实现,它的行为取决于进程的状态.

prev->on_rq等于1还是0,取决于prev进程接下来希望继续执行还是更改状态并睡眠,如果进程是因为执行时间到了被迫让出CPU,prev->on_rq等于1,此时并不会将进程从进程优先级红黑树删除,而是调用__enqueue_entity根据它的vruntime重新插入红黑树.

进程的状态、是否on_rq和是否在红黑树上的关系:
||正在执行|可执行(就绪)|睡眠|
|on_rq|1|1|0|
|ontree|0|1|0|

prev之前执行过一段时间,所以它的vruntime有所增加,重新插入红黑树中时位置会发生变化,其他进程可能会得到执行机会.

3.pick_next
CFS调用pick_next_task_fair选择下一个进程,前面的讨论中已经提到了它的一部分内容,总结如下。
调用pick_next_entity选择进程优先级红黑树上最左的(cfs_rq->tasks_timeline.rb_leftmost)进程,实际的过程要复杂一些,这里跳过了cfs_rq->last、cfs_rq->next和cfs_rq->skip等的讨论。

调用set_next_entity更新cfs_rq当前执行的进程,首先如果进程在红黑树中(se->on_rq),调用__dequeue_entity将进程从红黑树中删除(正在执行的进程不在红黑树中),然后更新部分字段.

## dl_sched_class
最后期限调度类dl_sched_class的优先级比实时调度类高,将它放在完全公平调度类后介绍的原因是它与完全公平调度类有更多共同
点,但比完全公平调度类简单。

dl_rq也使用红黑树管理进程,树的根为它的root.rb_root字段。

```c
// https://elixir.bootlin.com/linux/v6.6.29/source/include/linux/sched.h#L607
struct sched_dl_entity {
	struct rb_node			rb_node; // 将sched_dl_entity链接到红黑树

	/*
	 * Original scheduling parameters. Copied here from sched_attr
	 * during sched_setattr(), they will remain the same until
	 * the next sched_setattr().
	 */
	u64				dl_runtime;	/* Maximum runtime for each instance	*/ // 最大运行时间
	u64				dl_deadline;	/* Relative deadline of each instance	*/ // 相对截止时间
	u64				dl_period;	/* Separation of two instances (period) */
	u64				dl_bw;		/* dl_runtime / dl_period		*/
	u64				dl_density;	/* dl_runtime / dl_deadline		*/

	/*
	 * Actual scheduling parameters. Initialized with the values above,
	 * they are continuously updated during task execution. Note that
	 * the remaining runtime could be < 0 in case we are in overrun.
	 */
	s64				runtime;	/* Remaining runtime for this instance	*/ // 剩余运行时间
	u64				deadline;	/* Absolute deadline for this instance	*/ // 绝对截止时间
	unsigned int			flags;		/* Specifying the scheduler behaviour	*/

	/*
	 * Some bool flags:
	 *
	 * @dl_throttled tells if we exhausted the runtime. If so, the
	 * task has to wait for a replenishment to be performed at the
	 * next firing of dl_timer.
	 *
	 * @dl_yielded tells if task gave up the CPU before consuming
	 * all its available runtime during the last job.
	 *
	 * @dl_non_contending tells if the task is inactive while still
	 * contributing to the active utilization. In other words, it
	 * indicates if the inactive timer has been armed and its handler
	 * has not been executed yet. This flag is useful to avoid race
	 * conditions between the inactive timer handler and the wakeup
	 * code.
	 *
	 * @dl_overrun tells if the task asked to be informed about runtime
	 * overruns.
	 */
	unsigned int			dl_throttled      : 1;
	unsigned int			dl_yielded        : 1;
	unsigned int			dl_non_contending : 1;
	unsigned int			dl_overrun	  : 1;

	/*
	 * Bandwidth enforcement timer. Each -deadline task has its
	 * own bandwidth to be enforced, thus we need one timer per task.
	 */
	struct hrtimer			dl_timer;

	/*
	 * Inactive timer, responsible for decreasing the active utilization
	 * at the "0-lag time". When a -deadline task blocks, it contributes
	 * to GRUB's active utilization until the "0-lag time", hence a
	 * timer is needed to decrease the active utilization at the correct
	 * time.
	 */
	struct hrtimer inactive_timer;

#ifdef CONFIG_RT_MUTEXES
	/*
	 * Priority Inheritance. When a DEADLINE scheduling entity is boosted
	 * pi_se points to the donor, otherwise points to the dl_se it belongs
	 * to (the original one/itself).
	 */
	struct sched_dl_entity *pi_se;
#endif
};
```

关联task_struct和dl_rt的是task_struct的dl字段,类型为sched_dl_entity.

deadline 是 由 dl_deadline 加 上 当 前 时 间 得 到 的 , runtime 随 着task_tick减少。

cfs_rq的红黑树比较的是sched_entity.vruntim,dl_rq的红黑树比较的则是sched_dl_entity.deadline,deadline小的优先执行。

最后期限调度类有以下两点特别之处.

首先,task_fork_dl定义为空,意味着不能创建一个由最后期限调度类管理的进程。那么deadline进程从何而来?一般由已存在的进程调
用sched_setattr变成deadline进程.

其次,我们在创建进程过程中调用的sched_fork函数的第3步,if(dl_prio(p->prio)) return -EAGAIN,意味着最后期限调度类进程不能创
建进程,除非它的task_struct ->sched_reset_on_fork字段置1.

## idle_sched_class
idle_sched_class作为优先级最低的sched_class,它的使命就是没有其他进程需要执行的时候占用CPU,有进程需要执行的时候让出
CPU。它不接受dequeue和enqueue操作,只负责idle进程,由rq->idle指定。pick_next_task_idle返回rq->idle,check_preempt_curr_idle,则直接
调用resched_curr(rq).