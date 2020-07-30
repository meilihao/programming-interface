# vhost
参考:
- [VIRTIO & VHOST](https://zhuanlan.zhihu.com/p/38370357)
- [virtIO之VHOST工作原理简析](https://www.cnblogs.com/ck1020/p/7204769.html)
- [*Virtio和Vhost介绍*](https://forum.huawei.com/enterprise/zh/thread-465473.html)
- [Virtio网络的演化之路](https://cloud.tencent.com/developer/article/1540284)
- [详解vhost-user协议及其在OVS DPDK、QEMU和virtio-net驱动中的实现](http://www.jeepxie.net/article/378987.html)
- [Vhost-user详解](https://www.jianshu.com/p/ae54cb57e608)

vhost是一种 virtio 高性能的后端驱动实现. 原有virtio后端驱动的i/o要进过vmm(qemu)和host kernel, 但它进一步缩短了i/o路径, 不再经过vmm.

它是位于host kernel的一个模块，用于和guest直接通信，所以数据交换就在guest和host kernel间进行，减少了上下文的切换. vhost相对与virto之前架构，把virtio驱动后端驱动从用户态放到了内核态中（vhost的内核模块充当virtiO后端驱动).

![virtio和vhost（vhost-net时vhost架构中的网卡实现）架构下内核的不同工作流程](/misc/img/virt/4nry30ay3h.jpeg)

![vhost工作原理](/misc/img/virt/905w5l04dt.jpeg)

## vhost-user
在 vhost 的方案中，由于 vhost 实现在内核中，guest 与 vhost 的通信，相较于原生的 virtio 方式性能上有了一定程度的提升，从 guest 到 kvm.ko 的交互只有一次用户态的切换以及数据拷贝. 这个方案对于不同 host 之间的通信，或者 guest 到 host nic 之间的通信是比较好的，但是**对于某些用户态进程间的通信(因此vhost-user不适合所有情况)**，比如数据面的通信方案，openvswitch 和与之类似的 SDN 的解决方案，guest 需要和 host 用户态的 vswitch 进行数据交换，如果采用 vhost 的方案，guest 和 host 之间又存在多次的上下文切换和数据拷贝，为了避免这种情况，业界就想出将 vhost 从内核态移到用户态. 这就是 vhost-user 的实现.

vhost-user和vhost类似，只是使用一个用户态进程vhost-user代替了内核中的vhost模块. vhost-user进程和Guset之间时通过共享内存的方式进行数据操作. 

vhost-user 和 vhost 的实现原理是一样，都是采用 vring 完成共享内存，eventfd 机制完成事件通知. 不同在于 vhost 实现在内核中，而 vhost-user 实现在用户空间中，用于用户空间中两个进程之间的通信，其采用共享内存的通信方式.

vhost-user 基于 C/S 的模式，采用 UNIX 域套接字（UNIX domain socket）来完成进程间的事件通知和数据交互，相比 vhost 中采用 ioctl 的方式，vhost-user 采用 socket 的方式大大简化了操作.

vhost-user 基于 vring 这套通用的共享内存通信方案，只要 client 和 server 按照 vring 提供的接口实现所需功能即可，常见的实现方案是 client 实现在 guest OS 中，一般是集成在 virtio 驱动上，server 端实现在 qemu 中，也可以实现在各种数据面中，如 OVS，Snabbswitch 等虚拟交换机.

在软件实现的网络I/O半虚拟化中，vhost-user在性能、灵活性和兼容性等方面达到了近乎完美的权衡.

![vhost-user的工作原理](/misc/img/virt/s5hebv5w16.jpeg)