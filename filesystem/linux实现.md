# fs
参考:
- [一口气搞懂「文件系统」，就靠这 25 张图了](https://www.tuicool.com/articles/63qam22)

代码在`include/linux/`和`fs/`

## vfs
文件系统的种类众多，而操作系统希望 对用户提供一个统一的接口，于是在用户层与文件系统层引入了中间层，这个中间层就称为 虚拟文件系统(Virtual File System，VFS, 只存在内存中), 是驻留在用户进程和各种类型的linux 文件系统之间的一个抽象接口层, 对用户进程隐藏了实现每个文件系统的差异.

VFS 定义了一组所有文件系统都支持的数据结构和标准接口，这样程序员不需要了解文件系统的工作原理，只需要了解 VFS 提供的统一接口即可.

Linux为了实现这种VFS系统，采用面向对象的设计思路，主要抽象了四种对象类型：
1. 超级块对象(super_block)：代表一个文件系统,用于存储该文件系统的有关信息.

	fs中所有的inode都会链接到super_block的链表头.

	```c
	// https://elixir.bootlin.com/linux/v6.6.17/source/include/linux/fs.h#L1188
	// v6.5.2->v6.6.17无变化
	// s_dev和s_bdev是磁盘fs特有的
	struct super_block {
		struct list_head	s_list;		/* Keep this first */ //链入到所有超级块链表super_blocks(循环双向链表)的连接件
		dev_t			s_dev;		/* search index; _not_ kdev_t */ // 存储超级块的设备标识
		unsigned char		s_blocksize_bits; //表示s_blocksize的位数
		unsigned long		s_blocksize; //以B为单位的块大小
		loff_t			s_maxbytes;	/* Max file size */ //一个文件最大字节数
		struct file_system_type	*s_type; //指向文件系统类型
		const struct super_operations	*s_op; //操作超级块的函数集合
		const struct dquot_operations	*dq_op; //磁盘配额函数集合
		const struct quotactl_ops	*s_qcop; // 磁盘配额管理函数集合
		const struct export_operations *s_export_op; // 指向导出操作表. 支持s_export_op接口的文件系统都是存储设备文件系统，如ext3/4、ubifs等.
		unsigned long		s_flags; // 挂载标志
		unsigned long		s_iflags;	/* internal SB_I_* flags */
		unsigned long		s_magic; // fs magic number
		struct dentry		*s_root; // 挂载目录, 指向fs root dentry的指针
		struct rw_semaphore	s_umount; // 用于卸载的信号量
		int			s_count; // 引用计数, 可表示superblock能否被释放
		atomic_t		s_active; // 活动计数, 被mount了多少次
	#ifdef CONFIG_SECURITY
		void                    *s_security;
	#endif
		const struct xattr_handler **s_xattr; // 指向superblock扩展属性结构
	#ifdef CONFIG_FS_ENCRYPTION
		const struct fscrypt_operations	*s_cop;
		struct fscrypt_keyring	*s_master_keys; /* master crypto keys in use */
	#endif
	#ifdef CONFIG_FS_VERITY
		const struct fsverity_operations *s_vop;
	#endif
	#if IS_ENABLED(CONFIG_UNICODE)
		struct unicode_map *s_encoding;
		__u16 s_encoding_flags;
	#endif
		struct hlist_bl_head	s_roots;	/* alternate root dentries for NFS */
		struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */ // 挂载它的mount对象组成的链表的头
		struct block_device	*s_bdev; // 对于磁盘fs, 指向fs存在的块设备指针; 否则为NULL
		struct backing_dev_info *s_bdi; // 指向后备设备信息描述符. 对于某些磁盘fs, 指向块设备请求队列的内嵌后备设备信息; 某些网络fs会定义自己的后备设备信息, 而其他fs可能使用空操作
		struct mtd_info		*s_mtd; // 对于基于MTD的超级块, 指向MTD信息结构
		struct hlist_node	s_instances; // 链入file_system_type.fs_supers的连接件
		unsigned int		s_quota_types;	/* Bitmask of supported quota types */
		struct quota_info	s_dquot;	/* Diskquota specific options */ // 指向磁盘配额信息

		struct sb_writers	s_writers;

		/*
		 * Keep s_fs_info, s_time_gran, s_fsnotify_mask, and
		 * s_fsnotify_marks together for cache efficiency. They are frequently
		 * accessed and rarely modified.
		 */
		void			*s_fs_info;	/* Filesystem private info */ //fs info, 指向具体fs自定义的超级块对象

		/* Granularity of c/m/atime in ns (cannot be worse than a second) */
		u32			s_time_gran;
		/* Time limits for c/m/atime in seconds */
		time64_t		   s_time_min; // 最小时间限制
		time64_t		   s_time_max; // 最大时间限制
	#ifdef CONFIG_FSNOTIFY
		__u32			s_fsnotify_mask;
		struct fsnotify_mark_connector __rcu	*s_fsnotify_marks;
	#endif

		char			s_id[32];	/* Informational name */ //标志名称. 对于磁盘fs, 为块设备名; 否则为fs类型名
		uuid_t			s_uuid;		/* UUID */ //fs uuid

		unsigned int		s_max_links;

		/*
		 * The next field is for VFS *only*. No filesystems have any business
		 * even looking at it. You had been warned.
		 */
		struct mutex s_vfs_rename_mutex;	/* Kludge */

		/*
		 * Filesystem subtype.  If non-empty the filesystem type field
		 * in /proc/mounts will be "type.subtype"
		 */
		const char *s_subtype; // fs的子类型. 基于FUSE的fs会使用到. `/proc/mounts`中显示为`type.subtype`

		const struct dentry_operations *s_d_op; /* default d_op for dentries */

		struct shrinker s_shrink;	/* per-sb shrinker handle */

		/* Number of inodes with nlink == 0 but still referenced */
		atomic_long_t s_remove_count;

		/*
		 * Number of inode/mount/sb objects that are being watched, note that
		 * inodes objects are currently double-accounted.
		 */
		atomic_long_t s_fsnotify_connectors;

		/* Read-only state of the superblock is being changed */
		int s_readonly_remount;

		/* per-sb errseq_t for reporting writeback errors via syncfs */
		errseq_t s_wb_err;

		/* AIO completions deferred from interrupt context */
		struct workqueue_struct *s_dio_done_wq;
		struct hlist_head s_pins;

		/*
		 * Owning user namespace and default context in which to
		 * interpret filesystem uids, gids, quotas, device nodes,
		 * xattrs and security labels.
		 */
		struct user_namespace *s_user_ns;

		/*
		 * The list_lru structure is essentially just a pointer to a table
		 * of per-node lru lists, each of which has its own spinlock.
		 * There is no need to put them into separate cachelines.
		 */
		struct list_lru		s_dentry_lru; // fs的未使用dentry被链入的lru表头 by dentry.d_lru
		struct list_lru		s_inode_lru; // lru方式挂载的inode
		struct rcu_head		rcu;
		struct work_struct	destroy_work;

		struct mutex		s_sync_lock;	/* sync serialisation lock */ // 同步锁

		/*
		 * Indicates how deep in a filesystem stack this SB is
		 */
		int s_stack_depth;

		/* s_inode_list_lock protects s_inodes */
		spinlock_t		s_inode_list_lock ____cacheline_aligned_in_smp;
		struct list_head	s_inodes;	/* all inodes */ // 指向fs内所有的inode(by inode.i_sb_list), 通过它可遍历inode对象

		spinlock_t		s_inode_wblist_lock; // 回写inode的锁
		struct list_head	s_inodes_wb;	/* writeback inodes */ //挂载所有要回写的inode
	} __randomize_layout;

	// https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/fs.h#L1912
	// https://elixir.bootlin.com/linux/v6.6.17/source/include/linux/fs.h#L2054
	// v6.5.2->v6.6.17仅改变freeze_super和thaw_super
	struct super_operations {
	   	struct inode *(*alloc_inode)(struct super_block *sb);  //分配一个具体fs的inode
		void (*destroy_inode)(struct inode *); //销毁给定的索引节点
		void (*free_inode)(struct inode *); //释放给定的索引节点

	   	void (*dirty_inode) (struct inode *, int flags); //VFS将索引节点标记为脏(改变)时，会调用此函数
		int (*write_inode) (struct inode *, struct writeback_control *wbc);  //该函数用于将给定的索引节点写入磁盘. writeback_control是回写控制描述符, 通常包含表明写操作是否需要同步的标志, 并非所有的fs都检查该标志. 典型fs的write_inode都不执行I/O, 而是仅标记为脏
		int (*drop_inode) (struct inode *); //在最后一个指向索引节点的引用被释放后，VFS会调用该函数. 有些fs不想缓存inode, 则设为generic_delete_inode, 这样不论inode链接数是多少, 总能删除inode. 某些fs希望在删除前进行一些善后, 则需要实现该回调
		void (*evict_inode) (struct inode *);
		void (*put_super) (struct super_block *); //减少超级块计数调用, umount时调用
		int (*sync_fs)(struct super_block *sb, int wait); //同步文件系统调用. 日志fs通常需要实现它
		int (*freeze_super) (struct super_block *, enum freeze_holder who); //释放超级块调用
		int (*freeze_fs) (struct super_block *); // 锁住fs, 强制使它进入一致状态时调用. lvm有使用它
		int (*thaw_super) (struct super_block *, enum freeze_holder who);
		int (*unfreeze_fs) (struct super_block *); // 解锁fs
		int (*statfs) (struct dentry *, struct kstatfs *); // 获取文件系统状态
		int (*remount_fs) (struct super_block *, int *, char *); //当指定新的安装选项重新安装文件系统时，VFS会调用此函数
		void (*umount_begin) (struct super_block *); //卸载fs时调用. 该函数被网络文件系统使用，如NFS

		int (*show_options)(struct seq_file *, struct dentry *); // 显示fs信息
		int (*show_devname)(struct seq_file *, struct dentry *);
		int (*show_path)(struct seq_file *, struct dentry *);
		int (*show_stats)(struct seq_file *, struct dentry *);
	#ifdef CONFIG_QUOTA
		ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
		ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);
		struct dquot **(*get_dquots)(struct inode *);
	#endif
		long (*nr_cached_objects)(struct super_block *,
					  struct shrink_control *);
		long (*free_cached_objects)(struct super_block *,
					    struct shrink_control *);
		void (*shutdown)(struct super_block *sb);
	};
	```

	在文件系统被挂载到 VFS 的某个目录下时，VFS 会调用获取文件系统自己的超级块的函数，用具体文件系统的信息构造一个super_block实例，有了这个结构实例, VFS 就能感知到一个文件系统加入.

	super_operations 结构中所有函数指针所指向的函数，都应该要由一个具体文件系统实现.

	kernel有一个[super_blocks](https://elixir.bootlin.com/linux/v5.12.9/source/fs/super.c#L45), 所有super_block均在该双向链表中. fs中每个文件在打开时都会在内存分配一个inode并链接到super_block. 因此通过super_blocks可遍历os打开的所有inode.

1. 索引节点对象(inode)：代表具体的文件, 用于存储该文件的元信息

	VFS 用 inode 结构表示一个文件索引结点，它里面包含文件权限、文件所属用户、文件访问和修改时间、文件数据块号等一个文件的全部信息，一个 inode 结构就对应一个文件

	索引节点用来记录文件的元信息，比如 inode 编号、文件大小、访问权限、创建时间、修改时间、 数据在磁盘的位置, 对文件的读写函数, 文件的读写缓存 等等. **索引节点是文件的 唯一 标识**，它们之间一一对应，也同样都会被存储在硬盘中，即占用磁盘空间.

	为了加速文件的访问，通常会把索引节点加载到内存中.

	一个文件可以有多个dentry, 因为执行文件的路径可以有多个(考虑文件的链接), 但inode只有一个.

	`dentry + inode`可表示一个文件.

	super_block与inode是一对多.

	```c
	// https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/fs.h#L639
	// v6.5.2->v6.6.17仅改变__i_ctime
	/*
	 * Keep mostly read-only and often accessed (especially for
	 * the RCU path lookup and 'stat' data) fields at the beginning
	 * of the 'struct inode'
	 */
	struct inode {
		umode_t			i_mode; //文件访问权限, 见[这里](https://elixir.bootlin.com/linux/v5.12.9/source/include/uapi/linux/stat.h#L10): file_type(4bit)+setuid(1bit)+setgid(1bit)+sticky(1bit)+file_mode(9bit)
		unsigned short		i_opflags; // 打开file时的标志
		kuid_t			i_uid; // 创建该文件的UID
		kgid_t			i_gid; // 创建该文件的GID
		unsigned int		i_flags; // fs装载的标志

	#ifdef CONFIG_FS_POSIX_ACL
		struct posix_acl	*i_acl;
		struct posix_acl	*i_default_acl;
	#endif

		const struct inode_operations	*i_op; //操作inode的函数集合, 针对文件本身
		struct super_block	*i_sb; //指向所属超级块
		struct address_space	*i_mapping; //文件数据在内存中的页缓存. 缓存文件的内容 by radix tree. 对文件的读写操作首先在i_mapping中的缓存里查找. 如果缓存存在则从缓存获取, 不用访问存储设备, 这加速了文件操作.

	#ifdef CONFIG_SECURITY
		void			*i_security; // 指向inode的安全结构
	#endif

		/* Stat data, not accessed from path walking */
		unsigned long		i_ino; //inode编号, 在同一个fs中唯一
		/*
		 * Filesystems may only read i_nlink directly.  They shall use the
		 * following functions for modification:
		 *
		 *    (set|clear|inc|drop)_nlink
		 *    inode_(inc|dec)_link_count
		 */
		union {
			const unsigned int i_nlink; // inode的硬链接数
			unsigned int __i_nlink;
		};
		dev_t			i_rdev; //设备号, 如果该inode表示一个字符/块设备
		loff_t			i_size; //文件大小(B)
		struct timespec64	i_atime; // 最后访问时间
		struct timespec64	i_mtime; // 最后修改时间
		struct timespec64	__i_ctime; /* use inode_*_ctime accessors! */ // 最后修改时间
		spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */ // 文件的块数
		unsigned short          i_bytes; //以512字节的块为单位, 文件最后一个块的字节数
		u8			i_blkbits; //块大小(bit)
		u8			i_write_hint;
		blkcnt_t		i_blocks; // 文件的块数

	#ifdef __NEED_I_SIZE_ORDERED
		seqcount_t		i_size_seqcount; // 被SMP系统用来正确获取和设置文件长度
	#endif

		/* Misc */
		unsigned long		i_state; // inode状态
		struct rw_semaphore	i_rwsem;

		unsigned long		dirtied_when;	/* jiffies of first dirtying */ // 这个文件第一次(inode的某个page)脏的时间, 已jiffie为单位. 它被writeback用于确定是否将这个inode回写磁盘
		unsigned long		dirtied_time_when;

		struct hlist_node	i_hash; // 链入全局inode_hashtable的连接件
		struct list_head	i_io_list;	/* backing dev IO list */
	#ifdef CONFIG_CGROUP_WRITEBACK
		struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

		/* foreign inode detection, see wbc_detach_inode() */
		int			i_wb_frn_winner;
		u16			i_wb_frn_avg_time;
		u16			i_wb_frn_history;
	#endif
		struct list_head	i_lru;		/* inode LRU list */ // 用于链接描述inode当前状态的链表. 当创建一个新的inode时i_lru要链接到inode_in_use这个链表, 表示inode处于使用中, 同时i_sb_list要链接到super_block中的s_inodes链表
		struct list_head	i_sb_list; // 用于链接到super_block中的s_inodes链表的连接件
		struct list_head	i_wb_list;	/* backing dev writeback list */
		union {
			struct hlist_head	i_dentry; //引用这个inode的dentry链表(by dentry.d_alias)的表头. **一个文件可能对应多个dentry**, 这些dentry都要链接到这里
			struct rcu_head		i_rcu;
		};
		atomic64_t		i_version; //版本
		atomic64_t		i_sequence; /* see futex */
		atomic_t		i_count; //引用计数器
		atomic_t		i_dio_count; //直接io进程计数
		atomic_t		i_writecount; //用于写进程的使用计数器
	#if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
		atomic_t		i_readcount; /* struct files open RO */
	#endif
		union {
			const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */ //操作file的函数集合, 比如读写函数和异步io函数, 针对文件内容
			void (*free_inode)(struct inode *);
		};
		struct file_lock_context	*i_flctx;
		struct address_space	i_data;
		struct list_head	i_devices; // 如果inode是块设备, 则是链入块设备slave inode(block_device.bd_inodes)链表的连接件; 如果是字符设备, 是链入字符设备inode链表(cdev.list)的连接件
		union {
			struct pipe_inode_info	*i_pipe; // inode表示一个管道文件, 则指向管道信息
			struct cdev		*i_cdev; // inode表示一个字符设备, 则指向字符设备
			char			*i_link; // 链接的目标文件的路径
			unsigned		i_dir_seq;
		};

		__u32			i_generation; // inode版本号. 某些fs会使用

	#ifdef CONFIG_FSNOTIFY
		__u32			i_fsnotify_mask; /* all events this inode cares about */ // 该inode关心的所有事件
		struct fsnotify_mark_connector __rcu	*i_fsnotify_marks;
	#endif

	#ifdef CONFIG_FS_ENCRYPTION
		struct fscrypt_info	*i_crypt_info;
	#endif

	#ifdef CONFIG_FS_VERITY
		struct fsverity_info	*i_verity_info;
	#endif

		void			*i_private; /* fs or device private pointer */ // fs/设备驱动的私有数据指针
	} __randomize_layout;

	//https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/fs.h#L1826
	// v6.5.2->v6.6.18仅改变update_time和get_offset_ctx
	struct inode_operations {
		struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);  //该函数在特定目录中寻找索引节点，该索引节点要对应于dentry中给出的文件名. 只对代表目录的inode有意义
		const char * (*get_link) (struct dentry *, struct inode *, struct delayed_call *);
		int (*permission) (struct mnt_idmap *, struct inode *, int); //该函数用来检查给定的inode所代表的文件是否允许特定的访问模式，如果允许特定的访问模式，返回0，否则返回负值的错误码
		struct posix_acl * (*get_inode_acl)(struct inode *, int, bool);

		int (*readlink) (struct dentry *, char __user *,int); //被系统readlink()接口调用，拷贝数据到特定的缓冲buffer中. 拷贝的数据来自dentry指定的符号链接

		int (*create) (struct mnt_idmap *, struct inode *,struct dentry *,
			       umode_t, bool); //VFS通过系统create()和open()接口来调用该函数，从而为dentry对象创建一个新的索引节点. 只对代表目录的inode有意义, 用于创建常规文件
		int (*link) (struct dentry *,struct inode *,struct dentry *); //被系统link()接口调用，用来创建硬连接。硬链接名称由dentry参数指定
		int (*unlink) (struct inode *,struct dentry *); //被系统unlink()接口调用，删除由目录项dentry链接的索引节点对象
		int (*symlink) (struct mnt_idmap *, struct inode *,struct dentry *,
				const char *); //被系统symlik()接口调用，创建符号连接，该符号连接名称由symname指定，连接对象是dir目录中的dentry目录项
		int (*mkdir) (struct mnt_idmap *, struct inode *,struct dentry *,
			      umode_t); //被mkdir()接口调用，创建一个子目录
		int (*rmdir) (struct inode *,struct dentry *); //被rmdir()接口调用，删除dentry目录项代表的文件
		int (*mknod) (struct mnt_idmap *, struct inode *,struct dentry *,
			      umode_t,dev_t); //被mknod()接口调用，创建特殊文件(设备文件、命名管道或套接字)
		int (*rename) (struct mnt_idmap *, struct inode *, struct dentry *,
				struct inode *, struct dentry *, unsigned int); //VFS调用该函数来移动文件。文件源路径在old_dir目录中，源文件由old_dentry目录项所指定，目标路径在new_dir目录中，目标文件由new_dentry指定
		int (*setattr) (struct mnt_idmap *, struct dentry *, struct iattr *); //设置属性
		int (*getattr) (struct mnt_idmap *, const struct path *,
				struct kstat *, u32, unsigned int); //获取属性
		ssize_t (*listxattr) (struct dentry *, char *, size_t); //该函数将特定文件所有属性列表拷贝到一个缓冲列表中
		int (*fiemap)(struct inode *, struct fiemap_extent_info *, u64 start,
			      u64 len);
		int (*update_time)(struct inode *, struct timespec64 *, int);
		int (*atomic_open)(struct inode *, struct dentry *,
				   struct file *, unsigned open_flag,
				   umode_t create_mode);
		int (*tmpfile) (struct mnt_idmap *, struct inode *,
				struct file *, umode_t);
		struct posix_acl *(*get_acl)(struct mnt_idmap *, struct dentry *,
					     int);
		int (*set_acl)(struct mnt_idmap *, struct dentry *,
			       struct posix_acl *, int);
		int (*fileattr_set)(struct mnt_idmap *idmap,
				    struct dentry *dentry, struct fileattr *fa);
		int (*fileattr_get)(struct dentry *dentry, struct fileattr *fa);
	} ____cacheline_aligned;
	```

	inode 结构表示一个文件的全部信息，但这个 inode 结构是 VFS 使用的，跟某个具体文件系统上的“inode”结构并不是一一对应关系.

	VFS 通过定义 inode 结构和函数集合，并让具体文件系统实现这些函数，使得 VFS 及其上层只要关注 inode 结构，底层的具体文件系统根据自己的文件信息生成相应的 inode 结构，达到了 VFS 表示一个文件的目的

	kernel存在一个[inode_hashtable](https://elixir.bootlin.com/linux/v5.12.9/source/fs/inode.c#L60), 所有的inode均会链接到这里. 与它作用类似的还有[dentry_hashtable](https://elixir.bootlin.com/linux/v5.12.9/source/fs/dcache.c#L99).
1. 目录项对象(dentry)：代表一个目录项, 它和fs的目录不是同一个概念, 描述了该对象在内核所在fs树中的位置即文件系统的层次结构

	目录项用来记录文件的名字、 索引节点指针 以及与其他目录项的层级关联关系. 多个目录项关联起来，就会形成目录结构，但它与索引节点不同的是，**目录项是由内核维护的一个数据结构，不存放于磁盘，而是缓存在内存**

	一个文件可能不止一个dentry.
	一个文件路径的各组成部分都是一个目录项对象. 比如`/home/test/test.c`, kernel为home, test和test.c都创建了目录项对象.

	为了加快对dentry的查找, kernel使用了hash表来缓存dentry, 即dentry cache=[dentry_hashtable](https://elixir.bootlin.com/linux/v6.6.23/source/fs/dcache.c#L101)

	> 记dentry的父dentry为parent,那么dentry的d_hash字段会将它链接到由dentry_hashtable、parent和d_name的hash字段三者计算得到的哈希链表中

	inode反映fs对象的元数据, dentry反映fs对象在fs树中的位置, dentry和inode是多对一的关系, 比如多个硬连接共享一个inode.

	> 某一个文件系统可能有两套层级结构,一套存在于文件系统内部,由其自行维护;另一套由dentry表示,它来源于文件系统内部, 二者冲突时,以前者为准

	> dentry和inode存在于内存中, inode有fs实际文件对应, 而dentry没有

	```c
	//https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/dcache.h#L82
	// v6.5.2->v6.6.18无变化
	struct dentry {
		/* RCU lookup touched fields */
		unsigned int d_flags;		/* protected by d_lock */ //dentry cache标志. 可判断是否mounte: [d_mountpoint](https://elixir.bootlin.com/linux/v6.6.23/source/include/linux/dcache.h#L375). __lookup_mnt用于查找挂载点, 参考[__follow_mount_rcu](https://elixir.bootlin.com/linux/v6.6.23/source/fs/namei.c#L1508)
		seqcount_spinlock_t d_seq;	/* per dentry seqlock */
		struct hlist_bl_node d_hash;	/* lookup hash list */ //目录的hash链表, 链接到dentry cache的hash表. = v2.6.28的`struct hlist_node`
		struct dentry *d_parent;	/* parent directory */ //指向父dentry
		struct qstr d_name; //dentry名称. 打开一个文件时, 会根据这个名称来查找目标文件. qstr,全称为quick string,将文件的名字、名字的长度和哈希值保存,避免每次使用都需要计算一次
		struct inode *d_inode;		/* Where the name belongs to - NULL is
						 * negative */ //指向dentry关联的inode, inode与dentry共同描述了一个普通文件或目录文件
		unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */ //短目录名. 如果文件名超过36, 则申请内存,将名字存入d_name,d_name的name字段指向最终存储名字的位置

		/* Ref lookup also touches following */
		struct lockref d_lockref;	/* per-dentry lock and refcount */ //目录锁与计数
		const struct dentry_operations *d_op; //操作目录的函数集合
		struct super_block *d_sb;	/* The root of the dentry tree */ //指向所属fs superblock
		unsigned long d_time;		/* used by d_revalidate */ //被d_revalidate用来作为判断dentry是否还有效的依据
		void *d_fsdata;			/* fs-specific data */ //指向具体fs的dentry

		union {
			struct list_head d_lru;		/* LRU list */ // fs未使用的dentry被链入到一个lru(super_block.s_dentry_lru)中的连接件 
			wait_queue_head_t *d_wait;	/* in-lookup ones only */
		};
		struct list_head d_child;	/* child of parent list */ //挂入父目录的链表, dentry自身的链表头, 会链接到父dentry的d_subdirs. 但移动文件时需要将一个dentry从旧的父dentry链表中脱离, 再链接到新的父dentry的d_subdirs中
		struct list_head d_subdirs;	/* our children */ //挂载所有子dentry的链表
		/*
		 * d_alias and d_rcu can share memory
		 */
		union {
			struct hlist_node d_alias;	/* inode alias list */ // 链入到所属inode的i_dentry链表的连接件.
			struct hlist_bl_node d_in_lookup_hash;	/* only for in-lookup ones */
		 	struct rcu_head d_rcu;
		} d_u;
	} __randomize_layout;


	//https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/dcache.h#L128
	// v6.5.2->v6.6.18无变化
	struct dentry_operations {
		int (*d_revalidate)(struct dentry *, unsigned int); //在路径查找时从dentry缓存中找到目录项后, 被用来判断目录对象是否有效
		int (*d_weak_revalidate)(struct dentry *, unsigned int);
		int (*d_hash)(const struct dentry *, struct qstr *); //该函数为目录项生成散列值，当目录项要加入散列表中时，VFS调用该函数. 用于在路径查找时提高效率
		int (*d_compare)(const struct dentry *,
				unsigned int, const char *, const struct qstr *); //VFS调用该函数来比较name1和name2两个文件名。多数文件系统使用VFS的默认操作，仅做字符串比较。对于有些文件系统，比如FAT，简单的字符串比较不能满足其需要，因为 FAT文件系统不区分大小写
		int (*d_delete)(const struct dentry *); //当目录项对象的计数值等于0时，VFS调用该函数
		int (*d_init)(struct dentry *);//当分配目录时调用
		void (*d_release)(struct dentry *); //当目录项对象要被释放时，VFS调用该函数，默认情况下，它什么也不做. 如果其d_fsdata有指向时, 应该在本函数里处理
		void (*d_prune)(struct dentry *);
		void (*d_iput)(struct dentry *, struct inode *); //当一个目录项对象失去和inode的关联时，VFS调用该函数。默认情况下VFS会调用iput()函数释放索引节点
		char *(*d_dname)(struct dentry *, char *, int); //当需要生成一个dentry的路径名时被调用
		struct vfsmount *(*d_automount)(struct path *); //当要遍历一个自动挂载时被调用（可选），这应该创建一个新的VFS挂载记录并将该记录返回给调用者
		int (*d_manage)(const struct path *, bool); //文件系统管理从dentry的过渡（可选）时，被调用
		struct dentry *(*d_real)(struct dentry *, const struct inode *); //叠加/联合类型的文件系统实现此方法
	} ____cacheline_aligned;
	```

	目录也是文件，需要用 inode 索引结构来管理目录文件数据.

	dentry_operations 结构中的函数，也需要具体文件系统实现，下层代码查找或者操作目录时 VFS 就会调用这些函数，让具体文件系统根据自己储存设备上的目录信息处理并设置 dentry 结构中的信息，这样文件系统中的目录就和 VFS 的目录对应了.
1. 文件对象(file)：代表进程已打开的文件. 用于建立进程与文件之间的对应关系.

	file结构体对应文件的内容,并不是说它包含了文件的内容,而是说它记录了文件的访问方法和访问状态.

	文件对象代表进程与具体文件交互的关系. kernel为每个打开的文件申请一个文件对象并返回该文件的fd. 每个进程有一个文件描述符表, 它用数组保存了进程打开的每个文件.

	当且仅当进程访问文件期间存在与内存中. 同一个文件可能对应多个文件对象, 但其对应的索引节点对象是唯一的.

	```c
	// https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/fs.h#L992
	// v6.5.2->v6.6.18无变化
	/*
	 * f_{lock,count,pos_lock} members can be highly contended and share
	 * the same cacheline. f_{lock,mode} are very frequently used together
	 * and so share the same cacheline as well. The read-mostly
	 * f_{path,inode,op} are kept on a separate cacheline.
	 */
	struct file {
		union {
			struct llist_node	f_llist;
			struct rcu_head 	f_rcuhead;
			unsigned int 		f_iocb_flags;
		};

		/*
		 * Protects f_ep, f_flags.
		 * Must not be taken from IRQ context.
		 */
		spinlock_t		f_lock; // 用于保护的自旋锁
		fmode_t			f_mode; //文件权限
		atomic_long_t		f_count; // 引用计数
		struct mutex		f_pos_lock; //文件读写位置锁
		loff_t			f_pos; //进程读写文件的偏移量. 比如对文件读取前10字节, f_pos就指向第11B
		unsigned int		f_flags; // 打开文件时指定的标志位
		struct fown_struct	f_owner; // 用于通过信息进行I/O事件通知的数据
		const struct cred	*f_cred;
		struct file_ra_state	f_ra; // 用于文件预读的状态
		struct path		f_path; //文件路径, 包括所在fs的装载实例和该文件关联的dentry
		struct inode		*f_inode;	/* cached value */ //对应的inode
		const struct file_operations	*f_op; //操作文件的函数集合, 从inode的i_fop获取

		u64			f_version; // 版本号, 在每次使用后自动递增
	#ifdef CONFIG_SECURITY
		void			*f_security; // 指向file安全结构
	#endif
		/* needed for tty driver, and maybe others */
		void			*private_data; // 用于fs或设备驱动的私有指针. 大多数linux驱动遵循: 将private_data指向设备结构体, 再用read/write/ioctl/llseek等函数通过private_data访问设备结构体

	#ifdef CONFIG_EPOLL
		/* Used by fs/eventpoll.c to link all the hooks to this file */
		struct hlist_head	*f_ep;
	#endif /* #ifdef CONFIG_EPOLL */
		struct address_space	*f_mapping; //指向一个address_space, 该结构封装了文件的读写缓存页面
		errseq_t		f_wb_err;
		errseq_t		f_sb_err; /* for syncfs */
	} __randomize_layout
	  __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */

	//https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/fs.h#L961
	struct file_operations {
		struct module *owner; //所在的module
		loff_t (*llseek) (struct file *, loff_t, int); //调整读写偏移. 出错时返回负数
		ssize_t (*read) (struct file *, char __user *, size_t, loff_t *); // 成功: 返回读到的字节数; 0: EOF; 否则返回负数. 与用户空间的read和fread对应.
		ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *); // 成功: 返回写入的字节数; 0: EOF; 否则返回负数(未实现时返回-EINVAL). 与用户空间的write和fwrite对应.
		ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
		ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
		int (*iopoll)(struct kiocb *kiocb, struct io_comp_batch *,
				unsigned int flags);
		int (*iterate_shared) (struct file *, struct dir_context *);
		__poll_t (*poll) (struct file *, struct poll_table_struct *); // poll, epoll, select这3个系统调用的后端实现
		long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long); // 设备控制相关命令的实现. 成功: 返回非负数. 与用户空间的fcntl, ioctl对应
		long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
		int (*mmap) (struct file *, struct vm_area_struct *); // 将设备内存映射到进程的虚拟地址空间. 未实现返回-ENODEV. 对帧缓冲等设备特别有意义, 应用程序直接访问而无需在kernel和用户空间复制数据. 与用户空间的mmap对应
		unsigned long mmap_supported_flags;
		int (*open) (struct inode *, struct file *); // 驱动程序可以不实现这个函数, 此时设备的打开操作永远成功. 与release对应
		int (*flush) (struct file *, fl_owner_t id); // 执行并等待设备上尚未完结的操作. 不要用用户空间的fsync操作混淆. 它指用于少数几个驱动, 比如scsi 磁带驱动
		int (*release) (struct inode *, struct file *); // file被释放时调用
		int (*fsync) (struct file *, loff_t, loff_t, int datasync); // fsync的后端实现. 驱动未实现时返回-EINVAL.
		int (*fasync) (int, struct file *, int); // 通知设备其FASYNC标志发生了编号. 设备不支持异步通知, 则为NULL
		int (*lock) (struct file *, int, struct file_lock *); // 实现文件锁定. 设备驱动几乎不实现它
		unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
		int (*check_flags)(int); // 允许模块检查传递给fcntl()调用的标志
		int (*flock) (struct file *, int, struct file_lock *);
		ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
		ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
		void (*splice_eof)(struct file *file);
		int (*setlease)(struct file *, long, struct file_lock **, void **);
		long (*fallocate)(struct file *file, int mode, loff_t offset,
				  loff_t len);
		void (*show_fdinfo)(struct seq_file *m, struct file *f);
	#ifndef CONFIG_MMU
		unsigned (*mmap_capabilities)(struct file *);
	#endif
		ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
				loff_t, size_t, unsigned int);
		loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
					   struct file *file_out, loff_t pos_out,
					   loff_t len, unsigned int remap_flags);
		int (*fadvise)(struct file *, loff_t, loff_t, int);
		int (*uring_cmd)(struct io_uring_cmd *ioucmd, unsigned int issue_flags);
		int (*uring_cmd_iopoll)(struct io_uring_cmd *, struct io_comp_batch *,
					unsigned int poll_flags);
	} __randomize_layout;
	```

	在进程结构中有个文件表，那个表其实就是 file 结构的指针数组，进程每打开一个文件就会建立一个 file 结构实例，并将其地址放入数组中，最后返回对应的数组下标，就是调用 open 函数返回的那个整数.

它们对应的操作对象分别是[super_operations](https://elixir.bootlin.com/linux/v5.12.9/source/include/linux/fs.h#L2009), [indoe_operations](https://elixir.bootlin.com/linux/v5.12.9/source/include/linux/fs.h#L1930), [dentry_operations](https://elixir.bootlin.com/linux/v5.12.9/source/include/linux/dcache.h#L136), [file_operations](https://elixir.bootlin.com/linux/v5.12.9/source/include/linux/fs.h#L1888), fs只要实现了这4个对象的操作方法即可注册到kernel. 每个对象都包含一组操作方法，用于操作相应的文件系统.

到此为止，有超级块、目录结构、文件索引节点，打开文件的实例，通过四大对象就可以描述抽象出一个文件系统了。而四大对象的对应的操作函数集合，又由具体的文件系统来实现，这两个一结合，一个文件系统的状态和行为都具备了.

文件系统demo见[trfs](https://gitee.com/lmos/cosmos/tree/master/lesson35).

linux fs会为每个文件分配两个数据结构： 索引节点(index node)和目录项(directory entry).

由于索引节点唯一标识一个文件，而目录项记录着文件的名，所以目录项和索引节点的关系是多对一，也就是说，一个文件可以有多个别字. 比如，硬链接的实现就是多个目录项中的索引节点指向同一个文件, inode中也记录了一个链接数nlink, 表示有多少个目录项指向了该inode. 对于删除操作, 只有当所有指向这个inode的全部硬链接都被删除时, 这个inode及其数据才会被删除.

注意，目录也是文件，也是用索引节点唯一标识，和普通文件不同的是，普通文件在磁盘里面保存的是文件数据，而目录文件在磁盘里面保存子目录或文件.

![索引节点、目录项以及文件数据的关系](/misc/img/fs/EzAVba.png!web.png)

不可能把超级块和索引节点区全部加载到内存，这样内存肯定撑不住，所以只有当需要使用的时候，才将其加载进内存，它们加载进内存的时机是不同的：
- 超级块：当文件系统挂载时进入内存
- 索引节点区：当文件被访问时进入内存

![Linux 文件系统中，用户空间、系统调用、虚拟机文件系统、缓存、文件系统以及存储之间的关系](/misc/img/fs/6RJRJjJ.png!web.png)

Linux 支持的文件系统也不少，根据存储位置的不同，可以把文件系统分为三类：
- 磁盘的文件系统 : 它是直接把数据存储在磁盘中，比如 Ext4、XFS、Btrfs 等都是这类文件系统
- 内存的文件系统 : 这类文件系统的数据不是存储在硬盘的，而是占用内存空间，经常用到的 /proc 和 /sys 文件系统都属于这一类，读写这类文件，实际上是读写内核中相关的数据数据
- 网络的文件系统 : 用来访问其他计算机主机数据的文件系统，比如 NFS、SMB 等等

文件系统从逻辑上可以分为两部分:
1. 虚拟文件系统(Virtual File System, VFS)
1. 挂载到VFS的实际文件系统

**文件系统首先要先挂载到某个目录才可以正常使用**，比如 Linux 系统在启动时，会把文件系统挂载到根目录.

## fs_context
ref:
- [sock文件系统](https://blog.csdn.net/sinat_34467747/article/details/109169108)

fs_context 引入了一种新的super block创建方式. 这个patch在2019年(linux 5.x)被合入，老的接口仍然可以兼容使用，但是目前大部分文件系统实现已经切换到了fs_context的新方式，见[VFS: Introduce filesystem context](https://lwn.net/Articles/780267/), [Filesystem Mount API](https://www.kernel.org/doc/html/latest/filesystems/mount_api.html)

## mount
参考:
- [EADME - 计算机专业性文章及回答总索引#新一代VFS mount系统调用](https://zhuanlan.zhihu.com/p/67686817)

```c
// https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/mount.h#L70
// 文件系统的挂载信息
struct vfsmount {
	struct dentry *mnt_root;	/* root of the mounted tree */ // 指向这个fs根目录的dentry
	struct super_block *mnt_sb;	/* pointer to superblock */ // 指向这个fs根目录的超级块
	int mnt_flags; // 装载标志
	struct mnt_idmap *mnt_idmap;
} __randomize_layout;

// https://elixir.bootlin.com/linux/v6.6.18/source/fs/namespace.c#L71
static struct hlist_head *mount_hashtable __read_mostly; // 除根vfsmount外, 所有vfsmount都会加入全局哈希表mount_hashtable, 其用于方便查找装载到特定装载点的fs
```

当一个fs被挂载时, 它的[vfsmount](https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/mount.h#L70)被链接到了kernel的一个全局链表[mount_hashtable](https://elixir.bootlin.com/linux/v5.12.9/source/fs/namespace.c#L70). mount_hashtable是一个数组, 它的每个成员都是一个hash链表.

当发现目录是一个挂载点时, 会从mount_hashtable中找到该fs的vfsmount, 然后挂载点目录的dentry会被替换为被挂载fs的root dentry.

### 新内核mount
参考:
- [深入理解 Linux 文件系統之文件系統掛載](https://webcache.googleusercontent.com/search?q=cache:EX2JdZE_xJgJ:https://www.readfog.com/a/1637370894679642112+&cd=3&hl=zh-CN&ct=clnk)
- [文件系统(六)—文件系统mount过程](https://blog.csdn.net/u012489236/article/details/124523247)
	使用fs_context前的mount


```
[mount即sys_mount](https://elixir.bootlin.com/linux/v6.6.18/source/fs/namespace.c#L3872)
-> [do_mount](https://elixir.bootlin.com/linux/v6.6.18/source/fs/namespace.c#L3677)
   -> [path_mount](https://elixir.bootlin.com/linux/v6.6.18/source/fs/namespace.c#L3598)
      -> [do_new_mount](https://elixir.bootlin.com/linux/v6.6.18/source/fs/namespace.c#L3299)

[mount即sys_mount](https://elixir.bootlin.com/linux/v5.12.9/source/fs/namespace.c#L3431)
-> [do_mount](https://elixir.bootlin.com/linux/v5.12.9/source/fs/namespace.c#L3237)
   -> [path_mount](https://elixir.bootlin.com/linux/v5.12.9/source/fs/namespace.c#L3158)
      -> [do_new_mount](https://elixir.bootlin.com/linux/v5.12.9/source/fs/namespace.c#L2862)

do_new_mount
-> type = get_fs_type(fstype)  // 根据已注册fs的名称查找fs
-> fc = fs_context_for_mount(type, sb_flags) //为需挂载的fs分配fs上下文 struct fs_context
 -> alloc_fs_context
   -> 分配fs_context fc = kzalloc(sizeof(struct fs_context), GFP_KERNEL)
   ->  设置 ...
   ->  fc->fs_type     = get_filesystem(fs_type);  // 赋值相应的fs类型
   ->  init_fs_context = **fc->fs_type->init_fs_context**;  //新內核使用fs_type->init_fs_context接口来初始化文件系统上下文
    if (!init_fs_context)   //init_fs_context回调, 主要用于初始化
        init_fs_context = **legacy_init_fs_context**;    //沒有 fs_type->init_fs_context接口 
   -> init_fs_context(fc)  //初始化fs上下文 (比如初始化一些回调函数共以后使用)
-> parse_monolithic_mount_data(fc, data)  //调用fc->ops->parse_monolithic  解析挂载选项
-> mount_capable(fc) //检查是否有挂载权限
-> vfs_get_tree(fc)  //fs/super.c 挂载重点, 调用fc->ops->get_tree(fc) `set fc->root`并创建super_block实例
-> do_new_mount_fc(fc, path, mnt_flags)  //创建mount实例, 关联挂载点和super_block, 添加到命名空间的挂载树中
   -> do_add_mount(real_mount(mnt), mp, mountpoint, mnt_flags) // 把源fs挂载到目的fs
      -> graft_tree(newmnt, parent, mp) // 把源fs的dentry树与目的fs的dentry树嫁接到一起
        -> attach_recursive_mnt(mnt, p, mp, false) // 执行挂载操作
          -> mnt_set_mountpoint(dest_mnt, dest_mp, source_mnt)
          -> commit_tree(source_mnt) // 把源vfsmount提交到全局hash链表
```

对应没有实现init_fs_context接口情况:
```
//fs/fs_context.c
init_fs_context = legacy_init_fs_context
->  fc->ops = &legacy_fs_context_ops   // 设置fs上下文操作
                    ->.get_tree               = legacy_get_tree  // get_tree用于读取磁盘superblock并在内存创建super_block, root dentry和root inode.
                        -> root = fc->fs_type->mount(fc->fs_type, fc->sb_flags,
                                         ¦     fc->source, ctx->legacy_data)  // 调用fs的mount方法创建super_block
                        -> fc->root = root


有一些文件系统使用原來的接口(fs_type.mount  = xxx_mount)：如ext2,ext4等
有一些文件系统使用新的接口(fs_type.init_fs_context =  xxx_init_fs_context)：xfs， proc， sys, tmpfs

无论使用哪一种, 都会在xxx_init_fs_contex中实现fc->ops =  &xxx_context_ops 接口, 后面也都会调用fc->ops.get_tree来创建super_block
```

```c
// https://elixir.bootlin.com/linux/v6.6.28/source/fs/namespace.c#L3299
/*
 * create a new mount for userspace and request it to be added into the
 * namespace's tree
 */
static int do_new_mount(struct path *path, const char *fstype, int sb_flags,
			int mnt_flags, const char *name, void *data)
{
	struct file_system_type *type;
	struct fs_context *fc;
	const char *subtype = NULL;
	int err = 0;

	if (!fstype)
		return -EINVAL;

	type = get_fs_type(fstype); // 找到对应的file_system_type对象
	if (!type)
		return -ENODEV;

	if (type->fs_flags & FS_HAS_SUBTYPE) {
		subtype = strchr(fstype, '.');
		if (subtype) {
			subtype++;
			if (!*subtype) {
				put_filesystem(type);
				return -EINVAL;
			}
		}
	}

	fc = fs_context_for_mount(type, sb_flags);
	put_filesystem(type);
	if (IS_ERR(fc))
		return PTR_ERR(fc);

	/*
	 * Indicate to the filesystem that the mount request is coming
	 * from the legacy mount system call.
	 */
	fc->oldapi = true;

	if (subtype)
		err = vfs_parse_fs_string(fc, "subtype",
					  subtype, strlen(subtype));
	if (!err && name)
		err = vfs_parse_fs_string(fc, "source", name, strlen(name));
	if (!err)
		err = parse_monolithic_mount_data(fc, data);
	if (!err && !mount_capable(fc))
		err = -EPERM;
	if (!err)
		err = vfs_get_tree(fc);
	if (!err)
		err = do_new_mount_fc(fc, path, mnt_flags);

	put_fs_context(fc);
	return err;
}

int finish_automount(struct vfsmount *m, const struct path *path)
{
	struct dentry *dentry = path->dentry;
	struct mountpoint *mp;
	struct mount *mnt;
	int err;

	if (!m)
		return 0;
	if (IS_ERR(m))
		return PTR_ERR(m);

	mnt = real_mount(m);
	/* The new mount record should have at least 2 refs to prevent it being
	 * expired before we get a chance to add it
	 */
	BUG_ON(mnt_get_count(mnt) < 2);

	if (m->mnt_sb == path->mnt->mnt_sb &&
	    m->mnt_root == dentry) {
		err = -ELOOP;
		goto discard;
	}

	/*
	 * we don't want to use lock_mount() - in this case finding something
	 * that overmounts our mountpoint to be means "quitely drop what we've
	 * got", not "try to mount it on top".
	 */
	inode_lock(dentry->d_inode);
	namespace_lock();
	if (unlikely(cant_mount(dentry))) {
		err = -ENOENT;
		goto discard_locked;
	}
	if (path_overmounted(path)) {
		err = 0;
		goto discard_locked;
	}
	mp = get_mountpoint(dentry);
	if (IS_ERR(mp)) {
		err = PTR_ERR(mp);
		goto discard_locked;
	}

	err = do_add_mount(mnt, mp, path, path->mnt->mnt_flags | MNT_SHRINKABLE);
	unlock_mount(mp);
	if (unlikely(err))
		goto discard;
	mntput(m);
	return 0;

discard_locked:
	namespace_unlock();
	inode_unlock(dentry->d_inode);
discard:
	/* remove m from any expiration list it may be on */
	if (!list_empty(&mnt->mnt_expire)) {
		namespace_lock();
		list_del_init(&mnt->mnt_expire);
		namespace_unlock();
	}
	mntput(m);
	mntput(m);
	return err;
}

// https://elixir.bootlin.com/linux/v6.6.28/source/fs/namespace.c#L3225
/*
 * add a mount into a namespace's mount tree
 */
static int do_add_mount(struct mount *newmnt, struct mountpoint *mp,
			const struct path *path, int mnt_flags)
{
	struct mount *parent = real_mount(path->mnt);

	mnt_flags &= ~MNT_INTERNAL_FLAGS;

	if (unlikely(!check_mnt(parent))) {
		/* that's acceptable only for automounts done in private ns */
		if (!(mnt_flags & MNT_SHRINKABLE))
			return -EINVAL;
		/* ... and for those we'd better have mountpoint still alive */
		if (!parent->mnt_ns)
			return -EINVAL;
	}

	/* Refuse the same filesystem on the same mount point */
	if (path->mnt->mnt_sb == newmnt->mnt.mnt_sb && path_mounted(path))
		return -EBUSY;

	if (d_is_symlink(newmnt->mnt.mnt_root))
		return -EINVAL;

	newmnt->mnt.mnt_flags = mnt_flags;
	return graft_tree(newmnt, parent, mp);
}
```

do_new_mount_fc 完 成 两 个 任 务 , 首 先 调 用 lock_mount 找 到mountpoint,然后调用do_add_mount里的graft_tree完善mount的关系.

lock_mount是一个循环,不断调用以path作为参数调用lookup_mnt,lookup_mnt判断是否有挂载到(path->mnt,path->dentry)上
的文件系统(记mount为m,遍历检查mount_hashtable对应链表上的元素, 满足 &m->mnt_parent->mnt == path->mnt &&m-
>mnt_mountpoint==path->dentry即找到),如果找到,返回该vfsmount信息,继续查找是否有挂载到它上的文件系统。如此反复,直到找不到为止

mountpoint的核心字段是m_dentry和m_count,分别表示挂载的dentry 和 dentry 被 mount的次数。get_mountpoint判断dentry对应的mountpoint是否已经存在,如果存在直接返回,否则创建一个新的并将其初始化返回.

得到了mountpoint之后,graft_tree就可以建立mount与mountpoint和它的父mount的关系了.

总结,mount包括两个步骤,第一步获得super_block和root. 第一步完成之后,文件系统并不可直接访问,需要调用do_add_mount将mount对象挂载到它的parent上.

挂载的反操作是卸载,umount执行mount的相反操作.

## 查找文件
查找某个路径一般包括三步:设置起点:查找中间路径和处理`尾巴`(最后一个path elem),对应的函数一般为path_init、link_path_walk和xxx_last.

```c
// https://elixir.bootlin.com/linux/v6.6.28/source/fs/namei.c#L2471
/* Returns 0 and nd will be valid on success; Retuns error, otherwise. */
static int path_lookupat(struct nameidata *nd, unsigned flags, struct path *path) // nd, 工作目录, 一般是为AT_FDCWD(Current Work Directory, 当前工作目录, -100), 也可以是一个正整数,表示fd(文件描述符),并没有AT_FDROOT,路径以‘/’字符开始即表示root. nd->name->name字段表示路径名
{
	const char *s = path_init(nd, flags); // 设置查找的起点,给nd与文件相关的字段赋初值
	int err;

	if (unlikely(flags & LOOKUP_DOWN) && !IS_ERR(s)) {
		err = handle_lookup_down(nd);
		if (unlikely(err < 0))
			s = ERR_PTR(err);
	}

	while (!(err = link_path_walk(s, nd)) &&
	       (s = lookup_last(nd)) != NULL)
		;
	if (!err && unlikely(nd->flags & LOOKUP_MOUNTPOINT)) {
		err = handle_lookup_down(nd);
		nd->state &= ~ND_JUMPED; // no d_weak_revalidate(), please...
	}
	if (!err)
		err = complete_walk(nd);

	if (!err && nd->flags & LOOKUP_DIRECTORY)
		if (!d_can_lookup(nd->path.dentry))
			err = -ENOTDIR;
	if (!err) {
		*path = nd->path;
		nd->path.mnt = NULL;
		nd->path.dentry = NULL;
	}
	terminate_walk(nd);
	return err;
}
```

查找标志:
- LOOKUP_FOLLOW: follow link
- LOOKUP_DIRECTORY: 查找的是目录
- LOOKUP_PARENT: 查找中间路径, 不处理`尾巴`
- LOOKUP_REVAL: 存储的dentry不可信, 进行real lookup
- LOOKUP_RCU: RCU查找

与RCU查找对应的是REF查找,官方名字分别为rcu-walk和ref-walk,二者的区别在于查找过程中使用的同步机制不同,前者使用rcu
同步机制,不需要使用lock,后者需要使用lock。LOOKUP_PARENT比较常用,比如“mkdira/b/d”,d应该是不存在的,所以只需要path_lookupat查找到`a/b`即可.

path_init并没有改变查找的路径,它只设置了查找的起点. 进入link_path_walk函数前,nd的path和inode等字段已经指向了查找的起点文件。以‘/’开始的路径名,nd此时表示的是“/”,否则可以理解
为“.”(当前目录)。

link_path_walk可以用来查找中间的路径,比如`a/b/c`中的`a/b`.

link_path_walk在while循环中遍历中间路径上的每一个单元,调用walk_component依次查找它们。walk_component失败时返回表示错误的负值,成功时返回值大于等于零,等于零表示查找一个单元结束,
大于零则表示查找到的文件是一个符号链接,需要在下一次循环中处理。

> 非链接文件的情况: nd->depth始终等于0

walk_component用来查找路径的中间单元.

walk_component的第一个参数nd的path字段表示上一次查找的结果(第一次调用时由path_init指定),也就是本轮查找的dentry的
parent。我们在文件系统的数据结构一节说过,dentry可以表示文件系统的层级结构,所以可以先根据dentry来查找目标文件,这就是
lookup_fast做的事情。

但是dentry并不是一开始就存在于内存中的,比如目录a下,有b、c和d三个文件,访问过a/b之后,b的dentry就在内存中了,下次可以不
用在文件系统内部查找,直接查找dentry就可以找到b了,但如果查找的是c,靠dentry是不够的。

dentry查不到的文件,并不意味着不存在,这就需要深入文件系统内部查找了,是lookup_slow做的事情。如果lookup_fast和
lookup_slow都没有找到文件,那就表示目标文件不存在.

lookup_fast顾名思义就是快速查找,它根据当前存在的dentry组成的层级结构来查找目标文件. lookup_fast主要包括lookup(第1步)和follow(第2步)两个
逻辑.

lookup的任务是在parent(nd->path.dentry)下按照名字查找要找的dentry(前面介绍过,dentry在dentry_hashtable指向的某个哈希
链表上)。如果没有找到dentry,返回0,表示在dentry层级结构中找不到文件。如果找到了dentry,并不能立即返回,因为它有可能是某
个文件系统的挂载点,而被隐藏.

lookup_fast如果查找不到dentry,返回值等于0,接下来就由lookup_slow继续查找。它的流程的主要逻辑分为两部分:在dentry的
层级结构中再次尝试查找目标;调用d_alloc申请新的dentry,调用parent(nd->last)的inode提供的lookup回调函数深入文件系统内部查
找。lookup_slow找到文件后,walk_component会在第4步调用
follow_managed“拆墙”。

walk_component调用lookup_fast和lookup_slow找到文件、“拆墙”,找不到文件函数出错返回;找到文件,如果文件是一个符号链接
(symlink),返回1由link_path_walk继续处理,否则调用path_to_nameidata为nameidata对象赋值作为本次查找的结果,返回0。
如果walk_component返回0,link_path_walk以得到的nameidata继续查找下一个单元;如果大于0,说明查找到的文件是符号链接,调用get_link查找它链接的文件.

经过了查找、“拆墙”和符号链接处理后,一个单元的查找终于结束,link_path_walk继续以该单元为parent查找下一个单元,直到倒数
第二个单元,并将查找结果返回,由path_lookupat处理“尾巴”(最后一个单元).

如果用户需要path_lookupat 处 理 “ 尾 巴 ” ( LOOKUP_PARENT 清零),它会负责查找最后一个单元,查找的逻辑与link_path_walk的一轮查找并没有本质区别, 没有合并是为了复用.

## mkdir
mkdir和mkdirat系统调用用来创建目录,入口分别是sys_mkdir和sys_mkdirat, 都调用do_mkdirat实现, 主要逻辑是调用user_path_create函数返回一个新的dentry,然后调用vfs_kdir深入文件系统创建目录
并为dentry赋值.

user_path_create调用filename_create函数,后者先查找目标目录的父目录(lookup),然后在父目录下查找文件,文件如果存在则返回-
EEXIST,否则创建一个新的dentry,将其返回。

记父目录的inode为parent,vfs_mkdir先判断parent->i_op->mkdir是否为空,如果为空则表明文件系统不支持mkdir操作,返回-EPERM,
否则调用parent->i_op->mkdir由文件系统创建目录并为dentry赋值.

rmdir,也是系统调用,入口为sys_rmdir,由do_rmdir函数实现. 它先调用filename_parentat找到目标目录的父目录,然后调用`__lookup_hash`找到目录,最后调用vfs_rmdir删除目录。与vfs_mkdir类
似,vfs_rmdir先判断parent->i_op->rmdir是否为空,如果为空则表明文件系统不支持rmdir操作,返回-EPERM;否则调用parent->i_op->rmdir删除目录。另外,如果要删除的目录上挂载了文件系统,vfs_rmdir会返回-EBUSY,rmdir失败.

mkdir和rmdir都是先找到文件(前者找parent,后者找parent和目标目录),然后调用parent(inode)的i_op提供的回调函数完成操作。
整个过程中,VFS负责维护dentry组成的层级结构,同时通知文件系统维护其内部的层级结构,二者的桥梁就是inode。所以inode除了表示它对应的文件之外,还是VFS与文件系统沟通的枢纽.

## mknod
mknod用来创建节点.

有mknod和mknodat两个系统调用,入口分别为sys_mknod和sys_mknodat,二者都由do_mknodat实现.

do_mknodat 先 调 用 user_path_create 找 到 需 要 创 建 节 点 所 在 的 目录,并创建文件的dentry,然后创建文件。可以创建的文件包含普通
文件、字符设备文件、块设备文件、FIFO文件和Socket文件,所以所谓的节点也就是这几种文件的代称。

如果目标是普通文件,则调用vfs_create创建它,与open创建普通文件无异。实际上,mknod的初衷并不是创建普通文件,而是后面四
种文件,我们如果需要创建普通文件,使用open更好,不会引起误解。

如果目标是后四种文件,则调用vfs_mknod创建它们,vfs_mknod会先判断parent->i_op->mknod(parent为目录的inode)是否为空,如
果为空则表明文件系统不支持mknod操作,返回-EPERM;否则调用parent->i_op->mknod,创建文件。devtmpfs是使用mknod很好的例子.

## 删除文件
unlink和unlinkat系统调用则用来删除文件,入口分别为sys_unlink和sys_unlinkat。unlinkat可以用来删除目录,它的第三个参数flag的AT_REMOVEDIR标志如果置位,表示删除目录,
会调用do_rmdir(只是套用了rmdir)。删除普通文件由do_unlinkat函数实现,有了以上几种操作的分析,读者应该可以猜到,它无非也是先找到文件,然后调用文件的回调函数。

的确如此,do_unlinkat的主要逻辑是调用filename_parentat找到文件的父目录,然后调用__lookup_hash找到文件,然后调用vfs_unlink深
入文件系统执行unlink操作。

vfs_unlink先判断dir->i_op->unlink是否为空,如果为空则表明文件不支持unlink操作,返回-EPERM;否则调用dir->i_op->unlink由文件系
统,执行unlink操作。如果成功返回,会调用d_delete处理dentry,d_delete会将dentry从哈希链表中删除,这样do_unlinkat调用的dput才会
将dentry彻底删除。

unlink执行完毕后,实际的文件是否还存在呢?unlink删除的是什么,比如我们创建两个硬链接a和b,然后unlink b, vfs_unlink中使用的
是b的dentry,所以对a没有影响,那么被删掉的只是硬链接b。既然unlink删除的只不过是硬链接,文件系统的unlink操作就需要判断硬链
接被删除后文件是否还有硬链接存在,如果有,则不能删除文件,否则就需要将文件删除.

## fcntl
用来控制文件的属性

fcntl函数功能： 
1. 复制一个现有的描述符(cmd=F_DUPFD/F_DUPFD_CLOEXEC). 
2. 获得/设置文件描述符标记(cmd=F_GETFD或F_SETFD). 
3. 获得/设置文件状态标记(cmd=F_GETFL或F_SETFL). 
4. 获得/设置记录锁(cmd=F_GETLK , F_SETLK或F_SETLKW). LK即lock
5. 获得/设置异步I/O所有权(cmd=F_GETOWN或F_SETOWN, F_GETOWN_EX或F_SETOWN_EX, F_GETOWNER_UIDS, F_GETSIG或F_SETSIG). 
6. 通知: F_NOTIFY
7. 修改pipe容量: F_SETPIPE_SZ和F_GETPIPE_SZ
8. Leases: F_GETTLEASE和F_SETTLEASE. 文件状态发生特定变化后, 持有对应lease的进程会收到通知.

常用的是前3种.

F_DUPFD用来复制文件描述符,得到的文件描述符应该不小于arg。实现比较简单,首先调用alloc_fd获取一个不小于arg的文件描述
符,然后调用fd_install建立新的文件描述符和file对象的对应关系。F_DUPFD_CLOEXEC除了F_DUPFD的功能外,还会将新文件描述符
相关的O_CLOEXEC标志(close-on-exec)置位。

注意,O_CLOEXEC是属于文件描述符的,而不是属于file对象的,也就是说每个文件描述符可以自行定义它的标志。进程的文件描
述符的使用状态由位图表示,文件描述符的标志也是由位图表示( task_struct->files->fdt->close_on_exec ) , 位的值为1,表示
O_CLOEXEC置位。

F_GETFD用来获取文件描述符的标志,F_SETFD根据arg的值设置文件描述符的O_CLOEXEC标志,目前文件描述符仅支持O_CLOEXEC一种标志.

F_GETFL获取文件状态标志,F_SETFL用来改变文件状态标志。文件的状态标志由file对象的f_flags字段表示,F_SETFL最终体现在该
字段上,或者使文件产生一些行为,但并不是所有的标志都是有效果的 , 仅 支 持 O_APPEND 、 O_NONBLOCK 、 O_NDELAY 、O_DIRECT、O_NOATIME和FASYNC,其他的标志应该在open的时候
设置。

为什么其他的标志(比如O_SYNC)会被忽略呢?因为用户使用fcntl改变了文件的标志,肯定期待着会改变文件的后续行为,但某些标志可能已经产生了影响,即使更改了这些标志,文件的既有行为可
能也无法完全改变。这种情况下,对用户宣称支持这些标志是不明智的,即使某些情况下可能成功.

## ioctl
用来控制设备的IO等操作

ioctl也是内核提供的系统调用,入口是sys_ioctl。sys_ioctl调用do_vfs_ioctl 实 现 , 它 先 处 理 内 核 定 义 的 普 遍 适 用 的 cmd , 比 如
FIOCLEX将文件描述符的close_on_exec置位;然后调用vfs_ioctl处理文 件 的 ioctl 定 义 专 属 的 命 令 , vfs_ioctl 函 数 调 用 filp->f_op->unlocked_ioctl完成.

ioctl命令需要保证它们的唯一性,而且内核和应用程序使用的同一个命令的值必须相等,二者包含同一份头文件可以满足这个条件。另外,内核提供了`_IO 、_IOR、_IOW和_IOWR`等宏,可以使用它们定义命令来保证唯一性.

## 相关扩展
- 从2.4.10开始, buffer cache不再是一个独立的缓存, 而是被包含在page cache中, 通过page cache来实现.

- 脏页(dirty page): 磁盘上的数据与内存中的数据不一致, 此时内存中的数据就是脏页, 需要尽快同步到磁盘中, 以防止意外丢失.

- [块层](https://zhuanlan.zhihu.com/p/25096747)处理所有与块设备操作相关的活动, 其关键数据结构是bio(block input output)结构. 它处理bio请求, 并放入`i/o`请求队列.
bio结构是在文件系统层和块层之间的一个接口.

- 如果一次IO操作起始的逻辑块地址logical block address （LBA）紧挨着上一次IO操作的终止 LBA，就是顺序访问，否则就是随机访问.

## 文件的使用
```c
// write 的过程
fd = open(name, flag); # 打开文件
...
write(fd,...);         # 写数据
...
close(fd);             # 关闭文件
```

读取一个文件的过程：
1. 用 open 系统调用打开文件， open 的参数中包含文件的路径名和文件名
1. 使用 write 写数据，其中 write 使用 open 所返回的 文件描述符 ，并不使用文件名作为参数
1. 使用完文件后，要用 close 系统调用关闭文件，避免资源的泄露

当应用打开了一个文件后，os就会为每个进程维护一个打开文件表，文件表里的每一项即"文件描述符"，所以说文件描述符是打开文件的标识.

操作系统在打开文件表中维护着打开文件的状态和信息：
- 文件指针：系统跟踪上次读写位置作为当前文件位置指针，这种指针对打开文件的某个进程来说是唯一的
- 文件打开计数器：文件关闭时，操作系统必须重用其打开文件表条目，否则表内空间不够用。因为多个进程可能打开同一个文件，所以系统在删除打开文件条目之前，必须等待最后一个进程关闭文件，该计数器跟踪打开和关闭的数量，当该计数为 0 时，系统关闭文件，删除该条目；
- 文件磁盘位置：绝大多数文件操作都要求系统修改文件数据，该信息保存在内存中，以免每个操作都从磁盘中读取；
- 访问权限：每个进程打开文件都需要有一个访问模式（创建、只读、读写、添加等），该信息保存在进程的打开文件表中，以便操作系统能允许或拒绝之后的 I/O 请求

在用户视角里，文件就是一个持久化的数据结构，但操作系统并不会关心用户想存在磁盘上的任何的数据结构，操作系统的视角是如何把文件数据和磁盘块对应起来.

所以，用户和操作系统对文件的读写操作是有差异的，用户习惯以字节的方式读写文件，而操作系统则是以数据块来读写文件，那屏蔽掉这种差异的工作就是文件系统了.

分别看一下，读文件和写文件的过程：
- 当用户进程从文件读取 1 个字节大小的数据时，文件系统则需要获取字节所在的数据块，再返回数据块对应的用户进程所需的数据部分
- 当用户进程把 1 个字节大小的数据写进文件时，文件系统则找到需要写入数据的数据块的位置，然后修改数据块中对应的部分，最后再把数据块写回磁盘

即**kernel只能基于块来访问fs, 块也被成为fs的最小寻址单位**.

在 x86_64 架构里，open 函数会执行 syscall 指令，从用户态转换到内核态，并且最终调用到 do_sys_open 函数，然进而调用 do_sys_openat2 函数.

## 文件的存储
文件的数据是要存储在硬盘上面的，数据在磁盘上的存放方式，就像程序在内存中存放的方式那样，有以下两种：
- 连续空间存放方式
- 非连续空间存放方式, **推荐**

其中，非连续空间存放方式又可以分为"链表方式"和"索引方式".

![不同的文件存储方式](/misc/img/fs/6reQjy7.png!web.png)

不同的存储方式，有各自的特点，重点是要分析它们的存储效率和读写性能，接下来分别对每种存储方式说一下.
### 连续空间存放方式
连续空间存放方式顾名思义，文件所在数据块的LBA是连续的, 但物理空间可能不连续, 比如磁盘是HDD的情况, 相关资料可查看[磁盘的交错因子Interleave-Factor](https://baike.baidu.com/item/%E4%BA%A4%E9%94%99%E5%9B%A0%E5%AD%90/4327559). 这种模式下，文件的数据都是紧密相连，读写效率很高 ，因为一次磁盘寻道就可以读出整个文件.

使用连续存放的方式有一个前提，必须先知道一个文件的大小，这样文件系统才会根据文件的大小在磁盘上找到一块连续的空间分配给文件.

所以，文件头(这里说的文件头是指该种存储方式时Linux inode的实现)里需要指定"起始块的位置"和"长度"，有了这两个信息就可以很好的表示文件存放方式是一块连续的磁盘空间.

连续空间存放的方式虽然读写效率高，但是有**"磁盘空间碎片"和"文件长度不易扩展"的缺陷**.

### 非连续空间存放方式
非连续空间存放方式分为"链表方式"和"索引方式".

#### 1. 链表方式
链表的方式存放是 离散的，不用连续的 ，于是就可以 消除磁盘碎片 ，可大大提高磁盘空间的利用率，同时 文件的长度可以动态扩展. 根据实现的方式的不同，链表可分为"隐式链表"和"显式链接 "两种形式:
- 隐式链表

	文件头要包含「第一块」和「最后一块」的位置，并且每个数据块里面留出一个指针空间，用来存放下一个数据块的位置 ，这样一个数据块连着一个数据块，从链头开是就可以顺着指针找到所有的数据块，所以存放的方式可以是不连续的.

	隐式链表的存放方式的 缺点在于无法直接访问数据块，只能通过指针顺序访问文件，以及数据块指针消耗了一定的存储空间. 隐式链接分配的 稳定性较差，系统在运行过程中由于软件或者硬件错误 导致链表中的指针丢失或损坏，会导致文件数据的丢失.
- 显式链接

	如果从隐式链表中取出每个磁盘块的指针，把它放在内存的一个表中，就可以解决上述隐式链表的两个不足, 这种实现方式是"显式链接. 它指 把用于链接文件各数据块的指针，显式地存放在内存的一张链接表中 ，该表在整个磁盘仅设置一张， 每个表项中存放链接指针，指向下一个数据块号.

	内存中的这样一个表格称为 文件分配表（ File Allocation Table，FAT ）.

	由于查找记录的过程是在内存中进行的，因而不仅显著地 提高了检索速度 ，而且 大大减少了访问磁盘的次数. 但也正是整个表都存放在内存中的关系，它的主要的缺点是 不适用于大磁盘.

#### 2. 索引方式
链表的方式解决了连续分配的磁盘碎片和文件动态扩展的问题，但是不能有效支持直接访问（FAT除外），索引的方式可以解决这个问题.

索引的实现是为每个文件创建一个"索引数据块"，里面存放的是 指向文件数据块的指针列表, 类似书的目录.

另外，文件头需要包含指向"索引数据块"的指针 ，这样就可以通过文件头知道索引数据块的位置，再通过索引数据块里的索引信息找到对应的数据块.

创建文件时，索引块的所有指针都设为空。当首次写入第 i 块时，先从空闲空间中取得一个块，再将其地址写到索引块的第 i 个条目.


索引的方式优点在于：
- 文件的创建、增大、缩小很方便
- 不会有碎片的问题
- 支持顺序读写和随机读写

由于索引数据也是存放在磁盘块的，如果文件很小，明明只需一块就可以存放的下，但还是需要额外分配一块来存放索引数据，所以缺陷之一就是存储索引带来的开销.

如果文件很大，大到一个索引数据块放不下索引信息，这时需要链表 + 索引的组合，这种组合称为"链式索引块"，它的实现方式是 在索引数据块留出一个存放下一个索引数据块的指针 ，于是当一个索引数据块的索引信息用完了，就可以通过指针的方式，找到下一个索引数据块的信息. 那这种方式也会出现前面提到的链表方式的问题，万一某个指针损坏了，后面的数据也就会无法读取了.

其实还有另一种组合: 索引 + 索引的方式，这种组合称为"多级索引块"，实现方式是 通过一个索引块来存放多个索引数据块，一层套一层索引，就像俄罗斯套娃.

#### extN
早期 Unix 文件系统:
![早期 Unix 文件系统](/misc/img/fs/A32emu.png!web.png)

它是根据文件的大小，存放的方式会有所变化：
- 如果存放文件所需的数据块小于 10 块，则采用直接查找的方式
- 如果存放文件所需的数据块超过 10 块，则采用一级间接索引方式
- 如果前面两种方式都不够存放大文件，则采用二级间接索引方式
- 如果二级间接索引也不够存放大文件，这采用三级间接索引方式

那么，文件头（ Inode ）就需要包含 13 个指针：
- 10 个指向数据块的指针；
- 第 11 个指向索引块的指针；
- 第 12 个指向二级索引块的指针；
- 第 13 个指向三级索引块的指针；

所以，这种方式能很灵活地支持小文件和大文件的存放：
- 对于小文件使用直接查找的方式可减少索引数据块的开销；
- 对于大文件则以多级索引的方式来支持，所以大文件在访问数据块时需要大量查询；

这个方案就用在了 Linux Ext 2/3 文件系统里，虽然解决大文件的存储，但是对于大文件的访问，需要大量的查询，效率比较低.

为了解决这个问题，Ext 4 做了一定的改变, 这里就不展开了.

## 空闲空间管理
前面说到的文件的存储是针对已经被占用的数据块组织和管理，接下来的问题是，如果如果要保存一个数据块，如何查找空闲空间. 几种常见的方法是:
- 空闲表法
- 空闲链表法
- 位图法

### 空闲表法
空闲表法就是为所有空闲空间建立一张表，表内容包括空闲区的第一个块号和该空闲区的块个数.

当请求分配磁盘空间时，系统依次扫描空闲表里的内容，直到找到一个合适的空闲区域为止. 当用户撤销一个文件时，系统回收文件空间. 这时，也需顺序扫描空闲表，寻找一个空闲表条目并将释放空间的第一个物理块号及它占用的块数填到这个条目中.

这种方法仅当有少量的空闲区时才有较好的效果. 因为，如果存储空间中有着大量的小的空闲区，则空闲表变得很大，这样查询效率会很低. 另外，这种分配技术适用于建立连续文件.

### 空闲链表法
我们也可以使用"链表"的方式来管理空闲空间，每一个空闲块里有一个指针指向下一个空闲块，这样也能很方便的找到空闲块并管理起来.

当创建文件需要一块或几块时，就从链头上依次取下一块或几块. 反之，当回收空间时，把这些空闲块依次接到链头上.

这种技术只要在主存中保存一个指针，令它指向第一个空闲块. 其特点是简单，但不能随机访问，工作效率低，因为每当在链上增加或移动空闲块时需要做很多 I/O 操作，同时数据块的指针消耗了一定的存储空间.

空闲表法和空闲链表法都不适合用于大型文件系统，因为这会使空闲表或空闲链表太大.

### 位图法
位图是利用二进制的一位来表示磁盘中一个盘块的使用情况，磁盘上所有的盘块都有一个二进制位与之对应: 当值为 0 时，表示对应的盘块空闲，值为 1 时，表示对应的盘块已分配.

在 Linux 文件系统就采用了位图的方式来管理空闲空间，不仅用于数据空闲块的管理，还用于 inode 空闲块的管理，因为 inode 也是存储在磁盘的，自然也要有对其管理.

## 文件系统的结构
前面提到 Linux 是用位图的方式管理空闲空间，用户在创建一个新文件时，Linux 内核会通过 inode 的位图找到空闲可用的 inode，并进行分配. 要存储数据时，会通过块的位图找到空闲的块，并分配，但仔细计算一下还是有问题的.

数据块的位图是放在磁盘块里的，假设是放在一个块里，一个块 4K，每位表示一个数据块，共可以表示 4 * 1024 * 8 = 2^15 个空闲块，由于 1 个数据块是 4K 大小，那么最大可以表示的空间为 2^15 * 4 * 1024 = 2^27 个 byte，也就是 128M.

也就是说按照上面的结构，如果采用"一个块的位图 + 一系列的块"，外加"一个块的 inode 的位图 + 一系列的 inode 的结构"能表示的最大空间也就 128M，这太少了，现在很多文件都比这个大.

在 Linux 文件系统，把这个结构称为一个 块组 ，那么有 N 多的块组，就能够表示 N 大的文件.

## 目录的存储
基于 Linux 一切皆文件的设计思想，目录其实也是个文件，你甚至可以通过 vim 打开它，它也有 inode，inode 里面也是指向一些块.

和普通文件不同的是， 普通文件的块里面保存的是文件数据，而目录文件的块里面保存的是目录里面一项一项的文件信息.

在目录文件的块中，最简单的保存格式就是 列表 ，就是一项一项地将目录下的文件信息（如文件名、文件 inode、文件类型等）列在表里.

列表中每一项就代表该目录下的文件的文件名和对应的 inode，通过这个 inode，就可以找到真正的文件.

![目录格式哈希表](/misc/img/fs/yEjqUf.png!web.png)

通常，第一项是"."，表示当前目录，第二项是".."，表示上一级目录，接下来就是一项一项的文件名和 inode.

如果一个目录有超级多的文件，我们要想在这个目录下找文件，按照列表一项一项的找，效率就不高了. 于是，保存目录的格式改成"哈希表" ，对文件名进行哈希计算，把哈希值保存起来，如果要查找一个目录下面的文件名，可以通过名称取哈希. 如果哈希能够匹配上，就说明这个文件的信息在相应的块里面.

Linux 系统的 ext 文件系统就是采用了哈希表，来保存目录的内容，这种方法的优点是查找非常迅速，插入和删除也较简单，不过需要一些预备措施来避免哈希冲突.

目录查询是通过在磁盘上反复搜索完成，需要不断地进行 I/O 操作，开销较大. 所以，为了减少 I/O 操作，把当前使用的文件目录缓存在内存，以后要使用该文件时只要在内存中操作，从而降低了磁盘操作次数，提高了文件系统的访问速度.

## 软链接和硬链接
有时候我们希望给某个文件取个别名，那么在 Linux 中可以通过 硬链接（ Hard Link ） 和 软链接（ Symbolic Link ） 的方式来实现，它们都是比较特殊的文件，但是实现方式也是不相同的.

硬链接是 多个目录项中的「索引节点」指向一个文件 ，也就是指向同一个 inode，但是 inode 是不可能跨越文件系统的，每个文件系统都有各自的 inode 数据结构和列表，所以 硬链接是不可用于跨文件系统的. 由于多个目录项都是指向一个 inode，那么 只有删除文件的所有硬链接以及源文件时，系统才会彻底删除该文件.

![硬链接](/misc/img/fs/qEbi6nE.png!web.png)

软链接相当于重新创建一个文件，这个文件有 独立的 inode ，但是这个 文件的内容是另外一个文件的路径 ，所以访问软链接的时候，实际上相当于访问到了另外一个文件，所以 软链接是可以跨文件系统的，甚至 目标文件被删除了，链接文件还是在的，只不过指向的文件找不到了而已.

![硬链接](/misc/img/fs/77veiqf.png!web.png)

## 文件 I/O

文件的读写方式各有千秋，对于文件的 I/O 分类也非常多，常见的有:
- 缓冲与非缓冲 I/O
- 直接与非直接 I/O
- 阻塞与非阻塞 I/O VS 同步与异步 I/O

![总结了以上几种 I/O 模型](/misc/img/fs/EbmeIrb.png!web.png)

接下来，分别对这些分类讨论讨论.
### 缓冲与非缓冲 I/O
文件操作的标准库是可以实现数据的缓存，那么 根据"是否利用标准库缓冲"，可以把文件 I/O 分为缓冲 I/O 和非缓冲 I/O ：
- 缓冲 I/O，利用的是标准库的缓存实现文件的加速访问，而标准库再通过系统调用访问文件
- 非缓冲 I/O，直接通过系统调用访问文件，不经过标准库缓存

这里所说的"缓冲"特指标准库内部实现的缓冲.

比方说，很多程序遇到换行时才真正输出，而换行前的内容，其实就是被标准库暂时缓存了起来，这样做的目的是，减少系统调用的次数，毕竟系统调用是有 CPU 上下文切换的开销的.


### 直接与非直接 I/O
我们都知道磁盘 I/O 是非常慢的，所以 Linux 内核为了减少磁盘 I/O 次数，在系统调用后，会把用户数据拷贝到内核中缓存起来，这个内核缓存空间也就是"页缓存"，只有当缓存满足某些条件的时候，才发起磁盘 I/O 的请求.

那么，根据是"否利用操作系统的缓存"，可以把文件 I/O 分为直接 I/O 与非直接 I/O：
- 直接 I/O，不会发生内核缓存和用户程序之间数据复制，而是直接经过文件系统访问磁盘
- 非直接 I/O，读操作时，数据从内核缓存中拷贝给用户程序，写操作时，数据从用户程序拷贝给内核缓存，再由内核决定什么时候写入数据到磁盘

如果在使用文件操作类的系统调用函数时，指定了 O_DIRECT 标志，则表示使用直接 I/O. 如果没有设置过，默认使用的是非直接 I/O.

如果用了非直接 I/O 进行写数据操作，内核什么情况下才会把缓存数据写入到磁盘？

以下几种场景会触发内核缓存的数据写入磁盘：
- 在调用 write 的最后，当发现内核缓存的数据太多的时候，内核会把数据写到磁盘上
- 用户主动调用 sync ，内核缓存会刷到磁盘上
- 当内存十分紧张，无法再分配页面时，也会把内核缓存的数据刷到磁盘上
- 内核缓存的数据的缓存时间超过某个时间时，也会把数据刷到磁盘上

### 阻塞与非阻塞 I/O VS 同步与异步 I/O
为什么把阻塞 / 非阻塞与同步与异步放一起说的呢？因为它们确实非常相似，也非常容易混淆，不过它们之间的关系还是有点微妙的.

先来看看 阻塞 I/O ，当用户程序执行 read ，线程会被阻塞，一直等到内核数据准备好，并把数据从内核缓冲区拷贝到应用程序的缓冲区中，当拷贝过程完成， read 才会返回.

阻塞等待的是"内核数据准备好"和"数据从内核态拷贝到用户态"这两个过程.

![阻塞 I/O](/misc/img/fs/miu6b2f.png!web.png)

非阻塞 I/O, 非阻塞的 read 请求在数据未准备好的情况下立即返回，可以继续往下执行，此时应用程序不断轮询内核，直到数据准备好，内核将数据拷贝到应用程序缓冲区， read 调用才可以获取到结果.

![非阻塞 I/O](/misc/img/fs/2Q7bQn6.png!web.png) 

**注意，上图最后一次 read 调用，获取数据的过程，是一个同步的过程，是需要等待的过程. 这里的同步指的是内核态的数据拷贝到用户程序的缓存区这个过程**.

举个例子，访问管道或 socket 时，如果设置了 O_NONBLOCK 标志，那么就表示使用的是非阻塞 I/O 的方式访问，而不做任何设置的话，默认是阻塞 I/O.

应用程序每次轮询内核的 I/O 是否准备好，感觉有点傻乎乎，因为轮询的过程中，应用程序啥也做不了，只是在循环.

为了解决这种傻乎乎轮询方式，于是 I/O 多路复用 技术就出来了，如 select、poll，它是通过 I/O 事件分发，当内核数据准备好时，再以事件通知应用程序进行操作.

这个做法大大改善了应用进程对 CPU 的利用率，在没有被通知的情况下，应用进程可以使用 CPU 做其他的事情.

下图是使用 select I/O 多路复用过程. 注意， read 获取数据的过程（数据从内核态拷贝到用户态的过程），也是一个 同步的过程 ，需要等待.
![I/O 多路复用](/misc/img/fs/JrEZju.png!web.png)

实际上，无论是阻塞 I/O、非阻塞 I/O，还是基于非阻塞 I/O 的多路复用 都是同步调用. 因为它们在 read 调用时，内核将数据从内核空间拷贝到应用程序空间，过程都是需要等待的，也就是说这个过程是同步的，如果内核实现的拷贝效率不高，read 调用就会在这个同步过程中等待比较长的时间.

而真正的 异步 I/O 是「内核数据准备好」和「数据从内核态拷贝到用户态」这两个过程都不用等待.

当我们发起 aio_read 之后，就立即返回，内核自动将数据从内核空间拷贝到应用程序空间，这个拷贝过程同样是异步的，内核自动完成的，和前面的同步操作不一样，应用程序并不需要主动发起拷贝动作. 过程如下图：
![异步 I/O](/misc/img/fs/juy67nQ.png!web.png)

在前面我们知道了，I/O 是分为两个过程的：
1. 数据准备的过程
1. 数据从内核空间拷贝到用户进程缓冲区的过程

阻塞 I/O 会阻塞在"过程 1"和"过程 2"，而非阻塞 I/O 和基于非阻塞 I/O 的多路复用只会阻塞在"过程 2"，所以这三个都可以认为是同步 I/O.

异步 I/O 则不同，"过程 1"和"过程 2"都不会阻塞.

## stat
```c
// from `man 2 stat`
int stat(const char *pathname, struct stat *statbuf);
int fstat(int fd, struct stat *statbuf);
int lstat(const char *pathname, struct stat *statbuf);

// https://elixir.bootlin.com/linux/v5.8-rc3/source/tools/include/nolibc/nolibc.h
/* The format of the struct as returned by the libc to the application, which
 * significantly differs from the format returned by the stat() syscall flavours.
 */
struct stat {
	dev_t     st_dev;     /* ID of device containing file */
	ino_t     st_ino;     /* inode number */
	mode_t    st_mode;    /* protection */
	nlink_t   st_nlink;   /* number of hard links */
	uid_t     st_uid;     /* user ID of owner */
	gid_t     st_gid;     /* group ID of owner */
	dev_t     st_rdev;    /* device ID (if special file) */
	off_t     st_size;    /* total size, in bytes */
	blksize_t st_blksize; /* blocksize for file system I/O */
	blkcnt_t  st_blocks;  /* number of 512B blocks allocated */
	time_t    st_atime;   /* time of last access */
	time_t    st_mtime;   /* time of last modification */
	time_t    st_ctime;   /* time of last status change */
};
```

函数 stat 和 lstat 返回的是通过文件名查到的状态信息. 这两个方法区别在于，stat 没有处理符号链接（软链接）的能力. 如果一个文件是符号链接，stat 会直接返回它所指向的文件的属性，而 lstat 返回的就是这个符号链接的内容，fstat 则是通过文件描述符获取文件对应的属性.

## minix
```c
// https://elixir.bootlin.com/linux/v6.6.18/source/fs/minix/minix.h#L28
// 内存中的超级块
/*
 * minix super-block data in memory
 */
struct minix_sb_info {
	unsigned long s_ninodes; // i节点数
	unsigned long s_nzones; // 逻辑块数
	unsigned long s_imap_blocks; // i节点位图占用块数
	unsigned long s_zmap_blocks; // 逻辑块位图占用块数
	unsigned long s_firstdatazone; // 数据区中第一个逻辑块号
	unsigned long s_log_zone_size; // log2(磁盘块数/逻辑块)
	int s_dirsize; // 目录项的长度(minix目录项包括文件名和对应inode编号)
	int s_namelen; // 文件名的最大长度
	struct buffer_head ** s_imap; // 指向i节点位图缓冲头指针数组的指针, 数组长度为i节点位图所占块数
	struct buffer_head ** s_zmap; // 指向逻辑块位图缓冲头指针数组的指针, 数组长度为逻辑块位图所占块数 
	struct buffer_head * s_sbh; // 指向超级块缓冲区
	struct minix_super_block * s_ms; // 指向磁盘上超级块. 当使用minix 3.0fs时, 是按minix3_super_block格式读取
	unsigned short s_mount_state;
	unsigned short s_version; // 版本
};

// https://elixir.bootlin.com/linux/v6.6.18/source/include/uapi/linux/minix_fs.h#L82
/*
 * V3 minix super-block data on disk
 */
struct minix3_super_block {
	__u32 s_ninodes; // i节点数
	__u16 s_pad0;
	__u16 s_imap_blocks; // i节点位图所占块数
	__u16 s_zmap_blocks; // 逻辑块位图所占块数 
	__u16 s_firstdatazone; // 数据区中第一个逻辑块号
	__u16 s_log_zone_size; // log2(磁盘块数/逻辑块)
	__u16 s_pad1;
	__u32 s_max_size; // 最大文件长度
	__u32 s_zones; // 逻辑块数
	__u16 s_magic; // fs magic
	__u16 s_pad2;
	__u16 s_blocksize; // 磁盘上逻辑块长度
	__u8  s_disk_version; // no used
};

// https://elixir.bootlin.com/linux/v6.6.18/source/fs/minix/minix.h#L17
// 内存inode
/*
 * minix fs inode data in memory
 */
struct minix_inode_info {
	union {
		__u16 i1_data[16]; // 文件所占用的逻辑块数组. 针对minix2.0
		__u32 i2_data[16]; // 文件所占用的逻辑块数组. 使用前10项, 即从minix2_inode.i_zone[10]复制
	} u;
	struct inode vfs_inode; // 内嵌inode
};

// https://elixir.bootlin.com/linux/v6.6.18/source/include/uapi/linux/minix_fs.h#L51
// 磁盘inode
/*
 * The new minix inode has all the time entries, as well as
 * long block numbers and a third indirect block (7+1+1+1
 * instead of 7+1+1). Also, some previously 8-bit values are
 * now 16-bit. The inode is now 64 bytes instead of 32.
 */
struct minix2_inode {
	__u16 i_mode; // fs的类型和模式
	__u16 i_nlinks; // 链接数
	__u16 i_uid;
	__u16 i_gid;
	__u32 i_size; // 文件长度(B)
	__u32 i_atime;
	__u32 i_mtime;
	__u32 i_ctime;
	__u32 i_zone[10]; // 文件所占用的逻辑块数组: 0~6是直接块号, 7是一次间接块号, 8是二次间接块号, 9是三次间接块号
};

// https://elixir.bootlin.com/linux/v6.6.18/source/include/uapi/linux/minix_fs.h#L103
// 磁盘上的dentry
struct minix3_dir_entry {
	__u32 inode; // i节点编号
	char name[]; // 文件名
};
```

## ramfs
ramfs是基于内存的文件系统，它与 tmpfs 类似，但它更简单、更轻量级.

shmfs 是一个共享内存文件系统，它允许多个进程共享同一块内存. shmfs 通常用于 IPC（进程间通信）.

## ext4
参考:
- [**Ext4文件系统架构分析(一)**](https://www.cnblogs.com/alantu2018/p/8461272.html)
- [**linux io过程自顶向下分析**](https://my.oschina.net/fileoptions/blog/3058792/print)
- [ext2 - <<Linux内核探秘 深入解析文件系统和设备驱动的架构与设计>> 第13章]()
- [EXT4文件系统的磁盘整体布局](https://bean-li.github.io/EXT4-packet-meta-blocks/)

> ext4 dax特性: nvdimm(非易失性双列直插式内存模块=dram+nand+超级电容), 再使用PageCache缓存数据变得累赘, 因此dax不使用缓存而是直接访问设备.

ext4的数据可以分为两类:
1. metadata(元数据,包括文件系统的布局、文件的组织等信息)
1. 文件的内容

ext4按照块(block)为单位管理磁盘,一般情况下,块的大小为4KB. 为了减少碎片,并使一个文件的内容可以落在相邻的块中以便提高访问效率,ext4引入了block group,每个block group包含多个block,其中一个block用来存放它包含的block的使用情况,这个block中每个bit对应一个block,为0说明block空闲,为1
则被占用。所以一个block group最多有4K×8=32,768个block,大小最大为32858×4K=128M.

![](/misc/img/fs/f81bf3e5a6cd060c3225a8ae1803a138.jpeg)

block group布局:
- Group 0 Padding

	GROUP 0 PADDING是第一个block group特有的,它的前1024字节用于存放x86的启动信息等,其他的block group没有padding.

- ext4 super block: 1 block

	包含整个磁盘文件系统的信息,大小为1024字节
- Group Descriptors: many blocks

	包含所有block group的信息,占用的block数目由磁盘大小决定
- Reserved GDT blocks: many blocks

	RESERVED GDT BLOCKS 留作未来扩展文件系统,占多个block
- Data Block Bitmap: 1 block

	存放block group包含的block的使用情况的区域,占1个block
- inode Bitmap: 1 block

	INODE BITMAP与DATA BLOCK BITMAP的作用类似,只不过它描述的是inode的使用情况,占1个block
- inode Table: many blocks

	INODETABLE描述block group内的所有inode的信息,它占用的大小等于block group的inode的数目与inode大小的乘积.

	注意: 这里所说的inode,指的是一个文件在磁盘中的信息,并不是内存中的inode结构体
- Data blocks: many blocks

	存放的是文件的内容

EXT4 SUPER BLOCK和GROUP DESCRIPTORS理论上只需要一份就足够,但为了防止blockgroup 0坏掉而丢失数据,需要保持适当的冗余。如果ext4的sparse_super特性被使能,标号为0,1和3,5,7,9的整数次方的block group会各保留一份ext4 super block和Group Descriptors的拷贝;否则每一个block group都会保留一份.

如果一个block group没有EXT4 SUPER BLOCK 和 GROUP DESCRIPTORS,它直接从DATA BLOCK BITMAP开始.

ext4还有flexible block groups特性,它把几个相邻的block group组成一组,称之为flex_bg. 一个flex_bg中的所有block group的DATA BLOCK BITMAP、INODE BITMAP和INODE TABLE均存放在该flex_bg中的第一个block group中. flex_bg可以让一个flex_bg中除了第一个block group之外大多数可以只包含DATA BLOCKS(有些可能要包含冗余的EXT4 SUPER BLOCK 和 GROUP DESCRIPTORS 等信息),这样可以形成更大的连续的数据块,有利于集中存放大文件或metadata,提高访问效率.

`dumpe2fs <device>`可以用来dump一个ext4文件系统super block 和各block group的信息:
- First inode等于11, 是因为[0到10的inode号都被占用了,对应特殊文件](https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4.h#L310)


ext4特性:
- Meta Block Groups

	block大小为4K的情况下,一个block group最大为128M,假设描述一个block group需要32字节(group descriptor),文件系统所有的group descriptors只能存在一个block group中,所以整个文件系统最大为128M/32 × 128M等于512T字节。

	为了解决这个限制,从ext3就引入了META_BG, Meta BlockGroups 。引入META_BG特性后,整个文件系统会 被分成多个metablock groups,每一个metablock group包含多个block group,它们的group descriptors存储在第一个block group中。这样block group的数目就没有了128M / 32的限制,在32位模式下,文件系统最大为128M × 2^32等于512P字节.

- Lazy Block Group Initialization

	每个block group均包含DATA BLOCK BITMAP、INODE BITMAP和INODE TABLE. 格式化(mkfs)磁盘的时候,需要初始化这些block的数据,但这会使格式化的时间变得漫长.

	Lazy Block Group Initialization可以有效解决这个问题,它实际上只是引入了三个标志: BLOCK_UNINIT 、 INODE_UNINIT 和 INODE_ZEROED:
	- BLOCK_UNINIT标志表示DATABLOCKBITMAP没有被初始化
	- INODE_UNINIT标志表示INODEBITMAP没有被初始化,
	- INODE_ZEROED标志则表示INODETABLE已经初始化(参考ext_group_desc的bg_flags字段)

	磁盘格式化过程中,对绝大多数block group 而言, 将BLOCK_UNINIT和INODE_UNINIT置位,将INODE_ZEROED清零, 就可以跳过1+1+512=514个block的初始化,消耗的时间会大大减少。在后续的使用过程中,内核负责根据情况更新这些标志.

- bigalloc

	块的默认大小是4K,如果一个文件系统是更多的较大文件,那么以多个块(一簇,cluster)为单元管理磁盘可以减少碎片化,也可以减少metadata占用的空间, 所以ext4引入了bigalloc. 用户在格式化磁盘的时候可以设置这个单元的大小(block cluster size),之后DATA BLOCK BITMAP的一位表示一个单元的状态. 当然,申请数据块也以一个单元作为最小单位, 即使文件需要的空间可能很小,甚至文件只是一个目录.


```c
// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4.h#L766
/*
 * Structure of an inode on the disk
 */
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */ // uid的低16位
	__le32	i_size_lo;	/* Size in bytes */ // 文件大小的低32位
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Inode Change time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */ // 删除时间
	__le16	i_gid;		/* Low 16 bits of Group Id */ // gid的低16位
	__le16	i_links_count;	/* Links count */ // 硬连接的数量. 默认下, ext4不超过65000(EXT4_LINK_MAX)个硬链接
	__le32	i_blocks_lo;	/* Blocks count */ // 占用的blk数目的低32位
	__le32	i_flags;	/* File flags */
	union {
		struct {
			__le32  l_i_version;
		} linux1;
		struct {
			__u32  h_i_translator;
		} hurd1;
		struct {
			__u32  m_i_reserved1;
		} masix1;
	} osd1;				/* OS dependent 1 */
	__le32	i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
	__le32	i_generation;	/* File version (for NFS) */
	__le32	i_file_acl_lo;	/* File ACL */
	__le32	i_size_high; // 文件大小的高32位
	__le32	i_obso_faddr;	/* Obsoleted fragment address */
	union {
		struct {
			__le16	l_i_blocks_high; /* were l_i_reserved1 */ // 占用的blk数目的高16位
			__le16	l_i_file_acl_high;
			__le16	l_i_uid_high;	/* these 2 fields */
			__le16	l_i_gid_high;	/* were reserved2[0] */
			__le16	l_i_checksum_lo;/* crc32c(uuid+inum+inode) LE */
			__le16	l_i_reserved;
		} linux2;
		struct {
			__le16	h_i_reserved1;	/* Obsoleted fragment number/size which are removed in ext4 */
			__u16	h_i_mode_high;
			__u16	h_i_uid_high;
			__u16	h_i_gid_high;
			__u32	h_i_author;
		} hurd2;
		struct {
			__le16	h_i_reserved1;	/* Obsoleted fragment number/size which are removed in ext4 */
			__le16	m_i_file_acl_high;
			__u32	m_i_reserved2[2];
		} masix2;
	} osd2;				/* OS dependent 2 */
	__le16	i_extra_isize;
	__le16	i_checksum_hi;	/* crc32c(uuid+inum+inode) BE */
	__le32  i_ctime_extra;  /* extra Change time      (nsec << 2 | epoch) */
	__le32  i_mtime_extra;  /* extra Modification time(nsec << 2 | epoch) */
	__le32  i_atime_extra;  /* extra Access time      (nsec << 2 | epoch) */
	__le32  i_crtime;       /* File Creation time */ // 文件创建时间
	__le32  i_crtime_extra; /* extra FileCreationtime (nsec << 2 | epoch) */
	__le32  i_version_hi;	/* high 32 bits for 64-bit version */
	__le32	i_projid;	/* Project ID */
};
```

ext4_inode结构体描述文件(inode),block group的ext4_inode以数组的形式存放在它的INODE TABLE.

ext4一个目录最多有64998个直接子目录;但如果文件系统支持EXT4_FEATURE_RO_COMPAT_DIR_NLINK (readonly-compatible
feature的一种)特性,对直接子目录不再有数目限制.

i_blocks_lo和i_blocks_high的值最终被转换为512字节大小的块的数目, 如果文件系统没有使能huge_file ( EXT4_FEATURE_RO_COMPAT_HUGE_FILE , readonly-compatible feature的一种)特性,文件占用i_blocks_lo个512字节块; 如果文件系统支持huge_file, 文件本身没有置位EXT4_HUGE_FILE_FL标志(i_flags字段),文件占用i_blocks_lo + (i_blocks_hi << 32)个512字节块;如果EXT4_HUGE_FILE_FL标志被置位,文件占用i_blocks_lo+(i_blocks_hi<<32)个block,以block大小等于4096为例,最终等于(i_blocks_lo+(i_blocks_hi<<32)) × 8个512字节块。以512字节为单位,是因为传统磁盘一个扇区大小为512字节.

ext4中一个文件的inode(为了与VFS的inode结构体区分开,下文称之为inode_on_disk , 它不等同于ext4_inode)占用的空间可以比sizeof(struct ext4_inode)大,也可以比它小(只包含ext4_inode的前128-EXT4_GOOD_OLD_INODE_SIZE个字节的字段), 由ext4_super_block的s_inode_size字段表示,一般为256字节。256个字节中,ext4_inode仅占160个字节(3.10版内核中等于156),其余字节可以留作他用。i_extra_isize表示inode占用的超过128字节的大小,它的典型值等于sizeof (ext4_inode)-128=32。

内核中,访问前128字节之外的字段时,需要先判断该字段是否在128+i_extra_isize范围内,由EXT4_FITS_IN_INODE宏实现,比如访问
i_crtime。这主要是为了兼容老的内核,使用老的内核创建的ext4文件系统可能并没有新版内核定义的某些字段,访问之前需要确保合法。

给定一个inode号为ino的文件,它所属的block group号为(ino - 1) /ext4_super_block->s_inodes_per_group , 它在block group内 的索引号index为(ino - 1) % ext4_super_block->s_inodes_per_group,所以文件的inode_on_disk 在 block group 的 INODE TABLE 的偏移量为index × ext4_super_block->s_inode_size字节. 没有 inode 0.

i_block字段大小为60字节,与文件的内容有关,使用比较复杂. 此处介绍一种最简单的用法:一个符号链接文件,
如果它链接的目标路径长度不超过59字节(最后一个字节为0),就可以将路径存入i_block字段.


从这个数据结构中，可以看出，inode 里面有文件的读写权限 i_mode，属于哪个用户 i_uid，哪个组 i_gid，大小是多少 i_size_io，占用多少个块 i_blocks_io. ls 命令列出来的权限、用户、大小这些信息，就是从这里面取出来的.

另外，这里面还有几个与文件相关的时间: i_atime 是 access time，是最近一次访问文件的时间；i_ctime 是 change time，是最近一次更改 inode 的时间；i_mtime 是 modify time，是最近一次更改文件的时间.

这里需要注意区分几个地方. 首先，访问了，不代表修改了，也可能只是打开看看，就会改变 access time. 其次，修改 inode，有可能修改的是用户和权限，没有修改数据部分，就会改变 change time. **只有数据也修改了，才改变 modify time**. 

磁盘保存文件时常说的“某个文件分成几块、每一块在哪里”，其实这些是在 inode 里面，应该保存在 i_block 里面. EXT4_N_BLOCKS 有如下的定义，计算下来一共有 15 项.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4.h#L398
/*
 * Constants relative to the data blocks
 */
#define	EXT4_NDIR_BLOCKS		12
#define	EXT4_IND_BLOCK			EXT4_NDIR_BLOCKS
#define	EXT4_DIND_BLOCK			(EXT4_IND_BLOCK + 1)
#define	EXT4_TIND_BLOCK			(EXT4_DIND_BLOCK + 1)
#define	EXT4_N_BLOCKS			(EXT4_TIND_BLOCK + 1)
```

![](/misc/img/fs/73349c0fab1a92d4e1ae0c684cfe06e2.jpeg)

在 ext2 和 ext3 中，其中前 12 项直接保存了块的位置，也就是说，可以直接通过 i_block[0-11]，直接得到保存文件内容的块. 对于大文件,可以让 i_block[12]指向一个块，这个块里面不放数据块，而是放数据块的位置，这个块被称为间接块. 也就是说，在 i_block[12]里面放间接块的位置，通过 i_block[12]找到间接块后，间接块里面放数据块的位置，通过间接块可以找到数据块. 如果文件再大一些，i_block[13]会指向一个块，我们可以用二次间接块. 二次间接块里面存放了间接块的位置，间接块里面存放了数据块的位置，数据块里面存放的是真正的数据. 如果文件再大一些，i_block[14]会指向三次间接块. 原理和之前都是一样的，需要一层一层展开才能拿到数据块. 对于大文件来讲，就要多次读取硬盘才能找到相应的块，这样访问速度就会比较慢. 为了解决这个问题，ext4 做了一定的改变. 它引入了一个新的概念，叫做 Extents.

Exents 其实会保存成一棵树:
![](/misc/img/fs/b8f184696be8d37ad6f2e2a4f12d002a.jpeg)

树有一个个的节点，有叶子节点，也有分支节点. 每个节点都有一个头，ext4_extent_header 可以用来描述某个节点.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4_extents.h#L85
/*
 * Each block (leaves and indexes), even inode-stored has header.
 */
struct ext4_extent_header {
	__le16	eh_magic;	/* probably will support different formats */
	__le16	eh_entries;	/* number of valid entries */
	__le16	eh_max;		/* capacity of store in entries */
	__le16	eh_depth;	/* has tree real underlying blocks? */
	__le32	eh_generation;	/* generation of the tree */
};

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4_extents.h#L63
/*
 * This is the extent on-disk structure.
 * It's used at the bottom of the tree.
 */
struct ext4_extent {
	__le32	ee_block;	/* first logical block extent covers */
	__le16	ee_len;		/* number of blocks covered by extent */
	__le16	ee_start_hi;	/* high 16 bits of physical block */
	__le32	ee_start_lo;	/* low 32 bits of physical block */
};

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4_extents.h#L74
/*
 * This is index on-disk structure.
 * It's used at all the levels except the bottom.
 */
struct ext4_extent_idx {
	__le32	ei_block;	/* index covers logical blocks from 'block' */
	__le32	ei_leaf_lo;	/* pointer to the physical block of the next *
				 * level. leaf or next index could be there */
	__le16	ei_leaf_hi;	/* high 16 bits of physical block */
	__u16	ei_unused;
};
```

eh_entries 表示这个节点里面有多少项. 这里的项分两种，如果是叶子节点，这一项会直接指向硬盘上的连续块的地址，称为数据节点 ext4_extent；如果是分支节点，这一项会指向下一层的分支节点或者叶子节点，称为索引节点 ext4_extent_idx. 这两种类型的项的大小都是 12 个 byte.

如果文件不大，inode 里面的 i_block (60B)中，可以放得下一个 ext4_extent_header 和 4 项 ext4_extent(60 = 12 + 12*4). 所以这个时候，eh_depth 为 0，也即 inode 里面的就是叶子节点，树高度为 0. 如果文件比较大，4 个 extent 放不下，就要分裂成为一棵树，eh_depth>0 的节点就是索引节点，其中根节点深度最大，在 inode 中, 最底层 eh_depth=0 的是叶子节点. 除了根节点，其他的节点都保存在一个块 4k 里面，4k 扣除 ext4_extent_header 的 12 个 byte，剩下的能够放 340 项，每个 extent 最大能表示 128MB 的数据，340 个 extent 会使你表示的文件达到 42.5GB. 这已经足够大了，如果再大，还可以增加树的深度.

## inode 位图和块位图
在文件系统里面，有专门的一个块来保存 inode 的位图. 在这 4k 里面，每一位对应一个 inode. 如果是 1，表示这个 inode 已经被用了；如果是 0，则表示没被用. 同样，同样fs也有一个块保存 block 的位图.

[open flags](https://man7.org/linux/man-pages/man2/open.2.html):
- os.O_RDONLY: 以只读的方式打开
- os.O_WRONLY: 以只写的方式打开
- os.O_RDWR : 以读写的方式打开
- os.O_CREAT:  文件不存在, 创建文件
- os.O_EXCL: 若O_CREAT置位, 且文件已存在, 返回-EEXIST
- os.O_TRUNC: 打开一个文件并截断它的长度为零（必须有写权限）
- os.O_APPEND: 以追加写的方式打开
- os.O_NONBLOCK: 打开时不阻塞
- os.O_DSYNC: 写操作等待物理I/O完成, fdatasync
- os.O_SYNC : 写操作等待物理I/O完成, 文件属性更新完毕, fsync
- os.O_DIRECT: 直接I/O, 绕过缓冲, 直接写入文件
- os.O_SHLOCK: 自动获取共享锁
- os.O_EXLOCK: 自动获取独立锁
- os.O_CLOEXEC: close-on-exec, 调用exec成功后, 自动关闭
- os.O_NOFOLLOW: 不追踪软链接
- os.O_DIRECTORY: 如果目标文件不是目录, 打开失败

一个进程打开某个文件,返回的文件描述符具有唯一性,所以内核需要记录哪些文件描述符已被占用,哪些未被使用,由位图实现
(files_struct.fdt->open_fds)。files_struct.next_fd记录着下一个可能可用的文件描述符,在__alloc_fd调用find_next_zero_bit以它作为起始点(也可以指定一个大于它的起始点),在位图中查找下一个可用的文
件描述符.

在用户空间中可以使用文件描述符定位已打开的文件,内核维护了它和文件本身数据结构的对应关系:files_struct.fdt的fd字段是一个file结构体指针的数组,以文件描述符作为下标,即可得到文件的file
对象(current->files->fdt->fd[fd]).

open系统调用的入口为sys_open,后者调用do_sys_open(openat也是如此) . do_sys_open主要完成三个任务,首先调用get_unused_fd_flags(由__alloc_fd实现)获得一个新的可用的文件描
述符,然后调用do_filp_open获得文件的file对象,最后调用fd_install建立文件描述符和file对象的关系.

do_filp_open和它调用的path_openat函数主要逻辑与查找类似,分为三部分,调用path_init设置查找起点,调用link_path_walk查找路径的中间单元,最后调用do_last处理“尾巴”。简单地讲,就是先找到目
标文件的父目录,然后处理文件.

do_last逻辑:
第1步显然是要找到目标文件,如果O_CREAT没有置位,使用lookup_fast查找文件,找到则进入第5步。如果O_CREAT置位,或者lookup_fast没有找到文件(提醒下,没有找到和出错是不同的,返回
值也不同,前者等于0,后者小于0,参考查找一节的介绍),进入lookup_open.

lookup_open先尝试在父目录中查找文件,调用d_lookup在dentry层级结构中查找,记父目录的inode为parent,找不到则调用parent-
>i_op->lookup深入文件系统内部查找,仍未找到则说明文件不存在。如果文件不存在(!dentry->d_inode),且O_CREAT置位,parent-
>i_op->create为空,则表明文件系统不支持create操作,返回-EACCES;否则调用parent->i_op->create由文件系统创建文件并为
dentry赋值。

lookup_open实际上并没有open的动作,主要逻辑只是查找文件,如果文件不存在,需要的情况下,创建文件并返回。

第3步,如果文件是新创建的(file->f_mode|=FMODE_CREATED),那么它不可能是某个文件系统的挂载点
(mountpoint),也不可能是一个符号链接(symlink,由symlink创建,而不是open),就可以跳过中间的步骤到finish_open_created.

如果文件不是新创建的,那么它有可能是某个文件系统的挂载点,就需要调用follow_managed“拆墙”。它也有可能是一个符号链
接,由第4步和第5步处理。如果它是符号链接,step_into返回1,当前do_last结束,path_openat会调用trailing_symlink获得链接的目标文件路
径,然后继续调用link_path_walk和do_last,如此循环,直到找到最终的目标文件.

第6步,vfs_open调用do_dentry_open,它们为file对象赋值,调用文件系统的open回调函数.

f->f_mode的初始值是由open时的flags参数决定的:在path_init之前,由alloc_empty_file函数调用__alloc_file计算得到。至此,do_sys_open的第二个任务完成,接下来它调用fd_install建
立fd和file对象的对应关系, open结束.

close入口为sys_close.

如果file对象的引用数(file->f_count字段)不小于2,说明除了当前文件描述符外,还有其他文件描述符使用它,此时不能将其释放,
将f_count的值减1然后返回。只有当file对象所有的文件描述符都close的情况下,才需要继续3之后的步骤(file对象和文件描述符实际上并
不是1对1的关系)。

flush会执行并等待设备文件完成未完的操作,fasync用于异步通知 。dput判断是否还有其他file对象引用dentry(dentry-
>d_lockref.count>1),如果有,则说明dentry需要继续存在,直接返回;否则根据dentry的状态和文件系统的策略对其处理。

close的逻辑比较简单,但这里有一个值得深入讨论的问题:close之后,没有被引用的文件(dentry->d_lockref.count==0),是否应该
将它的dentry释放?dentry维护了一个层级结构,我们肯定希望尽可能地把它留在内存中,否则下次再次使用文件就需要深入文件系统再查
询一次。

一般在两种情况下删除dentry,第一种是文件被删除(d_unhashed),dentry无效;第二种是dentry->d_op->d_delete返回值
非0。很多文件系统并没有定义d_delete操作,所以close一般不会将dentry删除;如果需要定义d_delete操作,返回值要慎重,否则会给VFS增加负担.

内核提供了link和linkat创建硬链接,它们都是系统调用,入口分别是sys_link和sys_linkat,都通过do_linkat实现.

sys_linkat的逻辑比较清晰,先调用user_path_at找到目标文件,然后调用user_path_create在新的路径下创建dentry,最后调用vfs_link完成link.

vfs_link最终会调用dir->i_op->link,link由文件系统自由实现,主要涉及以下三点.

首先,链接并不是创建文件,文件已经存在,就是old_dentry->d_inode,要做的肯定不是在dir下面创建新的文件,而是让文件“归
属 ” 于 dir 。 其 次 , old_dentry 和 new_dentry 最 终 都 要 与 old_dentry->d_inode 关 联 ( hlist_add_head(&dentry->d_u.d_alias, &inode-
>i_dentry)),可以调用d_instantiate实现。最终,link还需要调整文件的硬链接数目(inode的__i_nlink字段).

创建硬链接有诸多限制,从以上代码段中可以看到,硬链接不能跨文件系统、不能给目录创建硬链接等。

跨文件系统: inode仅在同一个fs保证唯一.

不给目录创建硬连接: 有可能出现闭环链接关系, 这样遍历的时候可能会陷入死循环中.

Linux并没有刻意区分普通文件和目录,普通文件的inode的i_nlink(`__i_nlink与i_nlink`组成一个union,前者写,后者读)
字段表示硬链接数目,目录的inode也有i_nlink字段是直接指向该目录的目录数.

symlink 和 symlinkat系统调用用来创建符号链接,入口分别为sys_symlink和sys_symlinkat,二者都调用do_symlinkat实现.

> 快捷方式本身是一个文件, 它包含目标文件路径信息.

,link_path_walk调用get_link查找符号链接的目标文件的路径,get_link先查看inode->i_link字段,不为空的情
况下直接使用它。否则调用inode->i_op->get_link,得到目标文件的路径。

得到路径后,在接下来的循环中继续调用walk_component找到目标文件,该路径中可能还会存在符号链接,这种情况下继续调用get_link和walk_component,直到找到最终的目标文件.

do_symlinkat调用user_path_create函数返回一个新的dentry,然后调用vfs_symlink深入文件系统,创建符号链接并为dentry赋值。

记 父 目 录 的 inode 为 parent , vfs_symlink 先 判 断 parent->i_op->symlink是否为空,如果为空则表明文件系统不支持symlink操作,返
回-EPERM;否则调用parent->i_op->symlink,由文件系统创建symlink并为dentry赋值。

symlink需要在parent目录下创建一个符号链接类型的文件,并根据文件系统自身的策略将目标文件的路径反映(不一定是保存)出
来,这与快捷方式的创建过程也是类似的。

需要注意的是,在创建符号链接的过程中,并没有验证目标文件是否存在,也就是说可以成功地给一个不存在的文件创建符号链接, 甚至可以先创建符号链接,再创建文件.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/open.c#L1199
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
	return ksys_open(filename, flags, mode);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/syscalls.h#L1383
static inline long ksys_open(const char __user *filename, int flags,
			     umode_t mode)
{
	if (force_o_largefile())
		flags |= O_LARGEFILE;
	return do_sys_open(AT_FDCWD, filename, flags, mode);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/open.c#L1192
long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
	struct open_how how = build_open_how(flags, mode);
	return do_sys_openat2(dfd, filename, &how);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/open.c#L1163
static long do_sys_openat2(int dfd, const char __user *filename,
			   struct open_how *how)
{
	struct open_flags op;
	int fd = build_open_flags(how, &op);
	struct filename *tmp;

	if (fd)
		return fd;

	tmp = getname(filename); // 将文件路径名从用户空间复制到内核空间
	if (IS_ERR(tmp))
		return PTR_ERR(tmp);

	fd = get_unused_fd_flags(how->flags); // 在当前进程的打开文件表中找到一个可用的文件句柄, 并复制给fd
	if (fd >= 0) {
		struct file *f = do_filp_open(dfd, tmp, &op); // 打开文件
		if (IS_ERR(f)) {
			put_unused_fd(fd);
			fd = PTR_ERR(f);
		} else {
			fsnotify_open(f);
			fd_install(fd, f); // 将打开文件安装到打开文件表, 即关联f和fd
		}
	}
	putname(tmp);
	return fd;
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3379
struct file *do_filp_open(int dfd, struct filename *pathname,
		const struct open_flags *op)
{
	struct nameidata nd;
	int flags = op->lookup_flags;
	struct file *filp;

	set_nameidata(&nd, dfd, pathname); // 沿着要打开文件名的整个路径, 一层层解析路径, 最后得到文件的dentry和vfsmount, 再保存到nd中.
	filp = path_openat(&nd, op, flags | LOOKUP_RCU); // 根据nameidata, 获得一个file
	if (unlikely(filp == ERR_PTR(-ECHILD)))
		filp = path_openat(&nd, op, flags);
	if (unlikely(filp == ERR_PTR(-ESTALE)))
		filp = path_openat(&nd, op, flags | LOOKUP_REVAL);
	restore_nameidata();
	return filp;
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3340
static struct file *path_openat(struct nameidata *nd,
			const struct open_flags *op, unsigned flags)
{
	struct file *file;
	int error;

	file = alloc_empty_file(op->open_flag, current_cred());
	if (IS_ERR(file))
		return file;

	if (unlikely(file->f_flags & __O_TMPFILE)) {
		error = do_tmpfile(nd, flags, op, file);
	} else if (unlikely(file->f_flags & O_PATH)) {
		error = do_o_path(nd, flags, file);
	} else {
		const char *s = path_init(nd, flags);
		while (!(error = link_path_walk(s, nd)) && // link_path_walk
		       (s = open_last_lookups(nd, file, op)) != NULL)
			;
		if (!error)
			error = do_open(nd, file, op);
		terminate_walk(nd);
	}
	if (likely(!error)) {
		if (likely(file->f_mode & FMODE_OPENED))
			return file;
		WARN_ON(1);
		error = -EINVAL;
	}
	fput(file);
	if (error == -EOPENSTALE) {
		if (flags & LOOKUP_RCU)
			error = -ECHILD;
		else
			error = -ESTALE;
	}
	return ERR_PTR(error);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3111
static const char *open_last_lookups(struct nameidata *nd,
		   struct file *file, const struct open_flags *op)
{
	struct dentry *dir = nd->path.dentry;
	int open_flag = op->open_flag;
	bool got_write = false;
	unsigned seq;
	struct inode *inode;
	struct dentry *dentry;
	const char *res;
	int error;

	nd->flags |= op->intent;

	if (nd->last_type != LAST_NORM) {
		if (nd->depth)
			put_link(nd);
		return handle_dots(nd, nd->last_type);
	}

	if (!(open_flag & O_CREAT)) {
		if (nd->last.name[nd->last.len])
			nd->flags |= LOOKUP_FOLLOW | LOOKUP_DIRECTORY;
		/* we _can_ be in RCU mode here */
		dentry = lookup_fast(nd, &inode, &seq);
		if (IS_ERR(dentry))
			return ERR_CAST(dentry);
		if (likely(dentry))
			goto finish_lookup;

		BUG_ON(nd->flags & LOOKUP_RCU);
	} else {
		/* create side of things */
		if (nd->flags & LOOKUP_RCU) {
			error = unlazy_walk(nd);
			if (unlikely(error))
				return ERR_PTR(error);
		}
		audit_inode(nd->name, dir, AUDIT_INODE_PARENT);
		/* trailing slashes? */
		if (unlikely(nd->last.name[nd->last.len]))
			return ERR_PTR(-EISDIR);
	}

	if (open_flag & (O_CREAT | O_TRUNC | O_WRONLY | O_RDWR)) {
		error = mnt_want_write(nd->path.mnt);
		if (!error)
			got_write = true;
		/*
		 * do _not_ fail yet - we might not need that or fail with
		 * a different error; let lookup_open() decide; we'll be
		 * dropping this one anyway.
		 */
	}
	if (open_flag & O_CREAT)
		inode_lock(dir->d_inode);
	else
		inode_lock_shared(dir->d_inode);
	dentry = lookup_open(nd, file, op, got_write);
	if (!IS_ERR(dentry) && (file->f_mode & FMODE_CREATED))
		fsnotify_create(dir->d_inode, dentry);
	if (open_flag & O_CREAT)
		inode_unlock(dir->d_inode);
	else
		inode_unlock_shared(dir->d_inode);

	if (got_write)
		mnt_drop_write(nd->path.mnt);

	if (IS_ERR(dentry))
		return ERR_CAST(dentry);

	if (file->f_mode & (FMODE_OPENED | FMODE_CREATED)) {
		dput(nd->path.dentry);
		nd->path.dentry = dentry;
		return NULL;
	}

finish_lookup:
	if (nd->depth)
		put_link(nd);
	res = step_into(nd, WALK_TRAILING, dentry, inode, seq);
	if (unlikely(res))
		nd->flags &= ~(LOOKUP_OPEN|LOOKUP_CREATE|LOOKUP_EXCL);
	return res;
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3003
/*
 * Look up and maybe create and open the last component.
 *
 * Must be called with parent locked (exclusive in O_CREAT case).
 *
 * Returns 0 on success, that is, if
 *  the file was successfully atomically created (if necessary) and opened, or
 *  the file was not completely opened at this time, though lookups and
 *  creations were performed.
 * These case are distinguished by presence of FMODE_OPENED on file->f_mode.
 * In the latter case dentry returned in @path might be negative if O_CREAT
 * hadn't been specified.
 *
 * An error code is returned on failure.
 */
static struct dentry *lookup_open(struct nameidata *nd, struct file *file,
				  const struct open_flags *op,
				  bool got_write)
{
	struct dentry *dir = nd->path.dentry;
	struct inode *dir_inode = dir->d_inode;
	int open_flag = op->open_flag;
	struct dentry *dentry;
	int error, create_error = 0;
	umode_t mode = op->mode;
	DECLARE_WAIT_QUEUE_HEAD_ONSTACK(wq);

	if (unlikely(IS_DEADDIR(dir_inode)))
		return ERR_PTR(-ENOENT);

	file->f_mode &= ~FMODE_CREATED;
	dentry = d_lookup(dir, &nd->last);
	for (;;) {
		if (!dentry) {
			dentry = d_alloc_parallel(dir, &nd->last, &wq);
			if (IS_ERR(dentry))
				return dentry;
		}
		if (d_in_lookup(dentry))
			break;

		error = d_revalidate(dentry, nd->flags);
		if (likely(error > 0))
			break;
		if (error)
			goto out_dput;
		d_invalidate(dentry);
		dput(dentry);
		dentry = NULL;
	}
	if (dentry->d_inode) {
		/* Cached positive dentry: will open in f_op->open */
		return dentry;
	}

	/*
	 * Checking write permission is tricky, bacuse we don't know if we are
	 * going to actually need it: O_CREAT opens should work as long as the
	 * file exists.  But checking existence breaks atomicity.  The trick is
	 * to check access and if not granted clear O_CREAT from the flags.
	 *
	 * Another problem is returing the "right" error value (e.g. for an
	 * O_EXCL open we want to return EEXIST not EROFS).
	 */
	if (unlikely(!got_write))
		open_flag &= ~O_TRUNC;
	if (open_flag & O_CREAT) {
		if (open_flag & O_EXCL)
			open_flag &= ~O_TRUNC;
		if (!IS_POSIXACL(dir->d_inode))
			mode &= ~current_umask();
		if (likely(got_write))
			create_error = may_o_create(&nd->path, dentry, mode);
		else
			create_error = -EROFS;
	}
	if (create_error)
		open_flag &= ~O_CREAT;
	if (dir_inode->i_op->atomic_open) {
		dentry = atomic_open(nd, dentry, file, open_flag, mode);
		if (unlikely(create_error) && dentry == ERR_PTR(-ENOENT))
			dentry = ERR_PTR(create_error);
		return dentry;
	}

	if (d_in_lookup(dentry)) {
		struct dentry *res = dir_inode->i_op->lookup(dir_inode, dentry,
							     nd->flags);
		d_lookup_done(dentry);
		if (unlikely(res)) {
			if (IS_ERR(res)) {
				error = PTR_ERR(res);
				goto out_dput;
			}
			dput(dentry);
			dentry = res;
		}
	}

	/* Negative dentry, just create the file */
	if (!dentry->d_inode && (open_flag & O_CREAT)) {
		file->f_mode |= FMODE_CREATED;
		audit_inode_child(dir_inode, dentry, AUDIT_TYPE_CHILD_CREATE);
		if (!dir_inode->i_op->create) {
			error = -EACCES;
			goto out_dput;
		}
		error = dir_inode->i_op->create(dir_inode, dentry, mode,
						open_flag & O_EXCL);
		if (error)
			goto out_dput;
	}
	if (unlikely(create_error) && !dentry->d_inode) {
		error = create_error;
		goto out_dput;
	}
	return dentry;

out_dput:
	dput(dentry);
	return ERR_PTR(error);
}
```

调用链：do_sys_open -> do_sys_openat2 -> do_filp_open->path_openat->open_last_lookups->lookup_open. 这个调用链的逻辑是，要打开一个文件，先要根据路径找到文件夹. 如果发现文件夹下面没有这个文件，同时又设置了 O_CREAT，就说明我们要在这个文件夹下面创建一个文件，那就需要一个新的 inode.

想要创建新的 inode，就要调用 dir_inode，也就是文件夹的 inode 的 create 函数.

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/namei.c#L4056
/*
 * directories can handle most operations...
 */
const struct inode_operations ext4_dir_inode_operations = {
	.create		= ext4_create,
	.lookup		= ext4_lookup,
	.link		= ext4_link,
	.unlink		= ext4_unlink,
	.symlink	= ext4_symlink,
	.mkdir		= ext4_mkdir,
	.rmdir		= ext4_rmdir,
	.mknod		= ext4_mknod,
	.tmpfile	= ext4_tmpfile,
	.rename		= ext4_rename2,
	.setattr	= ext4_setattr,
	.getattr	= ext4_getattr,
	.listxattr	= ext4_listxattr,
	.get_acl	= ext4_get_acl,
	.set_acl	= ext4_set_acl,
	.fiemap         = ext4_fiemap,
};
```

ext4_dir_inode_operations里面定义了如果文件夹 inode 要做一些操作，每个操作对应应该调用哪些函数. 这里 create 操作调用的是 ext4_create.

接下来的调用链是这样的：[ext4_create](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/namei.c#L2602)->[ext4_new_inode_start_handle](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4.h#L2629)->[__ext4_new_inode](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ialloc.c#L755).在 __ext4_new_inode 函数中，会创建新的 inode.


__ext4_new_inode里面一个重要的逻辑就是，从文件系统里面读取 inode 位图，然后找到下一个为 0 的 inode，就是空闲的 inode. 对于 block 位图，在写入文件的时候，也会有这个过程.

## 文件系统的格式
数据块的位图是放在一个块里面的，共 4k. 每位表示一个数据块，共可以表示 4∗1024∗8=215 个数据块. 如果每个数据块也是按默认的 4K，最大可以表示空间为 215∗4∗1024=227 个 byte，也就是 128M.

如果采用“一个块的位图 + 一系列的块”，外加“一个块的 inode 的位图 + 一系列的 inode 的结构”，最多能够表示 128M, 先把这个结构称为一个块组. 有 N 多的块组，就能够表示 N 大的文件.

对于块组，同样也需要一个数据结构来表示为 [ext4_group_desc](https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4.h#L1289). 这里面对于一个块组里的 inode 位图 bg_inode_bitmap_lo、块位图 bg_block_bitmap_lo、inode 列表 bg_inode_table_lo，都有相应的成员变量.

每一个block group都有一个group descriptor与之对应, 由ext4_group_desc结构体表示。没有使能META_BG的情况下,所有的
group descriptors都按照先后以数组的形式存放在block group的GROUPDESCRIPTORS中,紧挨着EXT4 SUPER BLOCK.

32位模式下,group descriptor大小为32字节;64位模式下,大小为64到1024字节。ext4_group_desc结构体本身大小为64字节,也就是
说32位模式下,读写的只是它的前32字节包含的字段,64位模式下,除了它本身外,还可以包含其他信息. 64位模式下, group descriptor的大小由ext4_super_block的s_desc_size字段表示,32位模式下,该字段一般为0.

```c
// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4.h#L1289
/*
 * Structure of a blocks group descriptor
 */
struct ext4_group_desc
{
	__le32	bg_block_bitmap_lo;	/* Blocks bitmap block */ // block group的block bitmap所在的block号的低32位
	__le32	bg_inode_bitmap_lo;	/* Inodes bitmap block */ // block group的inode bitmap所在的block号的低32位
	__le32	bg_inode_table_lo;	/* Inodes table block */ //  block group的inode table所在的block号的低32位
	__le16	bg_free_blocks_count_lo;/* Free blocks count */ // 空闲的block数目
	__le16	bg_free_inodes_count_lo;/* Free inodes count */ // 空闲的inde数目
	__le16	bg_used_dirs_count_lo;	/* Directories count */ // 目录的数目
	__le16	bg_flags;		/* EXT4_BG_flags (INODE_UNINIT, etc) */
	__le32  bg_exclude_bitmap_lo;   /* Exclude bitmap for snapshots */
	__le16  bg_block_bitmap_csum_lo;/* crc32c(s_uuid+grp_num+bbitmap) LE */
	__le16  bg_inode_bitmap_csum_lo;/* crc32c(s_uuid+grp_num+ibitmap) LE */
	__le16  bg_itable_unused_lo;	/* Unused inodes count */ // 没有被使用的inode table项的数目的低16位
	__le16  bg_checksum;		/* crc16(sb_uuid+group+desc) */
	__le32	bg_block_bitmap_hi;	/* Blocks bitmap block MSB */
	__le32	bg_inode_bitmap_hi;	/* Inodes bitmap block MSB */
	__le32	bg_inode_table_hi;	/* Inodes table block MSB */
	__le16	bg_free_blocks_count_hi;/* Free blocks count MSB */
	__le16	bg_free_inodes_count_hi;/* Free inodes count MSB */
	__le16	bg_used_dirs_count_hi;	/* Directories count MSB */
	__le16  bg_itable_unused_hi;    /* Unused inodes count MSB */
	__le32  bg_exclude_bitmap_hi;   /* Exclude bitmap block MSB */
	__le16  bg_block_bitmap_csum_hi;/* crc32c(s_uuid+grp_num+bbitmap) BE */
	__le16  bg_inode_bitmap_csum_hi;/* crc32c(s_uuid+grp_num+ibitmap) BE */
	__u32   bg_reserved;
};
```
bg_flags字段可以是三种标志的组合:
1. EXT4_BG_INODE_UNINIT(0x1): block group的INODE BITMAP没有初始化
1. EXT4_BG_BLOCK_UNINIT(0x2): BLOCKBITMAP没有初始化
1. EXT4_BG_INODE_ZEROED(0x4): INODE TABLE已经初始化


bg_free_inodes_count_lo和bg_itable_unused_lo的用途不同,前者反映inode的使用情况,后者用在block group的INODE TABLE初始化过程
中 , 如 果 已 使 用 的 项 数 ( ext4_super_block->s_inodes_per_group - bg_itable_unused)为0,只需要将整个INODE TABLE清零即可,否则要跳过已使用项占用的block.


这样一个个块组，就基本构成了我们整个文件系统的结构. 因为块组有多个，块组描述符也同样组成一个列表，我们把这些称为块组描述符表.

当然，还需要有一个数据结构，对整个文件系统的情况进行描述，这个就是超级块[ext4_super_block](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4.h#L1245). 这里面有整个文件系统一共有多少 inode，s_inodes_count；一共有多少块，s_blocks_count_lo，每个块组有多少 inode，s_inodes_per_group，每个块组有多少块，s_blocks_per_group 等. 这些都是这类的全局信息.

如果是一个启动盘，就还需要预留一块区域作为引导区，所以第一个块组的前面要留 1K，用于启动引导区.

最终，整个文件系统格式就是下面这个样子.
![](/misc/img/fs/e3718f0af6a2523a43606a0c4003631b.jpeg)

超级块和块组描述符表都是全局信息，而且这些数据很重要. 如果这些数据丢失了，整个文件系统都打不开了，这比一个文件的一个块损坏更严重. 所以，这两部分都需要备份，但是采取不同的策略.

默认情况下，超级块和块组描述符表都有副本保存在每一个块组里面.

如果开启了 sparse_super 特性，超级块和块组描述符表的副本只会保存在块组索引为 0、3、5、7 的整数幂里. 除了块组 0 中存在一个超级块外，在块组 1（3^0=1）的第一个块中存在一个副本；在块组 3（3^1=3）、块组 5（5^1=5）、块组 7（7^1=7）、块组 9（3^2=9）、块组 25（5^2=25）、块组 27（3^3=27）的第一个 block 处也存在一个副本.

对于超级块来讲，由于超级块不是很大，所以就算备份多了也没有太多问题. 但是，对于块组描述符表来讲，如果每个块组里面都保存一份完整的块组描述符表，一方面很浪费空间；另一个方面，由于一个块组最大 128M，而块组描述符表里面有多少项，这就限制了有多少个块组，128M * 块组的总数目是整个文件系统的大小，就被限制住了.

改进的思路就是引入 Meta Block Groups 特性.

首先，块组描述符表不会保存所有块组的描述符了，而是将块组分成多个组，我们称为元块组（Meta Block Group）. 每个元块组里面的块组描述符表仅仅包括自己的，一个元块组包含 64 个块组，这样一个元块组中的块组描述符表最多 64 项. 假设一共有 256 个块组，原来是一个整的块组描述符表，里面有 256 项，要备份就全备份，现在分成 4 个元块组，每个元块组里面的块组描述符表就只有 64 项了，这就小多了，而且四个元块组自己备份自己的.

![](/misc/img/fs/b0bf4690882253a70705acc7368983b9.jpeg)

根据图中，每一个元块组包含 64 个块组，块组描述符表也是 64 项，备份三份，在元块组的第一个，第二个和最后一个块组的开始处. 这样化整为零，就可以发挥出 ext4 的 48 位块寻址的优势了，在超级块 ext4_super_block 的定义中，我们可以看到块寻址分为高位和低位，均为 32 位，其中有用的是 48 位，2^48 个块是 1EB，足够用了.

ext4_super_block结构体表示文件系统的整体信息,它的对象存储在磁盘第1024字节,也就是布局中的EXT4 SUPER BLOCK部分。磁盘
中 其 他 block group还会存储它的冗余拷贝. ext4_super_block的大小为1024字节.

```c
// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4.h#L1289
/*
 * Structure of the super block
 */
struct ext4_super_block {
/*00*/	__le32	s_inodes_count;		/* Inodes count */
	__le32	s_blocks_count_lo;	/* Blocks count */ // block的低32位
	__le32	s_r_blocks_count_lo;	/* Reserved blocks count */
	__le32	s_free_blocks_count_lo;	/* Free blocks count */ 空闲的block的低32位
/*10*/	__le32	s_free_inodes_count;	/* Free inodes count */ 可用的inode的低32位
	__le32	s_first_data_block;	/* First Data Block */ 前1024个字节存放x86的启动信息, 如果block大小是1k, 应该是1, 其余情况一般是0
	__le32	s_log_block_size;	/* Block size */ // block大小=2^(10+s_log_block_size)
	__le32	s_log_cluster_size;	/* Allocation cluster size */ // bigalloc使能时, cluster大小=2^s_log_cluster_size, 否则其为0
/*20*/	__le32	s_blocks_per_group;	/* # Blocks per group */
	__le32	s_clusters_per_group;	/* # Clusters per group */
	__le32	s_inodes_per_group;	/* # Inodes per group */
	__le32	s_mtime;		/* Mount time */
/*30*/	__le32	s_wtime;		/* Write time */
	__le16	s_mnt_count;		/* Mount count */
	__le16	s_max_mnt_count;	/* Maximal mount count */
	__le16	s_magic;		/* Magic signature */
	__le16	s_state;		/* File system state */
	__le16	s_errors;		/* Behaviour when detecting errors */
	__le16	s_minor_rev_level;	/* minor revision level */
/*40*/	__le32	s_lastcheck;		/* time of last check */
	__le32	s_checkinterval;	/* max. time between checks */
	__le32	s_creator_os;		/* OS */ // linux=0
	__le32	s_rev_level;		/* Revision level */
/*50*/	__le16	s_def_resuid;		/* Default uid for reserved blocks */
	__le16	s_def_resgid;		/* Default gid for reserved blocks */
	/*
	 * These fields are for EXT4_DYNAMIC_REV superblocks only.
	 *
	 * Note: the difference between the compatible feature set and
	 * the incompatible feature set is that if there is a bit set
	 * in the incompatible feature set that the kernel doesn't
	 * know about, it should refuse to mount the filesystem.
	 *
	 * e2fsck's requirements are more strict; if it doesn't know
	 * about a feature in either the compatible or incompatible
	 * feature set, it must abort and not try to meddle with
	 * things it doesn't understand...
	 */
	__le32	s_first_ino;		/* First non-reserved inode */ // 第一个非预留的indode, 一般是11
	__le16  s_inode_size;		/* size of inode structure */ // ext4的inode的大小, 此inode并非VFS定义的inode
	__le16	s_block_group_nr;	/* block group # of this superblock */
	__le32	s_feature_compat;	/* compatible feature set */ // 支持兼容(compatible)特性的标志
/*60*/	__le32	s_feature_incompat;	/* incompatible feature set */ // 支持不兼容(incompatible)特性的标志
	__le32	s_feature_ro_compat;	/* readonly-compatible feature set */ // 支持只读的兼容(readonly-compatible)特性的标志
/*68*/	__u8	s_uuid[16];		/* 128-bit uuid for volume */
/*78*/	char	s_volume_name[EXT4_LABEL_MAX];	/* volume name */
/*88*/	char	s_last_mounted[64] __nonstring;	/* directory where last mounted */
/*C8*/	__le32	s_algorithm_usage_bitmap; /* For compression */
	/*
	 * Performance hints.  Directory preallocation should only
	 * happen if the EXT4_FEATURE_COMPAT_DIR_PREALLOC flag is on.
	 */
	__u8	s_prealloc_blocks;	/* Nr of blocks to try to preallocate*/
	__u8	s_prealloc_dir_blocks;	/* Nr to preallocate for dirs */
	__le16	s_reserved_gdt_blocks;	/* Per group desc for online growth */
	/*
	 * Journaling support valid if EXT4_FEATURE_COMPAT_HAS_JOURNAL set.
	 */
/*D0*/	__u8	s_journal_uuid[16];	/* uuid of journal superblock */
/*E0*/	__le32	s_journal_inum;		/* inode number of journal file */
	__le32	s_journal_dev;		/* device number of journal file */
	__le32	s_last_orphan;		/* start of list of inodes to delete */
	__le32	s_hash_seed[4];		/* HTREE hash seed */
	__u8	s_def_hash_version;	/* Default hash version to use */
	__u8	s_jnl_backup_type;
	__le16  s_desc_size;		/* size of group descriptor */ // 64bit模式下, group descriptor的大小
/*100*/	__le32	s_default_mount_opts;
	__le32	s_first_meta_bg;	/* First metablock block group */
	__le32	s_mkfs_time;		/* When the filesystem was created */
	__le32	s_jnl_blocks[17];	/* Backup of the journal inode */
	/* 64bit support valid if EXT4_FEATURE_INCOMPAT_64BIT */
/*150*/	__le32	s_blocks_count_hi;	/* Blocks count */ // block高32
	__le32	s_r_blocks_count_hi;	/* Reserved blocks count */
	__le32	s_free_blocks_count_hi;	/* Free blocks count */
	__le16	s_min_extra_isize;	/* All inodes have at least # bytes */
	__le16	s_want_extra_isize; 	/* New inodes should reserve # bytes */
	__le32	s_flags;		/* Miscellaneous flags */
	__le16  s_raid_stride;		/* RAID stride */
	__le16  s_mmp_update_interval;  /* # seconds to wait in MMP checking */
	__le64  s_mmp_block;            /* Block for multi-mount protection */
	__le32  s_raid_stripe_width;    /* blocks on all data disks (N*stride)*/
	__u8	s_log_groups_per_flex;  /* FLEX_BG group size */ // 一个flex_bg包含的block group的数目=2^s_log_groups_per_flex
	__u8	s_checksum_type;	/* metadata checksum algorithm used */
	__u8	s_encryption_level;	/* versioning level for encryption */
	__u8	s_reserved_pad;		/* Padding to next 32bits */
	__le64	s_kbytes_written;	/* nr of lifetime kilobytes written */
	__le32	s_snapshot_inum;	/* Inode number of active snapshot */
	__le32	s_snapshot_id;		/* sequential ID of active snapshot */
	__le64	s_snapshot_r_blocks_count; /* reserved blocks for active
					      snapshot's future use */
	__le32	s_snapshot_list;	/* inode number of the head of the
					   on-disk snapshot list */
#define EXT4_S_ERR_START offsetof(struct ext4_super_block, s_error_count)
	__le32	s_error_count;		/* number of fs errors */
	__le32	s_first_error_time;	/* first time an error happened */
	__le32	s_first_error_ino;	/* inode involved in first error */
	__le64	s_first_error_block;	/* block involved of first error */
	__u8	s_first_error_func[32] __nonstring;	/* function where the error happened */
	__le32	s_first_error_line;	/* line number where error happened */
	__le32	s_last_error_time;	/* most recent time of an error */
	__le32	s_last_error_ino;	/* inode involved in last error */
	__le32	s_last_error_line;	/* line number where error happened */
	__le64	s_last_error_block;	/* block involved of last error */
	__u8	s_last_error_func[32] __nonstring;	/* function where the error happened */
#define EXT4_S_ERR_END offsetof(struct ext4_super_block, s_mount_opts)
	__u8	s_mount_opts[64];
	__le32	s_usr_quota_inum;	/* inode for tracking user quota */
	__le32	s_grp_quota_inum;	/* inode for tracking group quota */
	__le32	s_overhead_clusters;	/* overhead blocks/clusters in fs */
	__le32	s_backup_bgs[2];	/* groups with sparse_super2 SBs */
	__u8	s_encrypt_algos[4];	/* Encryption algorithms in use  */
	__u8	s_encrypt_pw_salt[16];	/* Salt used for string2key algorithm */
	__le32	s_lpf_ino;		/* Location of the lost+found inode */
	__le32	s_prj_quota_inum;	/* inode for tracking project quota */
	__le32	s_checksum_seed;	/* crc32c(uuid) if csum_seed set */
	__u8	s_wtime_hi;
	__u8	s_mtime_hi;
	__u8	s_mkfs_time_hi;
	__u8	s_lastcheck_hi;
	__u8	s_first_error_time_hi;
	__u8	s_last_error_time_hi;
	__u8	s_first_error_errcode;
	__u8    s_last_error_errcode;
	__le16  s_encoding;		/* Filename charset encoding */
	__le16  s_encoding_flags;	/* Filename charset encoding flags */
	__le32  s_orphan_file_inum;	/* Inode for tracking orphan inodes */
	__le32	s_reserved[94];		/* Padding to the end of the block */
	__le32	s_checksum;		/* crc32c(superblock) */
};

// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4.h#L1480
/*
 * fourth extended-fs super-block data in memory
 */
struct ext4_sb_info {
	unsigned long s_desc_size;	/* Size of a group descriptor in bytes */ // group descriptor占用磁盘空间大小, 32位模式下是32
	unsigned long s_inodes_per_block;/* Number of inodes per block */ // 每个block包含的inode数
	unsigned long s_blocks_per_group;/* Number of blocks in a group */ // 每个block group 包含的block数
	unsigned long s_clusters_per_group; /* Number of clusters in a group */
	unsigned long s_inodes_per_group;/* Number of inodes in a group */
	unsigned long s_itb_per_group;	/* Number of inode table blocks per group */
	unsigned long s_gdb_count;	/* Number of group descriptor blocks */ // group descriptor包含的block数
	unsigned long s_desc_per_block;	/* Number of group descriptors per block */
	ext4_group_t s_groups_count;	/* Number of groups in the fs */ // fs中block group数
	ext4_group_t s_blockfile_groups;/* Groups acceptable for non-extent files */
	unsigned long s_overhead;  /* # of fs overhead clusters */
	unsigned int s_cluster_ratio;	/* Number of blocks per cluster */
	unsigned int s_cluster_bits;	/* log2 of s_cluster_ratio */
	loff_t s_bitmap_maxbytes;	/* max bytes for bitmap files */
	struct buffer_head * s_sbh;	/* Buffer containing the super block */
	struct ext4_super_block *s_es;	/* Pointer to the super block in the buffer */ // 指向ext4_super_block
	struct buffer_head * __rcu *s_group_desc; // 指向一个指针数组, 该数组的elem指向group descriptors各block的数据
	unsigned int s_mount_opt;
	unsigned int s_mount_opt2;
	unsigned long s_mount_flags;
	unsigned int s_def_mount_opt;
	unsigned int s_def_mount_opt2;
	ext4_fsblk_t s_sb_block;
	atomic64_t s_resv_clusters;
	kuid_t s_resuid;
	kgid_t s_resgid;
	unsigned short s_mount_state;
	unsigned short s_pad;
	int s_addr_per_block_bits;
	int s_desc_per_block_bits;
	int s_inode_size; // inode_on_disk大小
	int s_first_ino;
	unsigned int s_inode_readahead_blks;
	unsigned int s_inode_goal;
	u32 s_hash_seed[4];
	int s_def_hash_version;
	int s_hash_unsigned;	/* 3 if hash should be unsigned, 0 if not */
	struct percpu_counter s_freeclusters_counter;
	struct percpu_counter s_freeinodes_counter;
	struct percpu_counter s_dirs_counter;
	struct percpu_counter s_dirtyclusters_counter;
	struct percpu_counter s_sra_exceeded_retry_limit;
	struct blockgroup_lock *s_blockgroup_lock;
	struct proc_dir_entry *s_proc;
	struct kobject s_kobj;
	struct completion s_kobj_unregister;
	struct super_block *s_sb; // 指向super_block
	struct buffer_head *s_mmp_bh;

	/* Journaling */
	struct journal_s *s_journal;
	unsigned long s_ext4_flags;		/* Ext4 superblock flags */
	struct mutex s_orphan_lock;	/* Protects on disk list changes */
	struct list_head s_orphan;	/* List of orphaned inodes in on disk
					   list */
	struct ext4_orphan_info s_orphan_info;
	unsigned long s_commit_interval;
	u32 s_max_batch_time;
	u32 s_min_batch_time;
	struct block_device *s_journal_bdev;
#ifdef CONFIG_QUOTA
	/* Names of quota files with journalled quota */
	char __rcu *s_qf_names[EXT4_MAXQUOTAS];
	int s_jquota_fmt;			/* Format of quota to use */
#endif
	unsigned int s_want_extra_isize; /* New inodes should reserve # bytes */
	struct ext4_system_blocks __rcu *s_system_blks;

#ifdef EXTENTS_STATS
	/* ext4 extents stats */
	unsigned long s_ext_min;
	unsigned long s_ext_max;
	unsigned long s_depth_max;
	spinlock_t s_ext_stats_lock;
	unsigned long s_ext_blocks;
	unsigned long s_ext_extents;
#endif

	/* for buddy allocator */
	struct ext4_group_info ** __rcu *s_group_info;
	struct inode *s_buddy_cache;
	spinlock_t s_md_lock;
	unsigned short *s_mb_offsets;
	unsigned int *s_mb_maxs;
	unsigned int s_group_info_size;
	unsigned int s_mb_free_pending;
	struct list_head s_freed_data_list;	/* List of blocks to be freed
						   after commit completed */
	struct list_head s_discard_list;
	struct work_struct s_discard_work;
	atomic_t s_retry_alloc_pending;
	struct list_head *s_mb_avg_fragment_size;
	rwlock_t *s_mb_avg_fragment_size_locks;
	struct list_head *s_mb_largest_free_orders;
	rwlock_t *s_mb_largest_free_orders_locks;

	/* tunables */
	unsigned long s_stripe;
	unsigned int s_mb_max_linear_groups;
	unsigned int s_mb_stream_request;
	unsigned int s_mb_max_to_scan;
	unsigned int s_mb_min_to_scan;
	unsigned int s_mb_stats;
	unsigned int s_mb_order2_reqs;
	unsigned int s_mb_group_prealloc;
	unsigned int s_max_dir_size_kb;
	/* where last allocation was done - for stream allocation */
	unsigned long s_mb_last_group;
	unsigned long s_mb_last_start;
	unsigned int s_mb_prefetch;
	unsigned int s_mb_prefetch_limit;
	unsigned int s_mb_best_avail_max_trim_order;

	/* stats for buddy allocator */
	atomic_t s_bal_reqs;	/* number of reqs with len > 1 */
	atomic_t s_bal_success;	/* we found long enough chunks */
	atomic_t s_bal_allocated;	/* in blocks */
	atomic_t s_bal_ex_scanned;	/* total extents scanned */
	atomic_t s_bal_cX_ex_scanned[EXT4_MB_NUM_CRS];	/* total extents scanned */
	atomic_t s_bal_groups_scanned;	/* number of groups scanned */
	atomic_t s_bal_goals;	/* goal hits */
	atomic_t s_bal_len_goals;	/* len goal hits */
	atomic_t s_bal_breaks;	/* too long searches */
	atomic_t s_bal_2orders;	/* 2^order hits */
	atomic_t s_bal_p2_aligned_bad_suggestions;
	atomic_t s_bal_goal_fast_bad_suggestions;
	atomic_t s_bal_best_avail_bad_suggestions;
	atomic64_t s_bal_cX_groups_considered[EXT4_MB_NUM_CRS];
	atomic64_t s_bal_cX_hits[EXT4_MB_NUM_CRS];
	atomic64_t s_bal_cX_failed[EXT4_MB_NUM_CRS];		/* cX loop didn't find blocks */
	atomic_t s_mb_buddies_generated;	/* number of buddies generated */
	atomic64_t s_mb_generation_time;
	atomic_t s_mb_lost_chunks;
	atomic_t s_mb_preallocated;
	atomic_t s_mb_discarded;
	atomic_t s_lock_busy;

	/* locality groups */
	struct ext4_locality_group __percpu *s_locality_groups;

	/* for write statistics */
	unsigned long s_sectors_written_start;
	u64 s_kbytes_written;

	/* the size of zero-out chunk */
	unsigned int s_extent_max_zeroout_kb;

	unsigned int s_log_groups_per_flex; // 同ext4_super_block的s_log_groups_per_flex
	struct flex_groups * __rcu *s_flex_groups;
	ext4_group_t s_flex_groups_allocated;

	/* workqueue for reserved extent conversions (buffered io) */
	struct workqueue_struct *rsv_conversion_wq;

	/* timer for periodic error stats printing */
	struct timer_list s_err_report;

	/* Lazy inode table initialization info */
	struct ext4_li_request *s_li_request;
	/* Wait multiplier for lazy initialization thread */
	unsigned int s_li_wait_mult;

	/* Kernel thread for multiple mount protection */
	struct task_struct *s_mmp_tsk;

	/* record the last minlen when FITRIM is called. */
	unsigned long s_last_trim_minblks;

	/* Reference to checksum algorithm driver via cryptoapi */
	struct crypto_shash *s_chksum_driver;

	/* Precomputed FS UUID checksum for seeding other checksums */
	__u32 s_csum_seed;

	/* Reclaim extents from extent status tree */
	struct shrinker s_es_shrinker;
	struct list_head s_es_list;	/* List of inodes with reclaimable extents */
	long s_es_nr_inode;
	struct ext4_es_stats s_es_stats;
	struct mb_cache *s_ea_block_cache;
	struct mb_cache *s_ea_inode_cache;
	spinlock_t s_es_lock ____cacheline_aligned_in_smp;

	/* Journal triggers for checksum computation */
	struct ext4_journal_trigger s_journal_triggers[EXT4_JOURNAL_TRIGGER_COUNT];

	/* Ratelimit ext4 messages. */
	struct ratelimit_state s_err_ratelimit_state;
	struct ratelimit_state s_warning_ratelimit_state;
	struct ratelimit_state s_msg_ratelimit_state;
	atomic_t s_warning_count;
	atomic_t s_msg_count;

	/* Encryption policy for '-o test_dummy_encryption' */
	struct fscrypt_dummy_policy s_dummy_enc_policy;

	/*
	 * Barrier between writepages ops and changing any inode's JOURNAL_DATA
	 * or EXTENTS flag or between writepages ops and changing DELALLOC or
	 * DIOREAD_NOLOCK mount options on remount.
	 */
	struct percpu_rw_semaphore s_writepages_rwsem;
	struct dax_device *s_daxdev;
	u64 s_dax_part_off;
#ifdef CONFIG_EXT4_DEBUG
	unsigned long s_simulate_fail;
#endif
	/* Record the errseq of the backing block device */
	errseq_t s_bdev_wb_err;
	spinlock_t s_bdev_wb_lock;

	/* Information about errors that happened during this mount */
	spinlock_t s_error_lock;
	int s_add_error_count;
	int s_first_error_code;
	__u32 s_first_error_line;
	__u32 s_first_error_ino;
	__u64 s_first_error_block;
	const char *s_first_error_func;
	time64_t s_first_error_time;
	int s_last_error_code;
	__u32 s_last_error_line;
	__u32 s_last_error_ino;
	__u64 s_last_error_block;
	const char *s_last_error_func;
	time64_t s_last_error_time;
	/*
	 * If we are in a context where we cannot update the on-disk
	 * superblock, we queue the work here.  This is used to update
	 * the error information in the superblock, and for periodic
	 * updates of the superblock called from the commit callback
	 * function.
	 */
	struct work_struct s_sb_upd_work;

	/* Ext4 fast commit sub transaction ID */
	atomic_t s_fc_subtid;

	/*
	 * After commit starts, the main queue gets locked, and the further
	 * updates get added in the staging queue.
	 */
#define FC_Q_MAIN	0
#define FC_Q_STAGING	1
	struct list_head s_fc_q[2];	/* Inodes staged for fast commit
					 * that have data changes in them.
					 */
	struct list_head s_fc_dentry_q[2];	/* directory entry updates */
	unsigned int s_fc_bytes;
	/*
	 * Main fast commit lock. This lock protects accesses to the
	 * following fields:
	 * ei->i_fc_list, s_fc_dentry_q, s_fc_q, s_fc_bytes, s_fc_bh.
	 */
	spinlock_t s_fc_lock;
	struct buffer_head *s_fc_bh;
	struct ext4_fc_stats s_fc_stats;
	tid_t s_fc_ineligible_tid;
#ifdef CONFIG_EXT4_DEBUG
	int s_fc_debug_max_replay;
#endif
	struct ext4_fc_replay_state s_fc_replay_state;
};

// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4.h#L992
/*
 * fourth extended file system inode data in memory
 */
struct ext4_inode_info {
	__le32	i_data[15];	/* unconverted */ // 与ext4_inode的i_block的数据意义相同
	__u32	i_dtime;
	ext4_fsblk_t	i_file_acl;

	/*
	 * i_block_group is the number of the block group which contains
	 * this file's inode.  Constant across the lifetime of the inode,
	 * it is used for making block allocation decisions - we try to
	 * place a file's data blocks near its inode block, and new inodes
	 * near to their parent directory's inode.
	 */
	ext4_group_t	i_block_group; // 包含文件的inode_on_disk的block group号
	ext4_lblk_t	i_dir_start_lookup;
#if (BITS_PER_LONG < 64)
	unsigned long	i_state_flags;		/* Dynamic state flags */
#endif
	unsigned long	i_flags; // 与ext4_inode的i_flags意义相同

	/*
	 * Extended attributes can be read independently of the main file
	 * data. Taking i_rwsem even when reading would cause contention
	 * between readers of EAs and writers of regular file data, so
	 * instead we synchronize on xattr_sem when reading or changing
	 * EAs.
	 */
	struct rw_semaphore xattr_sem;

	/*
	 * Inodes with EXT4_STATE_ORPHAN_FILE use i_orphan_idx. Otherwise
	 * i_orphan is used.
	 */
	union {
		struct list_head i_orphan;	/* unlinked but open inodes */
		unsigned int i_orphan_idx;	/* Index in orphan file */
	};

	/* Fast commit related info */

	/* For tracking dentry create updates */
	struct list_head i_fc_dilist;
	struct list_head i_fc_list;	/*
					 * inodes that need fast commit
					 * protected by sbi->s_fc_lock.
					 */

	/* Start of lblk range that needs to be committed in this fast commit */
	ext4_lblk_t i_fc_lblk_start;

	/* End of lblk range that needs to be committed in this fast commit */
	ext4_lblk_t i_fc_lblk_len;

	/* Number of ongoing updates on this inode */
	atomic_t  i_fc_updates;

	/* Fast commit wait queue for this inode */
	wait_queue_head_t i_fc_wait;

	/* Protect concurrent accesses on i_fc_lblk_start, i_fc_lblk_len */
	struct mutex i_fc_lock;

	/*
	 * i_disksize keeps track of what the inode size is ON DISK, not
	 * in memory.  During truncate, i_size is set to the new size by
	 * the VFS prior to calling ext4_truncate(), but the filesystem won't
	 * set i_disksize to 0 until the truncate is actually under way.
	 *
	 * The intent is that i_disksize always represents the blocks which
	 * are used by this file.  This allows recovery to restart truncate
	 * on orphans if we crash during truncate.  We actually write i_disksize
	 * into the on-disk inode when writing inodes out, instead of i_size.
	 *
	 * The only time when i_disksize and i_size may be different is when
	 * a truncate is in progress.  The only things which change i_disksize
	 * are ext4_get_block (growth) and ext4_truncate (shrinkth).
	 */
	loff_t	i_disksize;

	/*
	 * i_data_sem is for serialising ext4_truncate() against
	 * ext4_getblock().  In the 2.4 ext2 design, great chunks of inode's
	 * data tree are chopped off during truncate. We can't do that in
	 * ext4 because whenever we perform intermediate commits during
	 * truncate, the inode and all the metadata blocks *must* be in a
	 * consistent state which allows truncation of the orphans to restart
	 * during recovery.  Hence we must fix the get_block-vs-truncate race
	 * by other means, so we have i_data_sem.
	 */
	struct rw_semaphore i_data_sem;
	struct inode vfs_inode;
	struct jbd2_inode *jinode;

	spinlock_t i_raw_lock;	/* protects updates to the raw inode */

	/*
	 * File creation time. Its function is same as that of
	 * struct timespec64 i_{a,c,m}time in the generic inode.
	 */
	struct timespec64 i_crtime;

	/* mballoc */
	atomic_t i_prealloc_active;
	struct rb_root i_prealloc_node;
	rwlock_t i_prealloc_lock;

	/* extents status tree */
	struct ext4_es_tree i_es_tree;
	rwlock_t i_es_lock;
	struct list_head i_es_list;
	unsigned int i_es_all_nr;	/* protected by i_es_lock */
	unsigned int i_es_shk_nr;	/* protected by i_es_lock */
	ext4_lblk_t i_es_shrink_lblk;	/* Offset where we start searching for
					   extents to shrink. Protected by
					   i_es_lock  */

	/* ialloc */
	ext4_group_t	i_last_alloc_group;

	/* allocation reservation info for delalloc */
	/* In case of bigalloc, this refer to clusters rather than blocks */
	unsigned int i_reserved_data_blocks;

	/* pending cluster reservations for bigalloc file systems */
	struct ext4_pending_tree i_pending_tree;

	/* on-disk additional length */
	__u16 i_extra_isize; // 与ext4_inode的i_extra_isize相同

	/* Indicate the inline data space. */
	u16 i_inline_off;
	u16 i_inline_size;

#ifdef CONFIG_QUOTA
	/* quota space reservation, managed internally by quota code */
	qsize_t i_reserved_quota;
#endif

	/* Lock protecting lists below */
	spinlock_t i_completed_io_lock;
	/*
	 * Completed IOs that need unwritten extents handling and have
	 * transaction reserved
	 */
	struct list_head i_rsv_conversion_list;
	struct work_struct i_rsv_conversion_work;
	atomic_t i_unwritten; /* Nr. of inflight conversions pending */

	spinlock_t i_block_reservation_lock;

	/*
	 * Transactions that contain inode's metadata needed to complete
	 * fsync and fdatasync, respectively.
	 */
	tid_t i_sync_tid;
	tid_t i_datasync_tid;

#ifdef CONFIG_QUOTA
	struct dquot __rcu *i_dquot[MAXQUOTAS];
#endif

	/* Precomputed uuid+inum+igen checksum for seeding inode checksums */
	__u32 i_csum_seed;

	kprojid_t i_projid;
};
```

s_feature_compat、s_feature_incompat和s_feature_ro_compat比较拗口,其区别在于如果文件系统在它们的字段上置位了某些它们不支持
的标志,产生的结果不同。s_feature_compat,仍然可以继续,不当作错 误 ;s_feature_incompat , 会当作错误, mount失败; s_feature_ro_compat,也会当作错误,但可以mount成只读文件系统。它们支持的标志集分别为EXT4_FEATURE_COMPAT_SUPP 、
EXT4_FEATURE_INCOMPAT_SUPP和EXT4_FEATURE_RO_COMPAT_SUPP.

ext4定义了几个中间的辅助数据结构,建立VFS的super_block和inode等数据结构和它们的关联关系:
- ext4_sb_info
- ext4_inode_info

ext4_sb_info (ext4 super block infomation)结构体关联VFS 的super_block和ext4_super_block,同时它还保存了文件系统的一些通用信息.

ext4_sb_info和ext4_super_block的很多字段相似(这里没有把二者放在一起,因为ext4_super_block、ext4_group_desc和ext_inode逻辑上更紧密),但也有些区别,前者的字段值很多是根据后者的字段计算得到的,比如s_inodes_per_block,就是通过block大小除以s_inode_size得到的. 虽然通过ext4_sb_info可以找到ext4_super_block,从而得到需要的值,但定义这些看似重复的字段可以省略重复的计算.

super_block结构体的s_fs_info字段指向ext4_sb_info,EXT4_SB宏就是通过该字段获取ext4_sb_info的. ext4_sb_info的s_sb和s_es字段分
别指向super_block和ext4_super_block,构成了VFS和ext4磁盘数据结构之间的通路.

s_groups_count字段表示block group数,是由block总数除以每个block group的block数得到的,采用的是进一法。s_gdb_count则是根据
s_groups_count的值除以s_desc_per_block得到,采用的同样是进一法. s_group_desc字段指向一个指针数组,数组的元素个数与s_gdb_count
字段的值相等,每一个元素都指向GROUP DESCRIPTORS相应的block中的数据.

由于group descriptor访问非常频繁,系统运行过程中,除非遇到意外错误或者对磁盘执行umount操作,各元素指向的内存数据会一直保持,以提高访问效率.

VFS的inode结构体与ext4_inode结构体之间也需要一座桥梁,就是ext4_inode_info结构体.

ext4_inode_info内嵌了inode,可以通过传递inode对象到EXT4_I宏来获得ext4_inode_info对象,但它并没有定义可以直接访问ext4_inode
的字段,只是拷贝了后者的i_block和i_flags等字段;同时,inode结构体也包含了与ext4_inode字段类似的字段,比如i_atime、i_mtime和
i_ctime等。所以它们三者的关系可以理解为ext4_inode_info内嵌了inode,二者一起“瓜分”了ext4_inode的信息.

## 目录的存储格式
目录本身也是个文件，也有 inode. inode 里面也是指向一些块. 和普通文件不同的是，普通文件的块里面保存的是文件数据，而目录文件的块里面保存的是目录里面一项一项的文件信息. 这些信息称为 [ext4_dir_entry](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4.h#L1245). 从代码来看，有两个版本，在成员来讲几乎没有差别，只不过第二个版本 ext4_dir_entry_2 是将一个 16 位的 name_len，变成了一个 8 位的 name_len 和 8 位的 file_type.

在目录文件的块中，最简单的保存格式是列表，就是一项一项地将 ext4_dir_entry_2 列在那里.

每一项都会保存这个目录的下一级的文件的文件名和对应的 inode，通过这个 inode，就能找到真正的文件. 第一项是“.”，表示当前目录，第二项是“…”，表示上一级目录，接下来就是一项一项的文件名和 inode.

有时候，如果一个目录下面的文件太多的时候，此时想在这个目录下找一个文件，按照列表一个个去找，太慢了，于是就添加了索引的模式. 如果在 inode 中设置 EXT4_INDEX_FL 标志，则目录文件的块的组织形式将发生变化，变成了下面定义的这个样子：
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/namei.c#L209
struct dx_entry
{
	__le32 hash;
	__le32 block;
};

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/namei.c#L221
struct dx_root
{
	struct fake_dirent dot;
	char dot_name[4];
	struct fake_dirent dotdot;
	char dotdot_name[4];
	struct dx_root_info
	{
		__le32 reserved_zero;
		u8 hash_version;
		u8 info_length; /* 8 */
		u8 indirect_levels;
		u8 unused_flags;
	}
	info;
	struct dx_entry	entries[];
};
```

接下来看索引项 dx_entry. 这个也很简单，其实就是文件名的哈希值和数据块的一个映射关系.

如果要查找一个目录下面的文件名，可以通过名称取哈希. 如果哈希能够匹配上，就说明这个文件的信息在相应的块里面, 然后打开这个块; 如果里面不再是索引，而是索引树的叶子节点的话，那里面还是 ext4_dir_entry_2 的列表，我们只要一项一项找文件名就行. 通过索引树，可以将一个目录下面的 N 多的文件分散到很多的块里面，可以很快地进行查找.

![](/misc/img/fs/3ea2ad5704f20538d9c911b02f42086d.jpeg)

## 软链接和硬链接的存储格式
所谓的链接（Link），可以认为是文件的别名，而链接又可分为两种，硬链接（Hard Link）和软链接（Symbolic Link）.

![](/misc/img/fs/45a6cfdd9d45e30dc2f38f0d2572be7b.jpeg)


硬链接与原始文件共用一个 inode 的，**但是 inode 是不跨文件系统的，每个文件系统都有自己的 inode 列表，因而硬链接是没有办法跨文件系统的**. 而软链接不同，软链接相当于重新创建了一个文件, 这个文件也有独立的 inode，只不过打开这个文件看里面内容的时候，内容指向另外的一个文件. 这就很灵活了, 也可以跨文件系统了，甚至目标文件被删除了，链接文件还是在的，只不过指向的文件找不到了而已.

## 读文件
```c
// https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/fs.h#L470
/**
 * struct address_space - Contents of a cacheable, mappable object.
 * @host: Owner, either the inode or the block_device.
 * @i_pages: Cached pages.
 * @invalidate_lock: Guards coherency between page cache contents and
 *   file offset->disk block mappings in the filesystem during invalidates.
 *   It is also used to block modification of page cache contents through
 *   memory mappings.
 * @gfp_mask: Memory allocation flags to use for allocating pages.
 * @i_mmap_writable: Number of VM_SHARED mappings.
 * @nr_thps: Number of THPs in the pagecache (non-shmem only).
 * @i_mmap: Tree of private and shared mappings.
 * @i_mmap_rwsem: Protects @i_mmap and @i_mmap_writable.
 * @nrpages: Number of page entries, protected by the i_pages lock.
 * @writeback_index: Writeback starts here.
 * @a_ops: Methods.
 * @flags: Error bits and flags (AS_*).
 * @wb_err: The most recent error which has occurred.
 * @private_lock: For use by the owner of the address_space.
 * @private_list: For use by the owner of the address_space.
 * @private_data: For use by the owner of the address_space.
 */
struct address_space {
	struct inode		*host; // 常规文件就是文件的inode, 块设备文件有两个address_space, 打开块设备文件时, 会将文件描述符的f_mapping指向主inode的address_space,  次inode的address_space实际没使用到
	struct xarray		i_pages;
	struct rw_semaphore	invalidate_lock;
	gfp_t			gfp_mask;
	atomic_t		i_mmap_writable; // 该地址空间中共享内存映射的数目
#ifdef CONFIG_READ_ONLY_THP_FOR_FS
	/* number of thp, only for non-shmem files */
	atomic_t		nr_thps;
#endif
	struct rb_root_cached	i_mmap; // 为便于查找, 一个共享映射文件的所有虚拟内存区间或私有映射文件的写时复制虚拟内存空间被组织成一个radix有些查找树(priority search tree), i_mmap就是树根
	unsigned long		nrpages; // 页面的总数
	pgoff_t			writeback_index; // 为了不占用过多资源， kernel将地址空间中页面回写的行为分成若干轮次. writeback_index记录了上次回写操作的最后页面索引, 下一次回写操作将从该位置开始
	const struct address_space_operations *a_ops; // address_space的操作函数
	unsigned long		flags;
	struct rw_semaphore	i_mmap_rwsem;
	errseq_t		wb_err;
	spinlock_t		private_lock; // 保护private_list
	struct list_head	private_list; // 地址空间的私有链表, 通常用来链接为文件的中间记录块所分配的buffer_head结构
	void			*private_data;
} __attribute__((aligned(sizeof(long)))) __randomize_layout;

// https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/fs.h#L470
struct address_space_operations {
	int (*writepage)(struct page *page, struct writeback_control *wbc); // 一般用于基于磁盘的fs, 将文件在内存页中的数据更新到磁盘
	int (*read_folio)(struct file *, struct folio *);

	/* Write back some dirty pages from this mapping. */
	int (*writepages)(struct address_space *, struct writeback_control *); // 将address_space的多个脏页写入磁盘

	/* Mark a folio dirty.  Return true if this dirtied it */
	bool (*dirty_folio)(struct address_space *, struct folio *);

	void (*readahead)(struct readahead_control *);

	int (*write_begin)(struct file *, struct address_space *mapping,
				loff_t pos, unsigned len,
				struct page **pagep, void **fsdata); // 被通用的缓冲I/O代码调用. vfs调用write_begin通知具体fs, 准备写文件的begin~end到给定页面
	int (*write_end)(struct file *, struct address_space *mapping,
				loff_t pos, unsigned len, unsigned copied,
				struct page *page, void *fsdata); // 在成功调用write_begin, 并完成数据复制后, write_end必须被调用. vfs调用write_end告诉fs, 数据已复制到页面, 可以提交到磁盘了

	/* Unfortunately this kludge is needed for FIBMAP. Don't use it */
	sector_t (*bmap)(struct address_space *, sector_t); // 将文件中的逻辑块扇区编号映射为对应设备上的物理块扇区编号. 它被用于FIBMAP ioctl和swap file一起工作
	void (*invalidate_folio) (struct folio *, size_t offset, size_t len);
	bool (*release_folio)(struct folio *, gfp_t);
	void (*free_folio)(struct folio *folio);
	ssize_t (*direct_IO)(struct kiocb *, struct iov_iter *iter); // 被通用read/write调用, 指向direct io
	/*
	 * migrate the contents of a folio to the specified target. If
	 * migrate_mode is MIGRATE_ASYNC, it must not block.
	 */
	int (*migrate_folio)(struct address_space *, struct folio *dst,
			struct folio *src, enum migrate_mode);
	int (*launder_folio)(struct folio *);
	bool (*is_partially_uptodate) (struct folio *, size_t from,
			size_t count);
	void (*is_dirty_writeback) (struct folio *, bool *dirty, bool *wb);
	int (*error_remove_page)(struct address_space *, struct page *);

	/* swapfile support */
	int (*swap_activate)(struct swap_info_struct *sis, struct file *file,
				sector_t *span);
	void (*swap_deactivate)(struct file *file);
	int (*swap_rw)(struct kiocb *iocb, struct iov_iter *iter);
};


// https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/mm_types.h#L74
struct page {
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */ // 原子标志, 用于某些异步更新
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
			 * lruvec->lru_lock.  Sometimes used as a generic list
			 * by the page owner.
			 */
			union {
				struct list_head lru;

				/* Or, for the Unevictable "LRU list" slot */
				struct {
					/* Always even, to negate PageTail */
					void *__filler;
					/* Count page's or folio's mlocks */
					unsigned int mlock_count;
				};

				/* Or, free page */
				struct list_head buddy_list;
				struct list_head pcp_list;
			};
			/* See page-flags.h for PAGE_MAPPING_FLAGS */
			struct address_space *mapping; // 指向所属地址空间
			union {
				pgoff_t index;		/* Our offset within mapping. */
				unsigned long share;	/* share count for fsdax */
			};
			/**
			 * @private: Mapping-private opaque data.
			 * Usually used for buffer_heads if PagePrivate.
			 * Used for swp_entry_t if PageSwapCache.
			 * Indicates order in the buddy system if PageBuddy.
			 */
			unsigned long private; // 指向和页面关联的第一个缓冲头的指针
		};
		struct {	/* page_pool used by netstack */
			/**
			 * @pp_magic: magic value to avoid recycling non
			 * page_pool allocated pages.
			 */
			unsigned long pp_magic;
			struct page_pool *pp;
			unsigned long _pp_mapping_pad;
			unsigned long dma_addr;
			union {
				/**
				 * dma_addr_upper: might require a 64-bit
				 * value on 32-bit architectures.
				 */
				unsigned long dma_addr_upper;
				/**
				 * For frag page support, not supported in
				 * 32-bit architectures with 64-bit DMA.
				 */
				atomic_long_t pp_frag_count;
			};
		};
		struct {	/* Tail pages of compound page */
			unsigned long compound_head;	/* Bit zero is set */
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
	};

	/* Usage count. *DO NOT USE DIRECTLY*. See page_ref.h */
	atomic_t _refcount;

#ifdef CONFIG_MEMCG
	unsigned long memcg_data;
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
					   not kmapped, ie. highmem) */ // 内核虚拟地址
#endif /* WANT_PAGE_VIRTUAL */

#ifdef CONFIG_KMSAN
	/*
	 * KMSAN metadata for this page:
	 *  - shadow page: every bit indicates whether the corresponding
	 *    bit of the original page is initialized (0) or not (1);
	 *  - origin page: every 4 bytes contain an id of the stack trace
	 *    where the uninitialized value was created.
	 */
	struct page *kmsan_shadow;
	struct page *kmsan_origin;
#endif

#ifdef LAST_CPUPID_NOT_IN_PAGE_FLAGS
	int _last_cpupid;
#endif
} _struct_page_alignment;
```

address_space表示地址空间, 其目的是将存储介质上(可能不连续的)数据连续page的方式呈现出来.

## linux io过程自顶向下分析
![](/misc/img/fs/3c506edf93b15341da3db658e9970773.jpeg)

### 挂载文件系统
ref:
- [linux多版本文件系统接口对比梳理](https://blog.csdn.net/m0_37637511/article/details/124328953)
- [在linux 5.x的内核中，实际文件系统的挂载采用新的挂载API，引入了struct fs_context用于内部文件系统挂载的信息](https://cloud.tencent.com/developer/article/2074558)
	
	ext4挂载举例

内核是不是支持某种类型的文件系统，需要先进行注册才能知道. 例如 ext4 文件系统，就需要通过 register_filesystem 注册到全局变量[file_systems](https://elixir.bootlin.com/linux/v5.12.9/source/fs/filesystems.c#L34)中，传入的参数是 ext4_fs_type，表示注册的是 ext4 类型的文件系统. 这里面最重要的一个成员变量就是 ext4_mount.


```c
// https://elixir.bootlin.com/linux/v6.6.17/source/fs/filesystems.c#L34
static struct file_system_type *file_systems; // 所有file_system_type都会加入该链表

// https://elixir.bootlin.com/linux/v6.6.17/source/include/linux/fs.h#L2358
// 与 mount 相关联的 file_system_type 中的属性达到变化: get_sb(2.6)->mount(4.9)->init_fs_context(5.10), 见erofs_fs_type, ext2_fs_type, 参考[[RFC PATCH 37/68] vfs: Convert apparmorfs to use the new mount API](https://lore.kernel.org/linux-security-module/155373033460.7602.12727592550663113967.stgit@warthog.procyon.org.uk/T/)
// file_system_type表示fs种类
struct file_system_type {
	const char *name;
	int fs_flags; // 标志
#define FS_REQUIRES_DEV		1 
#define FS_BINARY_MOUNTDATA	2
#define FS_HAS_SUBTYPE		4
#define FS_USERNS_MOUNT		8	/* Can be mounted by userns root */
#define FS_DISALLOW_NOTIFY_PERM	16	/* Disable fanotify permission events */
#define FS_ALLOW_IDMAP         32      /* FS has been updated to handle vfs idmappings. */
#define FS_RENAME_DOES_D_MOVE	32768	/* FS will handle d_move() during rename() internally. */
	int (*init_fs_context)(struct fs_context *);
	const struct fs_parameter_spec *parameters;
	struct dentry *(*mount) (struct file_system_type *, int,
		       const char *, void *);
	void (*kill_sb) (struct super_block *); // 在该fs实例卸载时调用
	struct module *owner; // 指向实现该fs的module
	struct file_system_type * next; // 指向单链表file_systems的下一项
	struct hlist_head fs_supers; // 该fs类型的所有super_block实例链表的表头

	struct lock_class_key s_lock_key; // 用于调试锁依赖性
	struct lock_class_key s_umount_key; // 同上
	struct lock_class_key s_vfs_rename_key; // 同上
	struct lock_class_key s_writers_key[SB_FREEZE_LEVELS]; // 同上

	struct lock_class_key i_lock_key; // 用于调试锁依赖性
	struct lock_class_key i_mutex_key; // 同上
	struct lock_class_key invalidate_lock_key; // 同上
	struct lock_class_key i_mutex_dir_key; // 同上
};

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/super.c#L6262
static struct file_system_type ext4_fs_type = {
	.owner		= THIS_MODULE,
	.name		= "ext4",
	.mount		= ext4_mount,
	.kill_sb	= kill_block_super,
	.fs_flags	= FS_REQUIRES_DEV,
};
MODULE_ALIAS_FS("ext4");

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/super.c#L6274
static int __init ext4_init_fs(void)
{
	int i, err;

	ratelimit_state_init(&ext4_mount_msg_ratelimit, 30 * HZ, 64);
	ext4_li_info = NULL;
	mutex_init(&ext4_li_mtx);

	/* Build-time check for flags consistency */
	ext4_check_flag_values();

	for (i = 0; i < EXT4_WQ_HASH_SZ; i++)
		init_waitqueue_head(&ext4__ioend_wq[i]);

	err = ext4_init_es();
	if (err)
		return err;

	err = ext4_init_pending();
	if (err)
		goto out7;

	err = ext4_init_post_read_processing();
	if (err)
		goto out6;

	err = ext4_init_pageio();
	if (err)
		goto out5;

	err = ext4_init_system_zone();
	if (err)
		goto out4;

	err = ext4_init_sysfs();
	if (err)
		goto out3;

	err = ext4_init_mballoc();
	if (err)
		goto out2;
	err = init_inodecache();
	if (err)
		goto out1;
	register_as_ext3();
	register_as_ext2();
	err = register_filesystem(&ext4_fs_type); // 注册
	if (err)
		goto out;

	return 0;
out:
	unregister_as_ext2();
	unregister_as_ext3();
	destroy_inodecache();
out1:
	ext4_exit_mballoc();
out2:
	ext4_exit_sysfs();
out3:
	ext4_exit_system_zone();
out4:
	ext4_exit_pageio();
out5:
	ext4_exit_post_read_processing();
out6:
	ext4_exit_pending();
out7:
	ext4_exit_es();

	return err;
}
```

如果一种文件系统的类型曾经在内核注册过，这就说明允许挂载并且使用这个文件系统.

从第一个 mount 系统调用开始解析.

> mount入口地址有两个：
> 1. 系统自带的 mount 命令会调用 /fs/namespace.c 中的 SYSCALL_DEFINE5
> 1. busybox mount 命令会调用 /fs/compat.c 中的 COMPAT_SYSCALL_DEFINE5

mount 系统调用的定义如下：
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namespace.c#L3386
SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name,
		char __user *, type, unsigned long, flags, void __user *, data)
{
	int ret;
	char *kernel_type;
	char *kernel_dev;
	void *options;

	kernel_type = copy_mount_string(type);
	ret = PTR_ERR(kernel_type);
	if (IS_ERR(kernel_type))
		goto out_type;

	kernel_dev = copy_mount_string(dev_name);
	ret = PTR_ERR(kernel_dev);
	if (IS_ERR(kernel_dev))
		goto out_dev;

	options = copy_mount_options(data);
	ret = PTR_ERR(options);
	if (IS_ERR(options))
		goto out_data;

	ret = do_mount(kernel_dev, dir_name, kernel_type, flags, options);

	kfree(options);
out_data:
	kfree(kernel_dev);
out_dev:
	kfree(kernel_type);
out_type:
	return ret;
}
```

接下来的调用链为：[do_mount](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namespace.c#L3118)->[do_new_mount](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namespace.c#L2833)->[vfs_get_tree](https://elixir.bootlin.com/linux/v5.8-rc3/C/ident/vfs_get_tree) + [do_new_mount_fc](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namespace.c#L2792)

[do_new_mount_fc](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namespace.c#L2792)
->[vfs_create_mount](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namespace.c#L949).

> mount_fs() deleted in v5.0.21~v5.1-rc1 on 9bc61ab18b1d41f26dc06b9e6d3c203e65f83fe6 for "vfs: Introduce fs_context, switch vfs_kern_mount() to it."

vfs_create_mount 会创建 [struct mount](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/mount.h#L40) 结构，每个挂载的文件系统都对应于这样一个结构.
```c
// https://elixir.bootlin.com/linux/v6.6.18/source/fs/mount.h#L39
// v5.8-rc3->v6.6.18无变化
struct mount {
	struct hlist_node mnt_hash; // 链入全局已装载fs哈希表的连接件
	struct mount *mnt_parent; // 指向被装载到的父fs
	struct dentry *mnt_mountpoint; // 指向被装载到的装载点目录的dentry
	struct vfsmount mnt;
	union {
		struct rcu_head mnt_rcu;
		struct llist_node mnt_llist;
	};
#ifdef CONFIG_SMP
	struct mnt_pcp __percpu *mnt_pcp;
#else
	int mnt_count;
	int mnt_writers;
#endif
	struct list_head mnt_mounts;	/* list of children, anchored here */ // 装载到这个fs的目录上所有子fs的链表的表头
	struct list_head mnt_child;	/* and going through their mnt_child */ // 链接到被装载到的父fs mnt_mounts链表的连接件
	struct list_head mnt_instance;	/* mount instance on sb->s_mounts */ // 将mount链接到super_block的s_mounts链表
	const char *mnt_devname;	/* Name of device e.g. /dev/dsk/hda1 */ // 保存fs的块设备的设备名, 或特殊fs的文件系统类型名
	struct list_head mnt_list; // 链入达到进程名字空间中已装载fs链表的连接件. 链表头是mnt_namespace.list
	struct list_head mnt_expire;	/* link in fs-specific expiry list */ // 链入到fs专有的过期链表的连接件, 用于NFS, CIFS, AFS等网络fs
	struct list_head mnt_share;	/* circular list of shared mounts */ // 链入到共享装载循环链表的连接件
	struct list_head mnt_slave_list;/* list of slave mounts */ // 这个fs的slave mount链表的表头
	struct list_head mnt_slave;	/* slave list entry */
	struct mount *mnt_master;	/* slave is on master->mnt_slave_list */
	struct mnt_namespace *mnt_ns;	/* containing namespace */
	struct mountpoint *mnt_mp;	/* where is it mounted */
	union {
		struct hlist_node mnt_mp_list;	/* list mounts with the same mountpoint */
		struct hlist_node mnt_umount;
	};
	struct list_head mnt_umounting; /* list entry for umount propagation */
#ifdef CONFIG_FSNOTIFY
	struct fsnotify_mark_connector __rcu *mnt_fsnotify_marks;
	__u32 mnt_fsnotify_mask;
#endif
	int mnt_id;			/* mount identifier */ // 装载标识符
	int mnt_group_id;		/* peer group identifier */
	int mnt_expiry_mark;		/* true if marked for expiry */
	struct hlist_head mnt_pins;
	struct hlist_head mnt_stuck_children;
} __randomize_layout;
```

同dentry, mount已存在父子关系. mount是将每个局部fs链接到全局fs树的连接件. 有了装载, fs的位置需`<mount, dentry>`共同决定文件位置的路径.

> 新kernel将原先的vfsmount(v2.6.39.4)拆成了vfsmount+mount.

其中，mnt_parent 是装载点所在的父文件系统，mnt_mountpoint 是装载点在父文件系统中的 dentry；struct dentry 表示目录，并和目录的 inode 关联；mnt_root 是当前文件系统根目录的 dentry，mnt_sb 是指向超级块的指针. 接下来，来看调用 mount_fs 挂载文件系统.

接下来看调用 vfs_get_tree 挂载文件系统。
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/super.c#L1536
/**
 * vfs_get_tree - Get the mountable root
 * @fc: The superblock configuration context.
 *
 * The filesystem is invoked to get or create a superblock which can then later
 * be used for mounting.  The filesystem places a pointer to the root to be
 * used for mounting in @fc->root.
 */
int vfs_get_tree(struct fs_context *fc)
{
	struct super_block *sb;
	int error;

	if (fc->root)
		return -EBUSY;

	/* Get the mountable root in fc->root, with a ref on the root and a ref
	 * on the superblock.
	 */
	error = fc->ops->get_tree(fc);
	if (error < 0)
		return error;

	if (!fc->root) {
		pr_err("Filesystem %s get_tree() didn't set fc->root\n",
		       fc->fs_type->name);
		/* We don't know what the locking state of the superblock is -
		 * if there is a superblock.
		 */
		BUG();
	}

	sb = fc->root->d_sb;
	WARN_ON(!sb->s_bdi);

	/*
	 * Write barrier is for super_cache_count(). We place it before setting
	 * SB_BORN as the data dependency between the two functions is the
	 * superblock structure contents that we just set up, not the SB_BORN
	 * flag.
	 */
	smp_wmb();
	sb->s_flags |= SB_BORN;

	error = security_sb_set_mnt_opts(sb, fc->security, 0, NULL);
	if (unlikely(error)) {
		fc_drop_locked(fc);
		return error;
	}

	/*
	 * filesystems should never set s_maxbytes larger than MAX_LFS_FILESIZE
	 * but s_maxbytes was an unsigned long long for many releases. Throw
	 * this warning for a little while to try and catch filesystems that
	 * violate this rule.
	 */
	WARN((sb->s_maxbytes < 0), "%s set sb->s_maxbytes to "
		"negative value (%lld)\n", fc->fs_type->name, sb->s_maxbytes);

	return 0;
}
EXPORT_SYMBOL(vfs_get_tree);

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/fs_context.c#L588
/*
 * Get a mountable root with the legacy mount command.
 */
static int legacy_get_tree(struct fs_context *fc)
{
	struct legacy_fs_context *ctx = fc->fs_private;
	struct super_block *sb;
	struct dentry *root;

	root = fc->fs_type->mount(fc->fs_type, fc->sb_flags,
				      fc->source, ctx->legacy_data);
	if (IS_ERR(root))
		return PTR_ERR(root);

	sb = root->d_sb;
	BUG_ON(!sb);

	fc->root = root;
	return 0;
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/fs_context.c#L619
const struct fs_context_operations legacy_fs_context_ops = {
	.free			= legacy_fs_context_free,
	.dup			= legacy_fs_context_dup,
	.parse_param		= legacy_parse_param,
	.parse_monolithic	= legacy_parse_monolithic,
	.get_tree		= legacy_get_tree,
	.reconfigure		= legacy_reconfigure,
};

/*
 * Initialise a legacy context for a filesystem that doesn't support
 * fs_context.
 */
static int legacy_init_fs_context(struct fs_context *fc)
{
	fc->fs_private = kzalloc(sizeof(struct legacy_fs_context), GFP_KERNEL);
	if (!fc->fs_private)
		return -ENOMEM;
	fc->ops = &legacy_fs_context_ops;
	return 0;
}
```

上面`fc->fs_type->mount`调用的是 ext4_fs_type 的 mount 函数，也就是 ext4_mount，从文件系统里面读取超级块. 在文件系统的实现中，每个在硬盘上的结构，在内存中也对应相同格式的结构. 当所有的数据结构都读到内存里面，内核就可以通过操作这些数据结构，来操作文件系统了.

假设根文件系统下面有一个目录 home，有另外一个文件系统 A 挂载在这个目录 home 下面. 在文件系统 A 的根目录下面有另外一个文件夹 hello. 由于文件系统 A 已经挂载到了目录 home 下面，所以就有了目录 /home/hello，然后有另外一个文件系统 B 挂载在 /home/hello 下面. 在文件系统 B 的根目录下面有另外一个文件夹 world，在 world 下面有个文件夹 data. 由于文件系统 B 已经挂载到了 /home/hello 下面，所以就有了目录 /home/hello/world/data. 为了维护这些关系，操作系统创建了这一系列数据结构:
![](/misc/img/fs/663b3c5903d15fd9ba52f6d049e0dc27.jpeg)

从上图可知, 文件系统是树形组织. 每一个文件和文件夹都有 dentry，用于和 inode 关联. 因为这个例子涉及两次文件系统的挂载，再加上启动的时候挂载的根文件系统，一共三个 mount. 每个打开的文件都有一个 file 结构，它里面有两个变量，一个指向相应的 mount，一个指向相应的 dentry.

从最上面往下看, 根目录 / 对应一个 dentry，根目录是在根文件系统上的，根文件系统是系统启动的时候挂载的，因而有一个 mount 结构. 这个 mount 结构的 mount point 指针和 mount root 指针都是指向根目录的 dentry. 根目录对应的 file 的两个指针，一个指向根目录的 dentry，一个指向根目录的挂载结构 mount.

再来看第二层, 下一层目录 home 对应了两个 dentry，而且它们的 parent 都指向第一层的 dentry. 这是因为文件系统 A 挂载到了这个目录下, 这使得这个目录有两个用处: 一方面，home 是根文件系统的一个挂载点；另一方面，home 是文件系统 A 的根目录. 因为还有一次挂载，因而又有了一个 mount 结构. 这个 mount 结构的 mount point 指针指向作为挂载点的那个 dentry. mount root 指针指向作为根目录的那个 dentry，同时 parent 指针指向第一层的 mount 结构. home 对应的 file 的两个指针，一个指向文件系统 A 根目录的 dentry，一个指向文件系统 A 的挂载结构 mount.

再来看第三层, 目录 hello 又挂载了一个文件系统 B，所以第三层的结构和第二层几乎一样.

接下来是第四层, 目录 world 就是一个普通的目录. 只要它的 dentry 的 parent 指针指向上一层就可以了. 由于挂载点不变，world 对应的 file 结构还是指向第三层的 mount 结构.

接下来是第五层, 对于文件 data，是一个普通的文件，它的 dentry 的 parent 指向第四层的 dentry. 对于 data 对应的 file 结构，由于挂载点不变，还是指向第三层的 mount 结构.

## 打开文件
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/open.c#L1199
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
	return ksys_open(filename, flags, mode);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/syscalls.h#L1383
static inline long ksys_open(const char __user *filename, int flags,
			     umode_t mode)
{
	if (force_o_largefile())
		flags |= O_LARGEFILE;
	return do_sys_open(AT_FDCWD, filename, flags, mode);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/open.c#L1192
long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
	struct open_how how = build_open_how(flags, mode);
	return do_sys_openat2(dfd, filename, &how);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/open.c#L1163
static long do_sys_openat2(int dfd, const char __user *filename,
			   struct open_how *how)
{
	struct open_flags op;
	int fd = build_open_flags(how, &op);
	struct filename *tmp;

	if (fd)
		return fd;

	tmp = getname(filename);
	if (IS_ERR(tmp))
		return PTR_ERR(tmp);

	fd = get_unused_fd_flags(how->flags);
	if (fd >= 0) {
		struct file *f = do_filp_open(dfd, tmp, &op);
		if (IS_ERR(f)) {
			put_unused_fd(fd);
			fd = PTR_ERR(f);
		} else {
			fsnotify_open(f);
			fd_install(fd, f);
		}
	}
	putname(tmp);
	return fd;
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3379
struct file *do_filp_open(int dfd, struct filename *pathname,
		const struct open_flags *op)
{
	struct nameidata nd;
	int flags = op->lookup_flags;
	struct file *filp;

	set_nameidata(&nd, dfd, pathname);
	filp = path_openat(&nd, op, flags | LOOKUP_RCU);
	if (unlikely(filp == ERR_PTR(-ECHILD)))
		filp = path_openat(&nd, op, flags);
	if (unlikely(filp == ERR_PTR(-ESTALE)))
		filp = path_openat(&nd, op, flags | LOOKUP_REVAL);
	restore_nameidata();
	return filp;
}

// https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/dcache.h#L49
// 目录项名字
/*
 * "quick string" -- eases parameter passing, but more importantly
 * saves "metadata" about the string (ie length and the hash).
 *
 * hash comes first so it snuggles against d_parent in the
 * dentry.
 */
struct qstr {
	union {
		struct {
			HASH_LEN_DECLARE;
		};
		u64 hash_len; // 字符串长度
	};
	const unsigned char *name;
};

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L502
// 路径查找上下文
// 作用: 1. 在查找过程中, 记录当前查找所作用的路径; 2. 在查找结束后, 记录最终的目标路径
struct nameidata {
	struct path	path; // 保存已找到的路径
	struct qstr	last; // 路径名中最后一个组件的名字(文件名或目录名, 在LOOKUP_PARENT标志位设置时使用)
	struct path	root; // 查询的根路径. 一般使用当前进程的根目录
	struct inode	*inode; /* path.dentry.d_inode */
	unsigned int	flags; // 查询标志
	unsigned	seq, m_seq, r_seq;
	int		last_type; // 路径名最后一个组件的类型(在LOOKUP_PARENT标志位设置时使用) // 有5中: LAST_NORM(文件), LAST_ROOT(rooot), LAST_DOT(`.`), LAST_DOTDON(`..`), LAST_BIND(用于follow_link)
	unsigned	depth; // 当前正在查找的符号链接的嵌套深度
	int		total_link_count;
	struct saved {
		struct path link;
		struct delayed_call done;
		const char *name;
		unsigned seq;
	} *stack, internal[EMBEDDED_LEVELS]; // stack, 保存查找信息
	struct filename	*name;
	struct nameidata *saved;
	unsigned	root_seq;
	int		dfd;
	kuid_t		dir_uid;
	umode_t		dir_mode;
} __randomize_layout;

// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/path.h#L8
// 表示路径
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry;
} __randomize_layout;

// https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/fdtable.h#L27
struct fdtable {
	unsigned int max_fds; // 文件描述符指针数组的项数
	struct file __rcu **fd;      /* current fd array */ // 文件描述符指针数组
	unsigned long *close_on_exec; // 文件描述符指针数组的close_on_exec位图
	unsigned long *open_fds; // 文件描述符指针数组的open_fds位图
	unsigned long *full_fds_bits;
	struct rcu_head rcu; // 用于rcu
};

// https://elixir.bootlin.com/linux/v6.6.18/source/include/linux/fdtable.h#L49
// 进程不直接使用打开文件表, 而是通过files_struct动态扩展需要的指针数组
/*
 * Open file table structure
 */
struct files_struct {
  /*
   * read mostly part
   */
	atomic_t count; // 引用计数
	bool resize_in_progress;
	wait_queue_head_t resize_wait;

	struct fdtable __rcu *fdt; // 指向打开文件表
	struct fdtable fdtab; // 内嵌的打开文件表
  /*
   * written part on a separate cache line in SMP
   */
	spinlock_t file_lock ____cacheline_aligned_in_smp;
	unsigned int next_fd; // 要分配的下一个句柄的值
	unsigned long close_on_exec_init[1]; // 内嵌的close_on_exec位图
	unsigned long open_fds_init[1]; // 内嵌的open_fds位图
	unsigned long full_fds_bits_init[1];
	struct file __rcu * fd_array[NR_OPEN_DEFAULT]; // 内嵌的文件对象指针数组
};
```

files_struct通过fdt指向它实际使用的打开文件表. 对于大多数进程, 打开文件数据不超过32个. 这是无需另外分配空间, 此时它指向fdtab.

要打开一个文件，首先要通过 get_unused_fd_flags 得到一个没有用的文件描述符. 如何获取这个文件描述符呢？在每一个进程的 task_struct 中，有一个指针 files，类型是 [files_struct](https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/fdtable.h#L48).

files_struct 里面最重要的是一个文件描述符列表，每打开一个文件，就会在这个列表中分配一项，下标就是文件描述符.

对于任何一个进程，默认情况下，文件描述符 0 表示 stdin 标准输入，文件描述符 1 表示 stdout 标准输出，文件描述符 2 表示 stderr 标准错误输出. 另外，再打开的文件，都会从这个列表中找一个空闲位置分配给它.

文件描述符列表的每一项都是一个指向 struct file 的指针，也就是说，每打开一个文件，都会有一个 struct file 对应.

do_sys_open 中调用 do_filp_open，就是创建这个 struct file 结构，然后 fd_install(fd, f) 是将文件描述符和这个结构关联起来.

do_filp_open 里面首先初始化了 struct nameidata 这个结构. 文件都是一串的路径名称，需要逐个解析, 这个结构在解析和查找路径的时候提供辅助作用.

在 struct nameidata 里面有一个关键的成员变量 struct path.

其中，struct vfsmount 和文件系统的挂载有关, 另一个 struct dentry，除了上面说的用于标识目录之外，还可以表示文件名，还会建立文件名及其 inode 之间的关联.

接下来就调用 [path_openat](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3340)，主要做了以下几件事情：
1. alloc_empty_file 生成一个 struct file 结构
1. path_init 初始化 nameidata，准备开始节点路径查找
1. link_path_walk 对于路径名逐层进行节点路径查找，这里面有一个大的循环，用`/`分隔逐层处理
1. [open_last_lookups](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3111) 获取文件对应的 inode 对象，并且初始化 file 对象

例如，文件`/root/hello/world/data`，link_path_walk 会解析前面的路径部分`/root/hello/world`，解析完毕的时候 nameidata 的 dentry 为路径名的最后一部分的父目录`/root/hello/world`，而 nameidata 的 filename 为路径名的最后一部分`data`. 最后一部分的解析和处理，交给 open_last_lookups.

在open_last_lookups里面，需要先查找文件路径最后一部分对应的 dentry. Linux 为了提高目录项对象的处理效率，设计与实现了目录项高速缓存 dentry cache，简称 dcache. 它主要由两个数据结构组成：
- 哈希表 dentry_hashtable：dcache 中的所有 dentry 对象都通过 d_hash 指针链到相应的 dentry 哈希链表中
- 未使用的 dentry 对象链表 s_dentry_lru：dentry 对象通过其 d_lru 指针链入 LRU 链表中. 只要有它，就说明长时间不使用，就应该释放了.

![](/misc/img/fs/82dd76e1e84915206eefb8fc88385859.jpeg)


这两个列表之间会产生复杂的关系：
- 引用为 0：一个在散列表中的 dentry 变成没有人引用了，就会被加到 LRU 表中去
- 再次被引用：一个在 LRU 表中的 dentry 再次被引用了，则从 LRU 表中移除
- 分配：当 dentry 在散列表中没有找到，则从 Slub 分配器中分配一个
- 过期归还：当 LRU 表中最长时间没有使用的 dentry 应该释放回 Slub 分配器
- 文件删除：文件被删除了，相应的 dentry 应该释放回 Slub 分配器
- 结构复用：当需要分配一个 dentry，但是无法分配新的，就从 LRU 表中取出一个来复用

所以，open_last_lookups() 在查找 dentry 的时候，当然先从缓存中查找，调用的是 lookup_fast.

如果缓存中没有找到，就需要真的到文件系统里面去找了，[lookup_open](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3003) 会创建一个新的 dentry，并且[调用上一级目录的 Inode 的 inode_operations 的 lookup 函数](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3074)，对于 ext4 来讲，调用的是 ext4_lookup，会到文件系统里面去找 inode. 最终找到后将新生成的 dentry 赋给 path 变量.

path_openat() 的最后一步是调用 [do_open](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L3201) -> [vfs_open](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/open.c#L939) 真正打开文件.


[vfs_open](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/open.c#L939) -> [do_dentry_open](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/open.c#L768) , do_dentry_open 里面最终要做的一件事情是，调用 f->f_op->open，也就是调用 [ext4_file_operations](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/file.c#L884)的[ext4_file_open](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/file.c#L813). 另外一件重要的事情是将打开文件的所有信息，填写到 [struct file](https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/fs.h#L943https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/fs.h#L943) 这个结构里面.

### 目录结构
ext4文件系统内部的文件层级结构的组织方式可通过ext4_lookup函数(ext4_dir_inode_operations.lookup)了解.

目录也是文件,它也拥有属于自己的block,只不过普通文件的block存放的是文件的内容,目录的block中存放的是目录包含的子文件
的信息. 注意, 无论是普通文件的内容,还是目录的子文件的信息,都是它们的数据,不属于metadata.

ext4的目录有两种组织方式,第一种是线性方式,即每一个block都按照数组形式存放ext4_dir_entry或者ext4_dir_entry_2对象.

ext4_dir_entry和ext4_dir_entry_2是可以兼容的.

#### 线性目录
```c
// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4.h#L2286
struct ext4_dir_entry {
	__le32	inode;			/* Inode number */
	__le16	rec_len;		/* Directory entry length */
	__le16	name_len;		/* Name length */
	char	name[EXT4_NAME_LEN];	/* File name */
};

...

/*
 * The new version of the directory entry.  Since EXT4 structures are
 * stored in intel byte order, and the name_len field could never be
 * bigger than 255 chars, it's safe to reclaim the extra byte for the
 * file_type field.
 */
struct ext4_dir_entry_2 {
	__le32	inode;			/* Inode number */
	__le16	rec_len;		/* Directory entry length */ // dir entry的长度
	__u8	name_len;		/* Name length */
	__u8	file_type;		/* See file type macros EXT4_FT_* below */
	char	name[EXT4_NAME_LEN];	/* File name */
};
```
因为支持的名字最大长度为255,只需要一个字节即可表示,所以拆分ext4_dir_entry.name_len的两个字节,得到ext4_dir_entry_2的name_len和file_type

结构体的0x7字节的值为0,结构体的类型为ext4_dir_entry,否则为ext4_dir_entry_2,它们的第一个字段inode表示对应文件的inode号,
name_len字段表示文件名字的长度,name字段则表示文件的名字。一个entry占用磁盘空间包括结构体的前8个字节、文件的名字和额外的对齐三部分,值由rec_len字段表示。

看到name和inode字段, 应该明白lookup的原理了,目录的所有子文件都对应一个entry,如果某个entry的name字段与我们查找的文
件名字匹配,它的inode字段的值就是要查找的文件的inode号ino,有了ino就可以得到目标文件了.

最后一个entry, 它的rec_len字段等于0xfd0=0x1000-0xc × 4,也就是说最后一个entry的rec_len等于block被前
面所有entry占用后剩下的空间大小。这是为了方便插入新的entry,不需要累加各entry计算block已经被占用的空间,只需要查看最后一个entry即可.

创建新文件是申请到可用的inode号,初始化,然后创建一个entry添加到目录的block的entry数组的末尾。只不过一个block满了,可能要申请新的block.

#### 哈希树目录
如果文件系统支持dir_index特性(EXT4_FEATURE_COMPAT_DIR_INDEX),目录有线性和哈希树两种选择,若它的ext4_inode的flags字段的EXT4_INDEX_FL置位,则它采用的是哈希树方式,否则是线性方式.

一般情况下,目录的子文件不会太多,一个block足够满足它们的entry对磁盘空间的需要,搜索某一个entry的代价不会太大,但如果子
文件多了起来,需要多个block的情况下,搜索一个entry可能要搜多个block,最坏的情况下,搜索一个不存在的entry,要把每一个block都遍
历一遍,效率非常低。

于是,ext文件系统引入了dir_index特性,它的主要目的就是为了解决在子文件太多的情况下,线性目录搜索效率低下的问题。dir_index根据子文件的名字,计算得到哈希值,根据哈希值来判断文件的entry可以存入哪个block,如果block满了,存入相关block,查找的时候同样根据这个哈希值查找相关的block(by ext4_dx_add_entry),不需要每一个block查找.

某个目录选择的方式并不是一成不变的,一般情况下,目录的子文件的entry不超过一个block的情况下,选择线性方式;超过一个block,内核会自动切换成哈希树方式,更新它的block。ext4为哈希树目录定义了几个结构体,表示整个树结构。dx_root表示树的根.

```c
struct fake_dirent
{
	__le32 inode;
	__le16 rec_len;
	u8 name_len;
	u8 file_type;
};

struct dx_countlimit
{
	__le16 limit;
	__le16 count;
};

struct dx_entry
{
	__le32 hash;
	__le32 block;
};

// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/namei.c#L246
/*
 * dx_root_info is laid out so that if it should somehow get overlaid by a
 * dirent the two low bits of the hash version will be zero.  Therefore, the
 * hash version mod 4 should never be 0.  Sincerely, the paranoia department.
 */

struct dx_root
{
	struct fake_dirent dot;
	char dot_name[4]; // `.\0\0\0`
	struct fake_dirent dotdot;
	char dotdot_name[4]; // `..\0\0`
	struct dx_root_info
	{
		__le32 reserved_zero;
		u8 hash_version; // 计算文件名hash的hash方法
		u8 info_length; /* 8 */ // info长度
		u8 indirect_levels; // 树的深度
		u8 unused_flags;
	}
	info;
	struct dx_entry	entries[]; // 线性分别的dx_entry
};

struct dx_node
{
	struct fake_dirent fake;
	struct dx_entry	entries[];
};
```

dx_root存放在目录的第一个block,开头按照ext4_dir_entry_2的格
式存入“.”和“..”两个文件的信息.

0x18加上info.info_length就是entries的开始,entries[0]的block表示哈希值为0时,对应的block号,该号并不是基于
磁盘的,而是文件的第几个block。由此开始,基于文件的block, 称之为逻辑block,其余block除非特别说明均为基于磁盘的。dx_entry
有hash和block两个字段,分别表示哈希值和对应的block号。entries[0]的hash字段被拆分成dx_countlimit结构体的limit和count字段,表示
dx_entry的数量限制和当前数量.

info.indirect_levels表示树的深度,该字段的最大值为1,表示最多有一层中间结点。当它为0时,dx_entry的block字段对应的block中存放
的是ext4_dir_entry_2的线性数组。当它等于1时,dx_root的dx_entry的block字段对应的block中存放的是dx_node对象.

dx_node包含的dx_entry的block字段对应的block按照线性方式存放ext4_dir_entry_2.

哈希树目录的结构比线性目录复杂得多,但最终还是ext4_dir_entry_2来表示一个子文件的entry,所以lookup和创建文件方
面二者并没有本质不同,区别仅在于搜索和插入entry的位置计算方式不同.

#### 硬链接
建立一个硬链接,只不过在目标目录下插入了一个ext4_dir_entry_2对象,其inode字段的值与原文件的inode号
相等,它们共享同一个ext4_inode。只要inode号相同,名字和路径无论怎么变,最终都是同一个文件.

### 文件io
文件的block是由ext4_inode的i_block字段表示的,该字段是个数组,长60字节(__le32[15]),用来表示文件的block时,有两种使用方式 : Direct/Indirect Map 和 Extent Tree (区段树). 如果文件的EXT4_EXTENTS_FL标记清零,使用的是前者,否则使用的是后者.

#### 映射
60个字节,如果每四个字节映射一个block号,最多只能有15个block。ext2和ext3开始,将60个字节分为两部分,数组的前12个元素共48个字节直接映射一个block,后三个元素分别采用1、2和3级间接映射。

所谓间接映射就是,元素映射的block中存放的并不是文件的内容或者目录的子信息,而是一个中间的映射表(Indirect Block)。4096个字节,每四个字节映射一个下级block,可达4096/ 4 = 1024项。下级block中存放的是数据还是中间的映射表,由映射的层级决定。

i_block[12]采用一级间接映射,i_block[12]到block_l1(level 1)完成了间接映射,所以block_l1映射的block中存放的是文件的数据。i_block[13]采用二级间接映射,i_block[13]到block_l1再到block_l2才能完成间接映射,所以block_l1映射的block中存放的是block_l2, block_l2映射的block中存放的是文件的数据。

i_block[14]采用三级间接映射,所以block_l1映射的block中存放的是block_l2,block_l2映射的block中存放的是block_l3,block_l3映射的
block中存放的才是文件的数据。

每四个字节(__le32)映射一个block,采用的是直接映射,四个字节的整数值就是基于磁盘的block号,所以采用这种方式的文件,它的block(包括中间参与映射的block)都必须在2^32 block内。文件的逻辑block与项(4字节为一项)的索引值是一致的,比如文件的block 1对应i_block[1],block 20对应i_block[12]指向的block_l1的第8项(从第0项开始算起).

#### 区段树
ext4由Direct/Indirect Map切换到了Extent Tree,前者一个项对应一个block,比较浪费空间。Extent单词的意思是区间,不言而喻,Extent
Tree就是一些表示block区间的数据结构组成的树。一个数据结构对应一个区间的block,而不再是Map中的一个项对应一个block的关系。
Extent Tree有点特别,它的每一个中间结点对应的block存放的都是一棵树 ,ext4定义了ext4_extent_header 、 ext4_extent_idx 和ext4_extent三个结构体表示树根、中间结点和树叶,以下分别简称header、idx和extent.

```c
// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4_extents.h#L63
/*
 * This is the extent on-disk structure.
 * It's used at the bottom of the tree.
 */
struct ext4_extent {
	__le32	ee_block;	/* first logical block extent covers */ // extent涵盖的文件的逻辑block区间的起始值
	__le16	ee_len;		/* number of blocks covered by extent */ // extent涵盖的block的数量
	__le16	ee_start_hi;	/* high 16 bits of physical block */ // extent指向的第一关block的block号的高16位
	__le32	ee_start_lo;	/* low 32 bits of physical block */ // extent指向的第一关block的block号的低32位
};

/*
 * This is index on-disk structure.
 * It's used at all the levels except the bottom.
 */
struct ext4_extent_idx {
	__le32	ei_block;	/* index covers logical blocks from 'block' */ // idx对应的逻辑block号
	__le32	ei_leaf_lo;	/* pointer to the physical block of the next *
				 * level. leaf or next index could be there */ // idx指向的block的block号的低32位
	__le16	ei_leaf_hi;	/* high 16 bits of physical block */  // idx指向的block的block号的高16位
	__u16	ei_unused;
};


// https://elixir.bootlin.com/linux/v6.6.29/source/fs/ext4/ext4_extents.h#L85
/*
 * Each block (leaves and indexes), even inode-stored has header.
 */
struct ext4_extent_header {
	__le16	eh_magic;	/* probably will support different formats */ // 0xF30A
	__le16	eh_entries;	/* number of valid entries */ // header后entry的数量
	__le16	eh_max;		/* capacity of store in entries */ // header后entry的最大数目
 	__le16	eh_depth;	/* has tree real underlying blocks? */ // 树的中间节点的高度
	__le32	eh_generation;	/* generation of the tree */ // ext4未使用
};
```
ext4_extent_header结构体表示树根,共12个字节.

第一个header对象存在i_block的前12个字节,后面48个字节可以存放idx或者extent,eh_entries字段表示某个header后跟随的entry的数
量,eh_max字段表示header后跟随的entry的最大数量,eh_depth字段表示树的中间结点高度,如果为0,说明紧随其后的是extent,最大值为5.

ext4_extent_idx结构体表示树的中间结点.

ei_leaf_lo和ei_leaf_hi组成目标块的block号,共48位。块中包含的是中间结点还是树叶由header的eh_depth字段决定,eh_depth大于0为中
间结点,等于0则为树叶。ei_block表示该idx涵盖的文件的逻辑block区间的起始值。

ext4_extent结构体表示树叶,它指向文件的一个block区间

extent表示的block区间是连续的,ee_len字段表示该区间的block数量,如果它的值不大于32768,extent已经初始化,实际的block数量与
该值相等;如果大于32768,则extent并没有初始化,实际的block数量等于ee_len-32868。ee_start_hi和ee_start_lo组成起始块的block号,也是
48位,所以采用Extent Tree的文件的block都应该在2^48 block内.

## 总结
![](/misc/img/fs/8070294bacd74e0ac5ccc5ac88be1bb9.png)

对于每一个进程，打开的文件都有一个文件描述符，在 files_struct 里面会有文件描述符数组. 每个一个文件描述符是这个数组的下标，里面的内容指向一个 file 结构，表示打开的文件。这个结构里面有这个文件对应的 inode，最重要的是这个文件对应的操作 file_operation. 如何操作这个文件，就看这个 file_operation 里面的定义了.

对于每一个打开的文件，都有一个 dentry 对应，虽然叫作 directory entry，但是不仅仅表示文件夹，也表示文件. 它最重要的作用就是指向这个文件对应的 inode.

如果说 file 结构是一个文件打开以后才创建的，dentry 是放在一个 dentry cache 里面的，文件关闭了，它依然存在，因而它可以更长期地维护内存中的文件的表示和硬盘上文件的表示之间的关系.

inode 结构就表示硬盘上的 inode，包括块设备号等. 几乎每一种结构都有自己对应的 operation 结构，里面都是一些方法，因而当后面遇到对于某种结构进行处理的时候，如果不容易找到相应的处理函数，就先找这个 operation 结构，就清楚了.

## 系统调用层和虚拟文件系统层
文件系统的读写，其实就是调用系统函数 read 和 write.

read系统调用的入口为sys_read,先调用file_pos_read获取file对象当前的fp,然后调用vfs_read读文件,最后调用file_pos_write更新fp。
读写位置是属于file对象的,而不是属于文件的,由file->f_pos字段表示.

open同一个文件两次之后,fd1和fd2对应两个不同的file对象。fd1的file对象的fp在write之后得到了更新,文件的内容为abcdefg;fd2的
file对象的在close(fd1)后仍然为0,所以write(fd2)写到了文件的开头,最终的结果是hijklfg.

vfs_read需要文件系统至少实现file->f_op->read和file->f_op->read_iter二者之一。如果文件系统定义了read,则调用read;否则调
用new_sync_read,后者调用read_iter实现。

write像极了read,它的系统调用入口为sys_write,先调用file_pos_read获取file对象当前fp,然后调用vfs_write写文件,最后调用
file_pos_write更新fp。vfs_write也与vfs_read类似,如果文件系统实现了file->f_op->write,则调用它;否则调用new_sync_write,
new_sync_write调用filp->f_op->write_iter实现写操作。

对文件读写实际上是一个复杂的过程,VFS仅仅是调用文件系统的回调函数,具体的逻辑由文件系统自行决定.

3.10版内核并没有read_iter和write_iter,有的是aio_read和aio_write,它们的功能类似。a表示asynchronized,意思是异步I/O。所
谓的异步I/O,就是不等I/O操作完成即返回,操作完成再行处理。new_sync_read和new_sync_write中的new,是相对于do_sync_read和
do_sync_write而言的。3.10版内核中,以read为例,如果文件系统定义了read,则调用read;否则调用do_sync_read,后者调用aio_read实现。

无论是new_sync_xxx还是do_sync_xxx都是synchronized,同步的,也就是等待I/O操作完成。aio_xxx是实现异步读写的,xxx_iter可
以实现同步、异步读写,也就是说new_sync_xxx和do_sync_xxx通过可以实现异步读写的操作来实现。

逻辑上看似有些矛盾,实际上,do_sync_xxx调用aio_xxx后并不会直接返回,如果aio_xxx的返回值等于-EIOCBQUEUED(iocb
queued,表示已经插入I/O操作),它会调用wait_on_sync_kiocb等待I/O操作结束再返回。

至于new_sync_xxx调用xxx_iter,内核不允许它们在这种情况下返回-EIOCBQU-EUED,也就是说文件系统定义xxx_iter的时候需要区分
当前的请求是同步I/O,还是异步I/O,根据请求决定策略,内核提供了is_sync_kiocb函数辅助判断请求是否为同步I/O。

需要说明的是,异步I/O和读写置位了O_NONBLOCK的文件并不是一回事,异步I/O并不要求进行实际的I/O操作,O_NONBLOCK的意
思是进行I/O操作时,发现“无事可做”,不再等待,它描述的问题是阻塞(Block)。比如要买菜市场的菜,你可以在网上点外送,然后继续
忙其他事情,这是异步I/O;也可以去楼下,发现去早了,菜市场还没开门。等待是阻塞,不等待是非阻塞。你去了,就是同步I/O,不管有没有买到菜.

aio的使用分为两种情况,一种是glibc定义的aio_xxx函数,另一种是内核定义的io_submit、io_getevents等系统调用,与read/write相比,
它们的使用场景较少.

```c
ssize_tksys_read(unsignedintfd, char __user *buf, size_t count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos, *ppos = file_ppos(f.file);
		if (ppos) {
			pos = *ppos;
			ppos = &pos;
		}
		ret = vfs_read(f.file, buf, count, ppos); // 指向文件读取
		if (ret >= 0 && ppos)
			f.file->f_pos = pos;
		fdput_pos(f);
	}
	return ret;
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/read_write.c#L596
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
	return ksys_read(fd, buf, count);
}

ssize_t ksys_write(unsigned int fd, const char __user *buf, size_t count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos, *ppos = file_ppos(f.file);
		if (ppos) {
			pos = *ppos;
			ppos = &pos;
		}
		ret = vfs_write(f.file, buf, count, ppos);
		if (ret >= 0 && ppos)
			f.file->f_pos = pos;
		fdput_pos(f);
	}

	return ret;
}

SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	return ksys_write(fd, buf, count);
}
```

对于 read 来讲，里面调用 [vfs_read](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/read_write.c#L447)->[__vfs_read](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/read_write.c#L422). 对于 write 来讲，里面调用 [vfs_write](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/read_write.c#L543)->[__vfs_write](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/read_write.c#L491).


之前讲过，每一个打开的文件，都有一个 struct file 结构. 这里面有一个 struct file_operations f_op，用于定义对这个文件做的操作. __vfs_read 会调用相应文件系统的 file_operations 里面的 read 操作，__vfs_write 会调用相应文件系统 file_operations 里的 write 操作.

对于ext4, f_op就是ext4_file_operations. 由于 ext4 没有定义 read 和 write 函数，于是会调用 [ext4_file_read_iter](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/file.c#L114) 和 [ext4_file_write_iter](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/file.c#L641). ext4_file_read_iter 会调用 [generic_file_read_iter](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/filemap.c#L2257)->[generic_file_buffered_read](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/filemap.c#L1991)，ext4_file_write_iter 会调用 [ext4_buffered_write_iter](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/file.c#L255) ->[generic_perform_write](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/filemap.c#L3258).

ext4_file_read_iter 和 ext4_file_write_iter有相似的逻辑，就是要区分是否用缓存.

缓存其实就是内存中的一块空间. 因为内存比硬盘快得多，Linux 为了改进性能，有时候会选择不直接操作硬盘，而是读写都在内存中，然后批量读取或者写入硬盘. 一旦能够命中内存，读写效率就会大幅度提高.

因此，根据是否使用内存做缓存，就可以把文件的 I/O 操作分为两种类型:
1. 缓存 I/O. 大多数文件系统的默认 I/O 操作都是缓存 I/O. 对于读操作来讲，操作系统会先检查，内核的缓冲区有没有需要的数据. 如果已经缓存了，那就直接从缓存中返回；否则从磁盘中读取，然后缓存在操作系统的缓存中. 对于写操作来讲，操作系统会先将数据从用户空间复制到内核空间的缓存中. 这时对用户程序来说，写操作就已经完成. 至于什么时候再写到磁盘中由操作系统决定，除非显式地调用了 sync 同步命令.
2. 直接 IO, 就是应用程序直接访问磁盘数据，而不经过内核缓冲区，从而减少了在内核缓存和用户程序之间数据复制.

如果在读的逻辑 ext4_file_read_iter 里面，发现设置了 IOCB_DIRECT，则会调用 [ext4_dio_read_iter](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/file.c#L52) 的函数，将直接从硬盘中读取数据. 类似的ext4_file_write_iter是走[ext4_dio_write_iter](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/file.c#L450).

> 以前ext4 direct io的ext4_direct_IO已被删除 on 378f32bab3714f04c4e0c3aee4129f6703805550.

ext4_dio_write_iter最终会调到[iomap_dio_rw](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/iomap/direct-io.c#L406)，这就跨过了缓存层，到了通用块层，最终到了文件系统的设备驱动层.

### 带缓存的写入操作
[ext4_buffered_write_iter](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/ext4/file.c#L255) ->[generic_perform_write](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/filemap.c#L3258), generic_perform_write函数里，是一个 while 循环. 需要找出这次写入影响的所有的页，然后依次写入. 对于每一个循环，主要做四件事情：
1. 对于每一页，先调用 [address_space_operations](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/inode.c#L3605) 的 write_begin 做一些准备
1. 调用 [iov_iter_copy_from_user_atomic](https://elixir.bootlin.com/linux/v5.8-rc3/source/lib/iov_iter.c#L987)，将写入的内容从用户态拷贝到内核态的页中
1. 调用 address_space 的 write_end 完成写操作
1. 调用 [balance_dirty_pages_ratelimited](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/page-writeback.c#L1877)，看脏页是否太多，需要写回硬盘. 所谓脏页，就是写入到缓存，但是还没有写入到硬盘的页面.

> address_space主要用于在内存映射的时候将文件和内存页产生关联.

第一步，对于 ext4 来讲，调用的是 ext4_write_begin. ext4 是一种日志文件系统，是为了防止突然断电的时候的数据丢失，引入了日志**（Journal）**模式. 日志文件系统比非日志文件系统多了一个 Journal 区域. 文件在 ext4 中分两部分存储，一部分是文件的元数据，另一部分是数据. 元数据和数据的操作日志 Journal 也是分开管理的. 可以在挂载 ext4 的时候，选择 Journal 模式. 这种模式在将数据写入文件系统前，必须等待元数据和数据的日志已经落盘才能发挥作用. 这样性能比较差，但是最安全.

另一种模式是 order 模式. 这个模式不记录数据的日志，只记录元数据的日志，但是在写元数据的日志前，必须先确保数据已经落盘. 这个折中，是默认模式.

还有一种模式是 writeback，不记录数据的日志，仅记录元数据的日志，并且不保证数据比元数据先落盘. 这个性能最好，但是最不安全. 在 ext4_write_begin，能看到对于 ext4_journal_start 的调用，就是在做日志相关的工作. 在 ext4_write_begin 中，还做了另外一件重要的事情，就是调用 [grab_cache_page_write_begin](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/filemap.c#L3240)，来得到应该写入的缓存页.

在内核中，缓存以页为单位放在内存里面，那如何知道，一个文件的哪些数据已经被放到缓存中了呢？每一个打开的文件都有一个 struct file 结构，每个 struct file 结构都有一个 struct [address_space](https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/fs.h#L447) 用于关联文件和内存，就是在这个结构里面，有一棵树，用于保存所有与这个文件相关的的缓存页. 查找的时候，往往需要根据文件中的偏移量找出相应的页面，而基数树 radix tree 这种数据结构能够快速根据一个长整型查找到其相应的对象，因而这里缓存页就放在 radix 基数树里面.

pagecache_get_page 就是根据 pgoff_t index 这个长整型，在这棵树里面查找缓存页，如果找不到就会创建一个缓存页.

第二步，调用 iov_iter_copy_from_user_atomic. 先将分配好的页面调用 kmap_atomic 映射到内核里面的一个虚拟地址，然后将用户态的数据拷贝到内核态的页面的虚拟地址中，调用 kunmap_atomic 把内核里面的映射删除.

第三步，调用 ext4_write_end 完成写入. 这里面会调用 ext4_journal_stop 完成日志的写入，会调用 block_write_end->__block_commit_write->mark_buffer_dirty，将修改过的缓存标记为脏页. 可以看出，其实所谓的完成写入，并没有真正写入硬盘，仅仅是写入缓存后，标记为脏页. 写操作由一个 timer 触发，那个时候，才调用 wb_workfn 往硬盘写入页面.

但是这里有一个问题，数据很危险，一旦宕机就没有了，所以需要一种机制，将写入的页面真正写到硬盘中，称为回写（Write Back）

第四步，调用 balance_dirty_pages_ratelimited，是回写脏页的一个很好的时机.

在 balance_dirty_pages_ratelimited 里面，发现脏页的数目超过了规定的数目，就调用 [balance_dirty_pages](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/page-writeback.c#L1555)->[wb_start_background_writeback](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/fs-writeback.c#L1107)，启动一个背后线程开始回写.
```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/fs-writeback.c#L152
static void wb_wakeup(struct bdi_writeback *wb)
{
	spin_lock_bh(&wb->work_lock);
	if (test_bit(WB_registered, &wb->state))
		mod_delayed_work(bdi_wq, &wb->dwork, 0);
	spin_unlock_bh(&wb->work_lock);
}

// https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/backing-dev.c#L35
struct workqueue_struct *bdi_wq;

// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/workqueue.h#L255
#define __INIT_DELAYED_WORK(_work, _func, _tflags)			\
	do {								\
		INIT_WORK(&(_work)->work, (_func));			\
		__init_timer(&(_work)->timer,				\
			     delayed_work_timer_fn,			\
			     (_tflags) | TIMER_IRQSAFE);		\
	} while (0)

// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/workqueue.h#L271
#define INIT_DELAYED_WORK(_work, _func)					\
	__INIT_DELAYED_WORK(_work, _func, 0)
```

通过上面的代码可以看出，bdi_wq 是一个全局变量，所有回写的任务都挂在这个队列上. mod_delayed_work 函数负责将一个回写任务 bdi_writeback 挂在这个队列上. bdi_writeback 有个成员变量 struct delayed_work dwork，bdi_writeback 就是以 delayed_work 的身份挂到队列上的，并且把 delay 设置为 0，意思就是一刻不等，马上执行.

> Linux 2.6.32内核之后，放弃了原有的pdflush机制，改成了bdi_writeback机制. bdi_writeback机制主要解决了原有fdflush机制存在的一个问题：在多磁盘的系统中，pdflush管理了所有磁盘的Cache，从而导致一定程度的I/O瓶颈. bdi_writeback机制为每个磁盘都创建了一个线程，专门负责这个磁盘的Page Cache的刷新工作，从而实现了每个磁盘的数据刷新在线程级的分离，提高了I/O性能. 回写机制存在的问题是回写不及时引发数据丢失（可由sync|fsync解决），回写期间读I/O性能很差.

这里的 bdi 的意思是 backing device info，用于描述后端存储相关的信息, 每个块设备都会有这样一个结构，并且在初始化块设备的时候，调用 bdi_init 初始化这个结构，在初始化 bdi 的时候，也会调用 [wb_init](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/backing-dev.c#L283) 初始化 bdi_writeback.

wb_init里面最重要的是 INIT_DELAYED_WORK. 其实就是初始化一个 timer，也即定时器，到时候就执行 wb_workfn 这个函数.

接下来的调用链为：[wb_workfn](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/fs-writeback.c#L2060)->[wb_do_writeback](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/fs-writeback.c#L2029)->[wb_writeback](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/fs-writeback.c#L1837)->[writeback_sb_inodes](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/fs-writeback.c#L1624)->[__writeback_single_inode](https://elixir.bootlin.com/linux/v5.8-rc4/source/fs/fs-writeback.c#L1441)->[do_writepages](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/page-writeback.c#L2346) -> [mapping->a_ops->writepages(mapping, wbc);](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/page-writeback.c#L2354)，在 do_writepages 中，要调用 mapping->a_ops->writepages，但实际调用的是 ext4_writepages，往设备层写入数据.

在调用 write 的最后，当发现缓存的数据太多的时候，会触发回写，这仅仅是回写的一种场景. 另外还有几种场景也会触发回写：
- 用户主动调用 sync，将缓存刷到硬盘上去，最终会调用 wakeup_flusher_threads，同步脏页
- 当内存十分紧张，以至于无法分配页面的时候，会调用 free_more_memory，最终会调用 wakeup_flusher_threads，释放脏页
- 脏页已经更新了较长时间，时间上超过了 timer，需要及时回写，保持内存和磁盘上数据一致性

### 带缓存的读操作
读取比写入总体而言简单一些，主要涉及预读的问题.

在 generic_file_buffered_read 函数中，需要先找到 page cache 里面是否有缓存页. 如果没有找到，不但读取这一页，还要进行预读，这需要在 page_cache_sync_readahead 函数中实现. 预读完了以后，再试一把查找缓存页，应该能找到了. 如果第一次找缓存页就找到了，还是要判断，是不是应该继续预读；如果需要，就调用 page_cache_async_readahead 发起一个异步预读. 最后，copy_page_to_iter 会将内容从内核缓存页拷贝到用户内存空间.

## 总结
在系统调用层的read 和 write, 在 VFS 层调用的是 vfs_read 和 vfs_write 并且调用 file_operation. 在 ext4 层调用的是 ext4_file_read_iter 和 ext4_file_write_iter.

接下来就是分叉, 需要区分缓存 I/O 和直接 I/O. 直接 I/O 读写的流程是一样的，调用 iomap_dio_rw，再往下就调用块设备层了. 缓存 I/O 读写的流程不一样. 对于读，从块设备读取到缓存中，然后从缓存中拷贝到用户态. 对于写，从用户态拷贝到缓存，设置缓存页为脏，然后启动一个线程写入块设备.