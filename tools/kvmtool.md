# kvmtool
ref:
- [kvmtool正式运行kvm](https://jerling.github.io/post/kvmtool%E8%BF%90%E8%A1%8Ckvm/)

kvmtool代码量只有5KLOC，是一个干净的、从头开始写的、轻量级虚拟化工具.

类似于qemu的作用，kvmtool是一个支持运行KVM Guest OS的 host os端用户态虚拟机工具，它是一个纯虚拟化工具，guest os不需要修改即可运行其上, 不过，由于KVM是基于CPU的硬件虚拟化支持的，所以类似于qemu-kvm，它只支持基于相同架构的Guest OS.

kvmtool启动时会将cs, ds等所有的段寄存器设为0x1000, ip为0, 因此它会将可执行程序加载到0x10000(1M)并从该地址开始执行.