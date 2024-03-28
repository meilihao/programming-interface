# md
ref:
- [<<存储技术原理分析 - 第6章 multi-disk(md)模块>>]

Multi-Disk(MD)是软raid技术, 利用多个块设备模拟单个块设备, 相关代码在`include/linux/raid`和`drivers/md`. 其管理工具是mdadm.

md分两层:
1. raid公共层

	提供各种级别的raid的公共特性, 依照块设备的实现模板向上层注册, 同时向raid个性层提供公共函数, 已经接口注册函数
1. raid个性层

	各种级别raid的个性实现, 它向raid公共层注册个性接口, 利用raid公共层提供的公共函数, 基于底层实现个性化功能. raid个性以独立模块的形式实现, 可以动态加载


md核心数据结构:
- mddev是raid设备保存自身信息的, 包括了完整的raid设备的信息
- md_rdev是组成md设备的底层块设备信息

md设备可以有不同的个性, 它指向一个md个性结构, 其中包含了这种个性md设备的操作表.

```c
// https://elixir.bootlin.com/linux/v6.6.17/source/drivers/md/md.h#L309
struct md_rdev {
	struct list_head same_set;	/* RAID devices within the same set */ // 链入所属md设备的所有成员磁盘链表的连接件

	sector_t sectors;		/* Device size (in 512bytes sectors) */ // 成员磁盘的有效长度, 以扇区为单位
	struct mddev *mddev;		/* RAID array if running */ // 指向所属md对象
	int last_events;		/* IO event timestamp */ // I/O事件时间戳, 用来判断md设备最近是否空闲, 以决定同步是否给正常I/O"让路"

	/*
	 * If meta_bdev is non-NULL, it means that a separate device is
	 * being used to store the metadata (superblock/bitmap) which
	 * would otherwise be contained on the same device as the data (bdev).
	 */
	struct block_device *meta_bdev;
	struct block_device *bdev;	/* block device handle */ // 指向该成员磁盘所对应的块设备

	struct page	*sb_page, *bb_page; // sb_page 指向保存该成员磁盘上的raid超级块的页面
	int		sb_loaded; // 1, 该成员磁盘的raid超级块已经被读入内存
	__u64		sb_events; // 该成员磁盘保存的raid超级块的更新计数器
	sector_t	data_offset;	/* start of data in array */ // 磁盘的阵列数据的起始位置
	sector_t	new_data_offset;/* only relevant while reshaping */
	sector_t	sb_start;	/* offset of the super block (in 512byte sectors) */ // raid超级块保存在成员磁盘上的起始扇区编号
	int		sb_size;	/* bytes in the superblock */ // 超级块的字节数
	int		preferred_minor;	/* autorun support */ // 在自动运行md设备时采用的次设备号

	struct kobject	kobj;

	/* A device can be in one of three states based on two flags:
	 * Not working:   faulty==1 in_sync==0
	 * Fully working: faulty==0 in_sync==1
	 * Working, but not
	 * in sync with array
	 *                faulty==0 in_sync==0
	 *
	 * It can never have faulty==1, in_sync==1
	 * This reduces the burden of testing multiple flags in many cases
	 */

	unsigned long	flags;	/* bit set of 'enum flag_bits' bits. */ // 成员磁盘的标志
	wait_queue_head_t blocked_wait; // 等待该成员设备解除阻塞的等待队列. 如果请求处理依赖于这个被阻塞的成员磁盘, 则在此队列上等待. 成员磁盘解除阻塞时唤醒队列的等待线程

	int desc_nr;			/* descriptor index in the superblock */ // 成员磁盘在md超级块总的描述符索引
	int raid_disk;			/* role of device in array */ /// 成员磁盘在阵列中的角色
	int new_raid_disk;		/* role that the device will have in
					 * the array after a level-change completes.
					 */
	int saved_raid_disk;		/* role that device used to have in the
					 * array and could again if we did a partial
					 * resync from the bitmap
					 */ // 成员磁盘过去在阵列中的角色
	union {
		sector_t recovery_offset;/* If this device has been partially
					 * recovered, this is where we were
					 * up to.
					 */ // 如果正设备已经被部分恢复, 该字段反映已恢复的位置
		sector_t journal_tail;	/* If this device is a journal device,
					 * this is the journal tail (journal
					 * recovery start point)
					 */
	};

	atomic_t	nr_pending;	/* number of pending requests.
					 * only maintained for arrays that
					 * support hot removal
					 */ // 正在处理的请求数目. 只为支持热"移除"的阵列维护
	atomic_t	read_errors;	/* number of consecutive read errors that
					 * we have tried to ignore.
					 */ // 连续读错误的次数. 当超过一个阈值时, 使该磁盘失效; 若有一次成功, 则清零, 重新计数
	time64_t	last_read_error;	/* monotonic time since our
						 * last read error
						 */ // 距上传出现读错误以来过去的时间, 被raid10个性用来修正读错误
	atomic_t	corrected_errors; /* number of corrected read errors,
					   * for reporting to userspace and storing
					   * in superblock.
					   */ // 纠正了读错误数量. 如果报告到用户空间, 并保存到超级块中

	struct serial_in_rdev *serial;  /* used for raid1 io serialization */

	struct kernfs_node *sysfs_state; /* handle for 'state'
					   * sysfs entry */ // 被用来向用户空间传递成员磁盘的次序改变的状态信息, 对于sysfs的成员磁盘目录下的state
	/* handle for 'unacknowledged_bad_blocks' sysfs dentry */
	struct kernfs_node *sysfs_unack_badblocks;
	/* handle for 'bad_blocks' sysfs dentry */
	struct kernfs_node *sysfs_badblocks;
	struct badblocks badblocks;

	struct {
		short offset;	/* Offset from superblock to start of PPL.
				 * Not used by external metadata. */
		unsigned int size;	/* Size in sectors of the PPL space */
		sector_t sector;	/* First sector of the PPL space */
	} ppl;
};

// https://elixir.bootlin.com/linux/v6.6.17/source/drivers/md/md.h#L309
struct mddev {
	void				*private; // 指向个性化数据: raid->linear_conf_t; raid0->raid0_conf_t; raid1->raid1_conf_t; raid5->raid5_conf_t
	struct md_personality		*pers; // 指向个性化操作的指针. 保存了raid级别相关信息, 包括raid级别名称以及各种raid级别相关的处理函数
	dev_t				unit; // 设备号
	int				md_minor; // 次设备号
	struct list_head		disks; // 这个md设备的所有成员设备链表的表头
	unsigned long			flags; // 设备标志
	unsigned long			sb_flags;

	int				suspended; // 1. 表示已挂起
	struct percpu_ref		active_io; // 获取I/O计数器. 发给个性处理前加1, 个性处理结束后减1
	int				ro; // 0, 可写; 1, 只读; 2, 只读, 但在第一次写时自动转为可写. 即将元数据标志为"脏"
	int				sysfs_active; /* set when sysfs deletes
						       * are happening, so run/
						       * takeover/stop are not safe
						       */
	struct gendisk			*gendisk; // 指向通用磁盘结构

	struct kobject			kobj;
	int				hold_active; // 设备保持活动到什么时候. UNTIL_IOCTL保持到IOCTL结束; UNTIL_STOP保持到MD设备停止; 0, 可以释放
#define	UNTIL_IOCTL	1
#define	UNTIL_STOP	2

	/* Superblock information */
	int				major_version, // 超级块遵循的主版本号
					minor_version, // 超级块遵循的次版本号
					patch_version; // 超级块遵循的补丁号
	int				persistent; // 是否有持久化的超级块
	int				external;	/* metadata is
							 * managed externally */ // 1, 元数据由外部管理
	char				metadata_type[17]; /* externally set*/
	int				chunk_sectors; // md设备的数据被划分为多个chunk, 以扇区为单位的chunk长度
	time64_t			ctime, utime; // utime, 超级块的修改时间
	int				level, layout; // level, 级别; layout, 布局, 仅适用于某些raid个性. 比如raid5
	char				clevel[16]; // levle级别的字符串形式
	int				raid_disks; // 成员磁盘的数量
	int				max_disks; // 最大成员磁盘数量
	sector_t			dev_sectors;	/* used size of
							 * component devices */ // 这个成员磁盘用于阵列数据的扇区数
	sector_t			array_sectors; /* exported array size */ // 导出的阵列长度
	int				external_size; /* size managed
							* externally */ // 外部管理的长度
	__u64				events; // 更新计数器. 创建时清零. 每次发生大变更(启动/停止阵列, 添加设备, 备用盘激活等)递增1. 记录在md设备超级块中, 因此比较从各个成员磁盘读取的超级块的这个计数器就可以知道哪个成员磁盘更新了
	/* If the last 'event' was simply a clean->dirty transition, and
	 * we didn't write it to the spares, then it is safe and simple
	 * to just decrement the event count on a dirty->clean transition.
	 * So we record that possibility here.
	 */
	int				can_decrease_events;

	char				uuid[16]; // 唯一标识

	/* If the array is being reshaped, we need to record the
	 * new shape and an indication of where we are up to.
	 * This is written to the superblock.
	 * If reshape_position is MaxSector, then no reshape is happening (yet).
	 */
	sector_t			reshape_position; // 记录上次reshape到的位置, 下次启动raid设备时可以从这个位置继续开始reshape. MaxSector表示没有进行reshape或已完成reshape
	int				delta_disks, new_level, new_layout; // reshape: 对成员磁盘数目的变化, 新的raid级别, 新的布局
	int				new_chunk_sectors; // reshape: 新的chunk长度(以扇区为单位)
	int				reshape_backwards;

	struct md_thread __rcu		*thread;	/* management thread */ // 指向管理线程, 仅适用于某些raid个性, 比如raid5d循环处理stripe_head
	struct md_thread __rcu		*sync_thread;	/* doing resync or reconstruct */ // 指向同步线程, 仅适用于某些raid个性, 比如resync被用来处理同步, 恢复, reshape等

	/* 'last_sync_action' is initialized to "none".  It is set when a
	 * sync operation (i.e "data-check", "requested-resync", "resync",
	 * "recovery", or "reshape") is started.  It holds this value even
	 * when the sync thread is "frozen" (interrupted) or "idle" (stopped
	 * or finished).  It is overwritten when a new sync operation is begun.
	 */
	char				*last_sync_action;
	sector_t			curr_resync;	/* last block scheduled */ // 最近已调度的块
	/* As resync requests can complete out of order, we cannot easily track
	 * how much resync has been completed.  So we occasionally pause until
	 * everything completes, then set curr_resync_completed to curr_resync.
	 * As such it may be well behind the real resync mark, but it is a value
	 * we are certain of.
	 */
	sector_t			curr_resync_completed; // 因为同步请求块可能以随机次序完成, 不易追踪已经完成了多少同步. 因此, 时不时暂停一下, 等待所有已发起的同步请求都完成, 然后设置curr_resync_completed为curr_resync.
	unsigned long			resync_mark;	/* a recent timestamp */ // 用于计算同步速度的样本采样: 最近采集点的时间戳
	sector_t			resync_mark_cnt;/* blocks written at resync_mark */ // 用于计算同步速度的样本采样: 最近采集点的已同步块数
	sector_t			curr_mark_cnt; /* blocks scheduled now */ // 当前已调度的块数

	sector_t			resync_max_sectors; /* may be set by personality */ // 所需要同步的最大扇区数, 可由个性设置

	atomic64_t			resync_mismatches; /* count of sectors where
							    * parity/replica mismatch found
							    */ // 校验和检查发现不一致的扇区数

	/* allow user-space to request suspension of IO to regions of the array */
	sector_t			suspend_lo; // 它和suspend_hi都是扇区编号, 给出raid设备的一个范围, 落在该范围的I/O将被阻塞. 当前支持raid4/5/6
	sector_t			suspend_hi;
	/* if zero, use the system-wide default */
	int				sync_speed_min; // 为了充分利用cpu资源, 同时又不至于冲击正常I/O, 为raid的同步设定一个保证的速度范围. 同步过快则在同步中适当休眠. sync_speed_min是最小保证同步速度, sync_speed_max是最大保证同步速度
	int				sync_speed_max;

	/* resync even though the same disks are shared among md-devices */
	int				parallel_resync; // 1, 即使由其他共享相关的raid设备正在进行或准备开始同步时, 也允许本raid设备的同步进行

	int				ok_start_degraded; // 如果某些raid设备即"脏"又降级, 可能包含了没有被检测的数据损坏. 因此通常会拒绝启动该设备. 1, 绕过该检查机制

	unsigned long			recovery; // 同步/恢复等标志
	/* If a RAID personality determines that recovery (of a particular
	 * device) will fail due to a read error on the source device, it
	 * takes a copy of this number and does not attempt recovery again
	 * until this number changes.
	 */
	int				recovery_disabled; // 1, 禁止恢复尝试. raid1设备只有一个成员磁盘, 恢复没必要进行

	int				in_sync;	/* know to not need resync */ // 1, 该raid在同步状态, 不需要同步. 只有写操作才会导致条带不同步的情况(比如在没有同时写入数据单元和校验单元时掉电). 因此在发起写操作时需要将该字段清零, 在所有单元都写入成功后, 再次设置 
	/* 'open_mutex' avoids races between 'md_open' and 'do_md_stop', so
	 * that we are never stopping an array while it is open.
	 * 'reconfig_mutex' protects all other reconfiguration.
	 * These locks are separate due to conflicting interactions
	 * with disk->open_mutex.
	 * Lock ordering is:
	 *  reconfig_mutex -> disk->open_mutex
	 *  disk->open_mutex -> open_mutex:  e.g. __blkdev_get -> md_open
	 */
	struct mutex			open_mutex; // 用于保护的互斥量
	struct mutex			reconfig_mutex; // 用于保护的互斥量
	atomic_t			active;		/* general refcount */ // 一般的引用计数器
	atomic_t			openers;	/* number of active opens */ // 记录这个阵列被打开的次数

	int				changed;	/* True if we might need to
							 * reread partition info */ // 1, 需要重新读入分区信息
	int				degraded;	/* whether md should consider
							 * adding a spare
							 */ // 已故障的成员磁盘数目

	atomic_t			recovery_active; /* blocks scheduled, but not written */ // 已经调度, 但没有写入的块数. 在提交同步请求时增加, 在同步完成回调函数中减少
	wait_queue_head_t		recovery_wait; // 同步/恢复等待队列. 在同步/恢复中, 有时需要再此队列上等待, 直到已经发起的同步/恢复请求完成
	sector_t			recovery_cp; // 记录上次同步到的位置, 下次启动raid设备时可从该位置继续同步. MaxSector表示没有同步或已完成同步. 仅用于同步
	sector_t			resync_min;	/* user requested sync
							 * starts here */ // 用户请求同步从这里开始
	sector_t			resync_max;	/* resync should pause
							 * when it gets here */ // 用户请求同步从这里结束

	struct kernfs_node		*sysfs_state;	/* handle for 'array_state'
							 * file in sysfs.
							 */ // 被用来向用户空间传递raid设备的持续改变的状态信息, 对应sysfs raid设备目录下的array_state
	struct kernfs_node		*sysfs_action;  /* handle for 'sync_action' */ //  被用来向用户空间传递raid设备的持续改变的同步信息, 对应sysfs raid设备目录下的sync_action
	struct kernfs_node		*sysfs_completed;	/*handle for 'sync_completed' */
	struct kernfs_node		*sysfs_degraded;	/*handle for 'degraded' */
	struct kernfs_node		*sysfs_level;		/*handle for 'level' */

	struct work_struct del_work;	/* used for delayed sysfs removal */ //在销毁该结构时需要延迟

	/* "lock" protects:
	 *   flush_bio transition from NULL to !NULL
	 *   rdev superblocks, events
	 *   clearing MD_CHANGE_*
	 *   in_sync - and related safemode and MD_CHANGE changes
	 *   pers (also protected by reconfig_mutex and pending IO).
	 *   clearing ->bitmap
	 *   clearing ->bitmap_info.file
	 *   changing ->resync_{min,max}
	 *   setting MD_RECOVERY_RUNNING (which interacts with resync_{min,max})
	 */
	spinlock_t			lock; // 用于保护
	wait_queue_head_t		sb_wait;	/* for waiting on superblock updates */ // 超级块更新的等待队列, 要等待更新完成的进程将被挂在此队列上. 更新回调函数负责唤醒. 这个队列也可用于其他等待目的. 比如 md设备挂起需要等待所有发给md个性的I/O完成, 在屏障处理时后续请求需要等待屏障处理完成等
	atomic_t			pending_writes;	/* number of active superblock writes */ // 活动的超级块写的数目

	unsigned int			safemode;	/* if set, update "clean" superblock
							 * when no writes pending.
							 */ // 安全模式
	unsigned int			safemode_delay; // 用于安全模式的超时时间
	struct timer_list		safemode_timer; // 用于安全模式的定时器. 在完成写请求后的md_write_end函数中设置, 在开始写请求前的md_write_start函数中删除
	struct percpu_ref		writes_pending; // 正在处理的写请求数目. 在开始写请求前的md_write_start函数中递增, 在完成写请求后的md_write_end函数中递减
	int				sync_checkers;	/* # of threads checking writes_pending */
	struct request_queue		*queue;	/* for plugging ... */ // 请求队列

	struct bitmap			*bitmap; /* the bitmap for the device */ // 指向md设备的位图, 某些个性为NULL
	struct {
		struct file		*file; /* the bitmap file */ // 位图文件
		loff_t			offset; /* offset from superblock of
						 * start of bitmap. May be
						 * negative, but not '0'
						 * For external metadata, offset
						 * from start of device.
						 */ // 位图起始位置, 相对于超级块的偏移, 可以是负数, 但不能为0. 对于外部管理的元数据, 为相对于设备开始的偏移
		unsigned long		space; /* space available at this offset */
		loff_t			default_offset; /* this is the offset to use when
							 * hot-adding a bitmap.  It should
							 * eventually be settable by sysfs.
							 */ // 当热插入位图时使用的偏移量
		unsigned long		default_space; /* space available at
							* default offset */
		struct mutex		mutex; // 用于保护
		unsigned long		chunksize; // 位图每一位表示一个chunk的同步情况. 该字段记录chunk长度
		unsigned long		daemon_sleep; /* how many jiffies between updates? */ // 位图中设置的位可以延迟置零, 因为位的清理并不是很关键, 即使该信息丢失, 最多也不过是多余的同步操作而已, 没有副作用. 该字段记录了后天进程两次运行之间的间隔
		unsigned long		max_write_behind; /* write-behind mode */
		int			external; // 1, 使用外部管理位图
		int			nodes; /* Maximum number of nodes in the cluster */
		char                    cluster_name[64]; /* Name of the cluster */
	} bitmap_info;

	atomic_t			max_corr_read_errors; /* max read retries */ // 最大读重试次数
	struct list_head		all_mddevs; // 链接到所有md设备链表

	const struct attribute_group	*to_remove;

	struct bio_set			bio_set;
	struct bio_set			sync_set; /* for sync operations like
						   * metadata and bitmap writes
						   */
	struct bio_set			io_clone_set;

	/* Generic flush handling.
	 * The last to finish preflush schedules a worker to submit
	 * the rest of the request (without the REQ_PREFLUSH flag).
	 */
	struct bio *flush_bio;
	atomic_t flush_pending; // 等待处理的(针对成员磁盘) 冲刷次数
	ktime_t start_flush, prev_flush_start; /* prev_flush_start is when the previous completed
						* flush was started.
						*/
	struct work_struct flush_work;
	struct work_struct event_work;	/* used by dm to report failure event */
	mempool_t *serial_info_pool;
	void (*sync_super)(struct mddev *mddev, struct md_rdev *rdev);
	struct md_cluster_info		*cluster_info;
	unsigned int			good_device_nr;	/* good device num within cluster raid */
	unsigned int			noio_flag; /* for memalloc scope API */

	/*
	 * Temporarily store rdev that will be finally removed when
	 * reconfig_mutex is unlocked, protected by reconfig_mutex.
	 */
	struct list_head		deleting;

	/* Used to synchronize idle and frozen for action_store() */
	struct mutex			sync_mutex;
	/* The sequence number for sync thread */
	atomic_t sync_seq;

	bool	has_superblocks:1;
	bool	fail_last_dev:1;
	bool	serialize_policy:1;
};

// https://elixir.bootlin.com/linux/v6.6.17/source/drivers/md/md.h#L309
struct md_personality
{
	char *name; // 名称
	int level; // 级别
	struct list_head list; // 所有注册到系统的raid个性被加入一个全局链表pers_list. 该字段是连接件
	struct module *owner; // 指向实现了这个md个性的模块
	bool __must_check (*make_request)(struct mddev *mddev, struct bio *bio); // 在将请求传递给这个个性的md设备时被调用, 执行个性化的处理逻辑
	/*
	 * start up works that do NOT require md_thread. tasks that
	 * requires md_thread should go into start()
	 */
	int (*run)(struct mddev *mddev); // 在启动这个个性md设备时被调用
	/* start up works that require md threads */
	int (*start)(struct mddev *mddev);
	void (*free)(struct mddev *mddev, void *priv);
	void (*status)(struct seq_file *seq, struct mddev *mddev); // 在查询这个个性md设备时被调用
	/* error_handler must set ->faulty and clear ->in_sync
	 * if appropriate, and should abort recovery if needed
	 */
	void (*error_handler)(struct mddev *mddev, struct md_rdev *rdev); // 在检测到这个成员磁盘发生故障时被调用. 对不具有容错能力的md个性, 则是NULL
	int (*hot_add_disk) (struct mddev *mddev, struct md_rdev *rdev); // 在对md设备进行热添加磁盘时被调用. 对不支持热插拔的md个性, 则是NULL
	int (*hot_remove_disk) (struct mddev *mddev, struct md_rdev *rdev); // // 在对md设备进行热移除磁盘时被调用. 对不支持热插拔的md个性, 则是NULL
	int (*spare_active) (struct mddev *mddev); // 在md设备从故障中恢复, 而需要激活备用盘时被调用. 对不支持故障恢复的md个性, 则是NULL
	sector_t (*sync_request)(struct mddev *mddev, sector_t sector_nr, int *skipped); // 在md设备进行同步时被调用. 对不支持冗余的md个性, 则是NULL
	int (*resize) (struct mddev *mddev, sector_t sectors); //  在md设备进行变更容量时被调用. 对不支持在线变更的md个性, 则是NULL 
	sector_t (*size) (struct mddev *mddev, sector_t sectors, int raid_disks); //  在md设备进行计算长度时被调用. 返回已扇区为长度的md设备长度 
	int (*check_reshape) (struct mddev *mddev); // 检查有效性, 进行一些可以理解开始的处理, 比如添加设备到raid1
	int (*start_reshape) (struct mddev *mddev);  // 在md设备开始reshape时被调用. 对不支持reshape的md个性, 则是NULL
	void (*finish_reshape) (struct mddev *mddev);  // 在md设备结束reshape时被调用. 对不支持reshape的md个性, 则是NULL
	void (*update_reshape_pos) (struct mddev *mddev);
	void (*prepare_suspend) (struct mddev *mddev);
	/* quiesce suspends or resumes internal processing.
	 * 1 - stop new actions and wait for action io to complete
	 * 0 - return to normal behaviour
	 */
	void (*quiesce) (struct mddev *mddev, int quiesce);
	/* takeover is used to transition an array from one
	 * personality to another.  The new personality must be able
	 * to handle the data in the current layout.
	 * e.g. 2drive raid1 -> 2drive raid5
	 *      ndrive raid5 -> degraded n+1drive raid6 with special layout
	 * If the takeover succeeds, a new 'private' structure is returned.
	 * This needs to be installed and then ->run used to activate the
	 * array.
	 */
	void *(*takeover) (struct mddev *mddev); // 接管为另一种个性
	/* Changes the consistency policy of an active array. */
	int (*change_consistency_policy)(struct mddev *mddev, const char *buf);
};
```

系统中所有的md设备会链入全局的all_mddevs中. md所有成员磁盘也会组成一个链表即md的disks字段 by `md_rdev.same_set`.

每个成员磁盘保存了从其自身角度维护的raid超级块结构, 位置由sb_start和sb_size维护. 装载md设备时, 从所有成员磁盘读取raid超级块, 并从中选择最新的作为md设备的raid超级块.

各种raid级别即md个性. 在各种raid级别的模块加载时会调用register_md_personality注册其个性化结构, 并添加到pers_list, 注销个性用unregister_md_personality.

## 初始化
[md_init](https://elixir.bootlin.com/linux/v6.6.17/source/drivers/md/md.c#L9721)

> raid456模块依赖于md-mod模块

`__register_blkdev`注册块设备, 实际只维护主设备号和块设备名的关联, 对应关系在全局变量major_names里(见`/proc/devices`). major_names是一个哈希链表数组, 共256项, 链表成员是blk_major_name.

md_init注册了两种块设备:
1. md: 不可分区, 主设备号是MD_MAJOR(9)
1. mdp: 可分区, 主设备号动态分配, 一个mdp设备最多支持64个分区(包括零号分区)

register_reboot_notifier注册系统重启的回调函数, 其作用是变量所有md设备, 并停止它.

## md创建
ref:
- [md-raid5学习记录](http://www.lenzhao.com/topic/5a586ef2bedc2a8b075a6057)

在组软raid时，使用mdadm命令对md设备进行创建，通过查看mdadm的源码，可以看到md设备节点文件是由用户态调用mknod创建的. mdadm不同的命令操作都是通过ioctl进行的系统调用，最终调用到md_ioctl， md_ioctl会再根据传参的类型进行set_array_info 或者run_array等操作

以下为md的系统调用部分，md_ioctl 即为在md-mod.ko模块中&md_fops中注册的函数
```c
[mdadm 4.2源码(https://linux.die.net/man/8/mdadm)中的mdadm.c#main函数(其他的case类似)
	|case CREATE:										
		|Create
			|create_mddev  创建mddev
				|make_parts()
					|mknod()
			|set_array_info(mdfd, st, &info)							
				|ioctl(mdfd, SET_ARRAY_INFO, &inf)
-------------------------------内核态------------------------------
以下为ioctl的调用
|_SYSCALL_DEFINE3(ioctl,..) ioctl.char
	|_do_vfs_ioctl
		|_vfs_ioctl()
			|_filp->f_op->unlocked_ioctl(filp, cmd, arg);(针对blk设备block_dev.c注册.unlocked_ioctl=block_ioctl)
				|_block_ioctl
					|_blkdev_ioctl
						|_ __blkdev_driver_ioctl
							|_disk->fops->ioctl(bdev, mode, cmd, arg);
								|_md_ioctl
```

用strace追踪mdadm创建过程最直接.