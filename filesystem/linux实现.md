# fs
参考:
- [一口气搞懂「文件系统」，就靠这 25 张图了](https://www.tuicool.com/articles/63qam22)

## vfs
文件系统的种类众多，而操作系统希望 对用户提供一个统一的接口，于是在用户层与文件系统层引入了中间层，这个中间层就称为 虚拟文件系统(Virtual File System，VFS, 只存在内存中), 是驻留在用户进程和各种类型的linux 文件系统之间的一个抽象接口层, 对用户进程隐藏了实现每个文件系统的差异.

VFS 定义了一组所有文件系统都支持的数据结构和标准接口，这样程序员不需要了解文件系统的工作原理，只需要了解 VFS 提供的统一接口即可.

Linux为了实现这种VFS系统，采用面向对象的设计思路，主要抽象了四种对象类型：
1. 超级块对象(super_block)：代表一个文件系统,用于存储该文件系统的有关信息.

	fs中所有的inode都会链接到super_block的链表头.

	```c
	//https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/fs.h#L1154
	struct super_block {
		struct list_head	s_list;		/* Keep this first */ //超级块链表
		dev_t			s_dev;		/* search index; _not_ kdev_t */ // 设备标识
		unsigned char		s_blocksize_bits; //以bit为单位的块大小
		unsigned long		s_blocksize; //以B为单位的块大小
		loff_t			s_maxbytes;	/* Max file size */ //一个文件最大字节树
		struct file_system_type	*s_type; //文件系统类型
		const struct super_operations	*s_op; //操作超级块的函数集合
		const struct dquot_operations	*dq_op; //磁盘限额函数集合
		const struct quotactl_ops	*s_qcop;
		const struct export_operations *s_export_op; // 支持s_export_op接口的文件系统都是存储设备文件系统，如ext3/4、ubifs等.
		unsigned long		s_flags; // 挂载标志
		unsigned long		s_iflags;	/* internal SB_I_* flags */
		unsigned long		s_magic; // fs magic number
		struct dentry		*s_root; // 挂载目录, 指向fs root dentry的指针
		struct rw_semaphore	s_umount; // 卸载信号量
		int			s_count; // 引用计数
		atomic_t		s_active; // 活动计数
	#ifdef CONFIG_SECURITY
		void                    *s_security;
	#endif
		const struct xattr_handler **s_xattr;
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
		struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */
		struct block_device	*s_bdev; // 指向fs存在的块设备指针
		struct backing_dev_info *s_bdi;
		struct mtd_info		*s_mtd;
		struct hlist_node	s_instances;
		unsigned int		s_quota_types;	/* Bitmask of supported quota types */
		struct quota_info	s_dquot;	/* Diskquota specific options */

		struct sb_writers	s_writers;

		/*
		 * Keep s_fs_info, s_time_gran, s_fsnotify_mask, and
		 * s_fsnotify_marks together for cache efficiency. They are frequently
		 * accessed and rarely modified.
		 */
		void			*s_fs_info;	/* Filesystem private info */ //fs info

		/* Granularity of c/m/atime in ns (cannot be worse than a second) */
		u32			s_time_gran;
		/* Time limits for c/m/atime in seconds */
		time64_t		   s_time_min; // 最小时间限制
		time64_t		   s_time_max; // 最大时间限制
	#ifdef CONFIG_FSNOTIFY
		__u32			s_fsnotify_mask;
		struct fsnotify_mark_connector __rcu	*s_fsnotify_marks;
	#endif

		char			s_id[32];	/* Informational name */ //标志名称
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
		const char *s_subtype;

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
		struct list_lru		s_dentry_lru; // lru方式挂载的目录
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
		struct list_head	s_inodes;	/* all inodes */ // 指向fs内所有的inode, 通过它可遍历inode对象

		spinlock_t		s_inode_wblist_lock; // 回写inode的锁
		struct list_head	s_inodes_wb;	/* writeback inodes */ //挂载所有要回写的inode
	} __randomize_layout;

	// https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/fs.h#L1912
	struct super_operations {
	   	struct inode *(*alloc_inode)(struct super_block *sb);  //分配一个新的索引结点结构
		void (*destroy_inode)(struct inode *); //销毁给定的索引节点
		void (*free_inode)(struct inode *); //释放给定的索引节点

	   	void (*dirty_inode) (struct inode *, int flags); //VFS在索引节点为脏(改变)时，会调用此函数
		int (*write_inode) (struct inode *, struct writeback_control *wbc);  //该函数用于将给定的索引节点写入磁盘
		int (*drop_inode) (struct inode *); //在最后一个指向索引节点的引用被释放后，VFS会调用该函数
		void (*evict_inode) (struct inode *);
		void (*put_super) (struct super_block *); //减少超级块计数调用
		int (*sync_fs)(struct super_block *sb, int wait); //同步文件系统调用
		int (*freeze_super) (struct super_block *); //释放超级块调用
		int (*freeze_fs) (struct super_block *); //释放文件系统调用
		int (*thaw_super) (struct super_block *);
		int (*unfreeze_fs) (struct super_block *);
		int (*statfs) (struct dentry *, struct kstatfs *); //VFS通过调用该函数，获取文件系统状态
		int (*remount_fs) (struct super_block *, int *, char *); //当指定新的安装选项重新安装文件系统时，VFS会调用此函数
		void (*umount_begin) (struct super_block *); //VFS调用该函数中断安装操作。该函数被网络文件系统使用，如NFS

		int (*show_options)(struct seq_file *, struct dentry *);
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

	```c
	//https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/fs.h#L608
	/*
	 * Keep mostly read-only and often accessed (especially for
	 * the RCU path lookup and 'stat' data) fields at the beginning
	 * of the 'struct inode'
	 */
	struct inode {
		umode_t			i_mode; //文件访问权限, 见[这里](https://elixir.bootlin.com/linux/v5.12.9/source/include/uapi/linux/stat.h#L10)
		unsigned short		i_opflags; // 打开file时的标志
		kuid_t			i_uid;
		kgid_t			i_gid;
		unsigned int		i_flags;

	#ifdef CONFIG_FS_POSIX_ACL
		struct posix_acl	*i_acl;
		struct posix_acl	*i_default_acl;
	#endif

		const struct inode_operations	*i_op; //操作inode的函数集合
		struct super_block	*i_sb; //指向所属超级块
		struct address_space	*i_mapping; //文件数据在内存中的页缓存. 缓存文件的内容 by radix tree. 对文件的读写操作首先在i_mapping中的缓存里查找. 如果缓存存在则从缓存获取, 不用访问存储设备, 这加速了文件操作.

	#ifdef CONFIG_SECURITY
		void			*i_security;
	#endif

		/* Stat data, not accessed from path walking */
		unsigned long		i_ino; //inode
		/*
		 * Filesystems may only read i_nlink directly.  They shall use the
		 * following functions for modification:
		 *
		 *    (set|clear|inc|drop)_nlink
		 *    inode_(inc|dec)_link_count
		 */
		union {
			const unsigned int i_nlink;
			unsigned int __i_nlink;
		};
		dev_t			i_rdev; //实际设备
		loff_t			i_size; //文件大小(B)
		struct timespec64	i_atime;
		struct timespec64	i_mtime;
		struct timespec64	i_ctime;
		spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
		unsigned short          i_bytes; //使用的字节数
		u8			i_blkbits; //块大小(bit)
		u8			i_write_hint;
		blkcnt_t		i_blocks;

	#ifdef __NEED_I_SIZE_ORDERED
		seqcount_t		i_size_seqcount;
	#endif

		/* Misc */
		unsigned long		i_state;
		struct rw_semaphore	i_rwsem;

		unsigned long		dirtied_when;	/* jiffies of first dirtying */
		unsigned long		dirtied_time_when;

		struct hlist_node	i_hash;
		struct list_head	i_io_list;	/* backing dev IO list */
	#ifdef CONFIG_CGROUP_WRITEBACK
		struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

		/* foreign inode detection, see wbc_detach_inode() */
		int			i_wb_frn_winner;
		u16			i_wb_frn_avg_time;
		u16			i_wb_frn_history;
	#endif
		struct list_head	i_lru;		/* inode LRU list */ // 用于链接描述inode当前状态的链表. 当创建一个新的inode时i_lru要链接到inode_in_use这个链表, 表示inode处于使用中, 同时i_sb_list要链接到super_block中的s_inodes链表
		struct list_head	i_sb_list; // 用于链接到super_block中的s_inodes链表
		struct list_head	i_wb_list;	/* backing dev writeback list */
		union {
			struct hlist_head	i_dentry; //一个文件可能对应多个dentry, 这些dentry都要链接到这里
			struct rcu_head		i_rcu;
		};
		atomic64_t		i_version; //版本
		atomic64_t		i_sequence; /* see futex */
		atomic_t		i_count; //计数
		atomic_t		i_dio_count; //直接io进程计数
		atomic_t		i_writecount; //写进程计数
	#if defined(CONFIG_IMA) || defined(CONFIG_FILE_LOCKING)
		atomic_t		i_readcount; /* struct files open RO */
	#endif
		union {
			const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */ //操作file的函数集合
			void (*free_inode)(struct inode *);
		};
		struct file_lock_context	*i_flctx;
		struct address_space	i_data;
		struct list_head	i_devices;
		union {
			struct pipe_inode_info	*i_pipe;
			struct cdev		*i_cdev;
			char			*i_link;
			unsigned		i_dir_seq;
		};

		__u32			i_generation;

	#ifdef CONFIG_FSNOTIFY
		__u32			i_fsnotify_mask; /* all events this inode cares about */
		struct fsnotify_mark_connector __rcu	*i_fsnotify_marks;
	#endif

	#ifdef CONFIG_FS_ENCRYPTION
		struct fscrypt_info	*i_crypt_info;
	#endif

	#ifdef CONFIG_FS_VERITY
		struct fsverity_info	*i_verity_info;
	#endif

		void			*i_private; /* fs or device private pointer */ //私有数据指针
	} __randomize_layout;

	//https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/fs.h#L1826
	struct inode_operations {
		struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);  //该函数在特定目录中寻找索引节点，该索引节点要对应于dentry中给出的文件名
		const char * (*get_link) (struct dentry *, struct inode *, struct delayed_call *);
		int (*permission) (struct mnt_idmap *, struct inode *, int); //该函数用来检查给定的inode所代表的文件是否允许特定的访问模式，如果允许特定的访问模式，返回0，否则返回负值的错误码
		struct posix_acl * (*get_inode_acl)(struct inode *, int, bool);

		int (*readlink) (struct dentry *, char __user *,int); //被系统readlink()接口调用，拷贝数据到特定的缓冲buffer中。拷贝的数据来自dentry指定的符号链接

		int (*create) (struct mnt_idmap *, struct inode *,struct dentry *,
			       umode_t, bool); //VFS通过系统create()和open()接口来调用该函数，从而为dentry对象创建一个新的索引节点
		int (*link) (struct dentry *,struct inode *,struct dentry *); //被系统link()接口调用，用来创建硬连接。硬链接名称由dentry参数指定
		int (*unlink) (struct inode *,struct dentry *); //被系统unlink()接口调用，删除由目录项dentry链接的索引节点对象
		int (*symlink) (struct mnt_idmap *, struct inode *,struct dentry *,
				const char *); //被系统symlik()接口调用，创建符号连接，该符号连接名称由symname指定，连接对象是dir目录中的dentry目录项
		int (*mkdir) (struct mnt_idmap *, struct inode *,struct dentry *,
			      umode_t); //被mkdir()接口调用，创建一个新目录。
		int (*rmdir) (struct inode *,struct dentry *); //被rmdir()接口调用，删除dentry目录项代表的文件
		int (*mknod) (struct mnt_idmap *, struct inode *,struct dentry *,
			      umode_t,dev_t); //被mknod()接口调用，创建特殊文件(设备文件、命名管道或套接字)。
		int (*rename) (struct mnt_idmap *, struct inode *, struct dentry *,
				struct inode *, struct dentry *, unsigned int); //VFS调用该函数来移动文件。文件源路径在old_dir目录中，源文件由old_dentry目录项所指定，目标路径在new_dir目录中，目标文件由new_dentry指定
		int (*setattr) (struct mnt_idmap *, struct dentry *, struct iattr *); //被notify_change接口调用，在修改索引节点之后，通知发生了改变事件
		int (*getattr) (struct mnt_idmap *, const struct path *,
				struct kstat *, u32, unsigned int); //在通知索引节点需要从磁盘中更新时，VFS会调用该函数
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
1. 目录项对象(dentry)：代表一个目录项，描述了文件系统的层次结构.

	目录项用来记录文件的名字、 索引节点指针 以及与其他目录项的层级关联关系. 多个目录项关联起来，就会形成目录结构，但它与索引节点不同的是，**目录项是由内核维护的一个数据结构，不存放于磁盘，而是缓存在内存**

	一个文件可能不止一个dentry.
	一个文件路径的各组成部分都是一个目录项对象. 比如`/home/test/test.c`, kernel为home, test和test.c都创建了目录项对象.

	为了加快对dentry的查找, kernel使用了hash表来缓存dentry, 即dentry cache.

	```c
	//https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/dcache.h#L82
	struct dentry {
		/* RCU lookup touched fields */
		unsigned int d_flags;		/* protected by d_lock */ //目录标志
		seqcount_spinlock_t d_seq;	/* per dentry seqlock */
		struct hlist_bl_node d_hash;	/* lookup hash list */ //目录的hash链表, 链接到dentry cache的hash表. = v2.6.28的`struct hlist_node`
		struct dentry *d_parent;	/* parent directory */ //指向父目录
		struct qstr d_name; //目录名称. 打开一个文件时, 会根据这个名称来查找目标文件.
		struct inode *d_inode;		/* Where the name belongs to - NULL is
						 * negative */ //指向目录文件的inode, inode与dentry共同描述了一个普通文件或目录文件
		unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */ //短目录名

		/* Ref lookup also touches following */
		struct lockref d_lockref;	/* per-dentry lock and refcount */ //目录锁与计数
		const struct dentry_operations *d_op; //操作目录的函数集合
		struct super_block *d_sb;	/* The root of the dentry tree */ //指向超级块
		unsigned long d_time;		/* used by d_revalidate */ //时间
		void *d_fsdata;			/* fs-specific data */ //指向具体fs的数据

		union {
			struct list_head d_lru;		/* LRU list */
			wait_queue_head_t *d_wait;	/* in-lookup ones only */
		};
		struct list_head d_child;	/* child of parent list */ //挂入父目录的链表, dentry自身的链表头, 会链接到父dentry的d_subdirs. 但移动文件时需要将一个dentry从旧的父dentry链表中脱离, 再链接到新的父dentry的d_subdirs中
		struct list_head d_subdirs;	/* our children */ //挂载所有子目录的链表
		/*
		 * d_alias and d_rcu can share memory
		 */
		union {
			struct hlist_node d_alias;	/* inode alias list */
			struct hlist_bl_node d_in_lookup_hash;	/* only for in-lookup ones */
		 	struct rcu_head d_rcu;
		} d_u;
	} __randomize_layout;


	//https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/dcache.h#L128
	struct dentry_operations {
		int (*d_revalidate)(struct dentry *, unsigned int); //该函数判断目录对象是否有效
		int (*d_weak_revalidate)(struct dentry *, unsigned int);
		int (*d_hash)(const struct dentry *, struct qstr *); //该函数为目录项生成散列值，当目录项要加入散列表中时，VFS调用该函数
		int (*d_compare)(const struct dentry *,
				unsigned int, const char *, const struct qstr *); //VFS调用该函数来比较name1和name2两个文件名。多数文件系统使用VFS的默认操作，仅做字符串比较。对于有些文件系统，比如FAT，简单的字符串比较不能满足其需要，因为 FAT文件系统不区分大小写
		int (*d_delete)(const struct dentry *); //当目录项对象的计数值等于0时，VFS调用该函数
		int (*d_init)(struct dentry *);//当分配目录时调用
		void (*d_release)(struct dentry *); //当目录项对象要被释放时，VFS调用该函数，默认情况下，它什么也不做
		void (*d_prune)(struct dentry *);
		void (*d_iput)(struct dentry *, struct inode *); //当一个目录项对象丢失了相关索引节点时，VFS调用该函数。默认情况下VFS会调用iput()函数释放索引节点
		char *(*d_dname)(struct dentry *, char *, int); //当需要生成一个dentry的路径名时被调用
		struct vfsmount *(*d_automount)(struct path *); //当要遍历一个自动挂载时被调用（可选），这应该创建一个新的VFS挂载记录并将该记录返回给调用者
		int (*d_manage)(const struct path *, bool); //文件系统管理从dentry的过渡（可选）时，被调用
		struct dentry *(*d_real)(struct dentry *, const struct inode *); //叠加/联合类型的文件系统实现此方法
	} ____cacheline_aligned;
	```

	目录也是文件，需要用 inode 索引结构来管理目录文件数据.

	dentry_operations 结构中的函数，也需要具体文件系统实现，下层代码查找或者操作目录时 VFS 就会调用这些函数，让具体文件系统根据自己储存设备上的目录信息处理并设置 dentry 结构中的信息，这样文件系统中的目录就和 VFS 的目录对应了.
1. 文件对象(file)：代表进程已打开的文件. 用于建立进程与文件之间的对应关系.

	文件对象代表进程与具体文件交互的关系. kernel为每个打开的文件申请一个文件对象并返回该文件的fd. 每个进程有一个文件描述符表, 它用数组保存了进程打开的每个文件.

	当且仅当进程访问文件期间存在与内存中. 同一个文件可能对应多个文件对象, 但其对应的索引节点对象是唯一的.

	```c
	//https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/fs.h#L961
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
		spinlock_t		f_lock;
		fmode_t			f_mode; //文件权限
		atomic_long_t		f_count; //文件对象
		struct mutex		f_pos_lock; //文件读写位置锁
		loff_t			f_pos; //进程读写文件的当前位置. 比如对文件读取前10字节, f_pos就指向第11B
		unsigned int		f_flags;
		struct fown_struct	f_owner;
		const struct cred	*f_cred;
		struct file_ra_state	f_ra; // 用于文件预读的位置
		struct path		f_path; //文件路径
		struct inode		*f_inode;	/* cached value */ //对应的inode
		const struct file_operations	*f_op; //操作文件的函数集合

		u64			f_version;
	#ifdef CONFIG_SECURITY
		void			*f_security;
	#endif
		/* needed for tty driver, and maybe others */
		void			*private_data;

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
		loff_t (*llseek) (struct file *, loff_t, int); //调整读写偏移
		ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
		ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
		ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
		ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
		int (*iopoll)(struct kiocb *kiocb, struct io_comp_batch *,
				unsigned int flags);
		int (*iterate_shared) (struct file *, struct dir_context *);
		__poll_t (*poll) (struct file *, struct poll_table_struct *);
		long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
		long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
		int (*mmap) (struct file *, struct vm_area_struct *);
		unsigned long mmap_supported_flags;
		int (*open) (struct inode *, struct file *);
		int (*flush) (struct file *, fl_owner_t id);
		int (*release) (struct inode *, struct file *);
		int (*fsync) (struct file *, loff_t, loff_t, int datasync);
		int (*fasync) (int, struct file *, int);
		int (*lock) (struct file *, int, struct file_lock *);
		unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
		int (*check_flags)(int);
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

由于索引节点唯一标识一个文件，而目录项记录着文件的名，所以目录项和索引节点的关系是多对一，也就是说，一个文件可以有多个别字. 比如，硬链接的实现就是多个目录项中的索引节点指向同一个文件.

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

**文件系统首先要先挂载到某个目录才可以正常使用**，比如 Linux 系统在启动时，会把文件系统挂载到根目录.

## mount
参考:
- [EADME - 计算机专业性文章及回答总索引#新一代VFS mount系统调用](https://zhuanlan.zhihu.com/p/67686817)

当一个fs被挂载时, 它的[vfsmount](https://elixir.bootlin.com/linux/v5.12.9/source/include/linux/mount.h#L71)被链接到了kernel的一个全局链表[mount_hashtable](https://elixir.bootlin.com/linux/v5.12.9/source/fs/namespace.c#L70). mount_hashtable是一个数组, 它的每个成员都是一个hash链表.

当发现目录是一个挂载点时, 会从mount_hashtable中找到该fs的vfsmount, 然后挂载点目录的dentry会被替换为被挂载fs的root dentry.

### 新内核mount
参考:
- [深入理解 Linux 文件系統之文件系統掛載](https://webcache.googleusercontent.com/search?q=cache:EX2JdZE_xJgJ:https://www.readfog.com/a/1637370894679642112+&cd=3&hl=zh-CN&ct=clnk)

```
[mount](https://elixir.bootlin.com/linux/v5.12.9/source/fs/namespace.c#L3431)
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
-> vfs_get_tree(fc)  //fs/super.c 挂载重点, 调用fc->ops->get_tree(fc) 创建super_block实例
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

## ext4
参考:
- [*Ext4文件系统架构分析(一)](https://www.cnblogs.com/alantu2018/p/8461272.html)
- [*linux io过程自顶向下分析](https://my.oschina.net/fileoptions/blog/3058792/print)

> ext4 dax特性: nvdimm(非易失性双列直插式内存模块=dram+nand+超级电容), 再使用PageCache缓存数据变得累赘, 因此dax不使用缓存而是直接访问设备.

![](/misc/img/fs/f81bf3e5a6cd060c3225a8ae1803a138.jpeg)

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4.h#L752
/*
 * Structure of an inode on the disk
 */
struct ext4_inode {
	__le16	i_mode;		/* File mode */
	__le16	i_uid;		/* Low 16 bits of Owner Uid */
	__le32	i_size_lo;	/* Size in bytes */
	__le32	i_atime;	/* Access time */
	__le32	i_ctime;	/* Inode Change time */
	__le32	i_mtime;	/* Modification time */
	__le32	i_dtime;	/* Deletion Time */
	__le16	i_gid;		/* Low 16 bits of Group Id */
	__le16	i_links_count;	/* Links count */
	__le32	i_blocks_lo;	/* Blocks count */
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
	__le32	i_size_high;
	__le32	i_obso_faddr;	/* Obsoleted fragment address */
	union {
		struct {
			__le16	l_i_blocks_high; /* were l_i_reserved1 */
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
	__le32  i_crtime;       /* File Creation time */
	__le32  i_crtime_extra; /* extra FileCreationtime (nsec << 2 | epoch) */
	__le32  i_version_hi;	/* high 32 bits for 64-bit version */
	__le32	i_projid;	/* Project ID */
};
```

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

	set_nameidata(&nd, dfd, pathname); // 沿着要打开文件名的整个路径, 一层层解析路径, 最后得到文件的dentry和vfsmount, 再保存到nd中.
	filp = path_openat(&nd, op, flags | LOOKUP_RCU);
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
		while (!(error = link_path_walk(s, nd)) &&
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

对于块组，同样也需要一个数据结构来表示为 [ext4_group_desc](https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4.h#L324). 这里面对于一个块组里的 inode 位图 bg_inode_bitmap_lo、块位图 bg_block_bitmap_lo、inode 列表 bg_inode_table_lo，都有相应的成员变量.

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

```c
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/ext4/ext4.h#L1245
/*
 * Structure of the super block
 */
struct ext4_super_block {
/*00*/	__le32	s_inodes_count;		/* Inodes count */
	__le32	s_blocks_count_lo;	/* Blocks count */
	__le32	s_r_blocks_count_lo;	/* Reserved blocks count */
	__le32	s_free_blocks_count_lo;	/* Free blocks count */
/*10*/	__le32	s_free_inodes_count;	/* Free inodes count */
	__le32	s_first_data_block;	/* First Data Block */
	__le32	s_log_block_size;	/* Block size */
	__le32	s_log_cluster_size;	/* Allocation cluster size */
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
	__le32	s_creator_os;		/* OS */
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
	__le32	s_first_ino;		/* First non-reserved inode */
	__le16  s_inode_size;		/* size of inode structure */
	__le16	s_block_group_nr;	/* block group # of this superblock */
	__le32	s_feature_compat;	/* compatible feature set */
/*60*/	__le32	s_feature_incompat;	/* incompatible feature set */
	__le32	s_feature_ro_compat;	/* readonly-compatible feature set */
/*68*/	__u8	s_uuid[16];		/* 128-bit uuid for volume */
/*78*/	char	s_volume_name[16];	/* volume name */
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
	__le16  s_desc_size;		/* size of group descriptor */
/*100*/	__le32	s_default_mount_opts;
	__le32	s_first_meta_bg;	/* First metablock block group */
	__le32	s_mkfs_time;		/* When the filesystem was created */
	__le32	s_jnl_blocks[17];	/* Backup of the journal inode */
	/* 64bit support valid if EXT4_FEATURE_COMPAT_64BIT */
/*150*/	__le32	s_blocks_count_hi;	/* Blocks count */
	__le32	s_r_blocks_count_hi;	/* Reserved blocks count */
	__le32	s_free_blocks_count_hi;	/* Free blocks count */
	__le16	s_min_extra_isize;	/* All inodes have at least # bytes */
	__le16	s_want_extra_isize; 	/* New inodes should reserve # bytes */
	__le32	s_flags;		/* Miscellaneous flags */
	__le16  s_raid_stride;		/* RAID stride */
	__le16  s_mmp_update_interval;  /* # seconds to wait in MMP checking */
	__le64  s_mmp_block;            /* Block for multi-mount protection */
	__le32  s_raid_stripe_width;    /* blocks on all data disks (N*stride)*/
	__u8	s_log_groups_per_flex;  /* FLEX_BG group size */
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
	__le32	s_reserved[95];		/* Padding to the end of the block */
	__le32	s_checksum;		/* crc32c(superblock) */
};
```

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

## linux io过程自顶向下分析
![](/misc/img/fs/3c506edf93b15341da3db658e9970773.jpeg)

### 挂载文件系统
内核是不是支持某种类型的文件系统，需要先进行注册才能知道. 例如 ext4 文件系统，就需要通过 register_filesystem 注册到全局变量[file_systems](https://elixir.bootlin.com/linux/v5.12.9/source/fs/filesystems.c#L34)中，传入的参数是 ext4_fs_type，表示注册的是 ext4 类型的文件系统. 这里面最重要的一个成员变量就是 ext4_mount.

```c
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
// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/mount.h#L40
struct mount {
	struct hlist_node mnt_hash;
	struct mount *mnt_parent;
	struct dentry *mnt_mountpoint;
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
	struct list_head mnt_mounts;	/* list of children, anchored here */
	struct list_head mnt_child;	/* and going through their mnt_child */
	struct list_head mnt_instance;	/* mount instance on sb->s_mounts */
	const char *mnt_devname;	/* Name of device e.g. /dev/dsk/hda1 */
	struct list_head mnt_list;
	struct list_head mnt_expire;	/* link in fs-specific expiry list */
	struct list_head mnt_share;	/* circular list of shared mounts */
	struct list_head mnt_slave_list;/* list of slave mounts */
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
	int mnt_id;			/* mount identifier */
	int mnt_group_id;		/* peer group identifier */
	int mnt_expiry_mark;		/* true if marked for expiry */
	struct hlist_head mnt_pins;
	struct hlist_head mnt_stuck_children;
} __randomize_layout;
```

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

// https://elixir.bootlin.com/linux/v5.8-rc3/source/fs/namei.c#L502
struct nameidata {
	struct path	path;
	struct qstr	last;
	struct path	root;
	struct inode	*inode; /* path.dentry.d_inode */
	unsigned int	flags;
	unsigned	seq, m_seq, r_seq;
	int		last_type;
	unsigned	depth;
	int		total_link_count;
	struct saved {
		struct path link;
		struct delayed_call done;
		const char *name;
		unsigned seq;
	} *stack, internal[EMBEDDED_LEVELS];
	struct filename	*name;
	struct nameidata *saved;
	unsigned	root_seq;
	int		dfd;
	kuid_t		dir_uid;
	umode_t		dir_mode;
} __randomize_layout;

// https://elixir.bootlin.com/linux/v5.8-rc3/source/include/linux/path.h#L8
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry;
} __randomize_layout;
```

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

## 总结
![](/misc/img/fs/8070294bacd74e0ac5ccc5ac88be1bb9.png)

对于每一个进程，打开的文件都有一个文件描述符，在 files_struct 里面会有文件描述符数组. 每个一个文件描述符是这个数组的下标，里面的内容指向一个 file 结构，表示打开的文件。这个结构里面有这个文件对应的 inode，最重要的是这个文件对应的操作 file_operation. 如何操作这个文件，就看这个 file_operation 里面的定义了.

对于每一个打开的文件，都有一个 dentry 对应，虽然叫作 directory entry，但是不仅仅表示文件夹，也表示文件. 它最重要的作用就是指向这个文件对应的 inode.

如果说 file 结构是一个文件打开以后才创建的，dentry 是放在一个 dentry cache 里面的，文件关闭了，它依然存在，因而它可以更长期地维护内存中的文件的表示和硬盘上文件的表示之间的关系.

inode 结构就表示硬盘上的 inode，包括块设备号等. 几乎每一种结构都有自己对应的 operation 结构，里面都是一些方法，因而当后面遇到对于某种结构进行处理的时候，如果不容易找到相应的处理函数，就先找这个 operation 结构，就清楚了.

## 系统调用层和虚拟文件系统层
文件系统的读写，其实就是调用系统函数 read 和 write.

```c
ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos, *ppos = file_ppos(f.file);
		if (ppos) {
			pos = *ppos;
			ppos = &pos;
		}
		ret = vfs_read(f.file, buf, count, ppos);
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