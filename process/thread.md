# 线程(thread)
一个进程内的所有线程共享同一地址空间和若干资源及属性(文件描述符,栈以及与进程相关的属性).因此各线程需要同步共享数据来避免不一致性.

与进程相同,线程也有`id`.线程id只在它所属的进程内起作用.

> 多线程程序可通过gcc的`-D_REENRANT`使得函数使用线程安全(可重入)版本, 选项`-pthread`=`-lpthread -D_REENTRANT`. 最典型的是在一个多线程程序里，默认情况下，只有一个errno变量供所有的线程共享, 此时就需要该选项.

pthread_create, 创建线程, 一共有四个参数:
1. 线程对象
1. 线程的属性
1. 线程运行函数
1. 线程运行函数的参数

pthread_exit, 退出线程, 可传入一个转换为 (void *) 类型的参数, 表示线程退出的返回值.

线程销毁:
1. pthread_join : 阻塞直至线程销毁
1. pthreadA_detach : 不会阻塞

> pthread是api标准, NPTL是具体实现,已集成到glibc中, 其他两种实现LinuxThreads和NGPT已被放弃.

> 默认情况下线程栈大小为 8192（8MB）, 其大小可以通过命令 ulimit -a 查看, 也可使用命令 ulimit -s 修改; 在程序中可通过`
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);`修改

![](/misc/img/process/e38c28b0972581d009ef16f1ebdee2bd.jpg)
![](/misc/img/process/02a774d7c0f83bb69fec4662622d6d58.png)

> 主线程在内存中有一个栈空间，其他线程栈也拥有独立的栈空间. 为了避免线程之间的栈空间踩踏，线程栈之间还会有小块区域，用来隔离保护各自的栈空间, 一旦另一个线程踏入到这个隔离区，就会引发段错误.

## 线程同步
1. mutex(互斥量)
1. semaphore(信号量)

   信号量的值不能小于0, 且信号量为0时, `sem_wait()`会阻塞.

## 线程数据
分3类:
1. 线程栈上的本地数据, 比如函数执行过程中的局部变量
1. 在整个进程里共享的全局数据, 比如全局变量，虽然在不同进程中是隔离的，但是在一个进程中是共享的, 因此需要锁.
1. 线程私有数据（Thread Specific Data）

	key 一旦被创建，所有线程都可以访问它，表面看起来是一个全局变量, 但各线程可根据自己的需要往 key 中填入不同的值，而**它的值在每一个线程中又是单独存储**的, 这就相当于提供了一个同名而不同值的全局变量.

	key相关函数:
	- 创建 : int pthread_key_create(pthread_key_t *key, void (*destructor)(void*))
	- 设置 : int pthread_setspecific(pthread_key_t key, const void *value)
	- 获取 : void *pthread_getspecific(pthread_key_t key)