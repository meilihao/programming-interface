# fs
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

	set_nameidata(&nd, dfd, pathname);
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
内核是不是支持某种类型的文件系统，需要先进行注册才能知道. 例如 ext4 文件系统，就需要通过 register_filesystem 进行注册，传入的参数是 ext4_fs_type，表示注册的是 ext4 类型的文件系统. 这里面最重要的一个成员变量就是 ext4_mount.

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
generic_perform_write函数里，是一个 while 循环. 需要找出这次写入影响的所有的页，然后依次写入. 对于每一个循环，主要做四件事情：
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

第三步，调用 ext4_write_end 完成写入. 这里面会调用 ext4_journal_stop 完成日志的写入，会调用 block_write_end->__block_commit_write->mark_buffer_dirty，将修改过的缓存标记为脏页. 可以看出，其实所谓的完成写入，并没有真正写入硬盘，仅仅是写入缓存后，标记为脏页.

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

这里的 bdi 的意思是 backing device info，用于描述后端存储相关的信息, 每个块设备都会有这样一个结构，并且在初始化块设备的时候，调用 bdi_init 初始化这个结构，在初始化 bdi 的时候，也会调用 [wb_init](https://elixir.bootlin.com/linux/v5.8-rc3/source/mm/backing-dev.c#L283) 初始化 bdi_writeback.

wb_init里面最重要的是 INIT_DELAYED_WORK. 其实就是初始化一个 timer，也即定时器，到时候就执行 wb_workfn 这个函数.

接下来的调用链为：wb_workfn->wb_do_writeback->wb_writeback->writeback_sb_inodes->__writeback_single_inode->do_writepages，写入页面到硬盘.

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