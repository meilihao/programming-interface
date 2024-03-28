# concurrent
## 编译乱序/执行乱序
见[compile](compile/compile.md)的`优化`的`barrier`.

内存屏蔽指令: 解决多核间一个核的内存行为对另一个核可见的问题.

ARM的内存屏蔽指令包括:
- DMB(数据内存屏障): 在DMB之后的显示内存访问执行前, 保证所有在DMB指令之前的内存访问完成
- DSB(数据同步屏障): 等待所有在DSB指令之前的指令完成(位于此指令前的所有显示内存访问均完成, 位于此指令前的所有缓存, 跳转预测和TLB维护操作全备完成)
- ISB(指令同步屏障): flush流水线, 使得所有ISB之后执行的指令都是从缓存或内存中获得的

## 中断屏蔽
中断屏蔽会影响进程调度(包括抢占).

底层原理是让cpu不响应中断. 比如ARM是屏蔽CPSR的i位.

local_irq_disable/local_irq_enable仅操作本cpu内的中断, 并不能解决SMP引发的竞态.

local_irq_save(flags)与local_irq_restore(flags), 比上述的local_irq_xxx多了操作cpu的中断位, 对于ARM就是保存和恢复CPSR.

操作中断的底半部可使用local_bh_disable/local_bh_enable.

## 原子操作
分对位和整型变量的原子操作, 均是基于CPU的硬件.

ARM使用LDREX和SETEX指令, 同时适用于核内/多核的并发.

## 自旋锁
自旋锁主要针对SMP或单cpu但内核可抢占的情况, 对于单cpu和内核不支持抢占的系统, 自旋锁退化为空操作. 在单cpu和内核可抢占的系统中, 自旋锁持有期间中内核的抢占被禁止. 在多核SMP的情况下, 任何一个核拿到自旋锁, 那么该核上的抢占被暂时禁止, 但不影响其他核的抢占.

> 内核可抢占的单cpu的行为类似SMP, 因此自旋锁是有必要的.

自旋锁保证临界区不受别的cpu和本cpu内的抢占进程打扰, 但在临界区内执行时, 还可能受到中断和底半部(BH)的影响.

```
spin_lock_irq() = spin_lock() + local_irq_disable()
spin_unlock_irq() = spin_unlock() + local_irq_enable()
spin_lock_irqsave() = spin_lock() + local_irq_save()
spin_unlock_restore() = spin_unlock() + local_irq_restore()
spin_lock_bh() = spin_lock() + local_bh_disable()
spin_unlock_bh() = spin_unlock() + local_bh_enable()
```

注意:
1. 自旋锁是忙等锁. 但临界区很大, 或者共享设备的时候, 需要较长时间占用锁, 此时会降低系统性能
1. 递归使用自旋锁会导致系统死锁
1. 在自旋锁锁定期间不能调用可能引起进程调度的函数. 如果进程获取自旋锁后再阻塞, 比如调用copy_from_user(), copy_to_user(), kmalloc()和msleep()等函数, 则可能导致内核崩溃

ARM的自旋锁实现基于ldrex, strex, 内存屏蔽指令dmb和dsb, wfe和sev.

### 读写自旋锁(rwlock)
读写自旋锁比自旋锁颗粒度更小, 允许多读至多1写, 类似读写锁.

### 顺序锁(seqlock)
顺序锁是读写自旋锁的一种优化. 写之间互斥, 但读写不互斥.

```
write_seqlock_irq() = write_seqlock() + local_irq_disable()
write_sequnlock_irq() = write_sequnlock() + local_irq_enable()
write_seqlock_irqsave() = write_seqlock() + local_irq_save()
write_sequnlock_restore() = write_sequnlock() + local_irq_restore()
write_seqlock_bh() = write_seqlock() + local_bh_disable()
write_sequnlock_bh() = write_sequnlock() + local_bh_enable()
```

### 读-复制-更新(RCU, Read-Copy-Update)
不同于自旋锁, RCU的读端没有锁, 内存屏障, 原子指令类的开销, 几乎可以认为是直接读; RCU的写在访问它的共享资源前先复制一个副本, 然后对副本进行修改, 最后使用一个回调机制在适当时间(所有引用该数据的cpu都退出对共享数据读操作的时候, 等待适当时机的时间被称为宽限期即Grace Period)把指向原数据的指针指向副本.

RCU可看作rwlock的高性能版本, 但不能代替rwlock, 因为写比较多时, 对读执行的性能提高不能弥补写执行单元同步导致的损失, 因为写执行间的同步开销比较大.

## 信号量(semaphore)
信号量与os概念中的PV操作对应, 其值是0, 1或n.

与自旋锁相同的是, 只有得到信号量的进程才能执行临界区代码; 不同的是, 获取不到信号量时, 进程不会在原地打转而是进入休眠状态.

## 互斥锁
mutex_lock和mutex_lock_interruptible()的区别与down()和down_trylock()的区别完全一致. 前者引起的睡眠不能被信号打断, 而后者可以. mutex_trylock()用于尝试获取mutex, 获取不到时不会引起进程睡眠.

在严格意义上, 互斥锁和自旋锁是不同层次的互斥手段. 前者的实现依赖后者, 即自旋锁是更底层的手段.

互斥锁是进程级别的, 用于多个进程间对资源的互斥. 由于进程上下文切换的开销很大, 因此, 只有当进程占用资源时间较上时, 用互斥锁才是较好的选择. 当要保护的临界区访问时间较短时, 用自旋锁, 可以节省上下文切换的时间. 由此, 它俩的选择原则是:
1. 当锁不能被获取到时, 使用互斥锁的开销是进程上下文切换的时间, 使用自旋锁的开销是等待获取自旋锁. 若临界区较小, 使用自旋锁, 否则使用互斥锁.
1. 互斥锁所保护的临界区可能引起阻塞的代码, 而自旋锁则绝对要避免用来保护包含这样代码的临界区. 因为阻塞就意味着进程切换, 如果进程被切换出去后, 其他进程来获取本自旋锁会导致死锁.
1. 互斥锁存在于进程上下文. 因此, 如果被保护的共享资源需要在中断或者软中断情况下使用, 则只能选择自旋锁. 如果一定要使用互斥锁, 则只能通过 mutex_trylock() 进行, 不能获取就立即返回以避免阻塞.

## 完成量(completion)
用于一个执行单元等待另一个执行单元执行完某事.
