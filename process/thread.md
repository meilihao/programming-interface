# 线程(thread)
一个进程内的所有线程共享同一地址空间,文件描述符,栈以及与进程相关的属性.因此各线程需要同步共享数据来避免不一致性.

与进程相同,线程也有`id`.线程id只在它所属的进程内起作用.

> 多线程程序可通过gcc的`-D_REENRANT`使得函数使用线程安全(可重入)版本, 选项`-pthread`=`-lpthread -D_REENTRANT`. 最典型的是在一个多线程程序里，默认情况下，只有一个errno变量供所有的线程共享, 此时就需要该选项.

线程销毁:
1. pthread_join : 阻塞直至线程销毁
1. pthreadA_detach : 不会阻塞

> pthread是api标准, NPTL是具体实现,已集成到glibc中, 其他两种实现LinuxThreads和NGPT已被放弃.

## 线程同步
1. mutex(互斥量)
1. semaphore(信号量)

   信号量的值不能小于0, 且信号量为0时, `sem_wait()`会阻塞.
