# 多路复用
多路复用,是有效地利用通信线路,使得一个信道可以传输多路信号的技术. 相应的,I/O多路复用的意图就是在一个进程中,同时控制多个I/O操作,也被称作事件驱动型I/O.

严格意义上讲,I/O多路复用并不是I/O操作,它们只是与读写等操作捆绑,将目标文件可读、可写等事件告知应用,最终的读写依然由常规的读写操作独立完成.

并发的I/O操作有两种实现方案,第一种是创建多个进程,各进程独立负责各自的I/O,第二种就是I/O多路复用。从操作系统的角度,显然应该优选选择后者,不需要承担创建进程,维护进程的任务.

I/O多路复用有三种实现机制,分别为select、poll和epoll.

## select
select由select系统调用实现,最终调用do_select函数.

select的缺点主要有以下三方面:
1. 每次调用select,fd_set表示的位图都要完成由应用空间到内核的拷贝,拷贝的大小与最大的目标fd值有关,增加了内存管理的工作。同时也丧失了灵活性,不能单独添加或删除对某个文件的监
控,需求变更也必须传递整个位图.
1. select的位图参数既是输入也是输出,返回前会覆盖掉应用传递的值,所以应用每次调用select之前都需要重新设置参数.
1. 也是最为重要的是,select每次循环都要访问每一个相关的文件,执行它们的poll操作,当文件数目较多时,这无疑是在很大程度上降低了效率.

因为select的使用受限制、效率不够高,所以一直以来饱受争议, 这里不展开.

## poll
poll与select本质上是一样的,主要区别在于传递的参数不同. 与select传递位图让内核去检测包含哪些fd不同,poll只传递目标fd.

在内核的实现上,poll与select没有本质区别,它们每次循环都会回调每一个目标文件的poll操作,所以它们最主要的劣势是一样的,效率较低.

虽然poll与select拥有同样的硬伤,但从某些方面来讲,poll比select效率要高。首先是参数传递,select传递位图至内核,poll只传递目标文件的pollfd结构体,所以在灵活性方面poll虽然仍不能解决本质问题,但有一定的进步。其次,每次调用select之前都需要重新设置位图参数,而poll的输入输出参数是分开的,不存在该问题.

同样, 不展开poll.

# epoll
epoll,全称为eventpoll,它解决了select和poll的硬伤,是它们的升级版,相对它们有多方面的优势.

epoll涉及两个关键数据结构,eventpoll(以下简称ep) 和epitem(以下简称epi),二者是一对多的关系.

```c
// https://elixir.bootlin.com/linux/v6.6.30/source/fs/eventpoll.c#L130
/*
 * Each file descriptor added to the eventpoll interface will
 * have an entry of this type linked to the "rbr" RB tree.
 * Avoid increasing the size of this struct, there can be many thousands
 * of these on a server and we do not want this to take another cache line.
 */
struct epitem {
	union {
		/* RB tree node links this structure to the eventpoll RB tree */
		struct rb_node rbn; // 将epi插入红黑树
		/* Used to free the struct epitem */
		struct rcu_head rcu;
	};

	/* List header used to link this structure to the eventpoll ready list */
	struct list_head rdllink; // 已经就绪的文件的列表的节点

	/*
	 * Works together "struct eventpoll"->ovflist in keeping the
	 * single linked chain of items.
	 */
	struct epitem *next;

	/* The file descriptor information this item refers to */
	struct epoll_filefd ffd; // 对应的文件的信息

	/*
	 * Protected by file->f_lock, true for to-be-released epitem already
	 * removed from the "struct file" items list; together with
	 * eventpoll->refcount orchestrates "struct eventpoll" disposal
	 */
	bool dying;

	/* List containing poll wait queues */
	struct eppoll_entry *pwqlist;

	/* The "container" of this item */
	struct eventpoll *ep; // 所属的ep

	/* List header used to link this item to the "struct file" items list */
	struct hlist_node fllink;

	/* wakeup_source used when EPOLLWAKEUP is set */
	struct wakeup_source __rcu *ws;

	/* The structure that describe the interested events and the source fd */
	struct epoll_event event; // 感兴趣的事件的描述
};

/*
 * This structure is stored inside the "private_data" member of the file
 * structure and represents the main data structure for the eventpoll
 * interface.
 */
struct eventpoll {
	/*
	 * This mutex is used to ensure that files are not removed
	 * while epoll is using them. This is held during the event
	 * collection loop, the file cleanup path, the epoll file exit
	 * code and the ctl operations.
	 */
	struct mutex mtx;

	/* Wait queue used by sys_epoll_wait() */
	wait_queue_head_t wq; // 等待队列头, 用来唤醒epoll继续执行

	/* Wait queue used by file->poll() */
	wait_queue_head_t poll_wait; // 等待队列头, 当ep对应的文件作为普通文件被监控时使用

	/* List of ready file descriptors */
	struct list_head rdllist; // 已经就绪的文件的列表的头

	/* Lock which protects rdllist and ovflist */
	rwlock_t lock;

	/* RB tree root used to store monitored fd structs */
	struct rb_root_cached rbr; // rbr.rb_root epi组成的红黑树的根

	/*
	 * This is a single linked list that chains all the "struct epitem" that
	 * happened while transferring ready events to userspace w/out
	 * holding ->lock.
	 */
	struct epitem *ovflist;

	/* wakeup_source used when ep_scan_ready_list is running */
	struct wakeup_source *ws;

	/* The user that created the eventpoll descriptor */
	struct user_struct *user;

	struct file *file; // ep对应的文件

	/* used to optimize loop detection check */
	u64 gen;
	struct hlist_head refs;

	/*
	 * usage count, used together with epitem->dying to
	 * orchestrate the disposal of this struct
	 */
	refcount_t refcount;

#ifdef CONFIG_NET_RX_BUSY_POLL
	/* used to track busy poll napi_id */
	unsigned int napi_id;
#endif

#ifdef CONFIG_DEBUG_LOCK_ALLOC
	/* tracks wakeup nests for lockdep validation */
	u8 nests;
#endif
};
```

epoll 可 能 会 涉 及 多 个 等 待 队 列 , 其 中 两 个 等 待 队 列 由 wq 和poll_wait字段表示,其他等待队列由创建被监听文件的模块提供。wq表示的等待队列供ep自身使用,它负责在文件报告事件时,唤醒ep处理事件. poll_wait字段表示的等待队列是该ep对应的文件作为普通文件被监控时使用的,这时候它的角色就变成了创建被监听文件的模块.

epoll对被监听文件的poll操作的要求与poll和select是相同的, rdllist字段是它与后二者最大的不同,也是点睛之笔. 就绪的文件加入rdllist的链表,epoll不需要每次遍历每一个文件的poll操作,只需要访问该链表上的文件即可. 这是epoll相对于poll和select最大的优势, 执行效率大大提高. 涉及的文件数目越大,后二者的效率越低,而epoll受到的影响较小.

epoll有多个功能不同的函数:
```c
#include <sys/epoll.h>

int epoll_create(int size);
int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd,
                struct epoll_event *_Nullable event);
int epoll_wait(int epfd, struct epoll_event *events,
                int maxevents, int timeout);
int epoll_pwait(int epfd, struct epoll_event *events,
                int maxevents, int timeout,
                const sigset_t *_Nullable sigmask);
int epoll_pwait2(int epfd, struct epoll_event *events,
                int maxevents, const struct timespec *_Nullable timeout,
                const sigset_t *_Nullable sigmask);
```

epoll_create和epoll_create1就是用户用来创建一个epoll的函数,成功时返回该epoll的id,失败时返回负值,epoll_create的参数size必须大于0,其值并没有产生实际影响. epoll_create1的参数flags可以等于0或者EPOLL_CLOEXEC,EPOLL_CLOEXEC与O_CLOEXEC的值相等,含义也相同.

epoll的id实际上是内核为它创建的文件的描述符fd,内核初始化一个ep对象与epoll对应,调用anon_inode_getfile创建一个匿名文件,该文件的file对象的private_data字段指向ep,这样就可以通过fd来定位ep了.

内核同时为文件提供了文件操作eventpoll_fops,其中实现了poll操作ep_eventpoll_poll,该举动保证了epoll是可嵌套使用的。你没有看错,epoll_create返回的fd,同样可以作为普通文件描述符,被其他epoll监听(严格地讲,也可以被poll与select监听).

eventpoll_fops被当作epoll文件的标志,is_file_epoll函数根据它来判断一个文件是否为epoll文件.

epoll_ctl第一个参数就是epoll_create成功返回的id,用它来定位ep对象。第二个参数表示将要进行的操作,共有EPOLL_CTL_ADD、EPOLL_CTL_DEL和EPOLL_CTL_MOD三个可选值,fd为目标文件的
描述符,event是对感兴趣的事件的描述.

以EPOLL_CTL_ADD为例,内核首先做参数检查,epoll要求文件必须实现file->f_op->poll操作,否则返回错误,这点与select和poll不同。然后查看fd和它的file对象是否已经被ep监听,如果是则返回错误,否则调用ep_insert监听文件.

ep_insert使用ep、file和fd等信息初始化一个epi,调用文件的poll操作,这里是第一次调用,传递至poll操作的第二个参数(类型为poll_table指针)的_qproc字段是ep_ptable_queue_proc函数,poll操作调用 了 poll_wait ( 见 select 的 poll 操 作 的 示 例 代 码 ) , poll_wait 调 用ep_ptable_queue_proc,后者初始化一个wait_queue_entry_t对象挂载到
创建文件的模块提供的等待队列头上.

对一个epi而言,在ep_wait操作中可能也会调用文件的poll操作, 此时传递的第二个参数的_qproc字段被赋值为NULL,也就是说ep_ptable_queue_proc函数只会被调用一次。

完成poll操作后,epi会被插入到ep的rbr字段表示的红黑树中。

epoll_wait的第二个参数是输出参数,用来返回事件的结果, timeout的单位为毫秒。epoll_wait在参数检查后调用ep_poll,后者在没
有事件发生或者没有超时的情况下,会等待在ep的wq表示的等待队列上.

被监听的文件有事件发生后,唤醒自身模块的等待队列,该等待队 列 的 结 点 调 用 默 认 的 回 调 函 数 ep_poll_callback ( 由ep_ptable_queue_proc 指 定 ) , 它 会 将 对 应 的 epi 插 入 到 ep 的 rdllist 链表,唤醒ep的wq等待队列,这样ep_poll得以继续执行。ep_poll遍历rdllist链表上文件的poll操作来确定是否有感兴趣的事件发生,如果有
则将其返回至用户空间(ep_send_events函数).

另外,如果当前ep被监听,它所监听的文件有事件发生时,它也会唤醒监听它的ep,也就是等在ep->poll_wait上的其他ep.

对一个文件的监听可以选择边缘触发(Edge Triggered)、水平触发(Level Triggered)和一次触发(One Shot)三种模式,默认为水平触发。调用epoll_ctl的时候,将epoll_event类型的参数的events字段的EPOLLET或EPOLLONESHOT置位即表示边缘触发或一次触发.

如果选择水平触发,成功执行完毕文件的poll操作返回事件后,会将epi重新插入到ep的rdllist链表,这样下次调用epoll_wait的时候,该文件的poll操作至少会被执行一次。如果选择边缘触发,不会进行
重新插入epi的操作。一次触发,顾名思义就是文件产生过一次用户感兴趣的事件后,对文件不再感兴趣.

用户选择不同的触发方式要求不同的处理方法,一般而言,水平触发的处理方法相对简单。epoll_wait返回后,用户读数据之后,如果一次没有全部读出,再次调用epoll_wait,对水平触发方式而言,因为
epi依然在rdllist上,ep会执行文件的poll操作进而返回可读。但对边缘触发而言,除非新的事件到来导致epi重新插入到rdllist上,否则ep不会去执行文件的poll操作查询状态。所以选择了边缘触发,用户就需要负责每次都要把所有可读的数据读取完毕,而水平触发并没有这个要求.