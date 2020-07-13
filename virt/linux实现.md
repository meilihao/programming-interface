# virt
KVM 在内核里面需要有一个模块，来设置当前 CPU 是 Guest OS 在用，还是 Host OS 在用.

KVM 内核模块通过 /dev/kvm 暴露接口，用户态程序可以通过 ioctl 来访问这个接口.

![](/misc/img/virt/f5ee1a44d7c4890e411c2520507ddc62.png)

Qemu 将 KVM 整合进来，将有关 CPU 指令的部分交由内核模块来做，就是 qemu-kvm (qemu-system-XXX).

qemu 和 kvm 整合之后，CPU 的性能问题解决了. 另外 Qemu 还会模拟其他的硬件，如网络和硬盘. 同样，全虚拟化的方式也会影响这些设备的性能.

于是，qemu 采取半虚拟化的方式，让 Guest OS 加载特殊的驱动来做这件事情.

例如，网络需要加载 virtio_net，存储需要加载 virtio_blk，Guest 需要安装这些半虚拟化驱动，GuestOS 知道自己是虚拟机，所以数据会直接发送给半虚拟化设备，经过特殊处理（例如排队、缓存、批量处理等性能优化方式），最终发送给真正的硬件. 这在一定程度上提高了性能.

至此，整个关系如下图所示.
![](/misc/img/virt/f748fd6b6b84fa90a1044a92443c3522.png)

## 创建vm
1. 先要给虚拟机起一个名字，在 KVM 里面就是`-name ubuntutest`.
1. 设置一个内存大小，在 KVM 里面就是`-m 1024`.
1. 创建虚拟磁盘

	虚拟硬盘有两种格式，一个是动态分配，也即开始创建的时候，看起来很大，其实占用的空间很少，真实有多少数据，才真的占用多少空间. 一个是固定大小，一开始就占用指定的大小.

	在 KVM 中，创建一个虚拟机镜像，大小为 8G，其中 qcow2 格式为动态分配，raw 格式为固定大小, 命令是`qemu-img create -f qcow2 ubuntutest.img 8G`.

1. 挂载ubuntu iso, 在 KVM 里面是`-cdrom [ubuntu-xxx-server-amd64.iso](https://ubuntu.com/download/server)`
1. 配置网络

	1. 桥接网络

		虚拟机, host, 及其他机器在同一个网段, 且虚拟机和host连在同一个虚拟交换机上.

1. 显示在 KVM 里面，用 VNC 来做, 参数为` -vnc :19`

组合起来就是`qemu-system-x86_64 -enable-kvm -name ubuntutest  -m 2048 -hda ubuntutest.img -cdrom ubuntu-20.04-server-amd64.iso -boot d -vnc :19`

启动了虚拟机后，界面会显示在qemu console中, 但连接 VNC(127.0.0.1:19)，也能看到安装的过程.