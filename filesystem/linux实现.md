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

![](/misc/img/filesystem/73349c0fab1a92d4e1ae0c684cfe06e2.jpeg)

在 ext2 和 ext3 中，其中前 12 项直接保存了块的位置，也就是说，可以直接通过 i_block[0-11]，直接得到保存文件内容的块. 对于大文件,可以让 i_block[12]指向一个块，这个块里面不放数据块，而是放数据块的位置，这个块被称为间接块. 也就是说，在 i_block[12]里面放间接块的位置，通过 i_block[12]找到间接块后，间接块里面放数据块的位置，通过间接块可以找到数据块. 如果文件再大一些，i_block[13]会指向一个块，我们可以用二次间接块. 二次间接块里面存放了间接块的位置，间接块里面存放了数据块的位置，数据块里面存放的是真正的数据. 如果文件再大一些，i_block[14]会指向三次间接块. 原理和之前都是一样的，需要一层一层展开才能拿到数据块. 对于大文件来讲，就要多次读取硬盘才能找到相应的块，这样访问速度就会比较慢. 为了解决这个问题，ext4 做了一定的改变. 它引入了一个新的概念，叫做 Extents.

Exents 其实会保存成一棵树:
![](/misc/img/filesystem/b8f184696be8d37ad6f2e2a4f12d002a.jpeg)

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