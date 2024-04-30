# task
Linux实际上并没有从本质上将进程和线程分开,线程又被称为轻量级进程( Low WeightProcess,LWP).

所谓Linux的进程和线程并没有本质差别,指的是二者的实现,但二者的地位并不等同. 比如提起进程id,线程(轻量级进程)不被当作进程看待,类似的地方还有很多,在这类场景中,提起进程,不包含轻量级进程,而有些场景却是可以包含的. 究其原因,Linux有自身的特性,但为了程序的可移植性,它必须遵循POSIX等一系列标准.

比如getpid系统调用POSIX规定返回进程id,如果一个轻量级进程调用它返回的是线程id,那么在Linux上可以正常运行的程序,移植到其他操作系统就需要很多额外的工作. 为了不引起歧义,在此约定,本书中“进程id”指的是线程组id,而“进程的id”也可以是线程id.

在 Linux 里面，无论是进程，还是线程，到了内核里面，统一都叫任务（Task），由一个统一的结构 [task_struct](https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h#L632) 进行管理.

```c
// https://elixir.bootlin.com/linux/v6.6.29/source/include/linux/sched.h#L743
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
	/*
	 * For reasons of header soup (see current_thread_info()), this
	 * must be the first element of task_struct.
	 */
	struct thread_info		thread_info;
#endif
	unsigned int			__state;

#ifdef CONFIG_PREEMPT_RT
	/* saved state for "spinlock sleepers" */
	unsigned int			saved_state;
#endif

	/*
	 * This begins the randomizable portion of task_struct. Only
	 * scheduling-critical items should be added above here.
	 */
	randomized_struct_fields_start

	void				*stack; // 进程的内核栈
	refcount_t			usage;
	/* Per task flags (PF_*), defined further below: */
	unsigned int			flags;
	unsigned int			ptrace;

#ifdef CONFIG_SMP
	int				on_cpu;
	struct __call_single_node	wake_entry;
	unsigned int			wakee_flips;
	unsigned long			wakee_flip_decay_ts;
	struct task_struct		*last_wakee;

	/*
	 * recent_used_cpu is initially set as the last CPU used by a task
	 * that wakes affine another task. Waker/wakee relationships can
	 * push tasks around a CPU where each wakeup moves to the next one.
	 * Tracking a recently used CPU allows a quick search for a recently
	 * used CPU that may be idle.
	 */
	int				recent_used_cpu;
	int				wake_cpu;
#endif
	int				on_rq;

	int				prio; // 进程的优先级
	int				static_prio; // 同上
	int				normal_prio; // 同上
	unsigned int			rt_priority; // 同上

	struct sched_entity		se; // 进程调度相关
	struct sched_rt_entity		rt; // 同上
	struct sched_dl_entity		dl; // 同上
	const struct sched_class	*sched_class; // 进程所属的sched_class

#ifdef CONFIG_SCHED_CORE
	struct rb_node			core_node;
	unsigned long			core_cookie;
	unsigned int			core_occupation;
#endif

#ifdef CONFIG_CGROUP_SCHED
	struct task_group		*sched_task_group;
#endif

#ifdef CONFIG_UCLAMP_TASK
	/*
	 * Clamp values requested for a scheduling entity.
	 * Must be updated with task_rq_lock() held.
	 */
	struct uclamp_se		uclamp_req[UCLAMP_CNT];
	/*
	 * Effective clamp values used for a scheduling entity.
	 * Must be updated with task_rq_lock() held.
	 */
	struct uclamp_se		uclamp[UCLAMP_CNT];
#endif

	struct sched_statistics         stats;

#ifdef CONFIG_PREEMPT_NOTIFIERS
	/* List of struct preempt_notifier: */
	struct hlist_head		preempt_notifiers;
#endif

#ifdef CONFIG_BLK_DEV_IO_TRACE
	unsigned int			btrace_seq;
#endif

	unsigned int			policy;
	int				nr_cpus_allowed;
	const cpumask_t			*cpus_ptr;
	cpumask_t			*user_cpus_ptr;
	cpumask_t			cpus_mask;
	void				*migration_pending;
#ifdef CONFIG_SMP
	unsigned short			migration_disabled;
#endif
	unsigned short			migration_flags;

#ifdef CONFIG_PREEMPT_RCU
	int				rcu_read_lock_nesting;
	union rcu_special		rcu_read_unlock_special;
	struct list_head		rcu_node_entry;
	struct rcu_node			*rcu_blocked_node;
#endif /* #ifdef CONFIG_PREEMPT_RCU */

#ifdef CONFIG_TASKS_RCU
	unsigned long			rcu_tasks_nvcsw;
	u8				rcu_tasks_holdout;
	u8				rcu_tasks_idx;
	int				rcu_tasks_idle_cpu;
	struct list_head		rcu_tasks_holdout_list;
#endif /* #ifdef CONFIG_TASKS_RCU */

#ifdef CONFIG_TASKS_TRACE_RCU
	int				trc_reader_nesting;
	int				trc_ipi_to_cpu;
	union rcu_special		trc_reader_special;
	struct list_head		trc_holdout_list;
	struct list_head		trc_blkd_node;
	int				trc_blkd_cpu;
#endif /* #ifdef CONFIG_TASKS_TRACE_RCU */

	struct sched_info		sched_info;

	struct list_head		tasks;
#ifdef CONFIG_SMP
	struct plist_node		pushable_tasks;
	struct rb_node			pushable_dl_tasks;
#endif

	struct mm_struct		*mm; // 进程的内存信息
	struct mm_struct		*active_mm; // 同上

	int				exit_state; // 进程退出字段
	int				exit_code; // 同上
	int				exit_signal; // 同上
	/* The signal sent when the parent dies: */
	int				pdeath_signal;
	/* JOBCTL_*, siglock protected: */
	unsigned long			jobctl;

	/* Used for emulating ABI behavior of previous Linux versions: */
	unsigned int			personality;

	/* Scheduler bits, serialized by scheduler locks: */
	unsigned			sched_reset_on_fork:1;
	unsigned			sched_contributes_to_load:1;
	unsigned			sched_migrated:1;

	/* Force alignment to the next boundary: */
	unsigned			:0;

	/* Unserialized, strictly 'current' */

	/*
	 * This field must not be in the scheduler word above due to wakelist
	 * queueing no longer being serialized by p->on_cpu. However:
	 *
	 * p->XXX = X;			ttwu()
	 * schedule()			  if (p->on_rq && ..) // false
	 *   smp_mb__after_spinlock();	  if (smp_load_acquire(&p->on_cpu) && //true
	 *   deactivate_task()		      ttwu_queue_wakelist())
	 *     p->on_rq = 0;			p->sched_remote_wakeup = Y;
	 *
	 * guarantees all stores of 'current' are visible before
	 * ->sched_remote_wakeup gets used, so it can be in this word.
	 */
	unsigned			sched_remote_wakeup:1;

	/* Bit to tell LSMs we're in execve(): */
	unsigned			in_execve:1;
	unsigned			in_iowait:1;
#ifndef TIF_RESTORE_SIGMASK
	unsigned			restore_sigmask:1;
#endif
#ifdef CONFIG_MEMCG
	unsigned			in_user_fault:1;
#endif
#ifdef CONFIG_LRU_GEN
	/* whether the LRU algorithm may apply to this access */
	unsigned			in_lru_fault:1;
#endif
#ifdef CONFIG_COMPAT_BRK
	unsigned			brk_randomized:1;
#endif
#ifdef CONFIG_CGROUPS
	/* disallow userland-initiated cgroup migration */
	unsigned			no_cgroup_migration:1;
	/* task is frozen/stopped (used by the cgroup freezer) */
	unsigned			frozen:1;
#endif
#ifdef CONFIG_BLK_CGROUP
	unsigned			use_memdelay:1;
#endif
#ifdef CONFIG_PSI
	/* Stalled due to lack of memory */
	unsigned			in_memstall:1;
#endif
#ifdef CONFIG_PAGE_OWNER
	/* Used by page_owner=on to detect recursion in page tracking. */
	unsigned			in_page_owner:1;
#endif
#ifdef CONFIG_EVENTFD
	/* Recursion prevention for eventfd_signal() */
	unsigned			in_eventfd:1;
#endif
#ifdef CONFIG_IOMMU_SVA
	unsigned			pasid_activated:1;
#endif
#ifdef	CONFIG_CPU_SUP_INTEL
	unsigned			reported_split_lock:1;
#endif
#ifdef CONFIG_TASK_DELAY_ACCT
	/* delay due to memory thrashing */
	unsigned                        in_thrashing:1;
#endif

	unsigned long			atomic_flags; /* Flags requiring atomic access. */

	struct restart_block		restart_block;

	pid_t				pid;
	pid_t				tgid; // 进程所属的线程组的id

#ifdef CONFIG_STACKPROTECTOR
	/* Canary value for the -fstack-protector GCC feature: */
	unsigned long			stack_canary;
#endif
	/*
	 * Pointers to the (original) parent process, youngest child, younger sibling,
	 * older sibling, respectively.  (p->father can be replaced with
	 * p->real_parent->pid)
	 */

	/* Real parent process: */
	struct task_struct __rcu	*real_parent; // 进程的父进程

	/* Recipient of SIGCHLD, wait4() reports: */
	struct task_struct __rcu	*parent; // 同上

	/*
	 * Children/sibling form the list of natural children:
	 */
	struct list_head		children; // 进程的子进程组成的链表的表头
	struct list_head		sibling; // 将进程链接到兄弟进程组成的链表中, 表头为父进程的children字段
	struct task_struct		*group_leader; // 进程所在的线程组的领导进程

	/*
	 * 'ptraced' is the list of tasks this task is using ptrace() on.
	 *
	 * This includes both natural children and PTRACE_ATTACH targets.
	 * 'ptrace_entry' is this task's link on the p->parent->ptraced list.
	 */
	struct list_head		ptraced;
	struct list_head		ptrace_entry;

	/* PID/PID hash table linkage. */
	struct pid			*thread_pid; // 进程对应的pid
	struct hlist_node		pid_links[PIDTYPE_MAX];
	struct list_head		thread_group; // 将进程链接到线程组中, 链表的头为线程组领导进程的thread_group字段
	struct list_head		thread_node;

	struct completion		*vfork_done;

	/* CLONE_CHILD_SETTID: */
	int __user			*set_child_tid;

	/* CLONE_CHILD_CLEARTID: */
	int __user			*clear_child_tid;

	/* PF_KTHREAD | PF_IO_WORKER */
	void				*worker_private;

	u64				utime;
	u64				stime;
#ifdef CONFIG_ARCH_HAS_SCALED_CPUTIME
	u64				utimescaled;
	u64				stimescaled;
#endif
	u64				gtime;
	struct prev_cputime		prev_cputime;
#ifdef CONFIG_VIRT_CPU_ACCOUNTING_GEN
	struct vtime			vtime;
#endif

#ifdef CONFIG_NO_HZ_FULL
	atomic_t			tick_dep_mask;
#endif
	/* Context switch counts: */
	unsigned long			nvcsw;
	unsigned long			nivcsw;

	/* Monotonic time in nsecs: */
	u64				start_time;

	/* Boot based time in nsecs: */
	u64				start_boottime;

	/* MM fault and swap info: this can arguably be seen as either mm-specific or thread-specific: */
	unsigned long			min_flt;
	unsigned long			maj_flt;

	/* Empty if CONFIG_POSIX_CPUTIMERS=n */
	struct posix_cputimers		posix_cputimers;

#ifdef CONFIG_POSIX_CPU_TIMERS_TASK_WORK
	struct posix_cputimers_work	posix_cputimers_work;
#endif

	/* Process credentials: */

	/* Tracer's credentials at attach: */
	const struct cred __rcu		*ptracer_cred;

	/* Objective and real subjective task credentials (COW): */
	const struct cred __rcu		*real_cred; // credentials

	/* Effective (overridable) subjective task credentials (COW): */
	const struct cred __rcu		*cred; // 同上

#ifdef CONFIG_KEYS
	/* Cached requested key. */
	struct key			*cached_requested_key;
#endif

	/*
	 * executable name, excluding path.
	 *
	 * - normally initialized setup_new_exec()
	 * - access it with [gs]et_task_comm()
	 * - lock it with task_lock()
	 */
	char				comm[TASK_COMM_LEN];

	struct nameidata		*nameidata;

#ifdef CONFIG_SYSVIPC
	struct sysv_sem			sysvsem;
	struct sysv_shm			sysvshm;
#endif
#ifdef CONFIG_DETECT_HUNG_TASK
	unsigned long			last_switch_count;
	unsigned long			last_switch_time;
#endif
	/* Filesystem information: */
	struct fs_struct		*fs; // fs相关信息

	/* Open file information: */
	struct files_struct		*files; // 进程使用的文件信息

#ifdef CONFIG_IO_URING
	struct io_uring_task		*io_uring;
#endif

	/* Namespaces: */
	struct nsproxy			*nsproxy; // 管理进程的多种namespace

	/* Signal handlers: */
	struct signal_struct		*signal; // 信号处理
	struct sighand_struct __rcu		*sighand; // 同上
	sigset_t			blocked; // 同上
	sigset_t			real_blocked; // 同上
	/* Restored if set_restore_sigmask() was used: */
	sigset_t			saved_sigmask; // 同上
	struct sigpending		pending; // 同上
	unsigned long			sas_ss_sp;
	size_t				sas_ss_size;
	unsigned int			sas_ss_flags;

	struct callback_head		*task_works;

#ifdef CONFIG_AUDIT
#ifdef CONFIG_AUDITSYSCALL
	struct audit_context		*audit_context;
#endif
	kuid_t				loginuid;
	unsigned int			sessionid;
#endif
	struct seccomp			seccomp;
	struct syscall_user_dispatch	syscall_dispatch;

	/* Thread group tracking: */
	u64				parent_exec_id;
	u64				self_exec_id;

	/* Protection against (de-)allocation: mm, files, fs, tty, keyrings, mems_allowed, mempolicy: */
	spinlock_t			alloc_lock;

	/* Protection of the PI data structures: */
	raw_spinlock_t			pi_lock;

	struct wake_q_node		wake_q;

#ifdef CONFIG_RT_MUTEXES
	/* PI waiters blocked on a rt_mutex held by this task: */
	struct rb_root_cached		pi_waiters;
	/* Updated under owner's pi_lock and rq lock */
	struct task_struct		*pi_top_task;
	/* Deadlock detection and priority inheritance handling: */
	struct rt_mutex_waiter		*pi_blocked_on;
#endif

#ifdef CONFIG_DEBUG_MUTEXES
	/* Mutex deadlock detection: */
	struct mutex_waiter		*blocked_on;
#endif

#ifdef CONFIG_DEBUG_ATOMIC_SLEEP
	int				non_block_count;
#endif

#ifdef CONFIG_TRACE_IRQFLAGS
	struct irqtrace_events		irqtrace;
	unsigned int			hardirq_threaded;
	u64				hardirq_chain_key;
	int				softirqs_enabled;
	int				softirq_context;
	int				irq_config;
#endif
#ifdef CONFIG_PREEMPT_RT
	int				softirq_disable_cnt;
#endif

#ifdef CONFIG_LOCKDEP
# define MAX_LOCK_DEPTH			48UL
	u64				curr_chain_key;
	int				lockdep_depth;
	unsigned int			lockdep_recursion;
	struct held_lock		held_locks[MAX_LOCK_DEPTH];
#endif

#if defined(CONFIG_UBSAN) && !defined(CONFIG_UBSAN_TRAP)
	unsigned int			in_ubsan;
#endif

	/* Journalling filesystem info: */
	void				*journal_info;

	/* Stacked block device info: */
	struct bio_list			*bio_list;

	/* Stack plugging: */
	struct blk_plug			*plug;

	/* VM state: */
	struct reclaim_state		*reclaim_state;

	struct io_context		*io_context;

#ifdef CONFIG_COMPACTION
	struct capture_control		*capture_control;
#endif
	/* Ptrace state: */
	unsigned long			ptrace_message;
	kernel_siginfo_t		*last_siginfo;

	struct task_io_accounting	ioac;
#ifdef CONFIG_PSI
	/* Pressure stall state */
	unsigned int			psi_flags;
#endif
#ifdef CONFIG_TASK_XACCT
	/* Accumulated RSS usage: */
	u64				acct_rss_mem1;
	/* Accumulated virtual memory usage: */
	u64				acct_vm_mem1;
	/* stime + utime since last update: */
	u64				acct_timexpd;
#endif
#ifdef CONFIG_CPUSETS
	/* Protected by ->alloc_lock: */
	nodemask_t			mems_allowed;
	/* Sequence number to catch updates: */
	seqcount_spinlock_t		mems_allowed_seq;
	int				cpuset_mem_spread_rotor;
	int				cpuset_slab_spread_rotor;
#endif
#ifdef CONFIG_CGROUPS
	/* Control Group info protected by css_set_lock: */
	struct css_set __rcu		*cgroups;
	/* cg_list protected by css_set_lock and tsk->alloc_lock: */
	struct list_head		cg_list;
#endif
#ifdef CONFIG_X86_CPU_RESCTRL
	u32				closid;
	u32				rmid;
#endif
#ifdef CONFIG_FUTEX
	struct robust_list_head __user	*robust_list;
#ifdef CONFIG_COMPAT
	struct compat_robust_list_head __user *compat_robust_list;
#endif
	struct list_head		pi_state_list;
	struct futex_pi_state		*pi_state_cache;
	struct mutex			futex_exit_mutex;
	unsigned int			futex_state;
#endif
#ifdef CONFIG_PERF_EVENTS
	struct perf_event_context	*perf_event_ctxp;
	struct mutex			perf_event_mutex;
	struct list_head		perf_event_list;
#endif
#ifdef CONFIG_DEBUG_PREEMPT
	unsigned long			preempt_disable_ip;
#endif
#ifdef CONFIG_NUMA
	/* Protected by alloc_lock: */
	struct mempolicy		*mempolicy;
	short				il_prev;
	short				pref_node_fork;
#endif
#ifdef CONFIG_NUMA_BALANCING
	int				numa_scan_seq;
	unsigned int			numa_scan_period;
	unsigned int			numa_scan_period_max;
	int				numa_preferred_nid;
	unsigned long			numa_migrate_retry;
	/* Migration stamp: */
	u64				node_stamp;
	u64				last_task_numa_placement;
	u64				last_sum_exec_runtime;
	struct callback_head		numa_work;

	/*
	 * This pointer is only modified for current in syscall and
	 * pagefault context (and for tasks being destroyed), so it can be read
	 * from any of the following contexts:
	 *  - RCU read-side critical section
	 *  - current->numa_group from everywhere
	 *  - task's runqueue locked, task not running
	 */
	struct numa_group __rcu		*numa_group;

	/*
	 * numa_faults is an array split into four regions:
	 * faults_memory, faults_cpu, faults_memory_buffer, faults_cpu_buffer
	 * in this precise order.
	 *
	 * faults_memory: Exponential decaying average of faults on a per-node
	 * basis. Scheduling placement decisions are made based on these
	 * counts. The values remain static for the duration of a PTE scan.
	 * faults_cpu: Track the nodes the process was running on when a NUMA
	 * hinting fault was incurred.
	 * faults_memory_buffer and faults_cpu_buffer: Record faults per node
	 * during the current scan window. When the scan completes, the counts
	 * in faults_memory and faults_cpu decay and these values are copied.
	 */
	unsigned long			*numa_faults;
	unsigned long			total_numa_faults;

	/*
	 * numa_faults_locality tracks if faults recorded during the last
	 * scan window were remote/local or failed to migrate. The task scan
	 * period is adapted based on the locality of the faults with different
	 * weights depending on whether they were shared or private faults
	 */
	unsigned long			numa_faults_locality[3];

	unsigned long			numa_pages_migrated;
#endif /* CONFIG_NUMA_BALANCING */

#ifdef CONFIG_RSEQ
	struct rseq __user *rseq;
	u32 rseq_len;
	u32 rseq_sig;
	/*
	 * RmW on rseq_event_mask must be performed atomically
	 * with respect to preemption.
	 */
	unsigned long rseq_event_mask;
#endif

#ifdef CONFIG_SCHED_MM_CID
	int				mm_cid;		/* Current cid in mm */
	int				last_mm_cid;	/* Most recent cid in mm */
	int				migrate_from_cpu;
	int				mm_cid_active;	/* Whether cid bitmap is active */
	struct callback_head		cid_work;
#endif

	struct tlbflush_unmap_batch	tlb_ubc;

	/* Cache last used pipe for splice(): */
	struct pipe_inode_info		*splice_pipe;

	struct page_frag		task_frag;

#ifdef CONFIG_TASK_DELAY_ACCT
	struct task_delay_info		*delays;
#endif

#ifdef CONFIG_FAULT_INJECTION
	int				make_it_fail;
	unsigned int			fail_nth;
#endif
	/*
	 * When (nr_dirtied >= nr_dirtied_pause), it's time to call
	 * balance_dirty_pages() for a dirty throttling pause:
	 */
	int				nr_dirtied;
	int				nr_dirtied_pause;
	/* Start of a write-and-pause period: */
	unsigned long			dirty_paused_when;

#ifdef CONFIG_LATENCYTOP
	int				latency_record_count;
	struct latency_record		latency_record[LT_SAVECOUNT];
#endif
	/*
	 * Time slack values; these are used to round up poll() and
	 * select() etc timeout values. These are in nanoseconds.
	 */
	u64				timer_slack_ns;
	u64				default_timer_slack_ns;

#if defined(CONFIG_KASAN_GENERIC) || defined(CONFIG_KASAN_SW_TAGS)
	unsigned int			kasan_depth;
#endif

#ifdef CONFIG_KCSAN
	struct kcsan_ctx		kcsan_ctx;
#ifdef CONFIG_TRACE_IRQFLAGS
	struct irqtrace_events		kcsan_save_irqtrace;
#endif
#ifdef CONFIG_KCSAN_WEAK_MEMORY
	int				kcsan_stack_depth;
#endif
#endif

#ifdef CONFIG_KMSAN
	struct kmsan_ctx		kmsan_ctx;
#endif

#if IS_ENABLED(CONFIG_KUNIT)
	struct kunit			*kunit_test;
#endif

#ifdef CONFIG_FUNCTION_GRAPH_TRACER
	/* Index of current stored address in ret_stack: */
	int				curr_ret_stack;
	int				curr_ret_depth;

	/* Stack of return addresses for return function tracing: */
	struct ftrace_ret_stack		*ret_stack;

	/* Timestamp for last schedule: */
	unsigned long long		ftrace_timestamp;

	/*
	 * Number of functions that haven't been traced
	 * because of depth overrun:
	 */
	atomic_t			trace_overrun;

	/* Pause tracing: */
	atomic_t			tracing_graph_pause;
#endif

#ifdef CONFIG_TRACING
	/* Bitmask and counter of trace recursion: */
	unsigned long			trace_recursion;
#endif /* CONFIG_TRACING */

#ifdef CONFIG_KCOV
	/* See kernel/kcov.c for more details. */

	/* Coverage collection mode enabled for this task (0 if disabled): */
	unsigned int			kcov_mode;

	/* Size of the kcov_area: */
	unsigned int			kcov_size;

	/* Buffer for coverage collection: */
	void				*kcov_area;

	/* KCOV descriptor wired with this task or NULL: */
	struct kcov			*kcov;

	/* KCOV common handle for remote coverage collection: */
	u64				kcov_handle;

	/* KCOV sequence number: */
	int				kcov_sequence;

	/* Collect coverage from softirq context: */
	unsigned int			kcov_softirq;
#endif

#ifdef CONFIG_MEMCG
	struct mem_cgroup		*memcg_in_oom;
	gfp_t				memcg_oom_gfp_mask;
	int				memcg_oom_order;

	/* Number of pages to reclaim on returning to userland: */
	unsigned int			memcg_nr_pages_over_high;

	/* Used by memcontrol for targeted memcg charge: */
	struct mem_cgroup		*active_memcg;
#endif

#ifdef CONFIG_BLK_CGROUP
	struct gendisk			*throttle_disk;
#endif

#ifdef CONFIG_UPROBES
	struct uprobe_task		*utask;
#endif
#if defined(CONFIG_BCACHE) || defined(CONFIG_BCACHE_MODULE)
	unsigned int			sequential_io;
	unsigned int			sequential_io_avg;
#endif
	struct kmap_ctrl		kmap_ctrl;
#ifdef CONFIG_DEBUG_ATOMIC_SLEEP
	unsigned long			task_state_change;
# ifdef CONFIG_PREEMPT_RT
	unsigned long			saved_state_change;
# endif
#endif
	struct rcu_head			rcu;
	refcount_t			rcu_users;
	int				pagefault_disabled;
#ifdef CONFIG_MMU
	struct task_struct		*oom_reaper_list;
	struct timer_list		oom_reaper_timer;
#endif
#ifdef CONFIG_VMAP_STACK
	struct vm_struct		*stack_vm_area;
#endif
#ifdef CONFIG_THREAD_INFO_IN_TASK
	/* A live task holds one reference: */
	refcount_t			stack_refcount;
#endif
#ifdef CONFIG_LIVEPATCH
	int patch_state;
#endif
#ifdef CONFIG_SECURITY
	/* Used by LSM modules for access restriction: */
	void				*security;
#endif
#ifdef CONFIG_BPF_SYSCALL
	/* Used by BPF task local storage */
	struct bpf_local_storage __rcu	*bpf_storage;
	/* Used for BPF run context */
	struct bpf_run_ctx		*bpf_ctx;
#endif

#ifdef CONFIG_GCC_PLUGIN_STACKLEAK
	unsigned long			lowest_stack;
	unsigned long			prev_lowest_stack;
#endif

#ifdef CONFIG_X86_MCE
	void __user			*mce_vaddr;
	__u64				mce_kflags;
	u64				mce_addr;
	__u64				mce_ripv : 1,
					mce_whole_page : 1,
					__mce_reserved : 62;
	struct callback_head		mce_kill_me;
	int				mce_count;
#endif

#ifdef CONFIG_KRETPROBES
	struct llist_head               kretprobe_instances;
#endif
#ifdef CONFIG_RETHOOK
	struct llist_head               rethooks;
#endif

#ifdef CONFIG_ARCH_HAS_PARANOID_L1D_FLUSH
	/*
	 * If L1D flush is supported on mm context switch
	 * then we use this callback head to queue kill work
	 * to kill tasks that are not running on SMT disabled
	 * cores
	 */
	struct callback_head		l1d_flush_kill;
#endif

#ifdef CONFIG_RV
	/*
	 * Per-task RV monitor. Nowadays fixed in RV_PER_TASK_MONITORS.
	 * If we find justification for more monitors, we can think
	 * about adding more or developing a dynamic method. So far,
	 * none of these are justified.
	 */
	union rv_task_monitor		rv[RV_PER_TASK_MONITORS];
#endif

#ifdef CONFIG_USER_EVENTS
	struct user_event_mm		*user_event_mm;
#endif

	/*
	 * New fields for task_struct should be added above here, so that
	 * they are included in the randomized portion of task_struct.
	 */
	randomized_struct_fields_end

	/* CPU-specific state of this task: */
	struct thread_struct		thread; // 平台相关信息

	/*
	 * WARNING: on x86, 'thread_struct' contains a variable-sized
	 * structure.  It *MUST* be at the end of 'task_struct'.
	 *
	 * Do not put anything below here!
	 */
};
```

task_struct即进程描述符(process descriptor)也叫进程控制块(PCB))存在任务队列(task list, 双向循环链表)中. 它包含描述该进程内存资源、文件系统资源、文件资源、tty 资源、信号处理等的指针.

Linux下，进程与线程的最大不同是进程拥有独立的内存地址空间，而线程与其他线程共享内存地址空间. 除此之外，进程与线程的实现基本相同，都有task_struct结构，都被分配PID.

内核线程没有独立的地址空间，它们完成特定工作并接受内核的调度，不同于一般用户进程，它们不接收kill命令发送的信号.

## 成员
![task member](/misc/img/process/task.png)

### task id
```c
pid_t pid; // process id
pid_t tgid; // thread group ID, 可判断tast_struct代表的是一个进程还是一个线程
struct task_struct *group_leader;
```
任何一个进程,如果只有主线程,那pid是自己,tgid是自己,group_leader指向的还是自己; 但如果一个进程创建了其他线程, 那么线程有自己的pid,tgid就是进程的主线程的pid,group_leader指向的就是进程的主线程.

```c
// https://elixir.bootlin.com/linux/v6.6.29/source/include/linux/pid.h#L59
struct upid {
	int nr; // id
	struct pid_namespace *ns; // 命名空间
};

struct pid
{
	refcount_t count; // 引用计数
	unsigned int level; // pid的层级
	spinlock_t lock;
	/* lists of tasks that use this pid */
	struct hlist_head tasks[PIDTYPE_MAX]; // 链表数组
	struct hlist_head inodes;
	/* wait queue for pidfd notifications */
	wait_queue_head_t wait_pidfd;
	struct rcu_head rcu;
	struct upid numbers[]; // 每个层级的upid的信息
};
```

> 5.05版的内核中, upid去掉了pid_chain字段, pid_namespace结构体增加了类型为idr的idr字段,由它维护id和pid之间的一对一关系,这样只需要使用id在pid_namespace内查找即可,相对于3.10版本的内核步骤有所简化.

pid作为进程id和task_struct的桥梁. upid结构体存储每个层级的id和命名空间等信息.

进程的id是有空间的, 不同的空间中相同的id也可能表示不同的进程. 通常,进程都在一个level等于0的命名空间(pid_namespace)中.

pid的tasks字段表示四个链表的头,它们分别对应PIDTYPE_PID(进程)、PIDTYPE_TGID(线程组)、
PIDTYPE_PGID(进程组)和PIDTYPE_SID(会话)四种类型(pid_type);task_struct的pid_links字段是hlist_node数组,也分别对
应这四种类型,可以分别将进程链接到四个目标进程的pid对应的链表中.

一个进程拥有一个pid,也拥有一个task_struct,从这个角度来讲,pid和task_struct是一对一的关系。pid->tasks[PIDTYPE_PID]是进
程自身组成的链表的头,该链表上仅有一个元素,就是进程的task_struct->pid_links [PIDTYPE_PID],所以根据pid定位task_struct是
可行的,内核提供了pid_task函数完成该任务.

另外,从task_struct到pid也是通路,task_struct的signal->pids字段是pid*类型的数组,维护着四种类型对应的进程的pid。除此之外,还
可以通过task_struct的thread_pid字段访问进程的pid。内核提供了丰富的函数完成id、pid和task_struct的转换.

#### pid_namespace
亲属关系包括父子和兄弟两种。进程的task_struct有real_parent和parent两个字段表示它的父进程,其中real_parent指向它真正的父进程,这个父进程在进程被创建时就已经确定了,parent多数情况下与real_parent是相同的,但在进程被trace的时候,parent会被临时改变。

另外,进程的父进程并不是一直不变的,如果父进程退出,进程会被指定其他的进程作为父进程。

进程的task_struct会通过sibling字段链接到父进程的链表中,链表的头为父进程的task_struct的children字段,链表上的子进程都是同父的兄弟进程。

pid的tasks字段表示四个链表的头. PIDTYPE_PGID类型的链表由同一个进程组中的进程组成,它们通过task_struct->pid_links[PIDTYPE_PGID] 链 接 到 pid->tasks[PIDTYPE_PGID] 链 表中,pid对应的进程为进程组的领导进程。类似的,PIDTYPE_SID类型的链表由同一个会话中的进程组成,它们通过task_struct->pid_links[PIDTYPE_SID]链接到pid->tasks[PIDTYPE_SID]链表中,pid对应的进程为会话的领导进程。PIDTYPE_TGID类型的链表在3.10版本的内核中是不存在的,虽然名为线程组,但实际上只有线程组的领导进程的pid上存在该链表,而它本身是该链表的唯一元素.

线程并不是由进程组和会话管理的,它们由领导进程管理,所以线程的task_struct并没有链接到进程组和会话的链表中.

### `struct list_head		tasks;`
用于管理进程数据结构的双向链表`[struct list_head](https://elixir.bootlin.com/linux/v5.9-rc6/source/include/linux/types.h#L178) tasks`是一个很关键的进程链表.

struct list_head tasks 把所有的进程用双向链表链起来. 双向链表的第一个节点为 [init_task](https://elixir.bootlin.com/linux/v5.9-rc5/source/init/init_task.c#L64).

### signal处理
```c
// task_struct 中关于信号处理的字段
/* Signal handlers: */
struct signal_struct		*signal;
struct sighand_struct		*sighand; // 哪些信号正在通过信号处理函数进行处理: 处理的结果可以是忽略,可以是结束进程等等
sigset_t			blocked; // 哪些信号被阻塞暂不处理
sigset_t			real_blocked;
/* Restored if set_restore_sigmask() was used: */
sigset_t			saved_sigmask;
struct sigpending		pending; // 哪些信号尚等待处理
unsigned long			sas_ss_sp;
size_t				sas_ss_size;
unsigned int			sas_ss_flags;
```

信号处理函数默认使用用户态的函数栈, 也可以开辟新的栈专门用于信号处理,这就是sas_ss_xxx这三个变量的作用.

task_struct里的struct sigpending pending是本任务的; 而struct signal_struct *signal里的struct sigpending shared_pending是线程组共享的.

### task state
```c
/* -1 unrunnable, 0 runnable, >0 stopped: */
volatile long			state; // [state值的定义](https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h#L75)
int exit_state; // EXIT_ZOMBIE和EXIT_DEAD还可以出现在task_struct的exit_state字段中,意义相同
unsigned int flags; // [flags](https://elixir.bootlin.com/linux/v6.6.29/source/include/linux/sched.h#L1726)
```

state:
```c
// state是通过bitset的方式设置
/* Used in tsk->state: */
#define TASK_RUNNING			0x0000 // 正在执行或正准备执行
#define TASK_INTERRUPTIBLE		0x0001 // 阻塞
#define TASK_UNINTERRUPTIBLE		0x0002 // 与TASK_INTERRUPTIBLE一致, 但不能由信号唤醒
#define __TASK_STOPPED			0x0004 // 停止执行
#define __TASK_TRACED			0x0008 // 被监控
/* Used in tsk->exit_state: */
#define EXIT_DEAD			0x0010 // 进程已退出
#define EXIT_ZOMBIE			0x0020 // 僵尸进程, 执行被终止, 但父进程还没使用wait()等系统调用来获知它的终止信息
#define EXIT_TRACE			(EXIT_ZOMBIE | EXIT_DEAD)
/* Used in tsk->state again: */
#define TASK_PARKED			0x0040 // 主要用于内核线程
#define TASK_DEAD			0x0080 // 进程已死亡, 可以进行资源回收
#define TASK_WAKEKILL			0x0100 // 在收到致命信号时唤醒进程
#define TASK_WAKING			0x0200 // 进程正在被唤醒
#define TASK_NOLOAD			0x0400 // 进程不计算在负载中
#define TASK_NEW			0x0800 // 新创建的进程
#define TASK_STATE_MAX			0x1000

/* Convenience macros for the sake of set_current_state: */
#define TASK_KILLABLE			(TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
#define TASK_STOPPED			(TASK_WAKEKILL | __TASK_STOPPED)
#define TASK_TRACED			(TASK_WAKEKILL | __TASK_TRACED)

#define TASK_IDLE			(TASK_UNINTERRUPTIBLE | TASK_NOLOAD)

/* Convenience macros for the sake of wake_up(): */
#define TASK_NORMAL			(TASK_INTERRUPTIBLE | TASK_UNINTERRUPTIBLE)

/* get_task_state(): */
#define TASK_REPORT			(TASK_RUNNING | TASK_INTERRUPTIBLE | \
					 TASK_UNINTERRUPTIBLE | __TASK_STOPPED | \
					 __TASK_TRACED | EXIT_DEAD | EXIT_ZOMBIE | \
					 TASK_PARKED)
```

![state 流转](/misc/img/task_state_change.png)

![Linux进程状态转换](/misc/img/process/2020-11-12_18-03-09.png)

TASK_RUNNING并不是说进程正在运行, 而是`运行中+在运行队列中等待执行`. 当处于这个状态的进程获得时间片的时候,就是在运行中;如果没有获得时间片,就说明它被其他进程抢占了,在等待再次分配时间片. 

在运行中的进程,一旦要进行一些I/O操作,需要等待I/O完毕,这个时候会释放CPU,进入睡眠状态.

在Linux中,有两种睡眠状态:
- TASK_INTERRUPTIBLE ,可中断的睡眠状态
	进程在睡眠(即被阻塞), 等待某些条件达成. 一旦这些条件达成, kernel就会把进程状态设置为TASK_RUNNING. 处于此状态的进程会被信号唤醒而随时准备投入运行.

    **这是一种浅睡眠的状态,虽然在睡眠,等待I/O完成, 进程被信号唤醒后,不是继续刚才的操作,而是进行信号处理**. 当然也可以根据自己的意愿,来写信号处理函数,例如收到某些信号,就放弃等待这个I/O操作完成,直接退出,也可收到某些信息,继续等待

    > 可以被信号和wake_up()唤醒
- TASK_UNINTERRUPTIBLE ,不可中断的睡眠状态
    这是一种深度睡眠状态,**不可被信号唤醒**,只能死等I/O操作完成. 一旦I/O操作因为特殊原因不能完成,这个时候,谁也叫不醒这个进程了(kill本身也是一个信号,既然这个状态不可被信号唤醒,kill信号也被忽略了).除非重启电脑,没有其他办法. 因此,这其实是一个比较危险的事情,除非极其有把握,不然还是不要设置成该状态.

    > 只能被wake_up()唤醒

于是就有了一种新的进程睡眠状态,TASK_KILLABLE,可以终止的新睡眠状态. 进程处于这种状态中,它的运行原理类似TASK_UNINTERRUPTIBLE,只不过可以响应致命信号.从定义可以看出,TASK_WAKEKILL用于在接收到致命信号时唤醒进程,而TASK_KILLABLE相当于这两位都设置了.

TASK_STOPPED是在进程接收到SIGSTOP、SIGTTIN、SIGTSTP或者SIGTTOU信号之后进入该状态, 同时在调试期间接收到任何信号也会进入该状态. 进程在等待一个恢复信息比如SIGCONT.

TASK_TRACED表示进程被debugger(比如ptrace)等进程监视,进程执行被调试程序所停止. 当一个进程被另外的进程所监视,每一个信号都会让进程进入该状态.

**一旦一个进程要结束,先进入的是EXIT_ZOMBIE状态**,但是这个时候它的父进程还没有使用wait()等系统调用来获知它的终止信息及释放它的所有数据结构,此时进程就成了僵尸进程.

退出的进程要么是EXIT_ZOMBIE, 要么是EXIT_DEAD, 因此它们也可以用于exit_state. EXIT_DEAD是进程的最终状态.

state是和进程的运行、调度有关系, 还有其他的一些状态称为标志, 放在flags字段中,这些字段都被定义称为宏 ,以PF开头:
```c
// https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h#L1466
// PF =  process flag
/*
 * Per process flags
 */
#define PF_IDLE			0x00000002	/* I am an IDLE thread */ // idle进程
#define PF_EXITING		0x00000004	/* Getting shut down */ 表示正在退出. 当有这个flag的时候,在函数find_alive_thread中,找活着的线程时,遇到有这个flag的,就直接跳过
#define PF_EXITPIDONE		0x00000008	/* PI exit done on shut down */ // pi state(priority inheritance state)清理完毕
#define PF_VCPU			0x00000010	/* I'm a virtual CPU */ // 表示进程运行在虚拟CPU上. 在函数account_system_time中,统计进程的系统运行时间,如果有这个flag,就调用account_guest_time,按照客户机的时间进行统计
#define PF_WQ_WORKER		0x00000020	/* I'm a workqueue worker */ // 进程是一个workqueue的worker
#define PF_FORKNOEXEC		0x00000040	/* Forked but didn't exec */ // 表示fork完了,还没有exec. 在_do_fork函数里面调用copy_process,这个时候把flag设置为PF_FORKNOEXEC; 当exec中调用了load_elf_binary的时候又把这个flag去掉
#define PF_MCE_PROCESS		0x00000080      /* Process policy on mce errors */
#define PF_SUPERPRIV		0x00000100	/* Used super-user privileges */ // 进程拥有超级用户权限
#define PF_DUMPCORE		0x00000200	/* Dumped core */
#define PF_SIGNALED		0x00000400	/* Killed by a signal */ // 进程被某个信号杀掉
#define PF_MEMALLOC		0x00000800	/* Allocating memory */
#define PF_NPROC_EXCEEDED	0x00001000	/* set_user() noticed that RLIMIT_NPROC was exceeded */
#define PF_USED_MATH		0x00002000	/* If unset the fpu must be initialized before use */
#define PF_USED_ASYNC		0x00004000	/* Used async_schedule*(), used by module init */
#define PF_NOFREEZE		0x00008000	/* This thread should not be frozen */ // 进程不能被freeze
#define PF_FROZEN		0x00010000	/* Frozen for system suspend */ // 进程处于frozen状态
#define PF_KSWAPD		0x00020000	/* I am kswapd */
#define PF_MEMALLOC_NOFS	0x00040000	/* All allocation requests will inherit GFP_NOFS */
#define PF_MEMALLOC_NOIO	0x00080000	/* All allocation requests will inherit GFP_NOIO */
#define PF_LESS_THROTTLE	0x00100000	/* Throttle me less: I clean memory */
#define PF_KTHREAD		0x00200000	/* I am a kernel thread */ // 进程是一个内核线程
#define PF_RANDOMIZE		0x00400000	/* Randomize virtual address space */
#define PF_SWAPWRITE		0x00800000	/* Allowed to write to swap */
#define PF_MEMSTALL		0x01000000	/* Stalled due to lack of memory */
#define PF_UMH			0x02000000	/* I'm an Usermodehelper process */
#define PF_NO_SETAFFINITY	0x04000000	/* Userland is not allowed to meddle with cpus_allowed */
#define PF_MCE_EARLY		0x08000000      /* Early kill for mce process policy */
#define PF_MEMALLOC_NOCMA	0x10000000	/* All allocation request will have _GFP_MOVABLE cleared */
#define PF_FREEZER_SKIP		0x40000000	/* Freezer should not count it as freezable */
#define PF_SUSPEND_TASK		0x80000000      /* This thread called freeze_processes() and should not be frozen */
```

设置state: `set_current_state`, 除非必要(在SMP系统上), 它会设置内存屏障来强制其他cpu做重新排序, 否则等价于`task->state=state`

### do_exit
定义于`kernel/exit.c`, 工作内容:
1. 将task_struct中的flags设为PF_EXITING
1. call del_timer_sync() 删除内核定时器
1. BSD进程记账功能开启时, call acct_update_integrals()来输出记账信息
1. call exit_mm()释放进程占用的mm_struct, 如果mm_struct没有被共享的话, 就彻底释放它
1. call sem__exit(), 如果进程在排队等待IPC信号, 则取消排队
1. call exit_files()和exit_fs(), 递减文件描述符和文件系统数据的引用计数. 如果某个引用计数将为0, 则释放它.
1. 把存放在task_struct的exit_code中的任务退出码置位exit()提供的退出码(供父进程随时检索), 或着去完成其他由内核机制规定的退出动作.
1. call exit_notify()向父进程发送信号, 将task_struct的exit_state设为EXIT_ZOMBIE.
1. call schedule()切换到新的进程. 因为处于EXIT_ZOMBIE状态的进程不再被调度.

> exit_notify()会call forget_original_parent()-> find_new_reaper()来寻找父进程.

> 处于EXIT_ZOMBIE状态的进程所占用的所有内存就是内核栈, thread_info和task_struct, 此时进程存在的唯一作用是向它的父进程提供信息. 父进程获取信息或通知内核那是无用信息后, 进程持有的所有剩余内存被释放.

wait()->wait4(), 工作内容:
1. 挂起调用它的进程, 直到其中一个子进程退出, 此时得到返回的该子进程的pid, 即它设置的退出码.
1. call release_task()
1. call __exit_signal()->_unhash_process()->detach_pid()从pidhash上删除该进程, 同时也在task list上删除该进程.
1. __exit_signal()释放所有剩余的资源, 并进行最终统计和记录
1. 如果这个进程是线程组的最后一个进程, 并且领头进程已死, 那么release_task()会通知僵死的领头进程的父进程.
1. call put_task_struct()释放进程内核栈和thread_info, 并释放task_struct所占用的slab高速缓存.

孤儿进程处理: 为子进程在当前线程组内找一个线程作为父亲, 如果不行就让init来作为父进程.

### 调度
```c
//是否在运行队列上
int on_rq;
//优先级
int prio;
int static_prio;
int normal_prio;
unsigned int rt_priority;
//调度器类
const struct sched_class *sched_class;
//调度实体
struct sched_entity se;
struct sched_rt_entity rt;
struct sched_dl_entity dl;
//调度策略
unsigned int policy;
//可以使用哪些CPU
int nr_cpus_allowed;
cpumask_t cpus_allowed;
struct sched_info sched_info;
```

### 统计运行信息
```c
u64 utime;//用戶态消耗的CPU时间
u64 stime;//内核态消耗的CPU时间
unsigned long nvcsw;//自愿(voluntary)上下文切换计数
unsigned long nivcsw;//非自愿(involuntary)上下文切换计数
u64 start_time;//进程启动时间,不包含睡眠时间
u64 real_start_time;//进程启动时间,包含睡眠时间
```

### 进程亲缘关系

```c
struct task_struct __rcu *real_parent; /* real parent process */
struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */ // 指向其父进程. 当它终止时,必须向它的父进程发送信号
struct list_head children; /* list of my children */ // 表示链表的头部. 链表中的所有元素都是它的子进程
struct list_head sibling; /* linkage in my parent's children list */ // 用于把当前进程插入到兄弟链表中
```

通常情况下,real_parent和parent是一样的,但是也会有另外的情况存在. 例如,bash创建一个进程,那进程的parent和real_parent就都是bash. 如果在bash上使用GDB来debug一个进程,这个时候GDB是real_parent,bash是这个进程的parent.

![task tree](/misc/img/task_parent.png)

### 进程权限
```c
/* Objective and real subjective task credentials (COW): */
const struct cred __rcu		*real_cred; // 说明谁能操作我这个进程

/* Effective (overridable) subjective task credentials (COW): */
const struct cred __rcu		*cred; // 说明我这个进程能够操作谁
```
Objective是被操作对象, Subjective是操作者.

[cred定义](https://elixir.bootlin.com/linux/latest/source/include/linux/cred.h#L111):
```c
struct cred {
	atomic_t	usage;
#ifdef CONFIG_DEBUG_CREDENTIALS
	atomic_t	subscribers;	/* number of processes subscribed */
	void		*put_addr;
	unsigned	magic;
#define CRED_MAGIC	0x43736564
#define CRED_MAGIC_DEAD	0x44656144
#endif
	kuid_t		uid;		/* real UID of the task */
	kgid_t		gid;		/* real GID of the task */
	kuid_t		suid;		/* saved UID of the task */
	kgid_t		sgid;		/* saved GID of the task */
	kuid_t		euid;		/* effective UID of the task */
	kgid_t		egid;		/* effective GID of the task */
	kuid_t		fsuid;		/* UID for VFS ops */
	kgid_t		fsgid;		/* GID for VFS ops */
	unsigned	securebits;	/* SUID-less security management */
	kernel_cap_t	cap_inheritable; /* caps our children can inherit */
	kernel_cap_t	cap_permitted;	/* caps we're permitted */
	kernel_cap_t	cap_effective;	/* caps we can actually use */
	kernel_cap_t	cap_bset;	/* capability bounding set */
	kernel_cap_t	cap_ambient;	/* Ambient capability set */
#ifdef CONFIG_KEYS
	unsigned char	jit_keyring;	/* default keyring to attach requested
					 * keys to */
	struct key	*session_keyring; /* keyring inherited over fork */
	struct key	*process_keyring; /* keyring private to this process */
	struct key	*thread_keyring; /* keyring private to this thread */
	struct key	*request_key_auth; /* assumed request_key authority */
#endif
#ifdef CONFIG_SECURITY
	void		*security;	/* subjective LSM security */
#endif
	struct user_struct *user;	/* real user ID subscription */
	struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
	struct group_info *group_info;	/* supplementary groups for euid/fsgid */
	struct rcu_head	rcu;		/* RCU deletion hook */
} __randomize_layout;
```

从定义可以看出, 大部分是关于用户和用户所属的用户组信息:
1. uid和gid,注释是real user/group id. 一般情况下,谁启动的进程,就是谁的ID. 但是权限审核的时候,往往不比较这两个,也就是说不大起作用
1. euid和egid,注释是effective user/group id. 当这个进程要操作消息队列、共享内存、信号量等对象的时候,其实就是在比较这个用户和组是否有权限
1. fsuid和fsgid, 即filesystem user/group id. 这个是对文件操作会审核的权限.一般说来,fsuid、euid,和uid是一样的,fsgid、egid,和gid也是一样的. 因为谁启动的进程,就应该审核启动的用户到底有没有这个权限, 但是也有特殊的情况.比如`chmod u+s program`的情况即设置了 set-user-ID 的标识位, uid还是进程创建用户, 但euid和fsuid变成了`program`所有者的uid.

除了以用户和用户组控制权限(粗颗粒度),Linux还有另一个机制就是capabilities(细颗粒度), 它用位图表示权限,定义在[capability.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/capability.h#L113)里.

cap_permitted表示进程能够使用的权限, 但是真正起作用的是cap_effective. cap_permitted中可以包含cap_effective中没有的权限, 一个进程可以在必要的时候放弃自己的某些权限,这样更加安全.

cap_inheritable表示当可执行文件的扩展属性设置了inheritable位时,调用exec执行该程序会继承调用者的inheritable集合,并将其加入到permitted集合. 但在非root用户下执行exec时, 通常不会保留inheritable集合, 但是往往又是非root用户,才想保留权限,所以非常鸡肋.

cap_bset,也就是capability bounding set,是系统中所有进程允许保留的权限. 如果这个集合中不存在某个权限,那么系统中的所有进程都没有这个权限. 即使以超级用户权限执行的进程,也是一样的. 这样有很多好处。例如,系统启动以后,将加载内核模块的权限去掉,那所有进程都不能加载内核模块.

cap_ambient是比较新加入内核的,就是为了解决cap_inheritable鸡肋的状况,也就是,非root用户进程使用exec执行一个程序的时候,如何保留权限的问题. 当执行exec的时候,cap_ambient会被添加到cap_permitted中,同时设置到cap_effective中.

### 内存管理
> [linux内存布局和ASLR下的可分配地址空间](https://zsummer.github.io/2019/11/04/2019-11-04-aslr/)

参考:
- [x86_64 Memory Management](https://github.com/torvalds/linux/blob/master/Documentation/x86/x86_64/mm.rst)

```c
struct mm_struct *mm; // 进程的虚拟地址空间, 含用户态的页表结构.
struct mm_struct *active_mm;
```

`struct mm_struct`是和进程地址空间、内存管理相关的数据结构. 每个进程都有若干个数据段、代码段、堆栈段等，它们都是由这个数据结构统领起来的.

大多数计算机上系统的全部虚拟地址空间分为两个部分: 供用户态程序访问的虚拟地址空间和供内核访问的内核空间. 每当内核执行上下文切换时, 虚拟地址空间的用户层部分都会切换, 以便当前运行的进程匹配, 而内核空间不会放生切换.

对于普通用户进程来说，mm指向虚拟地址空间的用户空间部分，而对于内核线程，mm为NULL.

内核线程和普通的进程间的区别在于**内核线程没有独立的地址空间，mm指针被设置为NULL**(因此top命令上显示为0)；它只在 内核空间运行，从来不切换到用户空间去；并且和普通进程一样，可以被调度，也可以被抢占.

>为什么没有mm指针的进程称为惰性TLB进程?
>
>假如内核线程之后运行的进程与之前是同一个, 在这种情况下, 内核并不需要修改用户空间地址表。地址转换后备缓冲器(即TLB)中的信息仍然有效。只有在内核线程之后, 执行的进程是与此前不同的用户层进程时, 才需要切换(并对应清除TLB数据)

每个进程都有自己独立的虚拟内存空间, 用[mm_struct](https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L380)表示. 在 struct mm_struct 的`unsigned long task_size; /* size of task vm space */` 是区分用户态地址空间和内核态地址空间的分界线.
```c
// --- 32bit
// https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/processor.h#L855
#define TASK_SIZE		PAGE_OFFSET

// https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/page_types.h#L36
#define PAGE_OFFSET		((unsigned long)__PAGE_OFFSET)

// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/page_32_types.h#L18
#define __PAGE_OFFSET_BASE	_AC(CONFIG_PAGE_OFFSET, UL)
#define __PAGE_OFFSET		__PAGE_OFFSET_BASE

// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/Kconfig#L1420
choice
	prompt "Memory split" if EXPERT
	default VMSPLIT_3G
	depends on X86_32
	help
	  Select the desired split between kernel and user memory.

	  If the address range available to the kernel is less than the
	  physical memory installed, the remaining memory will be available
	  as "high memory". Accessing high memory is a little more costly
	  than low memory, as it needs to be mapped into the kernel first.
	  Note that increasing the kernel address space limits the range
	  available to user programs, making the address space there
	  tighter.  Selecting anything other than the default 3G/1G split
	  will also likely make your kernel incompatible with binary-only
	  kernel modules.

	  If you are not absolutely sure what you are doing, leave this
	  option alone!

	config VMSPLIT_3G
		bool "3G/1G user/kernel split"
	config VMSPLIT_3G_OPT
		depends on !X86_PAE
		bool "3G/1G user/kernel split (for full 1G low memory)"
	config VMSPLIT_2G
		bool "2G/2G user/kernel split"
	config VMSPLIT_2G_OPT
		depends on !X86_PAE
		bool "2G/2G user/kernel split (for full 2G low memory)"
	config VMSPLIT_1G
		bool "1G/3G user/kernel split"
endchoice

config PAGE_OFFSET
	hex
	default 0xB0000000 if VMSPLIT_3G_OPT
	default 0x80000000 if VMSPLIT_2G
	default 0x78000000 if VMSPLIT_2G_OPT
	default 0x40000000 if VMSPLIT_1G
	default 0xC0000000 // 3G = 0xC00 * 1M/1024 = 3072 *1M/1024
	depends on X86_32

// --- 64bit
// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/processor.h#L889
#define TASK_SIZE_MAX	((1UL << __VIRTUAL_MASK_SHIFT) - PAGE_SIZE)
...
#define TASK_SIZE		(test_thread_flag(TIF_ADDR32) ? \
					IA32_PAGE_OFFSET : TASK_SIZE_MAX)

// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/page_64_types.h#L55
#ifdef CONFIG_X86_5LEVEL
#define __VIRTUAL_MASK_SHIFT	(pgtable_l5_enabled() ? 56 : 47)
#else
#define __VIRTUAL_MASK_SHIFT	47
#endif

// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/page_64_types.h#L49
#define __START_KERNEL_map	_AC(0xffffffff80000000, UL)
```

对于 32 位系统，最大能够寻址 2^32=4G，其中用户态虚拟地址空间是 3G，内核态是 1G

不开启5级页表[`pgtable_l5_enabled()`](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/pgtable_64_types.h#L36)下, 对于 64 位系统，虚拟地址只使用了 48 位. 就像代码里面写的一样，1 左移了 47 位，就相当于 48 位地址空间一半的位置，0x0000800000000000，然后减去一个页，就是 0x00007FFFFFFFF000(2^47-4096)，共 128T.

`__START_KERNEL_map`是内核空间的起点, 但linux仅使用了48位虚拟地址, 因此, 实际的起点是`0xFFFF8000 00000000`. 内核空间大小同样是 128T.

内核空间和用户空间之间隔着很大的空隙.

64bit下, 这种用户/内核空间的切分方案叫`sign extension`. 比如当len(virtual address)=48时, 一个虚拟地址的低 48 位可以被用于寻址, 但第 48 位到第 63 位全是 0 或 1, 即此时这个虚拟地址空间被分为两部分：
- 内核空间(高位全1)
- 用户空间(高位全0)

> [五级页表PML5可参考这里](https://software.intel.com/sites/default/files/managed/2b/80/5-level_paging_white_paper.pdf)

![64 位系统下进程地址空间布局](/misc/img/space_linux_x64.png)

当执行一个新的进程的时候，会做该的设置`me->mm->task_size = TASK_SIZE`.
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/exec.c#L1509
void setup_new_exec(struct linux_binprm * bprm)
{
	/* Setup things that can depend upon the personality */
	struct task_struct *me = current;

	arch_pick_mmap_layout(me->mm, &bprm->rlim_stack);

	arch_setup_new_exec();

	/* Set the new mm task size. We have to do that late because it may
	 * depend on TIF_32BIT which is only updated in flush_thread() on
	 * some architectures like powerpc
	 */
	me->mm->task_size = TASK_SIZE;
	mutex_unlock(&me->signal->exec_update_mutex);
	mutex_unlock(&me->signal->cred_guard_mutex);
}
EXPORT_SYMBOL(setup_new_exec);
```

#### 用户态布局
```
在 struct mm_struct 里面，有下面这些变量定义了这些区域的统计信息和位置:
```c
// https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L380
unsigned long mmap_base;  /* base of mmap area */
unsigned long total_vm;    /* Total pages mapped */
unsigned long locked_vm;  /* Pages that have PG_mlocked set */
unsigned long pinned_vm;  /* Refcount permanently increased */
unsigned long data_vm;    /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
unsigned long exec_vm;    /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
unsigned long stack_vm;    /* VM_STACK */
unsigned long start_code, end_code, start_data, end_data;
unsigned long start_brk, brk, start_stack;
unsigned long arg_start, arg_end, env_start, env_end;
```

其中，total_vm 是总共映射的页的数目. 这么大的虚拟地址空间，不可能都有真实内存对应, 当内存吃紧的时候，有些页可以换出到硬盘上. 有的页因为比较重要，不能换出. locked_vm 就是被锁定不能换出，pinned_vm 是不能换出，也不能移动.

data_vm 是存放数据的页的数目，exec_vm 是存放可执行文件的页的数目，stack_vm 是栈所占的页的数目.

start_code 和 end_code 表示可执行代码的开始和结束位置，start_data 和 end_data 表示已初始化数据的开始位置和结束位置.

start_brk 是堆的起始位置，brk 是堆当前的结束位置. malloc 申请一小块内存的话，就是通过改变 brk 位置实现的.

start_stack 是栈的起始位置，栈的结束位置在寄存器的栈顶指针中.

arg_start 和 arg_end 是参数列表的位置， env_start 和 env_end 是环境变量的位置, 它们都位于栈中最高地址的地方.

mmap_base 表示虚拟地址空间中用于内存映射的起始地址. 一般情况下，这个空间是从高地址到低地址增长的. malloc 申请一大块内存的时候，就是通过 mmap 在这里映射一块区域到物理内存. 我们加载动态链接库 so 文件，也是在这个区域里面，映射一块区域到 so 文件. 具体布局参考上图`64 位系统下进程地址空间布局`即可.

除了位置信息之外，struct mm_struct 里面还专门有一个结构 vm_area_struct，来描述这些区域的属性:
```c
// https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L382
struct vm_area_struct *mmap;    /* list of VMAs */
struct rb_root mm_rb;
```

这里面一个是单链表，用于将这些区域串起来. 另外还有一个红黑树, 是为了快速查找一个内存区域，并在需要改变的时候，能够快速修改.

进程地址空间里的内存区域块被称为VMA(virtual memory area), 它描述了一个连续空间上的独立区间, 拥有一致的属性. 一个进程地址空间内的vms不允许发生地址重叠.

用`pmap <pid>`可查看进程的地址空间分布情况.

```c
// https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L297
/*
 * This struct describes a virtual memory area. There is one of these
 * per VM-area/task. A VM area is any part of the process virtual memory
 * space that has a special rule for the page-fault handlers (ie a shared
 * library, the executable area etc).
 */
struct vm_area_struct {
	/* The first cache line has the info for VMA tree walking. */

	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;

	struct rb_node vm_rb;

	/*
	 * Largest free memory gap in bytes to the left of this VMA.
	 * Either between this VMA and vma->vm_prev, or between one of the
	 * VMAs below us in the VMA rbtree and its ->vm_prev. This helps
	 * get_unmapped_area find a free area of the right size.
	 */
	unsigned long rb_subtree_gap;

	/* Second cache line starts here. */

	struct mm_struct *vm_mm;	/* The address space we belong to. */

	/*
	 * Access permissions of this VMA.
	 * See vmf_insert_mixed_prot() for discussion.
	 */
	pgprot_t vm_page_prot;
	unsigned long vm_flags;		/* Flags, see mm.h. */

	/*
	 * For areas with an address space and backing store,
	 * linkage into the address_space->i_mmap interval tree.
	 */
	struct {
		struct rb_node rb;
		unsigned long rb_subtree_last;
	} shared;

	/*
	 * A file's MAP_PRIVATE vma can be in both i_mmap tree and anon_vma
	 * list, after a COW of one of the file pages.	A MAP_SHARED vma
	 * can only be in the i_mmap tree.  An anonymous MAP_PRIVATE, stack
	 * or brk vma (with NULL file) can only be in an anon_vma list.
	 */
	struct list_head anon_vma_chain; /* Serialized by mmap_sem &
					  * page_table_lock */
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */

	/* Function pointers to deal with this struct. */
	const struct vm_operations_struct *vm_ops;

	/* Information about our backing store: */
	unsigned long vm_pgoff;		/* Offset (within vm_file) in PAGE_SIZE
					   units */
	struct file * vm_file;		/* File we map to (can be NULL). */
	void * vm_private_data;		/* was vm_pte (shared mem) */

#ifdef CONFIG_SWAP
	atomic_long_t swap_readahead_info;
#endif
#ifndef CONFIG_MMU
	struct vm_region *vm_region;	/* NOMMU mapping region */
#endif
#ifdef CONFIG_NUMA
	struct mempolicy *vm_policy;	/* NUMA policy for the VMA */
#endif
	struct vm_userfaultfd_ctx vm_userfaultfd_ctx;
} __randomize_layout;
```

vm_start 和 vm_end 指定了该区域在用户空间中的起始和结束地址, vm_next 和 vm_prev 将这个区域串在链表上. vm_rb 将这个区域放在红黑树上. vm_ops 里面是对这个内存区域可以做的操作的定义.

虚拟内存区域可以映射到物理内存，也可以映射到文件，映射到物理内存的时候称为匿名映射，anon_vma 中，anoy 就是 anonymous，匿名的意思，映射到文件就需要有 vm_file 指定被映射的文件.

vm_area_struct 是在 load_elf_binary 里面实现和上面的内存区域关联的.
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/binfmt_elf.c#L796

static int load_elf_binary(struct linux_binprm *bprm)
{
......
    setup_new_exec(bprm);

	/* Do this so that we can load the interpreter, if need be.  We will
	   change some of these later */
	retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP),
			executable_stack);
......
		error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt,
				elf_prot, elf_flags, total_size);
......
	retval = set_brk(elf_bss, elf_brk, bss_prot);
......
		elf_entry = load_elf_interp(interp_elf_ex,
					    interpreter,
					    load_bias, interp_elf_phdata,
					    &arch_state);
......
	mm = current->mm;
	mm->end_code = end_code;
	mm->start_code = start_code;
	mm->start_data = start_data;
	mm->end_data = end_data;
	mm->start_stack = bprm->p;
......
}
```
load_elf_binary 会完成以下的事情：
1. 调用 setup_new_exec，设置内存映射区 mmap_base
1. 调用 setup_arg_pages，设置栈的 vm_area_struct，这里面设置了 mm->arg_start 是指向栈底的，current->mm->start_stack 就是栈底
1. elf_map 会将 ELF 文件中的代码部分映射到内存中来
1. set_brk 设置了堆的 vm_area_struct，这里面设置了 current->mm->start_brk = current->mm->brk，也即堆里面还是空的
1. load_elf_interp 将依赖的 so 映射到内存中的内存映射区域

最终就形成下面这个内存映射图:
![](/misc/img/process/7af58012466c7d006511a7e16143314c.jpeg)

映射完毕后，什么情况下会修改呢？
1. 函数的调用，涉及函数栈的改变，主要是改变栈顶指针
2. 通过 malloc 申请一个堆内的空间，当然底层要么执行 brk，要么执行 mmap

brk 系统调用实现的入口是 sys_brk 函数:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/mmap.c#L190
SYSCALL_DEFINE1(brk, unsigned long, brk)
{
	unsigned long retval;
	unsigned long newbrk, oldbrk, origbrk;
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *next;
	unsigned long min_brk;
	bool populate;
	bool downgraded = false;
	LIST_HEAD(uf);

	if (mmap_write_lock_killable(mm))
		return -EINTR;

	origbrk = mm->brk;

#ifdef CONFIG_COMPAT_BRK
	/*
	 * CONFIG_COMPAT_BRK can still be overridden by setting
	 * randomize_va_space to 2, which will still cause mm->start_brk
	 * to be arbitrarily shifted
	 */
	if (current->brk_randomized)
		min_brk = mm->start_brk;
	else
		min_brk = mm->end_data;
#else
	min_brk = mm->start_brk;
#endif
	if (brk < min_brk)
		goto out;

	/*
	 * Check against rlimit here. If this check is done later after the test
	 * of oldbrk with newbrk then it can escape the test and let the data
	 * segment grow beyond its set limit the in case where the limit is
	 * not page aligned -Ram Gupta
	 */
	if (check_data_rlimit(rlimit(RLIMIT_DATA), brk, mm->start_brk,
			      mm->end_data, mm->start_data))
		goto out;

	newbrk = PAGE_ALIGN(brk);
	oldbrk = PAGE_ALIGN(mm->brk);
	if (oldbrk == newbrk) {
		mm->brk = brk;
		goto success;
	}

	/*
	 * Always allow shrinking brk.
	 * __do_munmap() may downgrade mmap_lock to read.
	 */
	if (brk <= mm->brk) {
		int ret;

		/*
		 * mm->brk must to be protected by write mmap_lock so update it
		 * before downgrading mmap_lock. When __do_munmap() fails,
		 * mm->brk will be restored from origbrk.
		 */
		mm->brk = brk;
		ret = __do_munmap(mm, newbrk, oldbrk-newbrk, &uf, true);
		if (ret < 0) {
			mm->brk = origbrk;
			goto out;
		} else if (ret == 1) {
			downgraded = true;
		}
		goto success;
	}

	/* Check against existing mmap mappings. */
	next = find_vma(mm, oldbrk);
	if (next && newbrk + PAGE_SIZE > vm_start_gap(next))
		goto out;

	/* Ok, looks good - let it rip. */
	if (do_brk_flags(oldbrk, newbrk-oldbrk, 0, &uf) < 0)
		goto out;
	mm->brk = brk;

success:
	populate = newbrk > oldbrk && (mm->def_flags & VM_LOCKED) != 0;
	if (downgraded)
		mmap_read_unlock(mm);
	else
		mmap_write_unlock(mm);
	userfaultfd_unmap_complete(mm, &uf);
	if (populate)
		mm_populate(oldbrk, newbrk - oldbrk);
	return brk;

out:
	retval = origbrk;
	mmap_write_unlock(mm);
	return retval;
}
```

堆是从低地址向高地址增长的，sys_brk 函数的参数 brk 是新的堆顶位置，而当前的 mm->brk 是原来堆顶的位置.

首先要做的第一个事情，将原来的堆顶和现在的堆顶，都按照页对齐地址，然后比较大小. 如果两者相同，说明这次增加的堆的量很小，还在一个页里面，不需要另行分配页，直接跳到 set_brk 那里，设置 mm->brk 为新的 brk 就可以了.

如果发现新旧堆顶不在一个页里面，麻烦了，这下要跨页了. 如果发现新堆顶小于旧堆顶，这说明不是新分配内存了，而是释放内存了，释放的还不小，至少释放了一页，于是调用 do_munmap 将这一页的内存映射去掉.

如果堆将要扩大，就要调用 find_vma. 如果打开这个函数，看到的是对红黑树的查找，找到的是原堆顶所在的 vm_area_struct 的下一个 vm_area_struct，看当前的堆顶和下一个 vm_area_struct 之间还能不能分配一个完整的页. 如果不能，没办法只好直接退出返回，内存空间都被占满了. 如果还有空间，就调用 do_brk 进一步分配堆空间，从旧堆顶开始，分配计算出的新旧堆顶之间的页数.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/mmap.c#L190
/*
 *  this is really a simplified "do_mmap".  it only handles
 *  anonymous maps.  eventually we may be able to do some
 *  brk-specific accounting here.
 */
static int do_brk_flags(unsigned long addr, unsigned long len, unsigned long flags, struct list_head *uf)
{
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *vma, *prev;
	struct rb_node **rb_link, *rb_parent;
	pgoff_t pgoff = addr >> PAGE_SHIFT;
	int error;
	unsigned long mapped_addr;

	/* Until we need other flags, refuse anything except VM_EXEC. */
	if ((flags & (~VM_EXEC)) != 0)
		return -EINVAL;
	flags |= VM_DATA_DEFAULT_FLAGS | VM_ACCOUNT | mm->def_flags;

	mapped_addr = get_unmapped_area(NULL, addr, len, 0, MAP_FIXED);
	if (IS_ERR_VALUE(mapped_addr))
		return mapped_addr;

	error = mlock_future_check(mm, mm->def_flags, len);
	if (error)
		return error;

	/*
	 * Clear old maps.  this also does some error checking for us
	 */
	while (find_vma_links(mm, addr, addr + len, &prev, &rb_link,
			      &rb_parent)) {
		if (do_munmap(mm, addr, len, uf))
			return -ENOMEM;
	}

	/* Check against address space limits *after* clearing old maps... */
	if (!may_expand_vm(mm, flags, len >> PAGE_SHIFT))
		return -ENOMEM;

	if (mm->map_count > sysctl_max_map_count)
		return -ENOMEM;

	if (security_vm_enough_memory_mm(mm, len >> PAGE_SHIFT))
		return -ENOMEM;

	/* Can we just expand an old private anonymous mapping? */
	vma = vma_merge(mm, prev, addr, addr + len, flags,
			NULL, NULL, pgoff, NULL, NULL_VM_UFFD_CTX);
	if (vma)
		goto out;

	/*
	 * create a vma struct for an anonymous mapping
	 */
	vma = vm_area_alloc(mm);
	if (!vma) {
		vm_unacct_memory(len >> PAGE_SHIFT);
		return -ENOMEM;
	}

	vma_set_anonymous(vma);
	vma->vm_start = addr;
	vma->vm_end = addr + len;
	vma->vm_pgoff = pgoff;
	vma->vm_flags = flags;
	vma->vm_page_prot = vm_get_page_prot(flags);
	vma_link(mm, vma, prev, rb_link, rb_parent);
out:
	perf_event_mmap(vma);
	mm->total_vm += len >> PAGE_SHIFT;
	mm->data_vm += len >> PAGE_SHIFT;
	if (flags & VM_LOCKED)
		mm->locked_vm += (len >> PAGE_SHIFT);
	vma->vm_flags |= VM_SOFTDIRTY;
	return 0;
}
```

在 do_brk_flags 中，调用 find_vma_links 找到将来的 vm_area_struct 节点在红黑树的位置，找到它的父节点、前序节点. 接下来调用 vma_merge，看这个新节点是否能够和现有树中的节点合并. 如果地址是连着的，能够合并，则不用创建新的 vm_area_struct 了，直接跳到 out，更新统计值即可；如果不能合并，则创建新的 vm_area_struct，既加到 anon_vma_chain 链表中，也加到红黑树中.

#### 内核态的布局
内核态的虚拟空间和某一个进程没有关系，所有进程通过系统调用进入到内核之后，看到的虚拟地址空间都是一样的.

在内核态，32 位和 64 位的布局差别比较大，主要是因为 32 位内核态空间太小了.

##### 32 bit
![32 位的内核态的布局](/misc/img/process/83a6511faf802014fbc2c02afc397a04.jpg)

32 位的内核态虚拟地址空间一共就 1G，占绝大部分的前 896M，我们称为直接映射区.

所谓的直接映射区，就是这一块空间是连续的，和物理内存是非常简单的映射关系，其实就是虚拟内存地址减去 3G，就得到物理内存的位置.

在内核里面，有两个宏:
- __pa(vaddr) 返回与虚拟地址 vaddr 相关的物理地址
- __va(paddr) 则计算出对应于物理地址 paddr 的虚拟地址

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/page.h#L59
#ifndef __pa
#define __pa(x)		__phys_addr((unsigned long)(x))
...
#ifndef __va
#define __va(x)			((void *)((unsigned long)(x)+PAGE_OFFSET))

// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/page_32.h#L13
#define __phys_addr_nodebug(x)	((x) - PAGE_OFFSET)
...
#define __phys_addr(x)		__phys_addr_nodebug(x)
```

这里虚拟地址和物理地址发生了关联关系，在物理内存的开始的 896M 的空间，会被直接映射到 3G 至 3G+896M 的虚拟地址，这样容易给人一种感觉，这些内存访问起来和物理内存差不多，别这样想，在大部分情况下，对于这一段内存的访问，在内核中，还是会使用虚拟地址的，并且将来也会为这一段空间建设页表，对这段地址的访问也会走分页地址的流程，只不过页表里面比较简单，是直接的一一对应而已

这 896M 还需要仔细分解. 在系统启动的时候，物理内存的前 1M 已经被占用了，从 1M 开始加载内核代码段，然后就是内核的全局变量、BSS 等，也是 ELF 里面涵盖的. 这样内核的代码段，全局变量，BSS 也就会被映射到 3G 后的虚拟地址空间里面. 具体的物理内存布局可以查看 /proc/iomem.

在内核运行的过程中，如果碰到系统调用创建进程，会创建 task_struct 这样的实例，内核的进程管理代码会将实例创建在 3G 至 3G+896M 的虚拟空间中，当然也会被放在物理内存里面的前 896M 里面，相应的页表也会被创建.

在内核运行的过程中，会涉及内核栈的分配，内核的进程管理的代码会将内核栈创建在 3G 至 3G+896M 的虚拟空间中，当然也就会被放在物理内存里面的前 896M 里面，相应的页表也会被创建.

896M 这个值在内核中被定义为 high_memory，在此之上常称为“高端内存”. 这是个很笼统的说法，到底是虚拟内存的 3G+896M 以上的是高端内存，还是物理内存 896M 以上的是高端内存呢？

这里仍然需要辨析一下，高端内存是物理内存的概念. 它仅仅是内核中的内存管理模块看待物理内存的时候的概念. 在内核中，除了内存管理模块直接操作物理地址之外，内核的其他模块，仍然要操作虚拟地址，而虚拟地址是需要内存管理模块分配和映射好的.

假设咱们的电脑有 2G 内存，现在如果内核的其他模块想要访问物理内存 1.5G 的地方，应该怎么办呢？首先，你不能使用物理地址。你需要使用内存管理模块给你分配的虚拟地址，但是虚拟地址的 0 到 3G 已经被用户态进程占用去了，你作为内核不能使用. 因为你写 1.5G 的虚拟内存位置，一方面你不知道应该根据哪个进程的页表进行映射；另一方面，就算映射了也不是你真正想访问的物理内存的地方，所以你发现你作为内核，能够使用的虚拟内存地址，只剩下 1G 减去 896M 的空间了.

于是，我们可以将剩下的虚拟内存地址分成下面这几个部分:
- 在 896M 到 VMALLOC_START 之间有 8M 的空间
- VMALLOC_START 到 VMALLOC_END 之间称为内核动态映射空间，也即内核想像用户态进程一样 malloc 申请内存，在内核里面可以使用 vmalloc. 假设物理内存里面，896M 到 1.5G 之间已经被用户态进程占用了，并且映射关系放在了进程的页表中，内核 vmalloc 的时候，只能从分配物理内存 1.5G 开始，就需要使用这一段的虚拟地址进行映射，映射关系放在专门给内核自己用的页表里面
- PKMAP_BASE 到 FIXADDR_START 的空间称为持久内核映射. 使用 alloc_pages() 函数的时候，在物理内存的高端内存得到 struct page 结构，可以调用 kmap 将其映射到这个区域
- FIXADDR_START 到 FIXADDR_TOP(0xFFFF F000) 的空间，称为固定映射区域，主要用于满足特殊需求
- 在最后一个区域可以通过 kmap_atomic 实现临时内核映射. 假设用户态的进程要映射一个文件到内存中，先要映射用户态进程空间的一段虚拟地址到物理内存，然后将文件内容写入这个物理内存供用户态进程访问. 给用户态进程分配物理内存页可以通过 alloc_pages()，分配完毕后，按说将用户态进程虚拟地址和物理内存的映射关系放在用户态进程的页表中，就完事大吉了. 这个时候，用户态进程可以通过用户态的虚拟地址，也即 0 至 3G 的部分，经过页表映射后访问物理内存，并不需要内核态的虚拟地址里面也划出一块来，映射到这个物理内存页. 但是如果要把文件内容写入物理内存，这件事情要内核来干了，这就只好通过 kmap_atomic 做一个临时映射，写入物理内存完毕后，再 kunmap_atomic 来解映射即可.

到此, 32 位的内核态布局就完了.

##### 64 bit
其实 64 位的内核布局反而简单，因为虚拟空间实在是太大了，根本不需要所谓的高端内存，因为内核是 128T，根本不可能有物理内存超过这个值.
![](/misc/img/process/7eaf620768c62ff53e5ea2b11b4940f6.jpg)

64 位的内核主要包含以下几个部分:
- 从 0xffff800000000000 开始就是内核的部分，只不过一开始有 8T 的空洞区域
- 从 __PAGE_OFFSET_BASE(0xffff880000000000) 开始的 64T 的虚拟地址空间是直接映射区域，也就是减去 PAGE_OFFSET 就是物理地址. 虚拟地址和物理地址之间的映射在大部分情况下还是会通过建立页表的方式进行映射
- 从 VMALLOC_START（0xffffc90000000000）开始到 VMALLOC_END（0xffffe90000000000）的 32T 的空间是给 vmalloc 的
- 从 VMEMMAP_START（0xffffea0000000000）开始的 1T 空间用于存放物理页面的描述结构 struct page 的
- 从 __START_KERNEL_map（0xffffffff80000000）开始的 512M 用于存放内核代码段、全局变量、BSS 等. 这里对应到物理内存开始的位置，减去 __START_KERNEL_map 就能得到物理内存的地址. 这里和直接映射区有点像，但是不矛盾，因为直接映射区之前有 8T 的空当区域，早就过了内核代码在物理内存中加载的位置.

到此, 64 位的内核态布局也完了.

#### 物理内存的管理
参考:
- [内存管理（上）](https://www.lagou.com/lgeduarticle/46096.html)
- [内存模型「memory model」](https://chasinglulu.github.io/2019/05/29/%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B%E3%80%8Cmemory-model%E3%80%8D/)
- [linux内存管理笔记(十七）----linux内存模型](https://blog.csdn.net/u012489236/article/details/106323088)

Linux目前支持三种内存模型(memory model)： FLATMEM（平坦内存模型）、DISCONTIGMEM（不连续内存模型）和SPARSEMEM（稀疏内存模型）. 不同Linux内存模型以不同方式管理和组织可用物理内存，具体方式取决于系统物理内存是否不连续且存在空隙. SPARSEMEM和DISCONTIGMEM实际上作用相同, 目前多采用SPARSEMEM, 其支持内存热插拔等新特性.

> memory model，其实就是从cpu的角度看其物理内存的分布情况, 在linux kernel中，使用什么的方式来管理这些物理内存.

每种内存模型的特点：
- FLATMEM

	- 内存连续且不存在空隙
    - 在大多数情况下，应用于UMA系统`Uniform Memory Access`
    - 通过CONFIG_FLATMEM配置

- DISCONTIGMEM

    - 多个内存节点不连续并且存在空隙`hole`
    - 适用于UMA系统和NUMA系统`Non-Uniform Memory Access`
    - [ARM在2010年已移除了对DISCONTIGMEM内存模型的支持](https://github.com/torvalds/linux/commit/be370302742ff9948f2a42b15cb2ba174d97b930)
    - 通过CONFIG_CONTIGMEM配置

- SPARSEMEM

    - 多个内存区域不连续并且存在空隙
    - 支持内存热插拔`hot-plug memory`，但性能稍逊色于DISCONTIGMEM
    - 在x86或ARM64内核采用了最近实现的SPARSEMEM_VMEMMAP变种，其性能比DISCONTIGMEM更优并且与FLATMEM相当
    - 对于ARM64内核，默认选择SPARSEMEM内存模型
    - 以section为单位管理online和hot-plug内存
    - 通过CONFIG_SPARSEMEM配置

![主流体系架构支持的内存模型](/misc/img/process/Kazam_screenshot_00000.png)
![确定内存模型的内核配置选项](/misc/img/process/memory_model_options.svg)

![](/misc/img/process/8f158f58dda94ec04b26200073e15449.jpeg)

SMP （Symmetric Multiprocessing, 对称多处理器）, 顾名思义, 在SMP中所有的处理器都是对等的, 它们通过公用总线连接共享同一块物理内存，而且距离都是一样的, 这也就导致了系统中所有资源(CPU、内存、I/O等)都是共享的. 它有一个显著的缺点，就是总线会成为瓶颈，因为数据都要走它. 其架构简单，但是拓展性能非常差，从linux 上能看到:
```bash
# ls /sys/devices/system/node/  # 如果只看到一个node0 那就是smp架构
```

为了提高性能和可扩展性，后来有了一种更高级的模式，NUMA（Non-uniform memory access），非一致性内存访问. 这种模型的是为了解决smp扩容性很差而提出的技术方案，如果说smp 相当于多个cpu 连接一个内存池导致请求经常发生冲突的话，numa 就是将cpu的资源分开，以node 为单位进行切割，每个node 里有着独有的core ，memory 等资源，CPU 访问本地内存不用过总线，因而速度要快很多，每个 CPU 和内存在一起，称为一个 NUMA 节点. 这也就导致了cpu在性能使用上的提升，但是同样存在问题就是2个node 之间的资源交互非常慢，比如在本地内存不足的情况下，每个 CPU 都可以去另外的 NUMA 节点申请内存，这个时候访问延时就会比较长.

**NUMA可体现在cpu里(AMD EPYC)或cpu+主板上, 本质都是numa相关的物理cpu(该cpu可能在一个大cpu里, 比如下图的AMD EPYC)的引脚与内存相连. 相关的硬件情况可用`numactl --hardware`查看.**
![](/misc/img/task/803522171f494d3e913485bef0c5e840.png)

这里仅说明NUMA方式, 且先上结论:

如果有多个 CPU，那就有多个节点. 每个节点用 struct pglist_data 表示，放在一个数组里面.

每个节点分为多个区域，每个区域用 struct zone 表示，也放在一个数组里面.

每个区域分为多个页. 为了方便分配，空闲页放在 struct free_area 里面，使用伙伴系统进行管理和分配，每一页用 struct page 表示.

![](/misc/img/process/3fa8123990e5ae2c86859f70a8351f4f.jpeg)

伙伴系统将多个连续的页面作为一个大的内存块分配给上层.
kswapd 负责物理页面的换入换出.
Slub Allocator 将从伙伴系统申请的大内存块切成小块，分配给其他系统.

![](/misc/img/process/527e5c861fd06c6eb61a761e4214ba54.jpeg)

kernel使用了`typedef struct pglist_data pg_data_t`来表示NUMA node:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/mmzone.h#L662
/*
 * On NUMA machines, each NUMA node would have a pg_data_t to describe
 * it's memory layout. On UMA machines there is a single pglist_data which
 * describes the whole memory.
 *
 * Memory statistics and page replacement data structures are maintained on a
 * per-zone basis.
 */
typedef struct pglist_data {
	/*
	 * node_zones contains just the zones for THIS node. Not all of the
	 * zones may be populated, but it is the full list. It is referenced by
	 * this node's node_zonelists as well as other node's node_zonelists.
	 */
	struct zone node_zones[MAX_NR_ZONES];

	/*
	 * node_zonelists contains references to all zones in all nodes.
	 * Generally the first zones will be references to this node's
	 * node_zones.
	 */
	struct zonelist node_zonelists[MAX_ZONELISTS];

	int nr_zones; /* number of populated zones in this node */
#ifdef CONFIG_FLAT_NODE_MEM_MAP	/* means !SPARSEMEM */
	struct page *node_mem_map;
#ifdef CONFIG_PAGE_EXTENSION
	struct page_ext *node_page_ext;
#endif
#endif
#if defined(CONFIG_MEMORY_HOTPLUG) || defined(CONFIG_DEFERRED_STRUCT_PAGE_INIT)
	/*
	 * Must be held any time you expect node_start_pfn,
	 * node_present_pages, node_spanned_pages or nr_zones to stay constant.
	 * Also synchronizes pgdat->first_deferred_pfn during deferred page
	 * init.
	 *
	 * pgdat_resize_lock() and pgdat_resize_unlock() are provided to
	 * manipulate node_size_lock without checking for CONFIG_MEMORY_HOTPLUG
	 * or CONFIG_DEFERRED_STRUCT_PAGE_INIT.
	 *
	 * Nests above zone->lock and zone->span_seqlock
	 */
	spinlock_t node_size_lock;
#endif
	unsigned long node_start_pfn;
	unsigned long node_present_pages; /* total number of physical pages */
	unsigned long node_spanned_pages; /* total size of physical page
					     range, including holes */
	int node_id;
	wait_queue_head_t kswapd_wait;
	wait_queue_head_t pfmemalloc_wait;
	struct task_struct *kswapd;	/* Protected by
					   mem_hotplug_begin/end() */
	int kswapd_order;
	enum zone_type kswapd_highest_zoneidx;

	int kswapd_failures;		/* Number of 'reclaimed == 0' runs */

#ifdef CONFIG_COMPACTION
	int kcompactd_max_order;
	enum zone_type kcompactd_highest_zoneidx;
	wait_queue_head_t kcompactd_wait;
	struct task_struct *kcompactd;
#endif
	/*
	 * This is a per-node reserve of pages that are not available
	 * to userspace allocations.
	 */
	unsigned long		totalreserve_pages;

#ifdef CONFIG_NUMA
	/*
	 * node reclaim becomes active if more unmapped pages exist.
	 */
	unsigned long		min_unmapped_pages;
	unsigned long		min_slab_pages;
#endif /* CONFIG_NUMA */

	/* Write-intensive fields used by page reclaim */
	ZONE_PADDING(_pad1_)
	spinlock_t		lru_lock;

#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
	/*
	 * If memory initialisation on large machines is deferred then this
	 * is the first PFN that needs to be initialised.
	 */
	unsigned long first_deferred_pfn;
#endif /* CONFIG_DEFERRED_STRUCT_PAGE_INIT */

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
	struct deferred_split deferred_split_queue;
#endif

	/* Fields commonly accessed by the page reclaim scanner */

	/*
	 * NOTE: THIS IS UNUSED IF MEMCG IS ENABLED.
	 *
	 * Use mem_cgroup_lruvec() to look up lruvecs.
	 */
	struct lruvec		__lruvec;

	unsigned long		flags;

	ZONE_PADDING(_pad2_)

	/* Per-node vmstats */
	struct per_cpu_nodestat __percpu *per_cpu_nodestats;
	atomic_long_t		vm_stat[NR_VM_NODE_STAT_ITEMS];
} pg_data_t;
```

它里面有以下的成员变量：
- 每一个节点都有自己的 ID：node_id
- node_mem_map 就是这个节点的 struct page 数组，用于描述这个节点里面的所有的页
- node_start_pfn 是这个节点的起始页号
- node_spanned_pages 是这个节点中包含不连续的物理内存地址的页面数
- node_present_pages 是真正可用的物理页面的数目

例如，64M 物理内存隔着一个 4M 的空洞，然后是另外的 64M 物理内存. 这样换算成页面数目就是，16K 个页面隔着 1K 个页面，然后是另外 16K 个页面. 这种情况下，node_spanned_pages 就是 33K 个页面，node_present_pages 就是 32K 个页面.

每一个节点分成一个个区域 zone，放在数组 node_zones 里面。这个数组的大小为 MAX_NR_ZONES.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/mmzone.h#L321
enum zone_type {
	/*
	 * ZONE_DMA and ZONE_DMA32 are used when there are peripherals not able
	 * to DMA to all of the addressable memory (ZONE_NORMAL).
	 * On architectures where this area covers the whole 32 bit address
	 * space ZONE_DMA32 is used. ZONE_DMA is left for the ones with smaller
	 * DMA addressing constraints. This distinction is important as a 32bit
	 * DMA mask is assumed when ZONE_DMA32 is defined. Some 64-bit
	 * platforms may need both zones as they support peripherals with
	 * different DMA addressing limitations.
	 *
	 * Some examples:
	 *
	 *  - i386 and x86_64 have a fixed 16M ZONE_DMA and ZONE_DMA32 for the
	 *    rest of the lower 4G.
	 *
	 *  - arm only uses ZONE_DMA, the size, up to 4G, may vary depending on
	 *    the specific device.
	 *
	 *  - arm64 has a fixed 1G ZONE_DMA and ZONE_DMA32 for the rest of the
	 *    lower 4G.
	 *
	 *  - powerpc only uses ZONE_DMA, the size, up to 2G, may vary
	 *    depending on the specific device.
	 *
	 *  - s390 uses ZONE_DMA fixed to the lower 2G.
	 *
	 *  - ia64 and riscv only use ZONE_DMA32.
	 *
	 *  - parisc uses neither.
	 */
#ifdef CONFIG_ZONE_DMA
	ZONE_DMA,
#endif
#ifdef CONFIG_ZONE_DMA32
	ZONE_DMA32,
#endif
	/*
	 * Normal addressable memory is in ZONE_NORMAL. DMA operations can be
	 * performed on pages in ZONE_NORMAL if the DMA devices support
	 * transfers to all addressable memory.
	 */
	ZONE_NORMAL,
#ifdef CONFIG_HIGHMEM
	/*
	 * A memory area that is only addressable by the kernel through
	 * mapping portions into its own address space. This is for example
	 * used by i386 to allow the kernel to address the memory beyond
	 * 900MB. The kernel will set up special mappings (page
	 * table entries on i386) for each page that the kernel needs to
	 * access.
	 */
	ZONE_HIGHMEM,
#endif
	ZONE_MOVABLE,
#ifdef CONFIG_ZONE_DEVICE
	ZONE_DEVICE,
#endif
	__MAX_NR_ZONES

};
```

ZONE_DMA 是指可用于作 DMA（Direct Memory Access，直接内存存取）的内存. DMA 是这样一种机制：要把外设的数据读入内存或把内存的数据传送到外设，原来都要通过 CPU 控制完成，但是这会占用 CPU，影响 CPU 处理其他事情，所以有了 DMA 模式. CPU 只需向 DMA 控制器下达指令，让 DMA 控制器来处理数据的传送，数据传送完毕再把信息反馈给 CPU，这样就可以解放 CPU.

对于 64 位系统，有两个 DMA 区域. 除了ZONE_DMA，还有 ZONE_DMA32.

ZONE_NORMAL 是直接映射区，就是从物理内存到虚拟内存的内核区域，通过加上一个常量直接映射. ZONE_HIGHMEM 是高端内存区，对于 32 位系统来说超过 896M 的地方，对于 64 位没必要有的一段区域.

ZONE_MOVABLE 是可移动区域，通过将物理内存划分为可移动分配区域和不可移动分配区域来避免内存碎片.

nr_zones 表示当前节点的区域的数量. node_zonelists 是备用节点和它的内存区域的情况. 如果numa节点内存不够就需要去其他节点进行分配. 毕竟，就算在备用节点里面选择，慢了点也比没有强. 既然整个内存被分成了多个节点，那 pglist_data 应该放在一个数组里面.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/mm/numa.c#L25
struct pglist_data *node_data[MAX_NUMNODES] __read_mostly;
EXPORT_SYMBOL(node_data);
```

到这里，我们把内存分成了节点，把节点分成了区域. 接下来看看, 一个区域里面是如何组织的. 表示区域的数据结构 zone 的定义如下:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/mmzone.h#L385
struct zone {
	/* Read-mostly fields */

	/* zone watermarks, access with *_wmark_pages(zone) macros */
	unsigned long _watermark[NR_WMARK];
	unsigned long watermark_boost;

	unsigned long nr_reserved_highatomic;

	/*
	 * We don't know if the memory that we're going to allocate will be
	 * freeable or/and it will be released eventually, so to avoid totally
	 * wasting several GB of ram we must reserve some of the lower zone
	 * memory (otherwise we risk to run OOM on the lower zones despite
	 * there being tons of freeable ram on the higher zones).  This array is
	 * recalculated at runtime if the sysctl_lowmem_reserve_ratio sysctl
	 * changes.
	 */
	long lowmem_reserve[MAX_NR_ZONES];

#ifdef CONFIG_NUMA
	int node;
#endif
	struct pglist_data	*zone_pgdat;
	struct per_cpu_pageset __percpu *pageset;

#ifndef CONFIG_SPARSEMEM
	/*
	 * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
	 * In SPARSEMEM, this map is stored in struct mem_section
	 */
	unsigned long		*pageblock_flags;
#endif /* CONFIG_SPARSEMEM */

	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
	unsigned long		zone_start_pfn;

	/*
	 * spanned_pages is the total pages spanned by the zone, including
	 * holes, which is calculated as:
	 * 	spanned_pages = zone_end_pfn - zone_start_pfn;
	 *
	 * present_pages is physical pages existing within the zone, which
	 * is calculated as:
	 *	present_pages = spanned_pages - absent_pages(pages in holes);
	 *
	 * managed_pages is present pages managed by the buddy system, which
	 * is calculated as (reserved_pages includes pages allocated by the
	 * bootmem allocator):
	 *	managed_pages = present_pages - reserved_pages;
	 *
	 * So present_pages may be used by memory hotplug or memory power
	 * management logic to figure out unmanaged pages by checking
	 * (present_pages - managed_pages). And managed_pages should be used
	 * by page allocator and vm scanner to calculate all kinds of watermarks
	 * and thresholds.
	 *
	 * Locking rules:
	 *
	 * zone_start_pfn and spanned_pages are protected by span_seqlock.
	 * It is a seqlock because it has to be read outside of zone->lock,
	 * and it is done in the main allocator path.  But, it is written
	 * quite infrequently.
	 *
	 * The span_seq lock is declared along with zone->lock because it is
	 * frequently read in proximity to zone->lock.  It's good to
	 * give them a chance of being in the same cacheline.
	 *
	 * Write access to present_pages at runtime should be protected by
	 * mem_hotplug_begin/end(). Any reader who can't tolerant drift of
	 * present_pages should get_online_mems() to get a stable value.
	 */
	atomic_long_t		managed_pages;
	unsigned long		spanned_pages;
	unsigned long		present_pages;

	const char		*name;

#ifdef CONFIG_MEMORY_ISOLATION
	/*
	 * Number of isolated pageblock. It is used to solve incorrect
	 * freepage counting problem due to racy retrieving migratetype
	 * of pageblock. Protected by zone->lock.
	 */
	unsigned long		nr_isolate_pageblock;
#endif

#ifdef CONFIG_MEMORY_HOTPLUG
	/* see spanned/present_pages for more description */
	seqlock_t		span_seqlock;
#endif

	int initialized;

	/* Write-intensive fields used from the page allocator */
	ZONE_PADDING(_pad1_)

	/* free areas of different sizes */
	struct free_area	free_area[MAX_ORDER];

	/* zone flags, see below */
	unsigned long		flags;

	/* Primarily protects free_area */
	spinlock_t		lock;

	/* Write-intensive fields used by compaction and vmstats. */
	ZONE_PADDING(_pad2_)

	/*
	 * When free pages are below this point, additional steps are taken
	 * when reading the number of free pages to avoid per-cpu counter
	 * drift allowing watermarks to be breached
	 */
	unsigned long percpu_drift_mark;

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* pfn where compaction free scanner should start */
	unsigned long		compact_cached_free_pfn;
	/* pfn where async and sync compaction migration scanner should start */
	unsigned long		compact_cached_migrate_pfn[2];
	unsigned long		compact_init_migrate_pfn;
	unsigned long		compact_init_free_pfn;
#endif

#ifdef CONFIG_COMPACTION
	/*
	 * On compaction failure, 1<<compact_defer_shift compactions
	 * are skipped before trying again. The number attempted since
	 * last failure is tracked with compact_considered.
	 */
	unsigned int		compact_considered;
	unsigned int		compact_defer_shift;
	int			compact_order_failed;
#endif

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
	/* Set to true when the PG_migrate_skip bits should be cleared */
	bool			compact_blockskip_flush;
#endif

	bool			contiguous;

	ZONE_PADDING(_pad3_)
	/* Zone statistics */
	atomic_long_t		vm_stat[NR_VM_ZONE_STAT_ITEMS];
	atomic_long_t		vm_numa_stat[NR_VM_NUMA_STAT_ITEMS];
} ____cacheline_internodealigned_in_smp;
```

在一个 zone 里面，zone_start_pfn 表示属于这个 zone 的第一个页.

spanned_pages = zone_end_pfn - zone_start_pfn，也即 spanned_pages 指的是不管中间有没有物理内存空洞，反正就是最后的页号减去起始的页号.

present_pages = spanned_pages - absent_pages(pages in holes)，也即 present_pages 是这个 zone 在物理内存中真实存在的所有 page 数目.

managed_pages = present_pages - reserved_pages，也即 managed_pages 是这个 zone 被伙伴系统管理的所有的 page 数目.

per_cpu_pageset 用于区分冷热页. 什么叫冷热页呢？为了让 CPU 快速访问段描述符，在 CPU 里面有段描述符缓存. CPU 访问这个缓存的速度比内存快得多. 同样对于页面来讲，也是这样的. 如果一个页被加载到 CPU 高速缓存里面，这就是一个热页（Hot Page），CPU 读起来速度会快很多，如果没有就是冷页（Cold Page）. 由于每个 CPU 都有自己的高速缓存，因而 per_cpu_pageset 也是每个 CPU 一个.

了解了区域 zone，接下来我们就到了组成物理内存的基本单位，页的数据结构 struct page:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/mm_types.h
struct page {
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	/*
	 * Five words (20/40 bytes) are available in this union.
	 * WARNING: bit 0 of the first word is used for PageTail(). That
	 * means the other users of this union MUST NOT use the bit to
	 * avoid collision and false-positive PageTail().
	 */
	union {
		struct {	/* Page cache and anonymous pages */
			/**
			 * @lru: Pageout list, eg. active_list protected by
			 * pgdat->lru_lock.  Sometimes used as a generic list
			 * by the page owner.
			 */
			struct list_head lru;
			/* See page-flags.h for PAGE_MAPPING_FLAGS */
			struct address_space *mapping;
			pgoff_t index;		/* Our offset within mapping. */
			/**
			 * @private: Mapping-private opaque data.
			 * Usually used for buffer_heads if PagePrivate.
			 * Used for swp_entry_t if PageSwapCache.
			 * Indicates order in the buddy system if PageBuddy.
			 */
			unsigned long private;
		};
		struct {	/* page_pool used by netstack */
			/**
			 * @dma_addr: might require a 64-bit value even on
			 * 32-bit architectures.
			 */
			dma_addr_t dma_addr;
		};
		struct {	/* slab, slob and slub */
			union {
				struct list_head slab_list;
				struct {	/* Partial pages */
					struct page *next;
#ifdef CONFIG_64BIT
					int pages;	/* Nr of pages left */
					int pobjects;	/* Approximate count */
#else
					short int pages;
					short int pobjects;
#endif
				};
			};
			struct kmem_cache *slab_cache; /* not slob */
			/* Double-word boundary */
			void *freelist;		/* first free object */
			union {
				void *s_mem;	/* slab: first object */
				unsigned long counters;		/* SLUB */
				struct {			/* SLUB */
					unsigned inuse:16;
					unsigned objects:15;
					unsigned frozen:1;
				};
			};
		};
		struct {	/* Tail pages of compound page */
			unsigned long compound_head;	/* Bit zero is set */

			/* First tail page only */
			unsigned char compound_dtor;
			unsigned char compound_order;
			atomic_t compound_mapcount;
		};
		struct {	/* Second tail page of compound page */
			unsigned long _compound_pad_1;	/* compound_head */
			atomic_t hpage_pinned_refcount;
			/* For both global and memcg */
			struct list_head deferred_list;
		};
		struct {	/* Page table pages */
			unsigned long _pt_pad_1;	/* compound_head */
			pgtable_t pmd_huge_pte; /* protected by page->ptl */
			unsigned long _pt_pad_2;	/* mapping */
			union {
				struct mm_struct *pt_mm; /* x86 pgds only */
				atomic_t pt_frag_refcount; /* powerpc */
			};
#if ALLOC_SPLIT_PTLOCKS
			spinlock_t *ptl;
#else
			spinlock_t ptl;
#endif
		};
		struct {	/* ZONE_DEVICE pages */
			/** @pgmap: Points to the hosting device page map. */
			struct dev_pagemap *pgmap;
			void *zone_device_data;
			/*
			 * ZONE_DEVICE private pages are counted as being
			 * mapped so the next 3 words hold the mapping, index,
			 * and private fields from the source anonymous or
			 * page cache page while the page is migrated to device
			 * private memory.
			 * ZONE_DEVICE MEMORY_DEVICE_FS_DAX pages also
			 * use the mapping, index, and private fields when
			 * pmem backed DAX files are mapped.
			 */
		};

		/** @rcu_head: You can use this to free a page by RCU. */
		struct rcu_head rcu_head;
	};

	union {		/* This union is 4 bytes in size. */
		/*
		 * If the page can be mapped to userspace, encodes the number
		 * of times this page is referenced by a page table.
		 */
		atomic_t _mapcount;

		/*
		 * If the page is neither PageSlab nor mappable to userspace,
		 * the value stored here may help determine what this page
		 * is used for.  See page-flags.h for a list of page types
		 * which are currently stored here.
		 */
		unsigned int page_type;

		unsigned int active;		/* SLAB */
		int units;			/* SLOB */
	};

	/* Usage count. *DO NOT USE DIRECTLY*. See page_ref.h */
	atomic_t _refcount;

#ifdef CONFIG_MEMCG
	struct mem_cgroup *mem_cgroup;
#endif

	/*
	 * On machines where all RAM is mapped into kernel address space,
	 * we can simply calculate the virtual address. On machines with
	 * highmem some memory is mapped into kernel virtual memory
	 * dynamically, so we need a place to store that address.
	 * Note that this field could be 16 bits on x86 ... ;)
	 *
	 * Architectures with slow multiplication can define
	 * WANT_PAGE_VIRTUAL in asm/page.h
	 */
#if defined(WANT_PAGE_VIRTUAL)
	void *virtual;			/* Kernel virtual address (NULL if
					   not kmapped, ie. highmem) */
#endif /* WANT_PAGE_VIRTUAL */

#ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
	int _last_cpupid;
#endif
} _struct_page_alignment;
```

这是一个特别复杂的结构，里面有很多的 union，union 结构是在 C 语言中被用于同一块内存根据情况保存不同类型数据的一种方式. 这里之所以用了 union，是因为一个物理页面使用模式有多种.

第一种模式，要用就用一整页. 这一整页的内存，或者直接和虚拟地址空间建立映射关系，我们把这种称为匿名页（Anonymous Page）. 或者用于关联一个文件，然后再和虚拟地址空间建立映射关系，这样的文件，我们称为内存映射文件（Memory-mapped File）.

如果某一页是这种使用模式，则会使用 union 中的以下变量:
- struct address_space *mapping 就是用于内存映射，如果是匿名页，最低位为 1；如果是映射文件，最低位为 0
- pgoff_t index 是在映射区的偏移量
- atomic_t _mapcount，每个进程都有自己的页表，这里指有多少个页表项指向了这个页
- struct list_head lru 表示这一页应该在一个链表上，例如这个页面被换出，就在换出页的链表中
- compound 相关的变量用于复合页（Compound Page），就是将物理上连续的两个或多个页看成一个独立的大页.

第二种模式，仅需分配小块内存. 有时候，我们不需要一下子分配这么多的内存，例如分配一个 task_struct 结构，只需要分配小块的内存，去存储这个进程描述结构的对象. 为了满足对这种小内存块的需要，Linux 系统采用了一种被称为 slab allocator 的技术，用于分配称为 slab 的一小块内存. 它的基本原理是从内存管理模块申请一整块页，然后划分成多个小块的存储池，用复杂的队列来维护这些小块的状态（状态包括：被分配了 / 被放回池子 / 应该被回收）.

也正是因为 slab allocator 对于队列的维护过于复杂，后来就有了一种不使用队列的分配器 slub allocator，但它里面还是用了很多 slab 的字眼，因为它保留了 slab 的用户接口，可以看成 slab allocator 的另一种实现. 还有一种小块内存的分配器称为 slob，非常简单，主要使用在小型的嵌入式系统.

如果某一页是用于分割成一小块一小块的内存进行分配的使用模式，则会使用 union 中的以下变量：
- s_mem 是已经分配了正在使用的 slab 的第一个对象
- freelist 是池子中的空闲对象
- rcu_head 是需要释放的列表

好了，上面讲了物理内存的组织，从节点到区域到页到小块. 接下来看物理内存的分配.

对于要分配比较大的内存，例如到分配页级别的，可以使用伙伴系统（Buddy System）.

Linux 中的内存管理的“页”大小为 4KB. 把所有的空闲页分组为 11 个页块链表，每个块链表分别包含很多个大小的页块，有 1、2、4、8、16、32、64、128、256、512 和 1024 个连续页的页块. 最大可以申请 1024 个连续页，对应 4MB 大小的连续内存. 每个页块的第一个页的物理地址是该页块大小的整数倍.

第 i 个页块链表中，页块中页的数目为 2^i. 在 struct zone 里面有以下的定义：`
struct free_area  free_area[MAX_ORDER];`, 其中[MAX_ORDER](https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/mmzone.h#L27)就是指数11.

当向内核请求分配 (2^(i-1)，2^i]数目的页块时，按照 2^i 页块请求处理. 如果对应的页块链表中没有空闲页块，那我们就在更大的页块链表中去找. 当分配的页块中有多余的页时，伙伴系统会根据多余的页块大小插入到对应的空闲页块链表中.

例如，要请求一个 128 个页的页块时，先检查 128 个页的页块链表是否有空闲块. 如果没有，则查 256 个页的页块链表；如果有空闲块的话，则将 256 个页的页块分成两份，一份使用，一份插入 128 个页的页块链表中. 如果还是没有，就查 512 个页的页块链表；如果有的话，就分裂为 128、128、256 三个页块，一个 128 的使用，剩余两个插入对应页块链表. 该过程，可以在分配页的函数 alloc_pages 中看到:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/gfp.h#L543
static inline struct page *
alloc_pages(gfp_t gfp_mask, unsigned int order)
{
	return alloc_pages_current(gfp_mask, order);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/mempolicy.c#L2277
/**
 * 	alloc_pages_current - Allocate pages.
 *
 *	@gfp:
 *		%GFP_USER   user allocation,
 *      	%GFP_KERNEL kernel allocation,
 *      	%GFP_HIGHMEM highmem allocation,
 *      	%GFP_FS     don't call back into a file system.
 *      	%GFP_ATOMIC don't sleep.
 *	@order: Power of two of allocation size in pages. 0 is a single page.
 *
 *	Allocate a page from the kernel page pool.  When not in
 *	interrupt context and apply the current process NUMA policy.
 *	Returns NULL when no page can be allocated.
 */
struct page *alloc_pages_current(gfp_t gfp, unsigned order)
{
	struct mempolicy *pol = &default_policy;
	struct page *page;

	if (!in_interrupt() && !(gfp & __GFP_THISNODE))
		pol = get_task_policy(current);

	/*
	 * No reference counting needed for current->mempolicy
	 * nor system default_policy
	 */
	if (pol->mode == MPOL_INTERLEAVE)
		page = alloc_page_interleave(gfp, order, interleave_nodes(pol));
	else
		page = __alloc_pages_nodemask(gfp, order,
				policy_node(gfp, pol, numa_node_id()),
				policy_nodemask(gfp, pol));

	return page;
}
EXPORT_SYMBOL(alloc_pages_current);
```

alloc_pages 会调用 alloc_pages_current，gfp 表示希望在哪个区域中分配这个内存:
- GFP_USER  用于分配一个页映射到用户进程的虚拟地址空间，并且希望直接被内核或者硬件访问，主要用于一个用户进程希望通过内存映射的方式，访问某些硬件的缓存，例如显卡缓存
- GFP_KERNEL 用于内核中分配页，主要分配 ZONE_NORMAL 区域，也即直接映射区
- GFP_HIGHMEM，顾名思义就是主要分配高端区域的内存

另一个参数 order，就是表示分配 2 的 order 次方个页.

接下来调用[__alloc_pages_nodemask](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/page_alloc.c#L4813). 这是伙伴系统的核心方法. 它会调用 get_page_from_freelist. 这里面的逻辑也很容易理解，就是在一个循环中先看当前节点的 zone. 如果找不到空闲页，则再看备用节点的 zone.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/page_alloc.c#L3679
/*
 * get_page_from_freelist goes through the zonelist trying to allocate
 * a page.
 */
static struct page *
get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
						const struct alloc_context *ac)
{
	struct zoneref *z;
	struct zone *zone;
	struct pglist_data *last_pgdat_dirty_limit = NULL;
	bool no_fallback;

retry:
	/*
	 * Scan zonelist, looking for a zone with enough free.
	 * See also __cpuset_node_allowed() comment in kernel/cpuset.c.
	 */
	no_fallback = alloc_flags & ALLOC_NOFRAGMENT;
	z = ac->preferred_zoneref;
	for_next_zone_zonelist_nodemask(zone, z, ac->zonelist,
					ac->highest_zoneidx, ac->nodemask) {
		struct page *page;
		unsigned long mark;

		if (cpusets_enabled() &&
			(alloc_flags & ALLOC_CPUSET) &&
			!__cpuset_zone_allowed(zone, gfp_mask))
				continue;
		/*
		 * When allocating a page cache page for writing, we
		 * want to get it from a node that is within its dirty
		 * limit, such that no single node holds more than its
		 * proportional share of globally allowed dirty pages.
		 * The dirty limits take into account the node's
		 * lowmem reserves and high watermark so that kswapd
		 * should be able to balance it without having to
		 * write pages from its LRU list.
		 *
		 * XXX: For now, allow allocations to potentially
		 * exceed the per-node dirty limit in the slowpath
		 * (spread_dirty_pages unset) before going into reclaim,
		 * which is important when on a NUMA setup the allowed
		 * nodes are together not big enough to reach the
		 * global limit.  The proper fix for these situations
		 * will require awareness of nodes in the
		 * dirty-throttling and the flusher threads.
		 */
		if (ac->spread_dirty_pages) {
			if (last_pgdat_dirty_limit == zone->zone_pgdat)
				continue;

			if (!node_dirty_ok(zone->zone_pgdat)) {
				last_pgdat_dirty_limit = zone->zone_pgdat;
				continue;
			}
		}

		if (no_fallback && nr_online_nodes > 1 &&
		    zone != ac->preferred_zoneref->zone) {
			int local_nid;

			/*
			 * If moving to a remote node, retry but allow
			 * fragmenting fallbacks. Locality is more important
			 * than fragmentation avoidance.
			 */
			local_nid = zone_to_nid(ac->preferred_zoneref->zone);
			if (zone_to_nid(zone) != local_nid) {
				alloc_flags &= ~ALLOC_NOFRAGMENT;
				goto retry;
			}
		}

		mark = wmark_pages(zone, alloc_flags & ALLOC_WMARK_MASK);
		if (!zone_watermark_fast(zone, order, mark,
				       ac->highest_zoneidx, alloc_flags)) {
			int ret;

#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
			/*
			 * Watermark failed for this zone, but see if we can
			 * grow this zone if it contains deferred pages.
			 */
			if (static_branch_unlikely(&deferred_pages)) {
				if (_deferred_grow_zone(zone, order))
					goto try_this_zone;
			}
#endif
			/* Checked here to keep the fast path fast */
			BUILD_BUG_ON(ALLOC_NO_WATERMARKS < NR_WMARK);
			if (alloc_flags & ALLOC_NO_WATERMARKS)
				goto try_this_zone;

			if (node_reclaim_mode == 0 ||
			    !zone_allows_reclaim(ac->preferred_zoneref->zone, zone))
				continue;

			ret = node_reclaim(zone->zone_pgdat, gfp_mask, order);
			switch (ret) {
			case NODE_RECLAIM_NOSCAN:
				/* did not scan */
				continue;
			case NODE_RECLAIM_FULL:
				/* scanned but unreclaimable */
				continue;
			default:
				/* did we reclaim enough */
				if (zone_watermark_ok(zone, order, mark,
					ac->highest_zoneidx, alloc_flags))
					goto try_this_zone;

				continue;
			}
		}

try_this_zone:
		page = rmqueue(ac->preferred_zoneref->zone, zone, order,
				gfp_mask, alloc_flags, ac->migratetype);
		if (page) {
			prep_new_page(page, order, gfp_mask, alloc_flags);

			/*
			 * If this is a high-order atomic allocation then check
			 * if the pageblock should be reserved for the future
			 */
			if (unlikely(order && (alloc_flags & ALLOC_HARDER)))
				reserve_highatomic_pageblock(page, zone, order);

			return page;
		} else {
#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
			/* Try again if zone has deferred pages */
			if (static_branch_unlikely(&deferred_pages)) {
				if (_deferred_grow_zone(zone, order))
					goto try_this_zone;
			}
#endif
		}
	}

	/*
	 * It's possible on a UMA machine to get through all zones that are
	 * fragmented. If avoiding fragmentation, reset and try again.
	 */
	if (no_fallback) {
		alloc_flags &= ~ALLOC_NOFRAGMENT;
		goto retry;
	}

	return NULL;
}
```

每一个 zone，都有伙伴系统维护的各种大小的队列，就像上面伙伴系统原理里讲的那样. 这里调用 rmqueue 就很好理解了，就是找到合适大小的那个队列，把页面取下来. 接下来的调用链是 rmqueue->__rmqueue->__rmqueue_smallest. 在__rmqueue_smallest里，我们能清楚看到伙伴系统的逻辑:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/page_alloc.c#L2249
/*
 * Go through the free lists for the given migratetype and remove
 * the smallest available page from the freelists
 */
static __always_inline
struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
						int migratetype)
{
	unsigned int current_order;
	struct free_area *area;
	struct page *page;

	/* Find a page of the appropriate size in the preferred list */
	for (current_order = order; current_order < MAX_ORDER; ++current_order) {
		area = &(zone->free_area[current_order]);
		page = get_page_from_free_area(area, migratetype);
		if (!page)
			continue;
		del_page_from_free_list(page, zone, current_order);
		expand(zone, page, order, current_order, migratetype);
		set_pcppage_migratetype(page, migratetype);
		return page;
	}

	return NULL;
}
```

从当前的 order，也即指数开始，在伙伴系统的 free_area 找 2^order 大小的页块. 如果链表的第一个不为空，就找到了；如果为空，就到更大的 order 的页块链表里面去找. 找到以后，除了将页块从链表中取下来，我们还要把多余部分放到其他页块链表里面. [expand](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/page_alloc.c#L2102) 就是干这个事情的. area就是伙伴系统那个表里面的前一项，前一项里面的页块大小是当前项的页块大小除以 2，size 右移一位也就是除以 2，list_add 就是加到链表上，nr_free++ 就是计数加 1.

##### 小内存的分配
> 理论上 SLUB 优于 SLAB ， 同时还保持了不错的向 SLAB 的兼容. 目前SLUB取代了SLAB，成为了默认的内存分配器.

参考:
- [linux 内核 内存管理 slub算法 （一） 原理](https://blog.csdn.net/lukuen/article/details/6935068)

如果遇到小的对象，会使用 slub 分配器进行分配.

创建进程的时候，会调用 dup_task_struct，它想要试图复制一个 task_struct 对象，需要先调用 alloc_task_struct_node，分配一个 task_struct 对象. 从这段代码可以看出，它调用了 kmem_cache_alloc_node 函数，在 task_struct 的缓存区域 task_struct_cachep 分配了一块内存.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L167
static inline struct task_struct *alloc_task_struct_node(int node)
{
	return kmem_cache_alloc_node(task_struct_cachep, GFP_KERNEL, node);
}


// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slab.c#L3573
/**
 * kmem_cache_alloc_node - Allocate an object on the specified node
 * @cachep: The cache to allocate from.
 * @flags: See kmalloc().
 * @nodeid: node number of the target node.
 *
 * Identical to kmem_cache_alloc but it will allocate memory on the given
 * node, which can improve the performance for cpu bound structures.
 *
 * Fallback to other node is possible if __GFP_THISNODE is not set.
 *
 * Return: pointer to the new object or %NULL in case of error
 */
void *kmem_cache_alloc_node(struct kmem_cache *cachep, gfp_t flags, int nodeid)
{
	void *ret = slab_alloc_node(cachep, flags, nodeid, _RET_IP_);

	trace_kmem_cache_alloc_node(_RET_IP_, ret,
				    cachep->object_size, cachep->size,
				    flags, nodeid);

	return ret;
}
EXPORT_SYMBOL(kmem_cache_alloc_node);
```

在系统初始化的时候，task_struct_cachep 会被 [kmem_cache_create](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slab_common.c#L561) 函数创建。这个函数也比较容易看懂，专门用于分配 task_struct 对象的缓存.

这个缓存区的名字就叫 task_struct. 缓存区中每一块的大小正好等于 task_struct 的大小，也即 arch_task_struct_size. 有了这个缓存区，每次创建 task_struct 的时候，我们不用到内存里面去分配，先在缓存里面看看有没有直接可用的，这就是 kmem_cache_alloc_node 的作用.

当一个进程结束，task_struct 也不用直接被销毁，而是放回到缓存中，这就是 [kmem_cache_free](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L3083) 的作用. 这样，新进程创建的时候，我们就可以直接用现成的缓存中的 task_struct 了.

缓存区 struct kmem_cache 长这样:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/slub_def.h#L82
/*
 * Slab cache management.
 */
struct kmem_cache {
	struct kmem_cache_cpu __percpu *cpu_slab;
	/* Used for retrieving partial slabs, etc. */
	slab_flags_t flags;
	unsigned long min_partial;
	unsigned int size;	/* The size of an object including metadata */
	unsigned int object_size;/* The size of an object without metadata */
	unsigned int offset;	/* Free pointer offset */
#ifdef CONFIG_SLUB_CPU_PARTIAL
	/* Number of per cpu partial objects to keep around */
	unsigned int cpu_partial;
#endif
	struct kmem_cache_order_objects oo;

	/* Allocation and freeing of slabs */
	struct kmem_cache_order_objects max;
	struct kmem_cache_order_objects min;
	gfp_t allocflags;	/* gfp flags to use on each alloc */
	int refcount;		/* Refcount for slab cache destroy */
	void (*ctor)(void *);
	unsigned int inuse;		/* Offset to metadata */
	unsigned int align;		/* Alignment */
	unsigned int red_left_pad;	/* Left redzone padding size */
	const char *name;	/* Name (only for display!) */
	struct list_head list;	/* List of slab caches */
#ifdef CONFIG_SYSFS
	struct kobject kobj;	/* For sysfs */
	struct work_struct kobj_remove_work;
#endif
#ifdef CONFIG_MEMCG
	struct memcg_cache_params memcg_params;
	/* For propagation, maximum size of a stored attr */
	unsigned int max_attr_size;
#ifdef CONFIG_SYSFS
	struct kset *memcg_kset;
#endif
#endif

#ifdef CONFIG_SLAB_FREELIST_HARDENED
	unsigned long random;
#endif

#ifdef CONFIG_NUMA
	/*
	 * Defragmentation by allocating from a remote node.
	 */
	unsigned int remote_node_defrag_ratio;
#endif

#ifdef CONFIG_SLAB_FREELIST_RANDOM
	unsigned int *random_seq;
#endif

#ifdef CONFIG_KASAN
	struct kasan_cache kasan_info;
#endif

	unsigned int useroffset;	/* Usercopy region offset */
	unsigned int usersize;		/* Usercopy region size */

	struct kmem_cache_node *node[MAX_NUMNODES];
};
```

在 struct kmem_cache 里面，有个变量 struct list_head list，这个结构我们已经看到过多次了. 我们可以想象一下，对于操作系统来讲，要创建和管理的缓存绝对不止 task_struct. 难道 mm_struct 就不需要吗？fs_struct 就不需要吗？都需要. 因此，所有的缓存最后都会放在一个链表里面，也就是 LIST_HEAD(slab_caches).

对于缓存来讲，其实就是分配了连续几页的大内存块，然后根据缓存对象的大小，切成小内存块.

所以，我们这里有三个 kmem_cache_order_objects 类型的变量. 这里面的 order，就是 2 的 order 次方个页面的大内存块，objects 就是能够存放的缓存对象的数量.

最终，我们将大内存块切分成小内存块.

每一项的结构都是缓存对象后面跟一个下一个空闲对象的指针，这样非常方便将所有的空闲对象链成一个链. 其实，这就相当于咱们数据结构里面学的，用数组实现一个可随机插入和删除的链表. 所以，这里面就有三个变量：size 是包含这个指针的大小，object_size 是纯对象的大小，offset 就是把下一个空闲对象的指针存放在这一项里的偏移量.

那这些缓存对象哪些被分配了、哪些在空着，什么情况下整个大内存块都被分配完了，需要向伙伴系统申请几个页形成新的大内存块？这些信息该由谁来维护呢？接下来就是最重要的两个成员变量出场的时候了. kmem_cache_cpu 和 kmem_cache_node，它们都是每个 NUMA 节点上有一个，我们只需要看一个节点里面的情况.

在分配缓存块的时候，要分两种路径，fast path 和 slow path，也就是快速通道和普通通道. 其中 kmem_cache_cpu 就是快速通道，kmem_cache_node 是普通通道. 每次分配的时候，要先从 kmem_cache_cpu 进行分配. 如果 kmem_cache_cpu 里面没有空闲的块，那就到 kmem_cache_node 中进行分配；如果还是没有空闲的块，才去伙伴系统分配新的页. kmem_cache_cpu 里面是如何存放缓存块的.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/slub_def.h#L41
struct kmem_cache_cpu {
	void **freelist;	/* Pointer to next available object */
	unsigned long tid;	/* Globally unique transaction id */
	struct page *page;	/* The slab from which we are allocating */
#ifdef CONFIG_SLUB_CPU_PARTIAL
	struct page *partial;	/* Partially allocated frozen slabs */
#endif
#ifdef CONFIG_SLUB_STATS
	unsigned stat[NR_SLUB_STAT_ITEMS];
#endif
};

#ifdef CONFIG_SLUB_CPU_PARTIAL
#define slub_percpu_partial(c)		((c)->partial)

#define slub_set_percpu_partial(c, p)		\
({						\
	slub_percpu_partial(c) = (p)->next;	\
})

#define slub_percpu_partial_read_once(c)     READ_ONCE(slub_percpu_partial(c))
#else
#define slub_percpu_partial(c)			NULL

#define slub_set_percpu_partial(c, p)

#define slub_percpu_partial_read_once(c)	NULL
#endif // CONFIG_SLUB_CPU_PARTIAL

/*
 * Word size structure that can be atomically updated or read and that
 * contains both the order and the number of objects that a slab of the
 * given order would contain.
 */
struct kmem_cache_order_objects {
	unsigned int x;
};
```

在这里，page 指向大内存块的第一个页，缓存块就是从里面分配的. freelist 指向大内存块里面第一个空闲的项. 按照上面说的，这一项会有指针指向下一个空闲的项，最终所有空闲的项会形成一个链表. partial 指向的也是大内存块的第一个页，之所以名字叫 partial（部分），就是因为它里面部分被分配出去了，部分是空的. 这是一个备用列表，当 page 满了，就会从这里找.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slab.h#L600
/*
 * The slab lists for all objects.
 */
struct kmem_cache_node {
	spinlock_t list_lock;

#ifdef CONFIG_SLAB
	struct list_head slabs_partial;	/* partial list first, better asm code */
	struct list_head slabs_full;
	struct list_head slabs_free;
	unsigned long total_slabs;	/* length of all slab lists */
	unsigned long free_slabs;	/* length of free slab list only */
	unsigned long free_objects;
	unsigned int free_limit;
	unsigned int colour_next;	/* Per-node cache coloring */
	struct array_cache *shared;	/* shared per node */
	struct alien_cache **alien;	/* on other nodes */
	unsigned long next_reap;	/* updated without locking */
	int free_touched;		/* updated without locking */
#endif

#ifdef CONFIG_SLUB
	unsigned long nr_partial;
	struct list_head partial;
#ifdef CONFIG_SLUB_DEBUG
	atomic_long_t nr_slabs;
	atomic_long_t total_objects;
	struct list_head full;
#endif
#endif

};
```

这里面也有一个 partial，是一个链表. 这个链表里存放的是部分空闲的内存块. 这是 kmem_cache_cpu 里面的 partial 的备用列表，如果那里没有，就到这里来找. 下面我们就来看看这个分配过程. kmem_cache_alloc_node 会调用 slab_alloc_node.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L2740
/*
 * Inlined fastpath so that allocation functions (kmalloc, kmem_cache_alloc)
 * have the fastpath folded into their functions. So no function call
 * overhead for requests that can be satisfied on the fastpath.
 *
 * The fastpath works by first checking if the lockless freelist can be used.
 * If not then __slab_alloc is called for slow processing.
 *
 * Otherwise we can simply pick the next object from the lockless free list.
 */
static __always_inline void *slab_alloc_node(struct kmem_cache *s,
		gfp_t gfpflags, int node, unsigned long addr)
{
	void *object;
	struct kmem_cache_cpu *c;
	struct page *page;
	unsigned long tid;

	s = slab_pre_alloc_hook(s, gfpflags);
	if (!s)
		return NULL;
redo:
	/*
	 * Must read kmem_cache cpu data via this cpu ptr. Preemption is
	 * enabled. We may switch back and forth between cpus while
	 * reading from one cpu area. That does not matter as long
	 * as we end up on the original cpu again when doing the cmpxchg.
	 *
	 * We should guarantee that tid and kmem_cache are retrieved on
	 * the same cpu. It could be different if CONFIG_PREEMPTION so we need
	 * to check if it is matched or not.
	 */
	do {
		tid = this_cpu_read(s->cpu_slab->tid);
		c = raw_cpu_ptr(s->cpu_slab);
	} while (IS_ENABLED(CONFIG_PREEMPTION) &&
		 unlikely(tid != READ_ONCE(c->tid)));

	/*
	 * Irqless object alloc/free algorithm used here depends on sequence
	 * of fetching cpu_slab's data. tid should be fetched before anything
	 * on c to guarantee that object and page associated with previous tid
	 * won't be used with current tid. If we fetch tid first, object and
	 * page could be one associated with next tid and our alloc/free
	 * request will be failed. In this case, we will retry. So, no problem.
	 */
	barrier();

	/*
	 * The transaction ids are globally unique per cpu and per operation on
	 * a per cpu queue. Thus they can be guarantee that the cmpxchg_double
	 * occurs on the right processor and that there was no operation on the
	 * linked list in between.
	 */

	object = c->freelist;
	page = c->page;
	if (unlikely(!object || !node_match(page, node))) {
		object = __slab_alloc(s, gfpflags, node, addr, c);
		stat(s, ALLOC_SLOWPATH);
	} else {
		void *next_object = get_freepointer_safe(s, object);

		/*
		 * The cmpxchg will only match if there was no additional
		 * operation and if we are on the right processor.
		 *
		 * The cmpxchg does the following atomically (without lock
		 * semantics!)
		 * 1. Relocate first pointer to the current per cpu area.
		 * 2. Verify that tid and freelist have not been changed
		 * 3. If they were not changed replace tid and freelist
		 *
		 * Since this is without lock semantics the protection is only
		 * against code executing on this cpu *not* from access by
		 * other cpus.
		 */
		if (unlikely(!this_cpu_cmpxchg_double(
				s->cpu_slab->freelist, s->cpu_slab->tid,
				object, tid,
				next_object, next_tid(tid)))) {

			note_cmpxchg_failure("slab_alloc", s, tid);
			goto redo;
		}
		prefetch_freepointer(s, next_object);
		stat(s, ALLOC_FASTPATH);
	}

	maybe_wipe_obj_freeptr(s, object);

	if (unlikely(slab_want_init_on_alloc(gfpflags, s)) && object)
		memset(object, 0, s->object_size);

	slab_post_alloc_hook(s, gfpflags, 1, &object);

	return object;
}
```

快速通道很简单，取出 cpu_slab 也即 kmem_cache_cpu 的 freelist，这就是第一个空闲的项，可以直接返回了. 如果没有空闲的了，则只好进入普通通道，调用 __slab_alloc.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L2740
/*
 * Slow path. The lockless freelist is empty or we need to perform
 * debugging duties.
 *
 * Processing is still very fast if new objects have been freed to the
 * regular freelist. In that case we simply take over the regular freelist
 * as the lockless freelist and zap the regular freelist.
 *
 * If that is not working then we fall back to the partial lists. We take the
 * first element of the freelist as the object to allocate now and move the
 * rest of the freelist to the lockless freelist.
 *
 * And if we were unable to get a new slab from the partial slab lists then
 * we need to allocate a new slab. This is the slowest path since it involves
 * a call to the page allocator and the setup of a new slab.
 *
 * Version of __slab_alloc to use when we know that interrupts are
 * already disabled (which is the case for bulk allocation).
 */
static void *___slab_alloc(struct kmem_cache *s, gfp_t gfpflags, int node,
			  unsigned long addr, struct kmem_cache_cpu *c)
{
	void *freelist;
	struct page *page;

	page = c->page;
	if (!page) {
		/*
		 * if the node is not online or has no normal memory, just
		 * ignore the node constraint
		 */
		if (unlikely(node != NUMA_NO_NODE &&
			     !node_state(node, N_NORMAL_MEMORY)))
			node = NUMA_NO_NODE;
		goto new_slab;
	}
redo:

	if (unlikely(!node_match(page, node))) {
		/*
		 * same as above but node_match() being false already
		 * implies node != NUMA_NO_NODE
		 */
		if (!node_state(node, N_NORMAL_MEMORY)) {
			node = NUMA_NO_NODE;
			goto redo;
		} else {
			stat(s, ALLOC_NODE_MISMATCH);
			deactivate_slab(s, page, c->freelist, c);
			goto new_slab;
		}
	}

	/*
	 * By rights, we should be searching for a slab page that was
	 * PFMEMALLOC but right now, we are losing the pfmemalloc
	 * information when the page leaves the per-cpu allocator
	 */
	if (unlikely(!pfmemalloc_match(page, gfpflags))) {
		deactivate_slab(s, page, c->freelist, c);
		goto new_slab;
	}

	/* must check again c->freelist in case of cpu migration or IRQ */
	freelist = c->freelist;
	if (freelist)
		goto load_freelist;

	freelist = get_freelist(s, page);

	if (!freelist) {
		c->page = NULL;
		stat(s, DEACTIVATE_BYPASS);
		goto new_slab;
	}

	stat(s, ALLOC_REFILL);

load_freelist:
	/*
	 * freelist is pointing to the list of objects to be used.
	 * page is pointing to the page from which the objects are obtained.
	 * That page must be frozen for per cpu allocations to work.
	 */
	VM_BUG_ON(!c->page->frozen);
	c->freelist = get_freepointer(s, freelist);
	c->tid = next_tid(c->tid);
	return freelist;

new_slab:

	if (slub_percpu_partial(c)) {
		page = c->page = slub_percpu_partial(c);
		slub_set_percpu_partial(c, page);
		stat(s, CPU_PARTIAL_ALLOC);
		goto redo;
	}

	freelist = new_slab_objects(s, gfpflags, node, &c);

	if (unlikely(!freelist)) {
		slab_out_of_memory(s, gfpflags, node);
		return NULL;
	}

	page = c->page;
	if (likely(!kmem_cache_debug(s) && pfmemalloc_match(page, gfpflags)))
		goto load_freelist;

	/* Only entered in the debug case */
	if (kmem_cache_debug(s) &&
			!alloc_debug_processing(s, page, freelist, addr))
		goto new_slab;	/* Slab failed checks. Next slab needed */

	deactivate_slab(s, page, get_freepointer(s, freelist), c);
	return freelist;
}
```

在这里，我们首先再次尝试一下 kmem_cache_cpu 的 freelist. 为什么呢？万一当前进程被中断，等回来的时候，别人已经释放了一些缓存，说不定又有空间了呢. 如果找到了，就跳到 load_freelist，在这里将 freelist 指向下一个空闲项，返回就可以了.

如果 freelist 还是没有，则跳到 new_slab 里面去. 这里面我们先去 kmem_cache_cpu 的 partial 里面看. 如果 partial 不是空的，那就将 kmem_cache_cpu 的 page，也就是快速通道的那一大块内存，替换为 partial 里面的大块内存. 然后 redo，重新试下. 这次应该就可以成功了. 如果真的还不行，那就要到 new_slab_objects 了.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L2499
static inline void *new_slab_objects(struct kmem_cache *s, gfp_t flags,
			int node, struct kmem_cache_cpu **pc)
{
	void *freelist;
	struct kmem_cache_cpu *c = *pc;
	struct page *page;

	WARN_ON_ONCE(s->ctor && (flags & __GFP_ZERO));

	freelist = get_partial(s, flags, node, c);

	if (freelist)
		return freelist;

	page = new_slab(s, flags, node);
	if (page) {
		c = raw_cpu_ptr(s->cpu_slab);
		if (c->page)
			flush_slab(s, c);

		/*
		 * No other reference to the page yet so we can
		 * muck around with it freely without cmpxchg
		 */
		freelist = page->freelist;
		page->freelist = NULL;

		stat(s, ALLOC_SLAB);
		c->page = page;
		*pc = c;
	}

	return freelist;
}
```

在这里面，[get_partial](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L1998) 会根据 node id，找到相应的 kmem_cache_node，然后调用 [get_partial_node](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L1885)，开始在这个节点进行分配.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L1885
/*
 * Try to allocate a partial slab from a specific node.
 */
static void *get_partial_node(struct kmem_cache *s, struct kmem_cache_node *n,
				struct kmem_cache_cpu *c, gfp_t flags)
{
	struct page *page, *page2;
	void *object = NULL;
	unsigned int available = 0;
	int objects;

	/*
	 * Racy check. If we mistakenly see no partial slabs then we
	 * just allocate an empty slab. If we mistakenly try to get a
	 * partial slab and there is none available then get_partials()
	 * will return NULL.
	 */
	if (!n || !n->nr_partial)
		return NULL;

	spin_lock(&n->list_lock);
	list_for_each_entry_safe(page, page2, &n->partial, slab_list) {
		void *t;

		if (!pfmemalloc_match(page, flags))
			continue;

		t = acquire_slab(s, n, page, object == NULL, &objects);
		if (!t)
			break;

		available += objects;
		if (!object) {
			c->page = page;
			stat(s, ALLOC_FROM_PARTIAL);
			object = t;
		} else {
			put_cpu_partial(s, page, 0);
			stat(s, CPU_PARTIAL_NODE);
		}
		if (!kmem_cache_has_cpu_partial(s)
			|| available > slub_cpu_partial(s) / 2)
			break;

	}
	spin_unlock(&n->list_lock);
	return object;
}
```

acquire_slab 会从 kmem_cache_node 的 partial 链表中拿下一大块内存来，并且将 freelist，也就是第一块空闲的缓存块，赋值给 t。并且当第一轮循环的时候，将 kmem_cache_cpu 的 page 指向取下来的这一大块内存，返回的 object 就是这块内存里面的第一个缓存块 t. 如果 kmem_cache_cpu 也有一个 partial，就会进行第二轮，再次取下一大块内存来，这次调用 put_cpu_partial，放到 kmem_cache_cpu 的 partial 里面.

如果 kmem_cache_node 里面也没有空闲的内存，这就说明原来分配的页里面都放满了，就要回到 new_slab_objects 函数，里面 [new_slab](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L1746) 函数会调用 [allocate_slab](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L1666).

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/slub.c#L1666
static struct page *allocate_slab(struct kmem_cache *s, gfp_t flags, int node)
{
	struct page *page;
	struct kmem_cache_order_objects oo = s->oo;
	gfp_t alloc_gfp;
	void *start, *p, *next;
	int idx;
	bool shuffle;

	flags &= gfp_allowed_mask;

	if (gfpflags_allow_blocking(flags))
		local_irq_enable();

	flags |= s->allocflags;

	/*
	 * Let the initial higher-order allocation fail under memory pressure
	 * so we fall-back to the minimum order allocation.
	 */
	alloc_gfp = (flags | __GFP_NOWARN | __GFP_NORETRY) & ~__GFP_NOFAIL;
	if ((alloc_gfp & __GFP_DIRECT_RECLAIM) && oo_order(oo) > oo_order(s->min))
		alloc_gfp = (alloc_gfp | __GFP_NOMEMALLOC) & ~(__GFP_RECLAIM|__GFP_NOFAIL);

	page = alloc_slab_page(s, alloc_gfp, node, oo);
	if (unlikely(!page)) {
		oo = s->min;
		alloc_gfp = flags;
		/*
		 * Allocation may have failed due to fragmentation.
		 * Try a lower order alloc if possible
		 */
		page = alloc_slab_page(s, alloc_gfp, node, oo);
		if (unlikely(!page))
			goto out;
		stat(s, ORDER_FALLBACK);
	}

	page->objects = oo_objects(oo);

	page->slab_cache = s;
	__SetPageSlab(page);
	if (page_is_pfmemalloc(page))
		SetPageSlabPfmemalloc(page);

	kasan_poison_slab(page);

	start = page_address(page);

	setup_page_debug(s, page, start);

	shuffle = shuffle_freelist(s, page);

	if (!shuffle) {
		start = fixup_red_left(s, start);
		start = setup_object(s, page, start);
		page->freelist = start;
		for (idx = 0, p = start; idx < page->objects - 1; idx++) {
			next = p + s->size;
			next = setup_object(s, page, next);
			set_freepointer(s, p, next);
			p = next;
		}
		set_freepointer(s, p, NULL);
	}

	page->inuse = page->objects;
	page->frozen = 1;

out:
	if (gfpflags_allow_blocking(flags))
		local_irq_disable();
	if (!page)
		return NULL;

	inc_slabs_node(s, page_to_nid(page), page->objects);

	return page;
}
```

在这里，我们看到了 alloc_slab_page 分配页面. 分配的时候，要按 kmem_cache_order_objects 里面的 order 来. 如果第一次分配不成功，说明内存已经很紧张了，那就换成 min 版本的 kmem_cache_order_objects.

#### 页面换出
另一个物理内存管理必须要处理的事情就是，页面换出. 每个进程都有自己的虚拟地址空间，无论是 32 位还是 64 位，虚拟地址空间都非常大，物理内存不可能有这么多的空间放得下. 所以，一般情况下，页面只有在被使用的时候，才会放在物理内存中。如果过了一段时间不被使用，即便用户进程并没有释放它，物理内存管理也有责任做一定的干预. 例如，将这些物理内存中的页面换出到硬盘上去；将空出的物理内存，交给活跃的进程去使用.

什么情况下会触发页面换出呢?

可以想象，最常见的情况就是，分配内存的时候，发现没有地方了，就试图回收一下. 例如，咱们解析申请一个页面的时候，会调用 get_page_from_freelist，接下来的调用链为 get_page_from_freelist->node_reclaim->__node_reclaim->shrink_node，通过这个调用链可以看出，页面换出也是以内存节点为单位的.

当然还有一种情况，就是作为内存管理系统应该主动去做的，而不能等真的出了事儿再做，这就是内核线程 [kswapd](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmscan.c#L3865). 这个内核线程，在系统初始化的时候就被创建. 这样它会进入一个无限循环，直到系统停止. 在这个循环中，如果内存使用没有那么紧张，那它就可以放心睡大觉；如果内存紧张了，就需要去检查一下内存，看看是否需要换出一些内存页.

kswapd的调用链是 [balance_pgdat](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmscan.c#L3545)->[kswapd_shrink_node](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmscan.c#L3497)->[shrink_node](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmscan.c#L2669)，是以内存节点为单位的. shrink_node 会调用 [shrink_node_memcgs](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmscan.c#L2611) -> [shrink_lruvec](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmscan.c#L2426). shrink_lruvec里面有一个循环处理页面的列表，看这个函数的注释，其实和上面我们想表达的内存换出是一样的.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmscan.c#L2426
static void shrink_lruvec(struct lruvec *lruvec, struct scan_control *sc)
{
	unsigned long nr[NR_LRU_LISTS];
	unsigned long targets[NR_LRU_LISTS];
	unsigned long nr_to_scan;
	enum lru_list lru;
	unsigned long nr_reclaimed = 0;
	unsigned long nr_to_reclaim = sc->nr_to_reclaim;
	struct blk_plug plug;
	bool scan_adjusted;

	get_scan_count(lruvec, sc, nr);

	/* Record the original scan target for proportional adjustments later */
	memcpy(targets, nr, sizeof(nr));

	/*
	 * Global reclaiming within direct reclaim at DEF_PRIORITY is a normal
	 * event that can occur when there is little memory pressure e.g.
	 * multiple streaming readers/writers. Hence, we do not abort scanning
	 * when the requested number of pages are reclaimed when scanning at
	 * DEF_PRIORITY on the assumption that the fact we are direct
	 * reclaiming implies that kswapd is not keeping up and it is best to
	 * do a batch of work at once. For memcg reclaim one check is made to
	 * abort proportional reclaim if either the file or anon lru has already
	 * dropped to zero at the first pass.
	 */
	scan_adjusted = (!cgroup_reclaim(sc) && !current_is_kswapd() &&
			 sc->priority == DEF_PRIORITY);

	blk_start_plug(&plug);
	while (nr[LRU_INACTIVE_ANON] || nr[LRU_ACTIVE_FILE] ||
					nr[LRU_INACTIVE_FILE]) {
		unsigned long nr_anon, nr_file, percentage;
		unsigned long nr_scanned;

		for_each_evictable_lru(lru) {
			if (nr[lru]) {
				nr_to_scan = min(nr[lru], SWAP_CLUSTER_MAX);
				nr[lru] -= nr_to_scan;

				nr_reclaimed += shrink_list(lru, nr_to_scan,
							    lruvec, sc);
			}
		}

		cond_resched();

		if (nr_reclaimed < nr_to_reclaim || scan_adjusted)
			continue;

		/*
		 * For kswapd and memcg, reclaim at least the number of pages
		 * requested. Ensure that the anon and file LRUs are scanned
		 * proportionally what was requested by get_scan_count(). We
		 * stop reclaiming one LRU and reduce the amount scanning
		 * proportional to the original scan target.
		 */
		nr_file = nr[LRU_INACTIVE_FILE] + nr[LRU_ACTIVE_FILE];
		nr_anon = nr[LRU_INACTIVE_ANON] + nr[LRU_ACTIVE_ANON];

		/*
		 * It's just vindictive to attack the larger once the smaller
		 * has gone to zero.  And given the way we stop scanning the
		 * smaller below, this makes sure that we only make one nudge
		 * towards proportionality once we've got nr_to_reclaim.
		 */
		if (!nr_file || !nr_anon)
			break;

		if (nr_file > nr_anon) {
			unsigned long scan_target = targets[LRU_INACTIVE_ANON] +
						targets[LRU_ACTIVE_ANON] + 1;
			lru = LRU_BASE;
			percentage = nr_anon * 100 / scan_target;
		} else {
			unsigned long scan_target = targets[LRU_INACTIVE_FILE] +
						targets[LRU_ACTIVE_FILE] + 1;
			lru = LRU_FILE;
			percentage = nr_file * 100 / scan_target;
		}

		/* Stop scanning the smaller of the LRU */
		nr[lru] = 0;
		nr[lru + LRU_ACTIVE] = 0;

		/*
		 * Recalculate the other LRU scan count based on its original
		 * scan target and the percentage scanning already complete
		 */
		lru = (lru == LRU_FILE) ? LRU_BASE : LRU_FILE;
		nr_scanned = targets[lru] - nr[lru];
		nr[lru] = targets[lru] * (100 - percentage) / 100;
		nr[lru] -= min(nr[lru], nr_scanned);

		lru += LRU_ACTIVE;
		nr_scanned = targets[lru] - nr[lru];
		nr[lru] = targets[lru] * (100 - percentage) / 100;
		nr[lru] -= min(nr[lru], nr_scanned);

		scan_adjusted = true;
	}
	blk_finish_plug(&plug);
	sc->nr_reclaimed += nr_reclaimed;

	/*
	 * Even if we did not try to evict anon pages at all, we want to
	 * rebalance the anon lru active/inactive ratio.
	 */
	if (total_swap_pages && inactive_is_low(lruvec, LRU_INACTIVE_ANON))
		shrink_active_list(SWAP_CLUSTER_MAX, lruvec,
				   sc, LRU_ACTIVE_ANON);
}
```

这里面有个 lru 列表. 从下面的定义，我们可以想象，所有的页面都被挂在 LRU 列表中. LRU 是 Least Recent Use，也就是最近最少使用. 也就是说，这个列表里面会按照活跃程度进行排序，这样就容易把不怎么用的内存页拿出来做处理.

内存页总共分两类，一类是匿名页，和虚拟地址空间进行关联；一类是内存映射，不但和虚拟地址空间关联，还和文件管理关联.

它们每一类都有两个列表，一个是 active，一个是 inactive. 顾名思义，active 就是比较活跃的，inactive 就是不怎么活跃的. 这两个里面的页会变化，过一段时间，活跃的可能变为不活跃，不活跃的可能变为活跃. 如果要换出内存，那就是从不活跃的列表中找出最不活跃的，换出到硬盘上.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/mmzone.h#L222
/*
 * We do arithmetic on the LRU lists in various places in the code,
 * so it is important to keep the active lists LRU_ACTIVE higher in
 * the array than the corresponding inactive lists, and to keep
 * the *_FILE lists LRU_FILE higher than the corresponding _ANON lists.
 *
 * This has to be kept in sync with the statistics in zone_stat_item
 * above and the descriptions in vmstat_text in mm/vmstat.c
 */
#define LRU_BASE 0
#define LRU_ACTIVE 1
#define LRU_FILE 2

enum lru_list {
	LRU_INACTIVE_ANON = LRU_BASE,
	LRU_ACTIVE_ANON = LRU_BASE + LRU_ACTIVE,
	LRU_INACTIVE_FILE = LRU_BASE + LRU_FILE,
	LRU_ACTIVE_FILE = LRU_BASE + LRU_FILE + LRU_ACTIVE,
	LRU_UNEVICTABLE,
	NR_LRU_LISTS
};

// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmscan.c#L2426
static unsigned long shrink_list(enum lru_list lru, unsigned long nr_to_scan,
				 struct lruvec *lruvec, struct scan_control *sc)
{
	if (is_active_lru(lru)) {
		if (sc->may_deactivate & (1 << is_file_lru(lru)))
			shrink_active_list(nr_to_scan, lruvec, sc, lru);
		else
			sc->skipped_deactivate = 1;
		return 0;
	}

	return shrink_inactive_list(nr_to_scan, lruvec, sc, lru);
}
```

从上面的代码可以看出，shrink_list 会先缩减活跃页面列表，再压缩不活跃的页面列表. 对于不活跃列表的缩减，shrink_inactive_list 就需要对页面进行回收；对于匿名页来讲，需要分配 swap，将内存页写入文件系统；对于内存映射关联了文件的，我们需要将在内存中对于文件的修改写回到文件中.


##### mmap 的原理
每一个进程都有一个列表 vm_area_struct，指向虚拟地址空间的不同的内存块，这个变量的名字叫 mmap.
```c
// https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L382
struct mm_struct {
	struct {
		struct vm_area_struct *mmap;		/* list of VMAs */
	...
	}
	...
}
```

其实内存映射不仅仅是物理内存和虚拟内存之间的映射，还包括将文件中的内容映射到虚拟内存空间. 这个时候，访问内存空间就能够访问到文件里面的数据. 而仅有物理内存和虚拟内存的映射，是一种特殊情况.

要申请小块内存，就用 brk. 如果申请一大块内存，就要用 mmap. 对于堆的申请来讲，mmap 是映射内存空间到物理内存. 另外，如果一个进程想映射一个文件到自己的虚拟内存空间，也要通过 mmap 系统调用. 这个时候 mmap 是映射内存空间到物理内存再到文件.


```c
// https://elixir.bootlin.com/linux/latest/source/mm/mmap.c#L1602
SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, pgoff)
{
	return ksys_mmap_pgoff(addr, len, prot, flags, fd, pgoff);
}

// https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/sys_x86_64.c#L89
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, off)
{
	long error;
	error = -EINVAL;
	if (off & ~PAGE_MASK)
		goto out;

	error = ksys_mmap_pgoff(addr, len, prot, flags, fd, off >> PAGE_SHIFT);
out:
	return error;
}

// https://elixir.bootlin.com/linux/latest/source/mm/mmap.c#L1553
unsigned long ksys_mmap_pgoff(unsigned long addr, unsigned long len,
			      unsigned long prot, unsigned long flags,
			      unsigned long fd, unsigned long pgoff)
{
	struct file *file = NULL;
	unsigned long retval;

	if (!(flags & MAP_ANONYMOUS)) {
		audit_mmap_fd(fd, flags);
		file = fget(fd);
		if (!file)
			return -EBADF;
		if (is_file_hugepages(file))
			len = ALIGN(len, huge_page_size(hstate_file(file)));
		retval = -EINVAL;
		if (unlikely(flags & MAP_HUGETLB && !is_file_hugepages(file)))
			goto out_fput;
	} else if (flags & MAP_HUGETLB) {
		struct user_struct *user = NULL;
		struct hstate *hs;

		hs = hstate_sizelog((flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
		if (!hs)
			return -EINVAL;

		len = ALIGN(len, huge_page_size(hs));
		/*
		 * VM_NORESERVE is used because the reservations will be
		 * taken when vm_ops->mmap() is called
		 * A dummy user value is used because we are not locking
		 * memory so no accounting is necessary
		 */
		file = hugetlb_file_setup(HUGETLB_ANON_FILE, len,
				VM_NORESERVE,
				&user, HUGETLB_ANONHUGE_INODE,
				(flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
		if (IS_ERR(file))
			return PTR_ERR(file);
	}

	flags &= ~(MAP_EXECUTABLE | MAP_DENYWRITE);

	retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
out_fput:
	if (file)
		fput(file);
	return retval;
}
```

如果要映射到文件，fd 会传进来一个文件描述符，并且 mmap_pgoff 里面通过 fget 函数，根据文件描述符获得 struct file。struct file 表示打开的一个文件. 接下来的调用链是 [vm_mmap_pgoff](https://elixir.bootlin.com/linux/latest/source/mm/util.c#L493)->[do_mmap_pgoff](https://elixir.bootlin.com/linux/latest/source/include/linux/mm.h#L2560)->[do_mmap](https://elixir.bootlin.com/linux/latest/source/mm/mmap.c#L1366). 这里面主要干了两件事情：调用 [get_unmapped_area](https://elixir.bootlin.com/linux/latest/source/mm/mmap.c#L2190) 找到一个没有映射的区域；调用 [mmap_region](https://elixir.bootlin.com/linux/latest/source/mm/mmap.c#L1687) 映射这个区域. 先来看 get_unmapped_area 函数:
```c
// https://elixir.bootlin.com/linux/latest/source/mm/mmap.c#L2190
unsigned long
get_unmapped_area(struct file *file, unsigned long addr, unsigned long len,
		unsigned long pgoff, unsigned long flags)
{
	unsigned long (*get_area)(struct file *, unsigned long,
				  unsigned long, unsigned long, unsigned long);

	unsigned long error = arch_mmap_check(addr, len, flags);
	if (error)
		return error;

	/* Careful about overflows.. */
	if (len > TASK_SIZE)
		return -ENOMEM;

	get_area = current->mm->get_unmapped_area;
	if (file) {
		if (file->f_op->get_unmapped_area)
			get_area = file->f_op->get_unmapped_area;
	} else if (flags & MAP_SHARED) {
		/*
		 * mmap_region() will call shmem_zero_setup() to create a file,
		 * so use shmem's get_unmapped_area in case it can be huge.
		 * do_mmap_pgoff() will clear pgoff, so match alignment.
		 */
		pgoff = 0;
		get_area = shmem_get_unmapped_area;
	}

	addr = get_area(file, addr, len, pgoff, flags);
	if (IS_ERR_VALUE(addr))
		return addr;

	if (addr > TASK_SIZE - len)
		return -ENOMEM;
	if (offset_in_page(addr))
		return -EINVAL;

	error = security_mmap_addr(addr);
	return error ? error : addr;
}
```

这里面如果是匿名映射，则调用 mm_struct 里面的 get_unmapped_area 函数. 这个函数其实是 arch_get_unmapped_area. 它会调用 find_vma_prev，在表示虚拟内存区域的 vm_area_struct 红黑树上找到相应的位置. 之所以叫 prev，是说这个时候虚拟内存区域还没有建立，找到前一个 vm_area_struct.

如果不是匿名映射，而是映射到一个文件，这样在 Linux 里面，每个打开的文件都有一个 struct file 结构，里面有一个 file_operations，用来表示和这个文件相关的操作. 如果是我们熟知的 ext4 文件系统，调用的是 thp_get_unmapped_area. 如果我们仔细看这个函数，最终还是调用 mm_struct 里面的 get_unmapped_area 函数. 殊途同归.

```c
// https://elixir.bootlin.com/linux/latest/source/fs/ext4/file.c#L884
const struct file_operations ext4_file_operations = {
	...
	.mmap		= ext4_file_mmap,
	.get_unmapped_area = thp_get_unmapped_area,
	...
};

// https://elixir.bootlin.com/linux/latest/source/mm/huge_memory.c#L569
unsigned long thp_get_unmapped_area(struct file *filp, unsigned long addr,
		unsigned long len, unsigned long pgoff, unsigned long flags)
{
	unsigned long ret;
	loff_t off = (loff_t)pgoff << PAGE_SHIFT;

	if (!IS_DAX(filp->f_mapping->host) || !IS_ENABLED(CONFIG_FS_DAX_PMD))
		goto out;

	ret = __thp_get_unmapped_area(filp, addr, len, off, flags, PMD_SIZE);
	if (ret)
		return ret;
out:
	return current->mm->get_unmapped_area(filp, addr, len, pgoff, flags);
}
EXPORT_SYMBOL_GPL(thp_get_unmapped_area);

// https://elixir.bootlin.com/linux/latest/source/mm/huge_memory.c#L533
static unsigned long __thp_get_unmapped_area(struct file *filp,
		unsigned long addr, unsigned long len,
		loff_t off, unsigned long flags, unsigned long size)
{
	loff_t off_end = off + len;
	loff_t off_align = round_up(off, size);
	unsigned long len_pad, ret;

	if (off_end <= off_align || (off_end - off_align) < size)
		return 0;

	len_pad = len + size;
	if (len_pad < len || (off + len_pad) < off)
		return 0;

	ret = current->mm->get_unmapped_area(filp, addr, len_pad,
					      off >> PAGE_SHIFT, flags);

	/*
	 * The failure might be due to length padding. The caller will retry
	 * without the padding.
	 */
	if (IS_ERR_VALUE(ret))
		return 0;

	/*
	 * Do not try to align to THP boundary if allocation at the address
	 * hint succeeds.
	 */
	if (ret == addr)
		return addr;

	ret += (off - ret) & (size - 1);
	return ret;
}
```

再来看 mmap_region，看它如何映射这个虚拟内存区域:

```c
// https://elixir.bootlin.com/linux/latest/source/mm/mmap.c#L1687
unsigned long mmap_region(struct file *file, unsigned long addr,
		unsigned long len, vm_flags_t vm_flags, unsigned long pgoff,
		struct list_head *uf)
{
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *vma, *prev;
	int error;
	struct rb_node **rb_link, *rb_parent;
	unsigned long charged = 0;

	/* Check against address space limit. */
	if (!may_expand_vm(mm, vm_flags, len >> PAGE_SHIFT)) {
		unsigned long nr_pages;

		/*
		 * MAP_FIXED may remove pages of mappings that intersects with
		 * requested mapping. Account for the pages it would unmap.
		 */
		nr_pages = count_vma_pages_range(mm, addr, addr + len);

		if (!may_expand_vm(mm, vm_flags,
					(len >> PAGE_SHIFT) - nr_pages))
			return -ENOMEM;
	}

	/* Clear old maps */
	while (find_vma_links(mm, addr, addr + len, &prev, &rb_link,
			      &rb_parent)) {
		if (do_munmap(mm, addr, len, uf))
			return -ENOMEM;
	}

	/*
	 * Private writable mapping: check memory availability
	 */
	if (accountable_mapping(file, vm_flags)) {
		charged = len >> PAGE_SHIFT;
		if (security_vm_enough_memory_mm(mm, charged))
			return -ENOMEM;
		vm_flags |= VM_ACCOUNT;
	}

	/*
	 * Can we just expand an old mapping?
	 */
	vma = vma_merge(mm, prev, addr, addr + len, vm_flags,
			NULL, file, pgoff, NULL, NULL_VM_UFFD_CTX);
	if (vma)
		goto out;

	/*
	 * Determine the object being mapped and call the appropriate
	 * specific mapper. the address has already been validated, but
	 * not unmapped, but the maps are removed from the list.
	 */
	vma = vm_area_alloc(mm);
	if (!vma) {
		error = -ENOMEM;
		goto unacct_error;
	}

	vma->vm_start = addr;
	vma->vm_end = addr + len;
	vma->vm_flags = vm_flags;
	vma->vm_page_prot = vm_get_page_prot(vm_flags);
	vma->vm_pgoff = pgoff;

	if (file) {
		if (vm_flags & VM_DENYWRITE) {
			error = deny_write_access(file);
			if (error)
				goto free_vma;
		}
		if (vm_flags & VM_SHARED) {
			error = mapping_map_writable(file->f_mapping);
			if (error)
				goto allow_write_and_free_vma;
		}

		/* ->mmap() can change vma->vm_file, but must guarantee that
		 * vma_link() below can deny write-access if VM_DENYWRITE is set
		 * and map writably if VM_SHARED is set. This usually means the
		 * new file must not have been exposed to user-space, yet.
		 */
		vma->vm_file = get_file(file);
		error = call_mmap(file, vma);
		if (error)
			goto unmap_and_free_vma;

		/* Can addr have changed??
		 *
		 * Answer: Yes, several device drivers can do it in their
		 *         f_op->mmap method. -DaveM
		 * Bug: If addr is changed, prev, rb_link, rb_parent should
		 *      be updated for vma_link()
		 */
		WARN_ON_ONCE(addr != vma->vm_start);

		addr = vma->vm_start;
		vm_flags = vma->vm_flags;
	} else if (vm_flags & VM_SHARED) {
		error = shmem_zero_setup(vma);
		if (error)
			goto free_vma;
	} else {
		vma_set_anonymous(vma);
	}

	vma_link(mm, vma, prev, rb_link, rb_parent);
	/* Once vma denies write, undo our temporary denial count */
	if (file) {
		if (vm_flags & VM_SHARED)
			mapping_unmap_writable(file->f_mapping);
		if (vm_flags & VM_DENYWRITE)
			allow_write_access(file);
	}
	file = vma->vm_file;
out:
	perf_event_mmap(vma);

	vm_stat_account(mm, vm_flags, len >> PAGE_SHIFT);
	if (vm_flags & VM_LOCKED) {
		if ((vm_flags & VM_SPECIAL) || vma_is_dax(vma) ||
					is_vm_hugetlb_page(vma) ||
					vma == get_gate_vma(current->mm))
			vma->vm_flags &= VM_LOCKED_CLEAR_MASK;
		else
			mm->locked_vm += (len >> PAGE_SHIFT);
	}

	if (file)
		uprobe_mmap(vma);

	/*
	 * New (or expanded) vma always get soft dirty status.
	 * Otherwise user-space soft-dirty page tracker won't
	 * be able to distinguish situation when vma area unmapped,
	 * then new mapped in-place (which must be aimed as
	 * a completely new data area).
	 */
	vma->vm_flags |= VM_SOFTDIRTY;

	vma_set_page_prot(vma);

	return addr;

unmap_and_free_vma:
	vma->vm_file = NULL;
	fput(file);

	/* Undo any partial mapping done by a device driver. */
	unmap_region(mm, vma, prev, vma->vm_start, vma->vm_end);
	charged = 0;
	if (vm_flags & VM_SHARED)
		mapping_unmap_writable(file->f_mapping);
allow_write_and_free_vma:
	if (vm_flags & VM_DENYWRITE)
		allow_write_access(file);
free_vma:
	vm_area_free(vma);
unacct_error:
	if (charged)
		vm_unacct_memory(charged);
	return error;
}
```

还记得咱们刚找到了虚拟内存区域的前一个 vm_area_struct，我们首先要看，是否能够基于它进行扩展，也即调用 vma_merge，和前一个 vm_area_struct 合并到一起.

如果不能，就需要调用 kmem_cache_zalloc，在 Slub 里面创建一个新的 vm_area_struct 对象，设置起始和结束位置，将它加入队列. 如果是映射到文件，则设置 vm_file 为目标文件，调用 call_mmap. 其实就是调用 file_operations 的 mmap 函数. 对于 ext4 文件系统，调用的是 ext4_file_mmap. 从这个函数的参数可以看出，这一刻文件和内存开始发生关系了. 这里我们将 vm_area_struct 的内存操作设置为文件系统操作，也就是说，读写内存其实就是读写文件系统.

再回到 mmap_region 函数. 最终，vma_link 函数将新创建的 vm_area_struct 挂在了 mm_struct 里面的红黑树上. 这个时候，从内存到文件的映射关系，至少要在逻辑层面建立起来. 那从文件到内存的映射关系呢？[vma_link](https://elixir.bootlin.com/linux/latest/source/mm/mmap.c#L643) 还做了另外一件事情，就是 [__vma_link_file](https://elixir.bootlin.com/linux/latest/source/mm/mmap.c#L615). 这个函数就是用于建立这层映射关系. 对于打开的文件，会有一个结构 [struct file](https://elixir.bootlin.com/linux/latest/source/include/linux/fs.h#L941) 来表示. 它有个成员指向 [struct address_space](https://elixir.bootlin.com/linux/latest/source/include/linux/fs.h#L445) 结构，这里面有棵变量名为 i_mmap 的红黑树，vm_area_struct 就挂在这棵树上.

到这里，内存映射的内容要告一段落了. 到目前为止，我们还没有开始真正访问内存呀！这个时候，内存管理并不直接分配物理内存，因为物理内存相对于虚拟地址空间太宝贵了，只有等你真正用的那一刻才会开始分配.

##### 用户态缺页异常
一旦开始访问虚拟内存的某个地址，如果我们发现，并没有对应的物理页，那就触发缺页中断，调用 handle_page_fault:
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/mm/fault.c#L1104
static __always_inline void
handle_page_fault(struct pt_regs *regs, unsigned long error_code,
			      unsigned long address)
{
	trace_page_fault_entries(regs, error_code, address);

	if (unlikely(kmmio_fault(regs, address)))
		return;

	/* Was the fault on kernel-controlled part of the address space? */
	if (unlikely(fault_in_kernel_space(address))) {
		do_kern_addr_fault(regs, error_code, address);
	} else {
		do_user_addr_fault(regs, error_code, address);
		/*
		 * User address page fault handling might have reenabled
		 * interrupts. Fixing up all potential exit points of
		 * do_user_addr_fault() and its leaf functions is just not
		 * doable w/o creating an unholy mess or turning the code
		 * upside down.
		 */
		local_irq_disable();
	}
}
```

在 handle_page_fault 里面，先要判断缺页中断是否发生在内核. 如果发生在内核则调用 do_kern_addr_fault, 而用户空间的部分用[`do_user_addr_fault`](https://elixir.bootlin.com/linux/latest/source/arch/x86/mm/fault.c#L1305).

先看do_user_addr_fault，它会找到你访问的那个地址所在的区域 vm_area_struct，然后调用 [handle_mm_fault](https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L4354) 来映射这个区域.

handle_mm_fault还会调用[__handle_mm_fault](https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L4354).

到这里，终于看到了熟悉的 PGD、P4G、PUD、PMD、PTE，因为暂且不考虑五级页表，所以先暂时忽略 P4G.


pgd_t 用于全局页目录项，pud_t 用于上层页目录项，pmd_t 用于中间页目录项，pte_t 用于直接页表项. 每个进程都有独立的地址空间，为了这个进程独立完成映射，每个进程都有独立的进程页表，这个页表的最顶级的 pgd 存放在 task_struct 中的 mm_struct 的 pgd 变量里面. 在一个进程新创建的时候，会调用 fork，对于内存的部分会调用 copy_mm，里面调用 dup_mm.

```c
// https://elixir.bootlin.com/linux/latest/source/kernel/fork.c#L1348
/**
 * dup_mm() - duplicates an existing mm structure
 * @tsk: the task_struct with which the new mm will be associated.
 * @oldmm: the mm to duplicate.
 *
 * Allocates a new mm structure and duplicates the provided @oldmm structure
 * content into it.
 *
 * Return: the duplicated mm or NULL on failure.
 */
static struct mm_struct *dup_mm(struct task_struct *tsk,
				struct mm_struct *oldmm)
{
	struct mm_struct *mm;
	int err;

	mm = allocate_mm();
	if (!mm)
		goto fail_nomem;

	memcpy(mm, oldmm, sizeof(*mm));

	if (!mm_init(mm, tsk, mm->user_ns))
		goto fail_nomem;

	err = dup_mmap(mm, oldmm);
	if (err)
		goto free_pt;

	mm->hiwater_rss = get_mm_rss(mm);
	mm->hiwater_vm = mm->total_vm;

	if (mm->binfmt && !try_module_get(mm->binfmt->module))
		goto free_pt;

	return mm;

free_pt:
	/* don't put binfmt in mmput, we haven't got module yet */
	mm->binfmt = NULL;
	mm_init_owner(mm, NULL);
	mmput(mm);

fail_nomem:
	return NULL;
}
```

在这里，除了创建一个新的 mm_struct，并且通过 memcpy 将它和父进程的弄成一模一样之外，还需要调用 [mm_init](https://elixir.bootlin.com/linux/latest/source/kernel/fork.c#L1009) 进行初始化. 接下来，mm_init 调用 [mm_alloc_pgd](https://elixir.bootlin.com/linux/latest/source/kernel/fork.c#L635)，分配全局页目录项，赋值给 mm_struct 的 pgd 成员变量.

[pgd_alloc](https://elixir.bootlin.com/linux/latest/source/arch/x86/mm/pgtable.c#L417) 里面除了分配 PGD 之外，还做了很重要的一个事情，就是调用 [pgd_ctor](https://elixir.bootlin.com/linux/latest/source/arch/x86/mm/pgtable.c#L116).

pgd_ctor 拷贝了对于 swapper_pg_dir 的引用. swapper_pg_dir 是内核页表的最顶级的全局页目录.

一个进程的虚拟地址空间包含用户态和内核态两部分. 为了从虚拟地址空间映射到物理页面，页表也分为用户地址空间的页表和内核页表，这就和上面遇到的 vmalloc 有关系了. 在内核里面，映射靠内核页表，这里内核页表会拷贝一份到进程的页表.

至此，一个进程 fork 完毕之后，有了内核页表，有了自己顶级的 pgd，但是对于用户地址空间来讲，还完全没有映射过. 这需要等到这个进程在某个 CPU 上运行，并且对内存访问的那一刻了.

当这个进程被调度到某个 CPU 上运行的时候，要调用 context_switch 进行上下文切换. 对于内存方面的切换会调用 switch_mm_irqs_off，这里面会调用 load_new_mm_cr3.

cr3 是 CPU 的一个寄存器，它会指向当前进程的顶级 pgd. 如果 CPU 的指令要访问进程的虚拟内存，它就会自动从 cr3 里面得到 pgd 在物理内存的地址，然后根据里面的页表解析虚拟内存的地址为物理内存，从而访问真正的物理内存上的数据.

这里需要注意两点:
1. cr3 里面存放当前进程的顶级 pgd，这个是硬件的要求. cr3 里面需要存放 pgd 在物理内存的地址，不能是虚拟地址. 因而 load_new_mm_cr3 里面会使用 __pa，将 mm_struct 里面的成员变量 pgd（mm_struct 里面存的都是虚拟地址）变为物理地址，才能加载到 cr3 里面去.

1. 用户进程在运行的过程中，访问虚拟内存中的数据，会被 cr3 里面指向的页表转换为物理地址后，才在物理内存中访问数据，这个过程都是在用户态运行的，地址转换的过程无需进入内核态.

只有访问虚拟内存的时候，发现没有映射到物理内存，页表也没有创建过，才触发缺页异常. 进入内核调用 handle_page_fault，一直调用到 __handle_mm_fault. 既然原来没有创建过页表，那只好补上这一课. 于是，__handle_mm_fault 调用 pud_alloc 和 pmd_alloc，来创建相应的页目录项，最后调用 handle_pte_fault 来创建页表项.

绕了一大圈，终于将页表整个机制的各个部分串了起来. 但物理的内存还没找到, 还得接着分析 [handle_pte_fault](https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L4171) 的实现.

```c
// https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L4171
/*
 * These routines also need to handle stuff like marking pages dirty
 * and/or accessed for architectures that don't do it in hardware (most
 * RISC architectures).  The early dirtying is also good on the i386.
 *
 * There is also a hook called "update_mmu_cache()" that architectures
 * with external mmu caches can use to update those (ie the Sparc or
 * PowerPC hashed page tables that act as extended TLBs).
 *
 * We enter with non-exclusive mmap_sem (to exclude vma changes, but allow
 * concurrent faults).
 *
 * The mmap_sem may have been released depending on flags and our return value.
 * See filemap_fault() and __lock_page_or_retry().
 */
static vm_fault_t handle_pte_fault(struct vm_fault *vmf)
{
	pte_t entry;

	if (unlikely(pmd_none(*vmf->pmd))) {
		/*
		 * Leave __pte_alloc() until later: because vm_ops->fault may
		 * want to allocate huge page, and if we expose page table
		 * for an instant, it will be difficult to retract from
		 * concurrent faults and from rmap lookups.
		 */
		vmf->pte = NULL;
	} else {
		/* See comment in pte_alloc_one_map() */
		if (pmd_devmap_trans_unstable(vmf->pmd))
			return 0;
		/*
		 * A regular pmd is established and it can't morph into a huge
		 * pmd from under us anymore at this point because we hold the
		 * mmap_sem read mode and khugepaged takes it in write mode.
		 * So now it's safe to run pte_offset_map().
		 */
		vmf->pte = pte_offset_map(vmf->pmd, vmf->address);
		vmf->orig_pte = *vmf->pte;

		/*
		 * some architectures can have larger ptes than wordsize,
		 * e.g.ppc44x-defconfig has CONFIG_PTE_64BIT=y and
		 * CONFIG_32BIT=y, so READ_ONCE cannot guarantee atomic
		 * accesses.  The code below just needs a consistent view
		 * for the ifs and we later double check anyway with the
		 * ptl lock held. So here a barrier will do.
		 */
		barrier();
		if (pte_none(vmf->orig_pte)) {
			pte_unmap(vmf->pte);
			vmf->pte = NULL;
		}
	}

	if (!vmf->pte) {
		if (vma_is_anonymous(vmf->vma))
			return do_anonymous_page(vmf);
		else
			return do_fault(vmf);
	}

	if (!pte_present(vmf->orig_pte))
		return do_swap_page(vmf);

	if (pte_protnone(vmf->orig_pte) && vma_is_accessible(vmf->vma))
		return do_numa_page(vmf);

	vmf->ptl = pte_lockptr(vmf->vma->vm_mm, vmf->pmd);
	spin_lock(vmf->ptl);
	entry = vmf->orig_pte;
	if (unlikely(!pte_same(*vmf->pte, entry)))
		goto unlock;
	if (vmf->flags & FAULT_FLAG_WRITE) {
		if (!pte_write(entry))
			return do_wp_page(vmf);
		entry = pte_mkdirty(entry);
	}
	entry = pte_mkyoung(entry);
	if (ptep_set_access_flags(vmf->vma, vmf->address, vmf->pte, entry,
				vmf->flags & FAULT_FLAG_WRITE)) {
		update_mmu_cache(vmf->vma, vmf->address, vmf->pte);
	} else {
		/*
		 * This is needed only for protection faults but the arch code
		 * is not yet telling us if this is a protection fault or not.
		 * This still avoids useless tlb flushes for .text page faults
		 * with threads.
		 */
		if (vmf->flags & FAULT_FLAG_WRITE)
			flush_tlb_fix_spurious_fault(vmf->vma, vmf->address);
	}
unlock:
	pte_unmap_unlock(vmf->pte, vmf->ptl);
	return 0;
}
```

这里面总的来说分了三种情况. 如果 PTE，也就是页表项，从来没有出现过，那就是新映射的页. 如果是匿名页，就是第一种情况，应该映射到一个物理内存页，在这里调用的是 do_anonymous_page. 如果是映射到文件，调用的就是 do_fault，这是第二种情况. 如果 PTE 原来出现过，说明原来页面在物理内存中，后来换出到硬盘了，现在应该换回来，调用的是 do_swap_page.

先看第一种情况，[do_anonymous_page](https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L3308). 对于匿名页的映射，需要先通过 pte_alloc 分配一个页表项，然后通过 [alloc_zeroed_user_highpage_movable](https://elixir.bootlin.com/linux/latest/source/include/linux/highmem.h#L205) 分配一个页. 之后它会调用 alloc_pages_vma，并最终调用 __alloc_pages_nodemask. 它是伙伴系统的核心函数，专门用来分配物理页面的. do_anonymous_page 接下来要调用 [mk_pte](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/pgtable.h#L857)，将页表项指向新分配的物理页，set_pte_at 会将页表项塞到页表里面.

第二种情况映射到文件 [do_fault](https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L3939), 最终会调用 [__do_fault](https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L3423).


它调用了 struct vm_operations_struct vm_ops 的 fault 函数. 对于 ext4 文件系统，vm_ops 指向了 ext4_file_vm_ops，也就是调用了 ext4_filemap_fault.
```c
// https://elixir.bootlin.com/linux/latest/source/fs/ext4/file.c#L733
static const struct vm_operations_struct ext4_file_vm_ops = {
	.fault		= ext4_filemap_fault,
	.map_pages	= filemap_map_pages,
	.page_mkwrite   = ext4_page_mkwrite,
};
```

[ext4_filemap_fault](https://elixir.bootlin.com/linux/latest/source/fs/ext4/inode.c#L6041) 里面的逻辑我们很容易就能读懂. vm_file 就是当时 mmap 的时候映射的那个文件，然后需要调用 [filemap_fault](https://elixir.bootlin.com/linux/latest/source/mm/filemap.c#L2461). 对于文件映射来说，一般这个文件会在物理内存里面有页面作为它的缓存，[find_get_page](https://elixir.bootlin.com/linux/latest/source/include/linux/pagemap.h#L255) 就是找那个页. 如果找到了，就调用 [do_async_mmap_readahead](https://elixir.bootlin.com/linux/latest/source/mm/filemap.c#L2416)，预读一些数据到内存里面；如果没有，就调用 [do_sync_mmap_readahead](https://elixir.bootlin.com/linux/latest/source/mm/filemap.c#L2368), 将文件内容读到内存中.

```c
// https://elixir.bootlin.com/linux/latest/source/mm/filemap.c#L2461
/**
 * filemap_fault - read in file data for page fault handling
 * @vmf:	struct vm_fault containing details of the fault
 *
 * filemap_fault() is invoked via the vma operations vector for a
 * mapped memory region to read in file data during a page fault.
 *
 * The goto's are kind of ugly, but this streamlines the normal case of having
 * it in the page cache, and handles the special cases reasonably without
 * having a lot of duplicated code.
 *
 * vma->vm_mm->mmap_sem must be held on entry.
 *
 * If our return value has VM_FAULT_RETRY set, it's because the mmap_sem
 * may be dropped before doing I/O or by lock_page_maybe_drop_mmap().
 *
 * If our return value does not have VM_FAULT_RETRY set, the mmap_sem
 * has not been released.
 *
 * We never return with VM_FAULT_RETRY and a bit from VM_FAULT_ERROR set.
 *
 * Return: bitwise-OR of %VM_FAULT_ codes.
 */
vm_fault_t filemap_fault(struct vm_fault *vmf)
{
	int error;
	struct file *file = vmf->vma->vm_file;
	struct file *fpin = NULL;
	struct address_space *mapping = file->f_mapping;
	struct file_ra_state *ra = &file->f_ra;
	struct inode *inode = mapping->host;
	pgoff_t offset = vmf->pgoff;
	pgoff_t max_off;
	struct page *page;
	vm_fault_t ret = 0;

	max_off = DIV_ROUND_UP(i_size_read(inode), PAGE_SIZE);
	if (unlikely(offset >= max_off))
		return VM_FAULT_SIGBUS;

	/*
	 * Do we have something in the page cache already?
	 */
	page = find_get_page(mapping, offset);
	if (likely(page) && !(vmf->flags & FAULT_FLAG_TRIED)) {
		/*
		 * We found the page, so try async readahead before
		 * waiting for the lock.
		 */
		fpin = do_async_mmap_readahead(vmf, page);
	} else if (!page) {
		/* No page in the page cache at all */
		count_vm_event(PGMAJFAULT);
		count_memcg_event_mm(vmf->vma->vm_mm, PGMAJFAULT);
		ret = VM_FAULT_MAJOR;
		fpin = do_sync_mmap_readahead(vmf);
retry_find:
		page = pagecache_get_page(mapping, offset,
					  FGP_CREAT|FGP_FOR_MMAP,
					  vmf->gfp_mask);
		if (!page) {
			if (fpin)
				goto out_retry;
			return VM_FAULT_OOM;
		}
	}

	if (!lock_page_maybe_drop_mmap(vmf, page, &fpin))
		goto out_retry;

	/* Did it get truncated? */
	if (unlikely(compound_head(page)->mapping != mapping)) {
		unlock_page(page);
		put_page(page);
		goto retry_find;
	}
	VM_BUG_ON_PAGE(page_to_pgoff(page) != offset, page);

	/*
	 * We have a locked page in the page cache, now we need to check
	 * that it's up-to-date. If not, it is going to be due to an error.
	 */
	if (unlikely(!PageUptodate(page)))
		goto page_not_uptodate;

	/*
	 * We've made it this far and we had to drop our mmap_sem, now is the
	 * time to return to the upper layer and have it re-find the vma and
	 * redo the fault.
	 */
	if (fpin) {
		unlock_page(page);
		goto out_retry;
	}

	/*
	 * Found the page and have a reference on it.
	 * We must recheck i_size under page lock.
	 */
	max_off = DIV_ROUND_UP(i_size_read(inode), PAGE_SIZE);
	if (unlikely(offset >= max_off)) {
		unlock_page(page);
		put_page(page);
		return VM_FAULT_SIGBUS;
	}

	vmf->page = page;
	return ret | VM_FAULT_LOCKED;

page_not_uptodate:
	/*
	 * Umm, take care of errors if the page isn't up-to-date.
	 * Try to re-read it _once_. We do this synchronously,
	 * because there really aren't any performance issues here
	 * and we need to check for errors.
	 */
	ClearPageError(page);
	fpin = maybe_unlock_mmap_for_io(vmf, fpin);
	error = mapping->a_ops->readpage(file, page);
	if (!error) {
		wait_on_page_locked(page);
		if (!PageUptodate(page))
			error = -EIO;
	}
	if (fpin)
		goto out_retry;
	put_page(page);

	if (!error || error == AOP_TRUNCATED_PAGE)
		goto retry_find;

	/* Things didn't work out. Return zero to tell the mm layer so. */
	shrink_readahead_size_eio(ra);
	return VM_FAULT_SIGBUS;

out_retry:
	/*
	 * We dropped the mmap_sem, we need to return to the fault handler to
	 * re-find the vma and come back and find our hopefully still populated
	 * page.
	 */
	if (page)
		put_page(page);
	if (fpin)
		fput(fpin);
	return ret | VM_FAULT_RETRY;
}
EXPORT_SYMBOL(filemap_fault);
```

[do_sync_mmap_readahead](https://elixir.bootlin.com/linux/latest/source/mm/filemap.c#L2368) -> [page_cache_sync_readahead](https://elixir.bootlin.com/linux/latest/source/mm/readahead.c#L509) -> [force_page_cache_readahead](https://elixir.bootlin.com/linux/latest/source/mm/readahead.c#L222) -> [__do_page_cache_readahead](https://elixir.bootlin.com/linux/latest/source/mm/readahead.c#L155) -> [read_pages](https://elixir.bootlin.com/linux/latest/source/mm/readahead.c#L116)

struct address_space_operations 对于 ext4 文件系统的定义如下所示.
```c
// https://elixir.bootlin.com/linux/latest/source/fs/ext4/inode.c#L3606
static const struct address_space_operations ext4_aops = {
	.readpage		= ext4_readpage,
	.readpages		= ext4_readpages,
	.writepage		= ext4_writepage,
	.writepages		= ext4_writepages,
	.write_begin		= ext4_write_begin,
	.write_end		= ext4_write_end,
	.set_page_dirty		= ext4_set_page_dirty,
	.bmap			= ext4_bmap,
	.invalidatepage		= ext4_invalidatepage,
	.releasepage		= ext4_releasepage,
	.direct_IO		= noop_direct_IO,
	.migratepage		= buffer_migrate_page,
	.is_partially_uptodate  = block_is_partially_uptodate,
	.error_remove_page	= generic_error_remove_page,
};
```

这么说来，上面的 readpage 调用的其实是 ext4_readpage, 最后会调用 [ext4_read_inline_page](https://elixir.bootlin.com/linux/latest/source/fs/ext4/inline.c#L464)，这里面有部分逻辑和内存映射有关.

在 ext4_read_inline_page 函数里，需要先调用 kmap_atomic，将物理内存映射到内核的虚拟地址空间，得到内核中的地址 kaddr. kmap_atomic是用来做临时内核映射的. 本来把物理内存映射到用户虚拟地址空间，不需要在内核里面映射一把. 但是，现在因为要从文件里面读取数据并写入这个物理页面，又不能使用物理地址，我们只能使用虚拟地址，这就需要在内核里面临时映射一把. 临时映射后，ext4_read_inline_data 读取文件到这个虚拟地址. 读取完毕后，我们取消这个临时映射 kunmap_atomic 就行了.

再来看第三种情况，do_swap_page. 如果长时间不用，这部分数据就要换出到硬盘，也就是 swap，现在这部分数据又要访问了，就得想办法再次读到内存中来.

[do_swap_page](https://elixir.bootlin.com/linux/latest/source/mm/memory.c#L3089) 函数会先查找 swap 文件有没有缓存页. 如果没有，就调用 [swapin_readahead](https://elixir.bootlin.com/linux/latest/source/mm/swap_state.c#L781)，将 swap 文件读到内存中来，形成内存页，并通过 mk_pte 生成页表项. set_pte_at 将页表项插入页表，swap_free 将 swap 文件清理. 因为重新加载回内存了，不再需要 swap 文件了. swapin_readahead 会最终调用 swap_readpage，在这里，我们看到了熟悉的 readpage 函数，也就是说读取普通文件和读取 swap 文件，过程是一样的，同样需要用 kmap_atomic 做临时映射.

通过上面复杂的过程，用户态缺页异常处理完毕了. 物理内存中有了页面，页表也建立好了映射. 接下来，用户程序在虚拟内存空间里面，可以通过虚拟地址顺利经过页表映射的访问物理页面上的数据了. 为了加快映射速度，我们不需要每次从虚拟地址到物理地址的转换都走一遍页表.

页表一般都很大，只能存放在内存中. 操作系统每次访问内存都要折腾两步，先通过查询页表得到物理地址，然后访问该物理地址读取指令、数据. 为了提高映射速度，引入了 TLB（Translation Lookaside Buffer），就是快表，专门用来做地址映射的硬件设备. 它不在内存中，可存储的数据比较少，但是比内存要快. 所以，可以认为，TLB 就是页表的 Cache，其中存储了当前最可能被访问到的页表项，其内容是部分页表项的一个副本. 有了 TLB 之后，先查快表，快表中有映射关系，然后直接转换为物理地址. 如果在 TLB 查不到映射关系时，才会到内存中查询页表.

![](/misc/img/process/78d351d0105c8e5bf0e49c685a2c1a44.jpg)

##### 内核页表
和用户态页表不同，在系统初始化的时候，就要创建内核页表了. 从内核页表的根 [swapper_pg_dir](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/pgtable_64.h#L29) 开始找线索，在 arch/x86/include/asm/pgtable_64.h 中就能找到它的定义.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/include/asm/pgtable_64.h#L19
extern p4d_t level4_kernel_pgt[512];
extern p4d_t level4_ident_pgt[512];
extern pud_t level3_kernel_pgt[512];
extern pud_t level3_ident_pgt[512];
extern pmd_t level2_kernel_pgt[512];
extern pmd_t level2_fixmap_pgt[512];
extern pmd_t level2_ident_pgt[512];
extern pte_t level1_fixmap_pgt[512 * FIXMAP_PMD_NUM];
extern pgd_t init_top_pgt[];

#define swapper_pg_dir init_top_pgt
```

swapper_pg_dir 指向内核最顶级的目录 pgd，同时出现的还有几个页表目录. 64 位系统的虚拟地址空间的布局，其中 XXX_ident_pgt 对应的是直接映射区，XXX_kernel_pgt 对应的是内核代码区，XXX_fixmap_pgt 对应的是固定映射区. 它们是在汇编语言的文件里面的[arch\x86\kernel\head_64.S](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/kernel/head_64.S#L388)里初始化.

内核页表的顶级目录 init_top_pgt，定义在 __INITDATA(即.init 区域) 里面. 可以看到，页表的根其实是全局变量，这就使得初始化的时候，甚至内存管理还没有初始化的时候，很容易就可以定位到

接下来，定义 init_top_pgt 包含哪些项，不懂汇编的人可以简单地认为，quad 是声明了一项的内容，org 是跳到了某个位置.

所以，init_top_pgt 有三项，上来先有一项，指向的是 level3_ident_pgt，也即直接映射区页表的三级目录. 为什么要减去 __START_KERNEL_map 呢？因为 level3_ident_pgt 是定义在内核代码里的，写代码的时候，写的都是虚拟地址，谁写代码的时候也不知道将来加载的物理地址是多少呀，对不对？

因为 level3_ident_pgt 是在虚拟地址的内核代码段里的，而 __START_KERNEL_map 正是虚拟地址空间的内核代码段的起始地址. 这样，level3_ident_pgt 减去 __START_KERNEL_map 才是物理地址. 第一项定义完了以后，接下来我们跳到 PGD_PAGE_OFFSET 的位置，再定义一项. 从定义可以看出，这一项就应该是 __PAGE_OFFSET_BASE 对应的. __PAGE_OFFSET_BASE 是虚拟地址空间里面内核的起始地址. 第二项也指向 level3_ident_pgt，直接映射区.

第二项定义完了以后，接下来跳到 PGD_START_KERNEL 的位置，再定义一项. 从定义可以看出，这一项应该是 __START_KERNEL_map 对应的项，__START_KERNEL_map 是虚拟地址空间里面内核代码段的起始地址. 第三项指向 level3_kernel_pgt，内核代码区.

内核页表定义完了，一开始这里面的页表能够覆盖的内存范围比较小. 例如，内核代码区 512M，直接映射区 1G. 这个时候，其实只要能够映射基本的内核代码和数据结构就可以了. 可以看出，里面还空着很多项，可以用于将来映射巨大的内核虚拟地址空间，等用到的时候再进行映射.

如果是用户态进程页表，会有 mm_struct 指向进程顶级目录 pgd，对于内核来讲，也定义了一个 mm_struct，指向 swapper_pg_dir.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/init-mm.c#L29
/*
 * For dynamically allocated mm_structs, there is a dynamically sized cpumask
 * at the end of the structure, the size of which depends on the maximum CPU
 * number the system can see. That way we allocate only as much memory for
 * mm_cpumask() as needed for the hundreds, or thousands of processes that
 * a system typically runs.
 *
 * Since there is only one init_mm in the entire system, keep it simple
 * and size this cpu_bitmask to NR_CPUS.
 */
struct mm_struct init_mm = {
	.mm_rb		= RB_ROOT,
	.pgd		= swapper_pg_dir,
	.mm_users	= ATOMIC_INIT(2),
	.mm_count	= ATOMIC_INIT(1),
	MMAP_LOCK_INITIALIZER(init_mm)
	.page_table_lock =  __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),
	.arg_lock	=  __SPIN_LOCK_UNLOCKED(init_mm.arg_lock),
	.mmlist		= LIST_HEAD_INIT(init_mm.mmlist),
	.user_ns	= &init_user_ns,
	.cpu_bitmap	= CPU_BITS_NONE,
	INIT_MM_CONTEXT(init_mm)
};
```

定义完了内核页表，接下来是初始化内核页表，在系统启动的时候 start_kernel 会调用 [setup_arch](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/kernel/setup.c#L789).

在 setup_arch 中，load_cr3(swapper_pg_dir) 说明内核页表要开始起作用了，并且刷新了 TLB，初始化 init_mm 的成员变量，最重要的就是 [init_mem_mapping](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/mm/init.c#L705). 最终它会调用 [kernel_physical_mapping_init](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/mm/init_64.c#L787) -> [__kernel_physical_mapping_init](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/mm/init_64.c#L730)

在 __kernel_physical_mapping_init 里，先通过 __va 将物理地址转换为虚拟地址，然后再创建虚拟地址和物理地址的映射页表. 你可能会问，怎么这么麻烦啊？既然对于内核来讲，我们可以用 __va 和 __pa 直接在虚拟地址和物理地址之间直接转来转去，为啥还要辛辛苦苦建立页表呢？因为这是 CPU 和内存的硬件的需求，也就是说，CPU 在保护模式下访问虚拟地址的时候，就会用 CR3 这个寄存器，这个寄存器是 CPU 定义的，作为操作系统，我们是软件，只能按照硬件的要求来. 你可能又会问了，按照咱们讲初始化的时候的过程，系统早早就进入了保护模式，到了 setup_arch 里面才 load_cr3，如果使用 cr3 是硬件的要求，那之前是怎么办的呢？如果你仔细去看 arch\x86\kernel\head_64.S，这里面除了初始化内核页表之外，在这之前，还有另一个页表 early_top_pgt. 看到关键字 early 了嘛？这个页表就是专门用在真正的内核页表初始化之前，为了遵循硬件的要求而设置的. 早期页表不是我们这节的重点，这里就不展开多说了.

##### vmalloc 和 kmap_atomic 原理
在用户态可以通过 malloc 函数分配内存，当然 malloc 在分配比较大的内存的时候，底层调用的是 mmap，当然也可以直接通过 mmap 做内存映射，在内核里面也有相应的函数. 在虚拟地址空间里面，有个 vmalloc 区域，从 VMALLOC_START 开始到 VMALLOC_END，可以用于映射一段物理内存.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmalloc.c#L2615
/**
 * vzalloc - allocate virtually contiguous memory with zero fill
 * @size:    allocation size
 *
 * Allocate enough pages to cover @size from the page level
 * allocator and map them into contiguous kernel virtual space.
 * The memory allocated is set to zero.
 *
 * For tight control over page level allocator and protection flags
 * use __vmalloc() instead.
 *
 * Return: pointer to the allocated memory or %NULL on error
 */
void *vzalloc(unsigned long size)
{
	return __vmalloc_node(size, 1, GFP_KERNEL | __GFP_ZERO, NUMA_NO_NODE,
				__builtin_return_address(0));
}
EXPORT_SYMBOL(vzalloc);

// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmalloc.c#L2581
/**
 * __vmalloc_node - allocate virtually contiguous memory
 * @size:	    allocation size
 * @align:	    desired alignment
 * @gfp_mask:	    flags for the page level allocator
 * @node:	    node to use for allocation or NUMA_NO_NODE
 * @caller:	    caller's return address
 *
 * Allocate enough pages to cover @size from the page level allocator with
 * @gfp_mask flags.  Map them into contiguous kernel virtual space.
 *
 * Reclaim modifiers in @gfp_mask - __GFP_NORETRY, __GFP_RETRY_MAYFAIL
 * and __GFP_NOFAIL are not supported
 *
 * Any use of gfp flags outside of GFP_KERNEL should be consulted
 * with mm people.
 *
 * Return: pointer to the allocated memory or %NULL on error
 */
void *__vmalloc_node(unsigned long size, unsigned long align,
			    gfp_t gfp_mask, int node, const void *caller)
{
	return __vmalloc_node_range(size, align, VMALLOC_START, VMALLOC_END,
				gfp_mask, PAGE_KERNEL, 0, node, caller);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/vmalloc.c#L2522
/**
 * __vmalloc_node_range - allocate virtually contiguous memory
 * @size:		  allocation size
 * @align:		  desired alignment
 * @start:		  vm area range start
 * @end:		  vm area range end
 * @gfp_mask:		  flags for the page level allocator
 * @prot:		  protection mask for the allocated pages
 * @vm_flags:		  additional vm area flags (e.g. %VM_NO_GUARD)
 * @node:		  node to use for allocation or NUMA_NO_NODE
 * @caller:		  caller's return address
 *
 * Allocate enough pages to cover @size from the page level
 * allocator with @gfp_mask flags.  Map them into contiguous
 * kernel virtual space, using a pagetable protection of @prot.
 *
 * Return: the address of the area or %NULL on failure
 */
void *__vmalloc_node_range(unsigned long size, unsigned long align,
			unsigned long start, unsigned long end, gfp_t gfp_mask,
			pgprot_t prot, unsigned long vm_flags, int node,
			const void *caller)
{
	struct vm_struct *area;
	void *addr;
	unsigned long real_size = size;

	size = PAGE_ALIGN(size);
	if (!size || (size >> PAGE_SHIFT) > totalram_pages())
		goto fail;

	area = __get_vm_area_node(real_size, align, VM_ALLOC | VM_UNINITIALIZED |
				vm_flags, start, end, node, gfp_mask, caller);
	if (!area)
		goto fail;

	addr = __vmalloc_area_node(area, gfp_mask, prot, node);
	if (!addr)
		return NULL;

	/*
	 * In this function, newly allocated vm_struct has VM_UNINITIALIZED
	 * flag. It means that vm_struct is not fully initialized.
	 * Now, it is fully initialized, so remove this flag here.
	 */
	clear_vm_uninitialized_flag(area);

	kmemleak_vmalloc(area, size, gfp_mask);

	return addr;

fail:
	warn_alloc(gfp_mask, NULL,
			  "vmalloc: allocation failure: %lu bytes", real_size);
	return NULL;
}
```

再来看内核的临时映射函数 kmap_atomic 的实现. 从下面的代码我们可以看出，如果是 32 位有高端地址的，就需要调用 set_pte 通过内核页表进行临时映射；如果是 64 位没有高端地址的，就调用 page_address，里面会调用 lowmem_page_address. 其实低端内存的映射，会直接使用 __va 进行临时映射.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/highmem.h#L92
/*
 * kmap_atomic/kunmap_atomic is significantly faster than kmap/kunmap because
 * no global lock is needed and because the kmap code must perform a global TLB
 * invalidation when the kmap pool wraps.
 *
 * However when holding an atomic kmap is is not legal to sleep, so atomic
 * kmaps are appropriate for short, tight code paths only.
 *
 * The use of kmap_atomic/kunmap_atomic is discouraged - kmap/kunmap
 * gives a more generic (and caching) interface. But kmap_atomic can
 * be used in IRQ contexts, so in some (very limited) cases we need
 * it.
 */
static inline void *kmap_atomic_prot(struct page *page, pgprot_t prot)
{
	preempt_disable();
	pagefault_disable();
	if (!PageHighMem(page))
		return page_address(page);
	return kmap_atomic_high_prot(page, prot);
}
#define kmap_atomic(page)	kmap_atomic_prot(page, kmap_prot)
```

##### 内核态缺页异常
可以看出，kmap_atomic 和 vmalloc 不同. kmap_atomic 发现，没有页表的时候，就直接创建页表进行映射了. 而 vmalloc 没有，它只分配了内核的虚拟地址. 所以，访问它的时候，会产生缺页异常. 内核态的缺页异常还是会调用 [handle_page_fault](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/mm/fault.c#L1104)，但是会走到咱们上面用户态缺页异常中没有解析的那部分 [do_kern_addr_fault](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/mm/fault.c#L1104)->[spurious_kernel_fault](https://elixir.bootlin.com/linux/v5.8-rc3/source/arch/x86/mm/fault.c#L976), 这个函数主要用于关联内核页表项.

### 文件和文件系统
```c
/* Filesystem information: */
struct fs_struct *fs;
/* Open file information: */
struct files_struct *files;
```

每个进程有一个文件系统的数据结构，还有一个打开文件的数据结构.

### 用户态/内核态的切换的依赖变量
```c
struct thread_info		thread_info;
void				*stack; // 内核栈

/* CPU-specific state of this task: */
struct thread_struct		thread; // 在 Linux 中，真的参与进程切换的寄存器很少，主要的就是栈顶寄存器. 于是，在 task_struct 里面保留了要切换进程的时候需要修改的寄存器
```

[struct thread_struct](https://elixir.bootlin.com/linux/v5.9-rc5/source/arch/x86/include/asm/processor.h#L489)是用来保存进程上下文中 CPU 相关的一些状态信息的数据结构, 在进程切换
时起着很重要的作用.

#### 用户态函数栈
在用户态中,程序的执行往往是一个函数调用另一个函数(通过指令跳转), 函数调用都是通过栈来进行的.

在进程的内存空间里面,栈是一个从高地址到低地址,往下增长的结构,也就是上面是栈底,下面是栈顶,入栈和出栈的操作都是从下面的栈顶开始的

![x86_64的栈示意图](/misc/img/stack_64.png)

对于64位操作系统, 其寄存器数目比32位多. rax用于保存函数调用的返回结果. 栈顶指针寄存器变成了rsp,指向栈顶位置. 堆栈的Pop和Push操作会自动调整rsp,栈基指针
寄存器变成了rbp,指向当前栈帧的起始位置. 

改变比较多的是参数传递. rdi、rsi、rdx、rcx、r8、r9这6个寄存器,用于传递存储函数调用时的6个参数. 如果超过6的时候,还是需要放到栈里面. 

然而,前6个参数有时候需要进行寻址,但是如果在寄存器里面,是没有地址的,因而还是会放到栈里面,只不过放到栈里面的操作是被调用函数做的

#### 内核态函数栈(kernel stack)
有内核栈,对应的也有用户栈。进程本身也有用户态和内核态之分,进程在执行应用程序时处于用户态,如果中断发生,或者进程执行了系统调用,需要切换至内核态.

需要栈切换的原因: 两种状态下CPU的特权级是不一样的,在x86上,CPU有0~3共四个特权级, 不同的指令需要的特权级是不一样的, 比如用户态下CPU的特权级不够,
无法处理中断.

内核栈就是进程在内核态下使用的栈,用户栈则是进程在用户态下使用的栈。除此之外,与用户栈相比,内核栈有其特殊性.

首先是大小不同,内核栈一般为8K,用户栈可以很大. 进程创建时,它的内核栈就已经分配,由进程管理.

其次,每次由用户栈切换到内核栈时,内核栈为空。切换至内核栈后,内核处理完中断或者进程的系统调用返回用户栈时,内核栈中的信息对进程已毫无价值,进程也绝不会期待某个时间点以此刻的状态继续执行,所以没有必要保留内核栈中的信息,每次切换后,内核栈就都是空的了。当然了,所谓的空并不是指将数据清零,而是每次都会回到栈起始的地方.

Linux 给每个 task 都分配了内核栈.
32位系统的THREAD_SIZE在[page_32_types.h](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/page_32_types.h), 是这样定义的: PAGE_SIZE 是 4K，左移一位就是乘以 2，也就是 8K.
64位系统的THREAD_SIZE在[page_64_types.h](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/page_64_types.h), 是这样定义的: 在PAGE_SIZE的基础上左移两位,也即16K,并且要求起始地址必须是8192的整数倍.

```c
#ifdef CONFIG_KASAN
#define KASAN_STACK_ORDER 1
#else
#define KASAN_STACK_ORDER 0
#endif

#define THREAD_SIZE_ORDER	(2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER) // HREAD_SIZE表示了整个内核栈的大小
```

> Linux内核在x86平台下，PAGE_SIZE为4KB(32位和64位相同).

> 用户态进程所用的栈，是在进程线性地址空间中；而内核栈是当进程从用户空间进入内核空间时，特权级发生变化，需要切换堆栈，那么内核空间中使用的就是这个内核栈. 因为内核控制路径使用很少的栈空间，所以只需要几千个字节的内核态栈.

![内核栈示意图](/misc/img/process/kernel_stack.png)

32/64位的工作模式:
- 在用户态,应用程序进行了至少一次函数调用. 32 位的就是用函数栈传参, 而64位的前6个参数用寄存器,其他的用函数栈
- 在内核态,32位和64位都使用内核栈,格式也稍有不同,主要集中在pt_regs结构上
- 在内核态,**32位和64位的内核栈和task_struct的关联关系不同, 32位主要靠thread_info,64位主要靠Per-CPU变量**

> thread_info结构是对task_struct结构的补充, 因为task_struct结构庞大但是通用,不同的体系结构就需要保存不同的东西,所以往往与体系结构有关的,都放在thread_info里面.

在内核代码里面有这样一个union: [thread_union](https://elixir.bootlin.com/linux/latest/source/include/linux/sched.h#L1650),x86_64下它将task_struct和stack放在一起:
```c
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK // 仅ia64体系结构启用此选项, 因此x86_64下thread_union会包含task_struct
	struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK // 会为x86_64体系结构启用此选项, 因此thread_info会包含在task_struct中
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

在内核栈的最高地址端,存放的是另一个结构[pt_regs](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/ptrace.h#L56),定义如下:
```c
struct pt_regs {
/*
 * C ABI says these regs are callee-preserved. They aren't saved on kernel entry
 * unless syscall needs a complete, fully filled "struct pt_regs".
 */
	unsigned long r15;
	unsigned long r14;
	unsigned long r13;
	unsigned long r12;
	unsigned long bp;
	unsigned long bx;
/* These regs are callee-clobbered. Always saved on kernel entry. */
	unsigned long r11;
	unsigned long r10;
	unsigned long r9;
	unsigned long r8;
	unsigned long ax;
	unsigned long cx;
	unsigned long dx;
	unsigned long si;
	unsigned long di;
/*
 * On syscall entry, this is syscall#. On CPU exception, this is error code.
 * On hw interrupt, it's IRQ number:
 */
	unsigned long orig_ax;
/* Return frame for iretq */
	unsigned long ip;
	unsigned long cs;
	unsigned long flags;
	unsigned long sp;
	unsigned long ss;
/* top of stack page */
};
```

当系统调用从用户态到内核态的时候,首先要做的第一件事情,就是将用户态运行过程中的CPU上下文保存起来,其实主要就是保存在这个结构的寄存器变量里. 这样当从内核系统调用返回的时候,才能让进程在刚才的地方接着运行下去. 系统调用的时候,压栈的值的顺序和struct pt_regs中寄存器定义的顺序是一样的.

在内核态时,CPU的寄存器ESP或者RSP,已经指向内核栈的栈顶,在内核态里的调用都有和用户态相似的过程.

#### [通过task_struct找内核栈的方法](https://elixir.bootlin.com/linux/latest/source/include/linux/sched/task_stack.h#L19)
```c
/*
 * When accessing the stack of a non-current task that might exit, use
 * try_get_task_stack() instead.  task_stack_page will return a pointer
 * that could get freed out from under you.
 */
static inline void *task_stack_page(const struct task_struct *task)
{
	return task->stack; // stack是指向内核栈的指针
}
```

#### [通过task_struct得到相应的pt_regs的方法](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/processor.h#L844)
```c
#define task_pt_regs(task) \
({									\
	unsigned long __ptr = (unsigned long)task_stack_page(task);	\
	__ptr += THREAD_SIZE - TOP_OF_KERNEL_STACK_PADDING;		\
	((struct pt_regs *)__ptr) - 1;					\
})
```

先从task_struct找到内核栈的开始位置(即task的首地址). 然后这个位置加上THREAD_SIZE就到了最后的位置(即thread_union的结束位置),然后转换为struct pt_regs的指针再减一,就相当于减去一个pt_regs的大小,就得到了pt_regs的首地址.

[TOP_OF_KERNEL_STACK_PADDING](https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/thread_info.h#L36):
```c
#ifdef CONFIG_X86_32
# ifdef CONFIG_VM86
#  define TOP_OF_KERNEL_STACK_PADDING 16
# else
#  define TOP_OF_KERNEL_STACK_PADDING 8
# endif
#else
# define TOP_OF_KERNEL_STACK_PADDING 0
#endif
``

也就是说,32 位机器上是 8, 其他是0. 这是因为压栈pt_regs有两种情况. 我们知道,CPU用ring来区分权限,从而Linux可以区分内核态和用户态.
1. 涉及从用户态到内核态的变化的系统调用来说. 因为涉及权限的改变,会压栈保存SS、ESP寄存器的,这两个寄存器共占用8个byte.
1. 不涉及权限的变化,就不会压栈这8个byte. 这样就会使得两种情况不兼容. 如果没有压栈还访问,就会报错,所以还不如预留在这里,保证安全. 在64位上,修改了这个问题,变成了定长的.

#### 获取当前在某个 CPU 上执行的进程的 task_struct
Linux内核引入percpu变量之后，逐渐通过percpu变量来实现current宏，并且从Linux 4.1开始，x86移除了kernel_stack(用current_task取代)，并逐渐开始简化thread_info结构体，直到Linux 4.9彻底不再通过thread_info获取task_struct指针，而是直接通过percpu(current_task)变量保存当前运行进程的task_struct, 可调用 this_cpu_read_stable 进行获取.

每个CPU运行的task_struct不再通过thread_info获取了,而是直接放在PerCPU 变量里面了.

多核情况下,CPU是同时运行的,但是它们共同使用其他的硬件资源的时候,我们需要解决多个CPU之间的同步问题.
PerCPU变量是内核中一种重要的同步机制. 顾名思义,PerCPU变量就是为每个CPU构造一个变量的副本,这样多个CPU各自操作自己的副本,互不干涉. 比如,**当前进程的变量current_task就被声明为PerCPU变量**:
```c
// 声明在https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/current.h#L11
DECLARE_PER_CPU(struct task_struct *, current_task);

static __always_inline struct task_struct *get_current(void)
{
	return this_cpu_read_stable(current_task);
}

#define current get_current() // currrent变量实际上是一个指向struct task_struct的指针

// 定义在https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/cpu/common.c#L1747
DEFINE_PER_CPU(struct task_struct *, current_task) = &init_task;
```

也就是说,系统刚刚初始化的时候,current_task都指向init_task

由于采用了percpu current_task来保存当前运行进程的task_struct，所以在当某个CPU上的进程进行切换的时候,current_task需被修改为将要切换到的目标进程. 例如,进程切换函数__switch_to就会改变current_task:
```c
// https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/process_64.c#L426
__visible __notrace_funcgraph struct task_struct *
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
{
......
this_cpu_write(current_task, next_p); // 切换时更新该变量
......
return prev_p;
}
```

当要获取当前的运行中的task_struct的时候,就需要调用this_cpu_read_stable进行读取.
```c
// https://elixir.bootlin.com/linux/latest/source/arch/x86/include/asm/percpu.h#L392
#define this_cpu_read_stable(var)	percpu_stable_op("mov", var)
```
好了,现在如果你是一个进程,正在某个CPU上运行,就能够轻松得到task_struct了.

## 进程的创建
![](/misc/img/process/9d9c5779436da40cabf8e8599eb85558.jpeg)

内核提供了不同的函数来满足创建进程、线程和内核线程的需求. 它们都调用_do_fork函数实现,区别在于传递给函数的参数不同. do_fork的逻辑并不
复杂,真正完成复杂任务的是copy_process.

fork 是一个系统调用，对应的是 sys_call_table 中找到相应的系统调用 sys_fork.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2535
#ifdef __ARCH_WANT_SYS_FORK
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
	struct kernel_clone_args args = {
		.exit_signal = SIGCHLD,
	};

	return _do_fork(&args);
#else
	/* can not support in nommu mode */
	return -EINVAL;
#endif
}
#endif

// https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2416
/*
 *  Ok, this is the main fork-routine.
 *
 * It copies the process, and if successful kick-starts
 * it and waits for it to finish using the VM if required.
 *
 * args->exit_signal is expected to be checked for sanity by the caller.
 */
long _do_fork(struct kernel_clone_args *args)
{
	u64 clone_flags = args->flags;
	struct completion vfork;
	struct pid *pid;
	struct task_struct *p;
	int trace = 0;
	long nr;

	/*
	 * Determine whether and which event to report to ptracer.  When
	 * called from kernel_thread or CLONE_UNTRACED is explicitly
	 * requested, no event is reported; otherwise, report if the event
	 * for the type of forking is enabled.
	 */
	if (!(clone_flags & CLONE_UNTRACED)) {
		if (clone_flags & CLONE_VFORK)
			trace = PTRACE_EVENT_VFORK;
		else if (args->exit_signal != SIGCHLD)
			trace = PTRACE_EVENT_CLONE;
		else
			trace = PTRACE_EVENT_FORK;

		if (likely(!ptrace_event_enabled(current, trace)))
			trace = 0;
	}

	p = copy_process(NULL, trace, NUMA_NO_NODE, args);
	add_latent_entropy();

	if (IS_ERR(p))
		return PTR_ERR(p);

	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	trace_sched_process_fork(current, p);

	pid = get_task_pid(p, PIDTYPE_PID);
	nr = pid_vnr(pid);

	if (clone_flags & CLONE_PARENT_SETTID)
		put_user(nr, args->parent_tid);

	if (clone_flags & CLONE_VFORK) {
		p->vfork_done = &vfork;
		init_completion(&vfork);
		get_task_struct(p);
	}

	wake_up_new_task(p);

	/* forking complete and child started to run, tell ptracer */
	if (unlikely(trace))
		ptrace_event_pid(trace, pid);

	if (clone_flags & CLONE_VFORK) {
		if (!wait_for_vfork_done(p, &vfork))
			ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
	}

	put_pid(pid);
	return nr;
}
```

copy_process的第一个参数clone_flags可以是多种标志的组合,它们在很大程度上决定了函数的行为:
1. CLONE_VM: 与当前进程共享vm
1. COONE_FS: 共享文件系统信息
1. CLONE_FILES: 共享打开的文件
1. CLONE_PARENT: 与当前进程共有相同的父进程
1. CLONE_THREAD: 与当前进程同属一个线程组, 即创建的是线程
1. CLONE_SYSVSEM: 共享sem_undo_list
1. CLONE_VFORK: 新进程会在当前进程之前"运行", 见_do_fork函数
1. CLONE_SETTLS: 设置新进程的TLS(thread local storage)
1. CLONE_PARENT_SETTID: 创建新进程成功,则存储它的pid到parent_tidptr
1. CLONE_CHILD_CLEARTID: 新进程退出时, 将child_tidptr指定的内存清零
1. CLONE_NEWUTS: 设置ns
1. CLONE_NEWIPC: 设置ns
1. CLONE_NEWUSER: 设置ns
1. CLONE_NEWPID: 设置ns
1. CLONE_NEWNET: 设置ns

抛开CLONE_VFORK等几个与资源管理没有直接关系的标志,其他的标志从名字上把它们分为两类:CLONE_XXX和CLONE_NEWXXX。一般在CLONE_XXX标志置1的情况下,新进程才会与当前进程共享相应的资源,CLONE_NEWXXX则相反.

_do_fork 里面做的第一件大事就是 [copy_process()](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1841).

新进程由当前进程创建,当前被作为参考模板。既然要创建进程,必然需要创建新的task_struct与之对应。copy_process在参数和权限等检查后,调用dup_task_struct创建新进程的task_struct.

它的[dup_task_struct(current, node)](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1937), 主要做了下面几件事情:
1. 调用 alloc_task_struct_node 分配一个 task_struct 结构
1. 调用 alloc_thread_stack_node 来创建内核栈，这里面调用 __vmalloc_node_range 分配一个连续的 THREAD_SIZE 的内存空间，赋值给 task_struct 的 void *stack 成员变量

	内核栈与thread_info的关系密切.

	在3.10版本的内核中,thread_info对象存在于内核栈中. 内核栈默认情况下大小为8K字节,8K字节的开始(低地址)存放的是thread_info对象.

	在5.05版本的内核中,CONFIG_THREAD_INFO_IN_TASK为真的情况下,thread_info变成了task_struct的一个字段. x86平台上该宏默认为真.

	栈的增长方向是从高地址到低地址, 所以在end_of_stack(task_struct->stack)处写入0x57AC6E9D(STACK_END_MAGIC)可以检查栈溢出.

1. 调用 arch_dup_task_struct(struct task_struct *dst, struct task_struct *src)，将 task_struct 进行复制，其实就是调用 memcpy

	arch_dup_task_struct与平台有关,但它一般至少要包含`*tsk=*orig`, 也就是将当前进程的task_struct的值复制给tsk,相当于给新的tsk继承了当前进程的值
1. 调用 setup_thread_stack 设置 thread_info.

	x86平台上已经将thread_info的作用弱化,setup_thread_stack实际为空.

到这里，整个 task_struct 复制了一份，而且内核栈也创建好了.

之后是设置权限[`retval = copy_creds(p, clone_flags);`](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1970):
1. 调用 prepare_creds，准备一个新的 struct cred *new, 其实还是从内存中分配一个新的 struct cred 结构，然后调用 memcpy 复制一份父进程的 cred.
1. 接着 p->cred = p->real_cred = get_cred(new)，将新进程的“我能操作谁”和“谁能操作我”两个权限都指向新的 cred.

cred结构体表示进程安全相关的上下文(context),记录着进程的uid(user)、gid(group)和capability等信息.

如果clone_flags的CLONE_THREAD标志置1(创建线程),新进程(实际是轻量级进程)与当前进程共享凭证。新进程的task_struct的cred字段已经指向目标cred(请注意,dup_task_struct函数已经复制了当前进程的task_struct到新进程,二者的字段的值相等),所以调用get_cred增加cred的引用计数即可返回.

如果CLONE_THREAD标志没有置位,新进程需要拥有自己的凭证。首先调用prepare_creds创建cred,prepare_creds为新的cred申请内
存,复制当前进程的cred给它赋值,并设置它的引用计数,第2步结束.

CLONE_NEWUSER 表示需要创建新的user namespace, 很少使用,create_user_ns也只在定义了CONFIG_USER_NS的情况下才有意
义,而CONFIG_USER_NS多数情况下是没有定义的(使用情况有限,比如vserver)。本书讨论CONFIG_USER_NS没有定义的情况,也
就是不支持创建新的user namespace。这并不是意味着user namespace不存在,而是所有的cred使用同一个user namespace,init_user_ns.

第4步,使用新的cred为新进程的task_struct的cred和real_cred字段赋值。real_cred和cred都指向cred对象,前者表示进程实际的凭证,后
者表示进程当前使用的凭证。二者大多数情况下是一致的,少数情况下,cred字段会被临时更改.

接下来，copy_process 重新设置进程运行的统计量:
```c
p->utime = p->stime = p->gtime = 0;
p->start_time = ktime_get_ns();
p->real_start_time = ktime_get_boot_ns();
```

utime: 进程在用户态下经历的节拍数. u=user
utimescaled: 进程在用户态下经历的节拍数, 以处理器的频率为刻度
stime: 进程在内核态下经历的节拍数. s=system
stimescaled: 进程在内核态下经历的节拍数, 以处理器的频率为刻度
gtime: 以节拍数计算的虚拟cpu运行时间. g=guest
start_time: 起始时间
real_start_time: 起始时间, 将系统睡眠时间计算在内

接下来，copy_process 开始设置调度相关的变量:
```c
retval = sched_fork(clone_flags, p);
```

[sched_fork](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2068) 主要做了下面几件事情：
1. 调用 __sched_fork，在这里面将 on_rq 设为 0，初始化 sched_entity(se, dl, rt)，将里面的 exec_start、sum_exec_runtime、prev_sum_exec_runtime、vruntime 都设为 0, 这几个变量涉及进程的实际运行时间和虚拟运行时间, 是否到时间应该被调度了，就靠它们几个
1. 设置进程的状态 p->state = TASK_NEW
1. 初始化优先级 prio、normal_prio、static_prio

	如果task_struct的sched_reset_on_fork字段为1,需要重置( reset )新进程的优先级和调度策略等字段为默认值。sched_reset_on_fork的值是从当前进程复制来的,也就是说如果一个进程的sched_reset_on_fork为1,由它创建的新进程都会经历重置操作。p->sched_reset_on_fork = 0表示新进程不会继续重置由它创建的进程.

1. 设置调度类，如果是普通进程，就设置为 p->sched_class = &fair_sched_class
1. 调用调度类的 task_fork 函数，对于 CFS 来讲，就是调用 task_fork_fair. 在这个函数里，先调用 update_curr，对于当前的进程进行统计量更新，然后把子进程和父进程的 vruntime 设成一样，最后调用 place_entity，初始化 sched_entity. 这里有一个变量 sysctl_sched_child_runs_first，可以设置父进程和子进程谁先运行. 如果设置了子进程先运行，即便两个进程的 vruntime 一样，也要把子进程的 sched_entity 放在前面，然后调用 resched_curr，标记当前运行的进程 TIF_NEED_RESCHED，也就是说，把父进程设置为应该被调度，这样下次调度的时候，父进程会被子进程抢占. 
1. `__set_task_cpu`建立新进程和CPU之间的关系, 设置task_struct的cpu字段

copy_process接下来执行一系列copy动作复制资源.

首先复制semundo, semundo与进程通信有关.

如果clone_flags的CLONE_SYSVSEM标志被置位,新进程与当前进程共享sem_undo_list, 先调用get_undo_list获取undo_list,
get_undo_list会先判断当前进程的undo_list是否为空,为空则申请一个新的undo_list赋值给当前进程并返回,否则直接返回。得到了undo_list
后,赋值给新进程,达到共享的目的.

如果CLONE_SYSVSEM标志没有置位,直接将新进程的undo_list置为NULL. 最后,线程并不具备独立的sem_undo_list,所以创建线程的时候
CLONE_SYSVSEM是被置位的.

接下来，copy_process 开始初始化与文件和文件系统相关的变量:
```c
retval = copy_files(clone_flags, p);
retval = copy_fs(clone_flags, p);
```

[copy_files](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1460) 主要用于复制一个进程打开的文件信息. 这些信息用一个结构 files_struct 来维护，每个打开的文件都有一个文件描述符. 在 copy_files 函数里面调用 dup_fd，在这里面会创建一个新的 files_struct，然后将所有的文件描述符数组 fdtable 拷贝一份.

如果clone_flags的CLONE_FILES标志被置位,新进程与当前进程共享files_struct,增加引用计数后直接返回。CLONE_FILES没有被置位的情况下,copy_files调用dup_fd为新进程创建files_struct并复制当前值为其赋值.

fdtable.max_fds字段表示进程当前可以打开文件的最大数量,默认值为NR_OPEN_DEFAULT , 等 于 BITS_PER_LONG 。 close_on_exec 和 open_fds指向两个位图,位图中的一位表示一个文件的信息,位的偏移量与文件的fd相等,前者表示文件的close on exec属性,后者表示文件的打开状态。full_fds_bits可以理解为指向一个高级位图。举个例子,假设BITS_PER_LONG等于32,当前进程max_fds等于32 × 3=96, 那么full_fds_bits只需要3个位,第0位置1表示fd等于[0, 31]的文件全部打开(full的含义),任何一个文件没有打开,则第0位清零,也就是full_fds_bits的1个位表示open_fds的32个位.

files_struct结构体内嵌了fdtable和file指针数组,数组的元素数等于 BITS_PER_LONG , close_on_exec_init 、 open_fds_init 和full_fds_bits_init可以表达的位数也等于BITS_PER_LONG。看到这几点你也许猜到了两个结构体的另一层关系。默认情况下,也就是新建files_struct的情况下,它的fdt指向它的fdtab;fdtable的fd指向files_struct的fd_array,close_on_exec、open_fds和full_fds_bits指向files_struct的close_on_exec_init、open_fds_init和full_fds_bits_init, 这就是dup_fd中第1步的含义.

这种预留内存的技巧可以快速满足多数需求,但如果进程打开的文件数超过NR_OPEN_DEFAULT,它就无法满足进程的需要了。这时候files_struct内嵌的fdtable、file指针数组和两个位图均不再使用,内核需要调用alloc_fdtable申请新的fdtable对象,并为它申请内存存放file指针和两种位图,最终更新max_fds字段的值并将fdtable对象赋值给files_struct的fdt字段,第2步正是该意图.

表面上,按照当前进程使用文件的情况申请了fdtable,应该就可以满足条件,while循环岂不多此一举?实际上,在申请fdtable的时候,当前进程的文件使用情况可能已经发生变化,得到fdtable后需要判断是否能够应对这些变化.

当前进程不是在创建新进程吗?文件使用情况怎么会变化?就像在copy_files函数中所说,files_struct是线程组共享的,即使当前进程无暇变动,其他线程所做的改动也会有影响。

dup_fd需要复制当前进程的文件信息给新进程,第2步中先调用count_open_files通过open_fds字段指向的位图计算需要复制多少个文件的信息。有一个特殊情况需要考虑,当进程打开过的文件关闭了,可能会在位图中间留下一个0,因为位图中位的偏移量与文件的描述符fd是相等的,所以必须将中间的0也复制给新进程。所以count_open_files计算得到的open_files并不是已打开的文件的数量,而是总共需要复制多少位的数据,不要被open这个名字蒙蔽。另外,三个位图字段都是unsigned long*型的,所以位图操作也是以long为单位的(进1法),复制的位数也应该是BITS_PER_LONG的整数倍。

第2步是复制之前的准备工作,第3步开始复制操作,首先由copy_fd_bitmaps函数复制三个位图,接下来for循环复制file指针(fd字段)。我们在文件系统的open一节分析过,打开一个需要三步,第一步是调用get_unused_fd_flags获取一个新的可用的文件描述符fd,第三步是调用fd_install建立fd和file对象的关系,第一步完成后fd就已经被标记为占用了,所以fd被占用并不表示文件已经被打开。所以for循环会判断fd是否对应有效的file,如果没有,清除fd的占用标记。

第4步就是扫尾工作了,前面只复制了前open_files个文件的信息,还需要将剩余部分的文件信息清零,共max_fds-open_files个。

dup_fd完成后,copy_files整个逻辑基本结束,将得到的新files_struct赋值给task_struct的files字段即可。一句话总结就是,新线程共享当前进程的文件信息,新进程复制当前进程的文件信息。共享和复制类似于传址与传值,共享意味着不独立,修改对彼此可见,类似函数传址;复制表示此刻相同,但此后彼此独立,互不干涉,类似函数传值.

[copy_fs](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1440) 主要用于复制一个进程的目录信息. 这些信息用一个结构 fs_struct 来维护. 一个进程有自己的根目录和根文件系统 root，也有当前目录 pwd 和当前目录的文件系统，都在 fs_struct 里面维护. copy_fs 函数里面调用 copy_fs_struct，创建一个新的 fs_struct，并复制原来进程的 fs_struct.

与copy_files的逻辑类似,如果clone_flags的CLONE_FS标志被置位,新进程与当前进程共享fs_struct,增加引用计数后直接返回. fs_struct表示进程与文件系统相关的信息,由task_struct的fs字段表示.

如果CLONE_FS标志没有被置位,则调用copy_fs_struct函数创建新的fs_struct并复制old_fs的值给它,最后把它赋值给当前进程task_struct的fs字段.

接下来，copy_process 开始初始化与信号相关的变量:
```c
init_sigpending(&p->pending);
retval = copy_sighand(clone_flags, p); // 涉及sighand_struct结构体,由tast_struct的sighand字段指向, sighand可理解为signal handler
retval = copy_signal(clone_flags, p); // 涉及signal_struct结构体,由tast_struct的signal字段指向,表示进程当前的信号信息
```

copy_sighand 会分配一个新的 sighand_struct, 最主要的是维护信号处理函数，在 copy_sighand 里面会调用 memcpy，将信号处理函数 sighand->action 从父进程复制到子进程.

init_sigpending 和 copy_signal 用于初始化，并且复制用于维护发给这个进程的信号的数据结构. copy_signal 函数会分配一个新的 signal_struct，并进行初始化.

copy_sighand和copy_signal,前者采用的是复制,后者采用的是重置。从逻辑上是可以讲通的,sighand表示进程处理它的信号的手段,
新进程复制当前进程的手段符合大多数需求;而signal是当前进程的信号信息,这些信号并不是发送给新进程的, 需要重置.

最后, copy_sighand 和 copy_signal检查的标志分别为CLONE_SIGHAND和CLONE_THREAD,标志置1的情况下,不会复制或重置;标志没有置位的情况下,前者复制,后者重置.

接下来，copy_process 开始复制进程内存空间. 
```c
retval = copy_mm(clone_flags, p);
```

进程都有自己的内存空间，用mm_struct结构来表示.copy_mm函数中调用dup_mm，分配一个新的mm_struct结构，调用memcpy复制这个结构.dup_mmap用于复制内存空间中内存映射的部分.mmap可以分配大块的内存，其实mmap也可以将一个文件映射到内存中，方便可以像读写内存一样读写文件.

task_struct的mm和active_mm两个字段与内存管理有关,它们都是指向mm_struct结构体的指针,前者表示进程所管理的内存的信息,后
者表示进程当前使用的内存的信息. 所属不同,mm管理的内存至少有一部分是属于进程本身的;active_mm,进程使用的内存,可能不属于进程。二者有可能不一致???

比如假设进程A借用了进程B的内存信息,进程B的mm_count加1。在进程A借用的这段时间,进程B是不能释放mm_struct的(mm_count等于0才可以释放),但如果进程B的线程组不再使用内存,它可以释放一部分内存资源.

它们的关系可以由mmput函数完美地阐述. mm_users为0的时候释放的是aio、mmap和exe_file等,mm_count为0的时候,释放的是pgd、context和mm_struct本身。它们能够释放的,也是它们负责保护的。另外,mmdrop函数不一定非由mmput调用,其他模块可以单独调用它,如此进程的内存有了相对完整的管理方案.

在创建线程的时候会将CLONE_VM标志置1,这种情况下copy_mm增加了mm_struct.mm_users的引用计数,然后赋值返回(第2步)。mm_users的作用其实就是表示共享当前内存资源的线程数,但此时mm_count并没有增加,也就是说一个线程组,实际上给mm_count带来的增益只是1,线程数增加对它没有影响。如果线程组的线程不再使用内存,mm_users减为0,会释放部分资源,mm_count的值减1.

copy_mm的第1步,如果当前进程的mm字段为NULL,则直接返回, 因为内核线程不需要自己管理内存.

第2步,调用dup_mm新建一个mm_struct,并复制当前进程的mm_struct的值。首先调用allocate_mm申请新的mm_struct对象,然后复制current->mm的字段的值给它(memcpy)。然后依次调用mm_init和dup_mmap等为新mm_struct对象的字段赋值.

mm_init主要的作用是初始化mm_struct的一些字段,其复杂之处在于它调用了mm_alloc_pgd调用pgd_alloc创建pgd,赋值给mm_struct的pgd字
段. pgd_alloc是平台相关的.

PREALLOCATED_PMDS表示预先申请的用于存放pmd项的内存的页数,只在定义了CONFIG_X86_PAE的情况下才有意义,其他情况
下为0。这是为什么呢?因为在使能PAE的情况下,物理上存在三级页表,分别对应pgd、pmd和pte。在禁用PAE的情况下,物理上存在二级
(X86_32)或四(五)级页表(X86_64),前者对应pgd和pte,后者对应pgd、pud、pmd和pte,都不符合PREALLOCATED_PMDS的目的.

PREALLOCATED_PMDS的值不会大于4,它的值取决于宏SHARED_KERNEL_PMD的值。我们知道4G虚拟空间中,内核只
占最高的1G,所以4项pgd中,前三项对应的是用户空间,最后一项对应内核,SHARED_KERNEL_PMD决定进程是否共享最后这个pgd项
指向的内存页--内核pmd页,也就是所谓的KERNEL_PMD。SHARED_KERNEL_PMD为1,意味着进程默认会共享内核pmd页,所
以只需要预申请3页内存即可,PREALLOCATED_PMDS默认为3

pgd_alloc在第1步中申请内存,用来存放pgd项。
第2步,preallocate_pmds预申请存放pmd项的内存
第3步,pgd_ctor复制内核对应的pgd项

一个进程的所有pgd项存放在一页内存中,但只有内存中后部分的项才与内核对应,前面的部分对应的是用户空间,它们的分界点就是
KERNEL_PGD_BOUNDARY,从第KERNEL_PGD_BOUNDARY项开始的项属于内核项,共KERNEL_PGD_PTRS项。
PAGETABLE_LEVELS==3&&SHARED_KERNEL_PMD正是使能PAE后的默认情况,此时KERNEL_PGD_BOUNDARY等于3,
KERNEL_PGD_PTRS等于1。clone_pgd_range从swapper_pg_dir复制KERNEL_PGD_PTRS个内核项到进程的pgd内存页,偏移量为KERNEL_PGD_BOUNDARY*sizeof(pgd_t)字节。swapper_pg_dir是系统第一个进程的pgd

第2步申请得到的存放pmd项的内存页并没有与pgd产生关联,这就是第4步pgd_prepopulate_pmd的作用.

所有的进程在该情况下都共享内核空间。X86_32禁用PAE和X86_64也是类似的,不过它们的PREALLOCATED_PMDS为0,用户空间使用的pgd没有设置,内核空间的pgd在第3步也完成了复制.

唯一的例外是X86_32使能PAE但SHARED_KERNEL_PMD为0的这种情况,preallocate_pmds预申请了4页内存,第4页对应内核, pgd_prepopulate_pmd复制swapper_pg_dir的内核pmd页(不是pgd项)给它,然后将它的地址写入pgd的最后一项。这种情况的策略是复制, 而不是共享,新进程创建时内核的内存信息与idle进程一致,但随后各自相对独立,需要额外的机制保证它们的内容一致

dup_mmap的作用是复制当前进程的内存映射信息给新进程.

mm_struct的mmap字段是进程的vma(vm_area_struct对象)组成的链表的头,dup_mmap的for循环遍历当前进程的链表,把符合条件
的vma复制给新进程。第1步申请新的vma,复制并赋值,第2步将vma插入新进程的链表的尾部,第3步将vma插入新进程的红黑树(红黑树
的根为mm_struct的mm_rb字段),第4步复制当前进程该vma涉及的pgd 、 p4d 、 pud 、 pmd 和 pte 项 给 新 的 进 程 , 调 用 copy_page_range 完
成.

copy_mm分析完毕。一句话总结就是新线程与当前进程共享内存,新进程复制当前进程的内存映射信息(复制),与当前进程共享内存的内核部分(共享),用户空间部分二者相互独立(重置),copy_mm比前几个copy都复杂

copy_namespaces实现比较简单,如下:
第1步,根据标志检查是否需要新的namespace,如果不需要则返回
第2步,验证权限,创建新的namespace需要admin的权限。
第3步,调用create_new_namespaces创建新的nsproxy对象,create_new_namespaces会依次调用copy_mnt_ns、copy_utsname、copy_ipcs、copy_pid_ns、copy_cgroup_ns和copy_net_ns,它们再根据各自标志是否置位决定创建新的namespace还是共享当前进程的namespace.

pid namespace由copy_pid_ns函数copy,它检查CLONE_NEWPID标志是否置位,没有置位则增加引用计数并返回,否则调用create_pid_namespace创建新的pid namespace.

pidmap结构体有nr_free和page两个字段,page字段是一页用作位图内存的地址,该页中每一位均表示一个pid,nr_free字段表示还有多
少空闲位。pid_namespace的pidmap字段是pidmap结构体组成的数组,第 2 步 申 请 了 一 页 内 存 给 数 组 的 第 一 个 元 素 , 共
BITS_PER_PAGE(4096)个可用的位,可用的位不足时才会继续申请内存。

pid_namespace按照父子关系分层级,由level字段表示,子namespace的level比父namespace的大1,第一个namespace为
init_pid_ns(又称为全局pidnamespace),它的level为0。

第3步创建的cache用作分配pid对象,pid结构体的numbers字段是一个数组,数组的元素数等于level+1,所以pid结构体并不是定长的,因此需要为其量身定制cache。第4步,字段赋值,置位pidmap[0]位图的0位,剩余可用位数减1,而pidmap数组余下的位图并没有分配内存,置其可用位数为BITS_PER_PAGE即可。除了init_pid_ns外,pid_namespace中的进程的id都是从1开始的,0位置1是为了防止有进程得到的id等于0。init_pid_ns是个特例,它的0进程是idle进程.

有了pidnamespace,终于可以完整地介绍id、pid和task_struct的转换了.

task_struct到pid:
- task_pid(): 进程pid
- task_tgid(): 进程所在的线程组的领导进程的pid
- task_pgrp(): 进程所在的进程组的领导进程的pid
- task_session():  进程所在的会话的领导进程的pid

task_struct到id:
- task_pid_nr(task)
- task_pid_nr_ns(task, ns)
- task_pid_vnr(task)
- task_tgid_nr(task)
- task_tgid_nr_ns(task, ns)
- task_tgid_vnr(task)
- task_pgrp_nr_ns(task, ns)
- task_pgrp_vnr(task)
- task_session_nr_ns(task, ns)
- task_session_vnr(task)

task_xxxid_nr(): 返回进程在全局pid namespace(init_pid_ns)中的id
task_xxxid_nr_ns(): 返回进程在ns指定的pid namespace中的id
task_xxxid_nr(): 返回进程在它当前的pid namespace中的id

id到pid:
- find_vpid(nr) : 在当前进程的namespace中查找id为nr的pid
- find_get_pid(nr) : 在当前进程的namespace中查找id为nr的pid
- find_pid_ns(nr, ns) : 在ns指定的pid namespace中查找id为nr的pid

id到task_struct:
- find_task_by_vpid(nr): 在当前进程的pid namespace中查找id为nr的进程
- find_get_task_by_vpid(nr): 在当前进程的pid namespace中查找id为nr的进程
- find_task_by_pid_ns(nr, ns): 在ns指定的pid namespace中查找id为nr的进程

copy_io检查clone_flags的CLONE_IO标志,如果置位则共享当前进程task_struct的io_context,否则为新进程创建io_context并重置.

copy_thread_tls与平台相关,它配置新进程的状态.

> 3.10版内核中还没有定义copy_thread_tls,取而代之的是copy_thread. 5.05版内核中,已经去掉了ret_from_kernel_thread和ip字段。
ret_from_kernel_thread的逻辑被并入ret_from_fork。ip字段的作用由新的结构体fork_frame实现,它的regs字段就是pt_regs,frame字段是
inactive_task_frame类型,表示目前没有运行的进程的某些寄存器状态.

copy_thread_tls在第1步中为sp和sp0字段赋值,sp紧挨着pt_regs或fork_frame对象,这也是进程使用内核栈的初始位置,内
核栈可用的范围为它和STACK_END_MAGIC之间。第2步和第3步分别针对内核线程和其他进程对pt_regs对象和
thread_struct对象赋值。需要注意以下几点。

首先,p->thread.ip和frame->ret_addr是新进程执行的起点。

其次,第3步中首先复制了当前进程的pt_regs对象的值给新进程,然后将其ax字段置为0,这是一个有趣的问题的答案。

最后,childregs->sp=sp,sp参数实际是传递至do_fork的第二个参数,表示新进程的用户栈

好了，copy_process要结束了，上面图中的组件也初始化的差不多了.

接下来，copy_process开始分配pid，设置tid，group_leader，并且建立进程之间的亲缘关系.

alloc_pid在p->nsproxy->pid_ns_for_children这个pid namespace中申请pid(并不一定是init_pid_ns).

第1步,从进程的ns的cache中申请得到pid对象;第2步为pid的numbers 数 组 ( upid 类 型 ) 赋 值 。 从 ns 开 始 , 到 它 的 parent , 直 到
init_pid_ns(level为0),申请id,返回给upid的nr字段,upid的ns字段则等于相应的namespace。也就是说,一个进程不仅在它所属的pid namespace中占用一个id,在该namespace的父namespace中,一直到init_pid_ns中,都占用一个id。

copy_process中,进程的id等于pid_nr(pid),后者展开为pid->numbers[0].nr,也就是进程在init_pid_ns中的id,

至于线程组id和tgid,如果创建的是进程,线程组id与进程的id相等;如果创建的是线程(CLONE_THREAD),新线程与当前进程属于同一个线程组,线程组id相同.

如果新建的是线程,它的exit_signal为-1,同一个线程组内,只有一个领导进程,它的exit_signal大于等于0。按照时间顺序,第一个进
程(非线程)就是领导进程,由它创建的线程都属于它的线程组(exit_signal=-1),但如果它退出,会由其他进程继承领导进程。
thread_group_leader函数可以用来判断一个进程是否是线程组的领导进程,判断的依据就是p->exit_signal>=0.

如果创建的是进程,group_leader就是它自己。如果创建的是线程( CLONE_THREAD ) , 它 与 当 前 进 程 属 于 同 一 个 线 程 组 , 它 的
group_leader等于当前进程的group_leader(线程组的领导进程)。换个说法,当前进程有可能是线程,也有可能不是,如果不是,那领导
进程就是它自己,否则领导进程就是它的领导进程。如果新进程是一个线程,它会在第4步被链接到领导进程的thread_group字段表示的链
表中.

第 2 步 , 设 置 新 进 程 的 父 进 程 , 如 果 clone_flags 的CLONE_PARENT 或 者 CLONE_THREAD 标 志 被 置 位 , 新 进 程 的
real_parent被置为当前进程的real_parent,否则它的real_parent就是当前进程。也就是说,新进程的父进程不一定是当前进程,创造它的却不
一定是它父亲,这与我们的常识并不一致。

接下来的ptrace_init_task会设置新进程task_struct的parent字段,clone_flags的CLONE_PTRACE标志没有被置位的情况下,parent保持
与real_parent一致,否则parent会被赋值为current->parent。

第3步,建立线程组、进程组和会话关系。如果新进程是线程组的领导进程(exit_signal >= 0),copy_process将它链接到父进程的链表
中,并调用3次attach_pid将进程链接到线程组、进程组和会话的链表中。有一点需要补充的是,既然新进程有可能继承当前进程的父进
程,也可能以当前进程为父进程,那么新进程与当前进程可能有两种关系,一种是父子关系,一种是兄弟关系,它们可能同在父进程的链表中。

请注意第3步的前提条件,新进程必须是线程组的领导进程,也就意味着它必须是一个进程,不能是线程。线程不会链接到进程组、会
话和父进程的链表中。

第4步,调用attach_pid,将其链接到pid的链表中,建立新进程和它的pid的关系.

copy_process就此完毕,主要包括初始化、copys和一些杂项.

copy_process 之 后 , _do_fork 会 调 用 wake_up_new_task 唤 醒 新 进程,这样新进程才会被调度执行。需要注意的是,唤醒后新进程并不
一定马上执行,它和当前进程的执行顺序也无法保证。不过,clone_flags的CLONE_VFORK标志被置位的情况下,如果当前进程在新进程之前得到运行,它会执行wait_for_vfork_done等待新进程结束或者调用execve。所以,CLONE_VFORK所谓的新进程先“运行”指的是宏观意义上的运行

[_do_fork]做的第二件大事 [`wake_up_new_task`](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/sched/core.c#L3012). 

首先，我们需要将进程的状态设置为 TASK_RUNNING. [activate_task](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/sched/core.c#L3035) 函数中会调用 enqueue_task.

如果是 CFS 的调度类，则执行相应的 [enqueue_task_fair](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/sched/fair.c#L5459).

在 enqueue_task_fair 中取出的队列就是 cfs_rq，然后调用 enqueue_entity. 在 enqueue_entity 函数里面，会调用 update_curr，更新运行的统计量，然后调用 __enqueue_entity，将 sched_entity 加入到红黑树里面，然后将 se->on_rq = 1 设置在队列上.

回到 enqueue_task_fair 后，将这个队列上运行的进程数目加一, 然后，wake_up_new_task 会调用 check_preempt_curr，看是否能够抢占当前进程. 

在 check_preempt_curr 中，会调用相应的调度类的 rq->curr->sched_class->check_preempt_curr(rq, p, flags). 对于 CFS 调度类来讲，调用的是 check_preempt_wakeup.

在 check_preempt_wakeup 函数中，前面调用 task_fork_fair 的时候，设置 sysctl_sched_child_runs_first 了，已经将当前父进程的 TIF_NEED_RESCHED 设置了，则直接返回; 否则，check_preempt_wakeup 还是会调用 update_curr 更新一次统计量，然后 wakeup_preempt_entity 将父进程和子进程 PK 一次，看是不是要抢占，如果要则调用 resched_curr 标记父进程为 TIF_NEED_RESCHED.

如果新创建的进程应该抢占父进程，因为 fork 是一个系统调用，从系统调用返回的时候，是抢占的一个好时机，如果父进程判断自己已经被设置为 TIF_NEED_RESCHED，就让子进程先跑，抢占自己.

子进程得到执行后,它的起点为ret_from_fork, 它会根据子进程内核栈中保存的pt_regs继续执行。子进程的pt_regs是从父进程复制来的,而父进程的pt_regs是
系统调用导致其切换到内核态时保存的,使它可以恢复到系统调用的下一条语句继续执行。既然是复制的,所以子进程从ret_from_fork退
出到用户态后,也是接着系统调用的下一条语句继续执行。但是,子进程得到的fork的返回值是0,与父进程的不同,因为它的返回值已经
被修改了childregs->ax=0.

所以实际上子进程并没有调用fork,而是从fork系统调用返回, 这就是`fork调用一次,返回两次`.

以前新进程从ret_from_fork退出就到了用户态, 但5.05版的内核中ret_from_fork包含了ret_from_kernel_thread的逻辑,所以从ret_from_fork退出后,进程可能处于用户态,也可能处于内核态.

## 线程的创建
env: glibc 2.31

![](/misc/img/process/14635b1613d04df9f217c3508ae8524b.jpeg)

创建一个线程调用的是 pthread_create, 本质是kernel创建一个新的task_struct, 并将新struct的所有资源指针都指向创建它的那个task_struct的资源指针.

其实，线程不是一个完全由内核实现的机制，它是由内核态和用户态合作完成的. pthread_create 不是一个系统调用，是 Glibc 库的一个函数.

### 用户态创建线程
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L623
int
__pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
               void *(*start_routine) (void *), void *arg)
{
 ...
}
versioned_symbol (libpthread, __pthread_create_2_1, 
```

首先处理的是线程的属性参数. 例如写程序的时候设置的线程栈大小. 如果没有传入线程属性，就取默认值.
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L628
const struct pthread_attr *iattr = (struct pthread_attr *) attr;
struct pthread_attr default_attr;
bool free_cpuset = false;
bool c11 = (attr == ATTR_C11_THREAD);
if (iattr == NULL || c11)
{
  ......
  iattr = &default_attr;
}
```

接下来，就像在内核里一样，每一个进程或者线程都有一个 task_struct 结构，在用户态也有一个用于维护线程的结构，就是这个 pthread 结构:
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L659
struct pthread *pd = NULL;
```

凡是涉及函数的调用，都要使用到栈, 每个线程也有自己的栈, 那接下来就是创建线程栈了:
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L660
int err = ALLOCATE_STACK (iattr, &pd);
```

[ALLOCATE_STACK](https://elixir.bootlin.com/glibc/latest/source/nptl/allocatestack.c#L53) 是一个宏，我们找到它的定义之后，发现它其实就是一个[函数allocate_stack](https://elixir.bootlin.com/glibc/latest/source/nptl/allocatestack.c#L53).

allocate_stack 主要做了以下这些事情:
1. 如果在线程属性里面设置过栈的大小，需要把设置的值拿出来
1. 为了防止栈的访问越界，在栈的末尾会有一块空间 guardsize，一旦访问到这里就错误了
1. 其实线程栈是在进程的堆里面创建的. 如果一个进程不断地创建和删除线程，我们不可能不断地去申请和清除线程栈使用的内存块，这样就需要有一个缓存. get_cached_stack 就是根据计算出来的 size 大小，看一看已经有的缓存中，有没有已经能够满足条件的；如果缓存里面没有，就需要调用 __mmap 创建一块新的
1. 线程栈也是自顶向下生长的，每个线程要有一个 pthread 结构，这个结构也是放在栈的空间里面的, 在栈底的位置，其实是地址最高位.
1. 计算出 guard 内存的位置，调用 setup_stack_prot 设置这块内存的是受保护的
1. 接下来，开始填充 pthread 这个结构里面的成员变量 stackblock、stackblock_size、guardsize、specific. 这里的 specific 是用于存放 Thread Specific Data 的，也即属于线程的全局变量
1. 将这个线程栈放到 stack_used 链表中，其实管理线程栈总共有两个链表，一个是 stack_used，也就是这个栈正被使用；另一个是 stack_cache，就是上面说的，一旦线程结束，先缓存起来，不释放，等有其他的线程创建的时候，给其他的线程用. 搞定了用户态栈的问题，其实用户态的事情基本搞定了一半.

> 如果要在堆里面 malloc 一块内存，比较大的话，用 __mmap

### 内核态创建任务
由于fork传递至_do_fork的clone_flags参数是固定的,所以它只能用来创建进程,内核提供了另一个系统调用clone,clone最终也调用_do_fork实现,与fork不同的是用户可以根据需要确定clone_flags, 因此可以使用它创建线程.

```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L688
pd->start_routine = start_routine;
```

start_routine 就是咱们给线程的函数，start_routine，start_routine 的参数 arg，以及调度策略都要赋值给 pthread. 接下来 __nptl_nthreads 加一，说明又多了一个线程.

真正创建线程的是调用 [create_thread](https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/createthread.c#L49) 函数，这个函数定义如下:
```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L785
retval = create_thread (pd, iattr, &stopped_start,
			      STACK_VARIABLES_ARGS, &thread_ran);
```

create_thread里面有很长的 clone_flags，里面的CLONE_THREAD意味着新线程和当前进程并不是父子关系, 然后就是 [ARCH_CLONE](https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/createthread.c#L34)，其实调用的是 [__clone](https://elixir.bootlin.com/glibc/latest/source/sysdeps/unix/sysv/linux/x86_64/clone.S#L50).

> x86_64只有__clone, 而不是__clone2.

我们能看到最后调用了 syscall，这一点 clone 和我们原来熟悉的其他系统调用几乎是一致的。但是，也有少许不一样的地方。

如果在进程的主线程里面调用其他系统调用，当前用户态的栈是指向整个进程的栈，栈顶指针也是指向进程的栈，指令指针也是指向进程的主线程的代码.此时此刻执行到这里，调用 clone 的时候，用户态的栈、栈顶指针、指令指针和其他系统调用一样，都是指向主线程的.

但是对于线程来说，这些都要变, 因为我们希望当 clone 这个系统调用成功的时候，除了内核里面有这个线程对应的 task_struct，当系统调用返回到用户态的时候，用户态的栈应该是线程的栈，栈顶指针应该指向线程的栈，指令指针应该指向线程将要执行的那个函数. 所以这些都需要我们自己做，将线程要执行的函数的参数和指令的位置都压到栈里面，当从内核返回，从栈里弹出来的时候，就从这个函数开始，带着这些参数执行下去.

内核里面对于 [clone 系统调用](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2564)的定义是这样的:
```c
#ifdef __ARCH_WANT_SYS_CLONE
#ifdef CONFIG_CLONE_BACKWARDS
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 unsigned long, tls,
		 int __user *, child_tidptr)
#elif defined(CONFIG_CLONE_BACKWARDS2)
SYSCALL_DEFINE5(clone, unsigned long, newsp, unsigned long, clone_flags,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
#elif defined(CONFIG_CLONE_BACKWARDS3)
SYSCALL_DEFINE6(clone, unsigned long, clone_flags, unsigned long, newsp,
		int, stack_size,
		int __user *, parent_tidptr,
		int __user *, child_tidptr,
		unsigned long, tls)
#else
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
#endif
{
	struct kernel_clone_args args = {
		.flags		= (lower_32_bits(clone_flags) & ~CSIGNAL),
		.pidfd		= parent_tidptr,
		.child_tid	= child_tidptr,
		.parent_tid	= parent_tidptr,
		.exit_signal	= (lower_32_bits(clone_flags) & CSIGNAL),
		.stack		= newsp,
		.tls		= tls,
	};

	if (!legacy_clone_args_valid(&args))
		return -EINVAL;

	return _do_fork(&args);
}
#endif
```

clone系统调用最终也通过_do_fork实现,所以它与创建进程的fork的区别仅限于因参数不同而导致的差异.

首先,vfork置位了CLONE_VM标志,导致新进程对局部变量的修改会影响当前进程,clone也置位了CLONE_VM,也有这个隐患
吗?答案是没有的,因为新线程指定了自己的用户栈,由stackaddr指定。copy_thread函数的sp参数就是stackaddr,childregs->sp=sp修改了
新线程的pt_regs,所以新线程在用户空间执行的时候,使用的栈与当前进程的不同,不会造成干扰。那为什么vfork不这么做,请参考vfork
的设计意图.

其次,fork返回了两次,clone也是一样,但它们都是返回到系统调用后开始执行,pthread_create如何让新线程执行start_routine的?
start_routine是由start_thread函数间接执行的,所以我们只需要清楚start_thread是如何被调用的. start_thread并没有传递给clone系统调用,所以它的调用与内核无关,答案就在__clone函数中.

复杂的标志位设定的影响:
- 对于 [copy_files](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1460)，原来是调用 dup_fd 复制一个 files_struct 的，现在因为 CLONE_FILES 标识位变成将原来的 files_struct 引用计数加一
- 对于 [copy_fs](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1440)，原来是调用 copy_fs_struct 复制一个 fs_struct，现在因为 CLONE_FS 标识位变成将原来的 fs_struct 的用户数加一
- 对于 [copy_sighand](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1513)，原来是创建一个新的 sighand_struct，现在因为 CLONE_SIGHAND 标识位变成将原来的 sighand_struct 引用计数加一
- 对于 [copy_signal](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1562)，原来是创建一个新的 signal_struct，现在因为 CLONE_THREAD 直接返回了
- 对于 [copy_mm](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1393)，原来是调用 dup_mm 复制一个 mm_struct，现在因为 CLONE_VM 标识位而直接指向了原来的 mm_struct
- 对于亲缘关系的影响，毕竟要识别多个线程是不是属于一个进程:

	```c
	// https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L2172
	p->pid = pid_nr(pid);
		if (clone_flags & CLONE_THREAD) {
			p->exit_signal = -1;
			p->group_leader = current->group_leader;
			p->tgid = current->tgid;
		} else {
			if (clone_flags & CLONE_PARENT)
				p->exit_signal = current->group_leader->exit_signal;
			else
				p->exit_signal = args->exit_signal;
			p->group_leader = p;
			p->tgid = p->pid;
		}

		p->nr_dirtied = 0;
		p->nr_dirtied_pause = 128 >> (PAGE_SHIFT - 10);
		p->dirty_paused_when = 0;

		p->pdeath_signal = 0;
		INIT_LIST_HEAD(&p->thread_group);
		p->task_works = NULL;

		/*
		* Ensure that the cgroup subsystem policies allow the new process to be
		* forked. It should be noted the the new process's css_set can be changed
		* between here and cgroup_post_fork() if an organisation operation is in
		* progress.
		*/
		retval = cgroup_can_fork(p, args);
		if (retval)
			goto bad_fork_put_pidfd;

		/*
		* From this point on we must avoid any synchronous user-space
		* communication until we take the tasklist-lock. In particular, we do
		* not want user-space to be able to predict the process start-time by
		* stalling fork(2) after we recorded the start_time but before it is
		* visible to the system.
		*/

		p->start_time = ktime_get_ns();
		p->start_boottime = ktime_get_boottime_ns();

		/*
		* Make it visible to the rest of the system, but dont wake it up yet.
		* Need tasklist lock for parent etc handling!
		*/
		write_lock_irq(&tasklist_lock);

		/* CLONE_PARENT re-uses the old parent */
		if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
			p->real_parent = current->real_parent;
			p->parent_exec_id = current->parent_exec_id;
		} else {
			p->real_parent = current;
			p->parent_exec_id = current->self_exec_id;
		}

	```

	从上面的代码可以看出，使用了 CLONE_THREAD 标识位之后，使得亲缘关系有了一定的变化:
	- 如果是新进程，那这个进程的 group_leader 就是它自己，tgid 是它自己的 pid，自己是线程组的头. 如果是新线程，group_leader 是当前进程的，group_leader，tgid 是当前进程的 tgid，也就是当前进程的 pid，这个时候还是拜原来进程为老大
	- 如果是新进程，新进程的 real_parent 是当前的进程，在进程树里面又见一辈人; 如果是新线程，线程的 real_parent 是当前的进程的 real_parent，其实是平辈的.

- 对于信号的处理

	如何保证发给进程的信号虽然可以被一个线程处理，但是影响范围应该是整个进程的. 例如，kill 一个进程，则所有线程都要被干掉. 如果一个信号是发给一个线程的 pthread_kill，则应该只有线程能够收到.
	
	在 copy_process 的主流程里面，无论是创建进程还是线程，都会初始化 struct sigpending pending by [`init_sigpending(&p->pending)`](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1992)，也就是每个 task_struct，都会有这样一个成员变量, 这就是一个信号列表. 如果这个 task_struct 是一个线程，这里面的信号就是发给这个线程的；如果这个 task_struct 是一个进程，这里面的信号是发给主线程的.
	
	另外，上面 copy_signal 的时候，可以看到，在创建进程的过程中，会初始化 signal_struct 里面的 struct sigpending shared_pending by [`
init_sigpending(&sig->shared_pending);`](https://elixir.bootlin.com/linux/v5.8-rc3/source/kernel/fork.c#L1584). 但是，在创建线程的过程中，连 signal_struct 都共享了. 也就是说，整个进程里的所有线程共享一个 shared_pending，这也是一个信号列表，是发给整个进程的，哪个线程处理都一样.

至此，clone 在内核的调用完毕，要返回系统调用，回到用户态.

### 用户态执行线程
根据 __clone 的第一个参数，回到用户态也不是直接运行我们指定的那个函数，而是一个通用的 [start_thread](https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L378)，这是所有线程在用户态的统一入口.

```c
// https://elixir.bootlin.com/glibc/latest/source/nptl/createthread.c#L22
#define START_THREAD_DEFN \
  static void __attribute__ ((noreturn)) start_thread (void)

// https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L378
START_THREAD_DEFN
{
	...
}
```

在 start_thread 入口函数中，才真正的调用用户提供的函数，在用户的函数执行完毕之后，会释放这个线程相关的数据. 例如，线程本地数据 thread_local variables，线程数目也减一. 如果这是最后一个线程了，就直接退出进程，另外 [__free_tcb](https://elixir.bootlin.com/glibc/latest/source/nptl/pthread_create.c#L344) 用于释放 pthread.

__free_tcb 会调用 [__deallocate_stack](https://elixir.bootlin.com/glibc/latest/source/nptl/allocatestack.c#L788) 来释放整个线程栈，这个线程栈要从当前使用线程栈的列表 stack_used 中拿下来，放到缓存的线程栈列表 stack_cache 中.

好了，整个线程的生命周期到这里就结束了.

## 创建内核线程
内核线程是一种只存在内核态的进程,它不会在用户态运行,多是一些内核中的服务进程。它们并不需要属于自己的内存,task_struct的mm字段为NULL,flags字段的PF_KTHREAD标志被置位表示它们的身份.

copy_mm先判断当前进程task_struct的mm字段是否等于NULL,等于NULL则直接返回,不等于NULL则共享或者复制内存信息。也就是说,如果当前进程不是内核线程,由它创建的进程就不是内核线程,因为无论共享还是复制,mm字段都不等于NULL,所以内核线程必须由内核线程创建。另外,如果当前进程是内核线程,那么它创建的进程也是内核线程.

第一个进程确实是内核线程,但是内核线程可以变成普通进程,执行do_execve即可.

内核线程必须由内核线程创建,所以内核提供了一个内核线程kthreadd,由它来接受创建内核线程的请求,为其他模块创建内核线程, 而不是直接使用kernel_thread.

kernel_thread也是通过调用_do_fork实现的.

kernel_thread传递给_do_fork的第二个参数是fn,创建线程的时候,第二个参数是线程的用户栈地址,这并不矛盾。kernel_thread创建
的是内核线程,所以copy_thread_tls函数中,frame->bx=sp,sp并没有被当作栈地址使用。另外,_do_fork的第3个参数arg,被赋值给了
frame->di,也就是说bx存储的是内核线程要执行的函数的地址,di存放的是传递给函数的参数,它们最终都会被ret_from_kernel使用。
kthreadd内核线程会执行kthreadd函数,该函数会循环等待其他模块创建内核线程的需求.

它会查看kthread_create_list链表,链表上的每一个元素都表示一个需求,如果链表为空,则调用schedule让出CPU,否则遍历链表上的
元素,调用create_kthread为它们创建内核线程.

create_kthread调用kernel_thread创建内核线程: kernel_thread (kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD),所以新的内核线程会以create为参数执行kthread函数.

新内核线程执行kthread,kthread先通知kthreadd内核线程创建成功(complete(&create->done)),然后schedule等待唤醒,根据下一步指示 , 退 出 或 者 在 合 适 的 条 件 下 以 create->data 为 参 数 执 行 create->threadfn指向的回调函数。

至此可以做总结了:首先,系统中的内核线程调用kernel_thread创建了kthreadd内核线程为我们服务。其次,kthreadd内核线程等待需
求,并在需求到来时调用kernel_thread创建新的内核线程,该内核线程执行其他模块期待的回调函数, kthreadd和其他模块之间依靠kthread_create_info对象传递需求。所以内核提供的函数只需要根据其他模块的参数,产生kthread_create_info对象,将它链接到kthread_create_list链表,然后唤醒kthreadd内核线程即可. 如果成功,它们均返回新内核线程的task_struct指针.

创建内核线程函数:
- kthread_create: 创建内核线程
- kthread_create_on_cpu: 创建内核线程
- kthread_create_on_node: 创建内核线程
- kthread_run: 创建内核线程, 并调用wake_up_process将它唤醒

kernel_thread调用的是_do_fork,而后者会调用 wake_up_new_task唤醒新内核线程,但是它被唤醒后执行的是kthread函数,进而进入睡眠,所以前三个函数返回后,内核线程处于睡眠状态.

## 特殊进程
### idle
第一个进程是idle进程,它不是动态创建的,程序员`固化`在系统中的[init_task](https://elixir.bootlin.com/linux/v6.6.29/source/init/init_task.c#L64)变量就是它的雏形. 系统初始化过程中,会设置init_task的相关成员.

它是一个内核线程(PF_KTHREAD,mm为NULL), 拥有很多init_xxx的变量,特权满满.

初始化完毕,idle会功成名就身退二线, 在此之前, 它拉起两个得力助手,其中一个是init进程,另一个就是内核线程一节介绍的kthreadd内核线程。kthreadd负责内核线程,init进程是第一个用户进程,负责其他进程.

init进程和kthreadd都是idle进程在rest_init函数中调用kernel_thread
创建的,所以init进程最初也是一个内核线程. 它被创建后,执行的函数是kernel_init, 里面有退出内核线程的逻辑.

kernel_init调用kernel_init_freeable,后者调用do_basic_setup,继而调用do_initcalls。do_initcalls是一个有趣的话题,它按照如下顺序调
用内核中的各种init(优先级递减).

常见的module_init的优先级与device_initcall相等。实现的原理是编译内核的时候把所有init按照同优先级同组、优先级高的组在前的顺序放在一起,do_initcalls只需要像遍历函数数组一样遍历它们即可.

第二点,kernel_init调用了do_execve执行可执行文件,导致init进程退出内核线程。可执行文件有/sbin/init、/etc/init、/bin/init和/bin/sh四
个,优先级依次降低。init文件在一个基于Linux内核的操作系统中是极为重要的,一般启动操作系统服务进程,初始化应用程序执行环境都是由它完成的.

## 进程退出
进程始于fork,终于exit,它所拥有的资源在退出时被回收.

### 退出方式
进程退出方式是在main函数中使用return,属于正常退出。除此之外,调用exit和_exit也属于正常退出.

```c
#include <stdlib.h>

[[noreturn]] void exit(int status);

#include <unistd.h>

[[noreturn]] void _exit(int status);

#include <stdlib.h>

[[noreturn]] void _Exit(int status);
```

_exit和_Exit是等同的,后者是c语言的库函数.

exit和_exit的区别在于前者会回调由atexit和on_exit注册的函数(顺序与它们注册时的顺序相反),刷新并关闭标准IO流(stdin、
stdout和stderr). exit也是c语言的库函数,最终由_exit完成.

_exit由系统调用实现,早期的glibc中,调用的是exit系统调用,从glibc 2.3版本开始,会调用exit_group系统调用,线程组退出.

如果atexit和on_exit注册的函数中有函数导致进程退出,后面的函数不会继续执行.

与正常退出对应,异常退出常见的方式包含abort和被信号终止两种. abort也是一个c语言库函数,它发送SIGABRT信号到进程,如果信号被忽略或者捕获(处理),恢复信号的处理策略为SIG_DFL,再次发送SIGABRT信号,实在不行调用_exit退出进程。另外, 在程序中使用的assert也是调用abort实现的.

### 退出过程
exit和exit_group两个系统调用完成进程退出操作,前者调用do_exit,后者对线程组中其他线程发送SIGKILL信号强制它们退出(zap_other_threads),然后调用do_exit.

do_exit用于完成进程退出的主要任务,它不仅仅为以上两个系统调用服务,其他模块也可以调用它。do_exit函数只有code一个参数, 会赋值给task_struct的exit_code字段,表示进程退出状态码.

用户空间通过exit类函数触发系统调用时传递的参数status与code有一定的换算关系,code=(status&0xff)<<8,也就是说status的低8位有
效,且code的低8位等于0,这可以与内核中直接调用do_exit退出的情况区分开,用以表明进程退出的原因.

do_exit的逻辑比较清晰,它首先调用一系列的exit函数将资源归还给系统,包括exit_signals、exit_mm、exit_sem、exit_shm、exit_files、
exit_fs 、 exit_task_namespaces 、 exit_thread 、 exit_notify 和exit_io_context等.

exit_notify:
第 1 步 , 调用forget_original_parent函数处理进程的子进程(线程),它的主要任务如下.

( 1 ) 为 子 进 程 选 择 新 的 父 进 程 , 由 find_child_reaper 和find_new_reaper函数完成,优先选择与当前进程属于同一个线程组的
进程,其次是祖先进程中,以PR_SET_CHILD_SUBREAPER为参数调用 prctl 将 自 己 设 为 child_subreaper 的 进 程 , 最 差 选 择 是 当 前
pid_namespace中的child reaper进程,也就是init进程.

(2)遍历父进程的子进程链表(parent->children),针对每一个子进程p(该链表不包括线程),更改它所管理的线程组中的线程的父
进程(包括它自己),并将它插入父进程的链表中。如果它的状态是EXIT_ZOMBIE(p->exit_state),且它所属的线程组内没有其他线
程,调用do_notify_parent(见下),由函数的返回值决定立即回收进程、还是等待它的父进程回收。

(3)若进程的退出导致某些进程组变成孤儿进程组(orphaned pgrp),如果它们之前有被停止的工作(has_stopped_jobs,意味着有进程处于TASK_STOPPED状态),发送SIGHUP和SIGCONT信号给它们,由kill_orphaned_pgrp函数完成。SIGHUP是挂起信号,默认会终止进 程 , 但 是 处 于 TASK_STOPPED 状 态 的 进 程 只 能 接 收 SIGCONT 信号,配合SIGCONT信号,SIGHUP信号才会起作用。

第2步,如果进程是线程组的领导进程,当线程组没有其他线程的情况下(请注意这个前提),调用do_notify_parent(期望发送给父进程的信号是tsk->exit_signal),函数返回值true时,进程的状态置为EXIT_DEAD,调用release_task回收进程;返回false时,进程状态置为
EXIT_ZOMBIE,等待它的父进程回收。

如果进程不是领导进程(是线程),处理方式与do_notify_parent函数返回true的情况一样。也就是说,线程是被直接回收的,只有进
程才可以被置为EXIT_ZOMBIE状态,等待父进程回收。

do_notify_parent负责以发送信号的方式将进程退出的事件通知父进程,如果父进程对事件不感兴趣,函数返回true,否则返回false。父
进程是否感兴趣取决于进程发送的信号sig(进程退出时不同情况下发送的信号可能不同)和父进程的处理策略,以tsk表示进程.

SIGCHLD的默认处理策略是SIG_IGN,所以只有父进程定义了sa_handler的情况下,子进程以SIGCHLD退出的时候才会被置为
EXIT_ZOMBIE。

执行完以上的各种exit后,do_exit会将进程的state字段置为TASK_DEAD,然后调用schedule进行进程切换,进程切换完毕后,被调度执行的进程在finish_task_switch函数中调用put_task_struct释放退出进程占用的资源.

目前已经有三处回收进程资源的地方了:第一处,exit_notify会调用release_task回收线程和父进程不感兴趣的进程;第二
处,子进程被置为EXIT_ZOMBIE被父进程回收;第三处,进程退出完成切换后,由替代者回收。前两处二选其一,第三处是必选项。为什么要回收两次呢?

release_task解除进程与其他进程的关系,然后调用put_task_struct。所以真正被调用了两次的是put_task_struct函数,它会
将进程的usage字段减1,字段值变为0的情况下才会释放task_struct等占用的资源。与一般的字段初始值为1不同,usage在进程被创建时初
始值等于2,所以进程经得起两次put_task_struct.

exit_notify的第1步的第2条任务,如果退出进程的子进程所在的线程组还有其他线程,即使它的状态是EXIT_ZOMBIE也不会
被回收。首先,这是合理的,因为线程组中其他线程的group_leader字段都依然指向它,不应该被回收。其次,这种情况确实存在,比如一
个进程创建了线程组后退出,等待父进程回收时父进程退出了,它被交给了init进程,init进程也不能马上回收它。那么它究竟在何时被回
收呢?

其实release_task中有一段代码处理这种情况. 当线程组中最后一个线程退出后(thread_group_empty),如果领导进程处于EXIT_ZOMBIE,调用do_notify_parent决定是否回收领导进程,如果是,repeat的过程会将它回收。也就是说,线程组的领导进程会在组内线程都退出时才会被回收.

### 使用wait等待子进程
父进程可以回收处于EXIT_ZOMBIE状态的进程, 使用的就是wait

```c
#include <sys/wait.h>

pid_t wait(int *_Nullable wstatus);
pid_t waitpid(pid_t pid, int *_Nullable wstatus, int options);

int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
				/* This is the glibc and POSIX interface; see
					NOTES for information on the raw system call. */
```

除了以上三个之外,还有wait3和wait4,不过它们已经废弃了,不建议直接使用.

wait是由wait4实现的(glibc),等待其中一个子进程退出即返回,wstatus存储着退出进程的退出状态码。
waitpid等待子进程的状态发生变化,包括子进程退出、子进程被信号停止(stopped)和子进程收到信号继续(continued)。参数pid用
于选择考虑的子进程.

pid和子进程关系表:
1. < -1 : 进程组id等于-pid的子进程
1. -1 : 任意子进程
1. 0 : 进程组id与当前进程的进程组id相等的子进程
1. > 0: 进程的id等于pid的进程

option的标志:
- WNOHANG: 即使没有等到有子进程退出也返回
- WUNTRACED: 有子进程stopped即可返回
- WCONTINUED: 有子进程continued即可返回
- _WNOTHREAD: 仅考虑当前进程的子进程, 不考虑同线程组内其他进程的子进程
- _WCLONE: 置为表示只考虑clone的子进程, 否则只考虑非clone的子进程
- _WALL: 无论clone还是非clone的子进程都考虑

clone的进程指的是进程退出时不发送信号给父进程,或者发送给父进程的信号不是SIGCHLD的进程。fork得到的进程都不是clone的,
fork传递给do_fork的第一个参数为SIGCHLD,会将进程的exit_signal字段设置为SIGCHLD.

可以通过WUNTRACED和WCONTINUED标志控制waitpid是否考虑子进程stopped和continued的情况,子进程退出的情况(WEXITED)是默认必须考虑的,waitpid系统调用会自动将WEXITED置位。但是,使用waitpid时不可置位WEXITED标志,否则
会出错(EINVAL)。

waitid对参数的控制更加精细,idtype有P_ALL、P_PID和P_PGID三种,分别表示所有子进程、pid等于第二个参数id的子进程和进程组
id等于id的子进程,P_ALL会忽略id,后两种id必须大于0。

参数options除了waitpid可以接受的option外,还可以包括WNOWAIT和WEXITED,其中WEXITED、WSTOPPED和WCONTINUED三者至少选其一.

第三个参数infop是输出参数,存储子进程状态变化的原因、状态和导致它状态产生变化的信号等信息。

内核定义了wait4、waitpid和waitid三个系统调用,其中wait4和waitpid都是通过kernel_wait4实现的,三者的主要逻辑都在do_wait函数
中。逻辑并不复杂,遍历每一个符号条件的子进程,询问它们的状态,由wait_task_zombie、wait_task_stopped和wait_task_continued判断
它们是否符合WEXITED、WSTOPPED和WCONTINUED条件。如果没有返回条件的子进程,则在current->signal->wait_chldexit等待队列上
等待子进程在状态变化时唤醒它.

注意点:
首先,在exit一节中讨论过,线程组的领导进程必须等到其他线程退出之后才能回收,所以即便子进程处于EXIT_ZOMBIE状态,在调
用wait_task_zombie之前仍然需要判断线程组内其他线程是否已经退出。由delay_group_leader宏实现,线程组内还有其他线程的情况下宏
为真。

其次,wait_task_zombie在WNOWAIT标志没有置位的情况下,会调用release_task回收子进程。反过来讲,以WNOWAIT标志调用
waitid,子进程并没有被回收,还需要wait。

最后,根据exit一节的讨论,对子进程而言,如果它的父进程对SIGCHLD的处理方式是SIG_IGN,或者sa_flags字段置位了
SA_NOCLDWAIT标志,do_notify_parent返回true,子进程会被回收。也就是说进程必须捕捉SIGCHLD,子进程才会在退出时报告。如果进
程没有捕捉SIGCHLD,wait会等到所有子进程退出,得到错误(ECHILD).

## FAQ
### current
内核定义了一个使用频率很高的宏current,它是指向当前进程的task_struct的指针.

current的实现方式与平台有关,在x86上是通过每CPU变量实现的,变量的名字为current_task,类型为struct task_struct*,current通过
获取当前CPU上变量的值得到.