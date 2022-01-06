# virt
参考:
 -[QEMU开源实战(四)](https://juejin.im/post/6844904113197547533)

KVM 在内核里面需要有一个模块，来设置当前 CPU 是 Guest OS 在用，还是 Host OS 在用.

KVM 内核模块通过 /dev/kvm 暴露接口，用户态程序可以通过 ioctl 来访问这个接口.

![](/misc/img/virt/f5ee1a44d7c4890e411c2520507ddc62.png)

Qemu 将 KVM 整合进来，将有关 CPU 指令的部分交由内核模块来做，就是 qemu-kvm (qemu-system-XXX).

qemu 和 kvm 整合之后，CPU 的性能问题解决了. 另外 Qemu 还会模拟其他的硬件，如网络和硬盘. 同样，全虚拟化的方式也会影响这些设备的性能.

于是，qemu 采取半虚拟化的方式，让 Guest OS 加载特殊的驱动来做这件事情.

例如，网络需要加载 virtio_net，存储需要加载 virtio_blk，Guest 需要安装这些半虚拟化驱动，GuestOS 知道自己是虚拟机，所以数据会直接发送给半虚拟化设备，经过特殊处理（例如排队、缓存、批量处理等性能优化方式），最终发送给真正的硬件. 这在一定程度上提高了性能.

> 完全虚拟化是很慢的，而通过内核的 KVM 技术和 EPT 技术，加速虚拟机对于物理 CPU 和内存的使用，即称为硬件辅助虚拟化, 因此它需要配备特殊的硬件才能工作

> 网络和存储主要使用特殊的半虚拟化驱动加速，需要加载特殊的驱动程序

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

		虚拟机, host, 及其他机器在同一个网段, 效果类似于虚拟机, host, 其他机器连在同一个虚拟交换机上.

		创建步骤:
		```bash
		# brctl addbr br0 # 在host上创建bridge br0
		# ip link set br0 up # 启动br0
		# tunctl -b # 创建tap device
		# ip link set tap0 up # 启动tap0
		# brctl addif br0 tap0 # 将tap0 加入到br0上
		# qemu-system-x86_64 -enable-kvm -name ubuntutest -m 2048 -hda ubuntutest.qcow2 -vnc :19 -net nic,model=virtio -nettap,ifname=tap0,script=no,downscript=no # 启动vm, vm添加tap0, tap0连接br0
		# ifconfig br0 192.168.57.1/24 # 虚拟机启动后，网卡没有配置，所以无法连接外网，给 br0 设置一个 ip
		# VNC 连上虚拟机，给网卡设置地址，重启虚拟机，可 ping 通 br0
		# 要想访问外网，在 Host 上设置 NAT，并且 enable ip forwarding，可以 ping 通外网网关
		# sysctl -p
		net.ipv4.ip_forward = 1
		sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
		# 如果 DNS 没配错，vm可以进行了 apt-get update
		```

1. 显示在 KVM 里面，用 VNC 来做, 参数为` -vnc :19`

组合起来就是`qemu-system-x86_64 -enable-kvm -name ubuntutest  -m 2048 -hda ubuntutest.img -cdrom ubuntu-20.04-server-amd64.iso -boot d -vnc :19`

启动了虚拟机后，界面会显示在qemu console中, 但连接 VNC(127.0.0.1:19)，也能看到安装的过程.

## qemu(v5.0.0)
参考:
- [虚拟机创建流程-qemu篇（上）](https://sq.163yun.com/blog/article/175668619278782464)
- [1. QEMU的核心初始化流程](https://hhb584520.github.io/kvm_blog/2017/05/16/create-guest.html)

qemu入口在`softmmu/main.c`, 其初始化工作在`softmmu/vl.c`的`qemu_init()`中完成.

![](/misc/img/virt/078dc698ef1b3df93ee9569e55ea2f30.png)
![MachineClass](/misc/img/virt/078dc698ef1b3df93ee9569e55ea2f30.png)

qemu每个模块都会有一个定义 TypeInfo，会通过 type_init 变为全局的 TypeImpl. TypeInfo 以及生成的 TypeImpl 有以下成员：
- name 表示当前类型的名称
- parent 表示父类的名称
- class_init 用于将 TypeImpl 初始化为 MachineClass
- instance_init 用于将 MachineClass 初始化为 MachineState

所以，以后遇到任何一个类型的时候，将父类和子类之间的关系，以及对应的初始化函数都要看好，这样就一目了然了.

### 1. 初始化所有的 Module
`qemu_init()`初始化所有的 Module时, 会调用函数`module_call_init(MODULE_INIT_QOM)`

qemu 为了模拟各种各样的设备，也需要管理各种各样的模块，这些模块也需要符合一定的格式. 定义一个 qemu 模块会调用 type_init. 例如，kvm 的模块要在 accel/kvm/kvm-all.c 文件里面实现. 在这个文件里面，有一行下面的代码：
```c
// https://elixir.bootlin.com/qemu/latest/source/accel/kvm/kvm-all.c#L3102
type_init(kvm_type_init);

// https://elixir.bootlin.com/qemu/latest/source/include/qemu/module.h#L28
#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_dso_module_init(function, type);                               \
}
#else
/* This should not be used directly.  Use block_init etc. instead.  */
#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}
#endif

// https://elixir.bootlin.com/qemu/latest/source/include/qemu/module.h#L56
#define type_init(function) module_init(function, MODULE_INIT_QOM)

// https://elixir.bootlin.com/qemu/latest/source/include/qemu/module.h#L68
void register_module_init(void (*fn)(void), module_init_type type);
void register_dso_module_init(void (*fn)(void), module_init_type type);

// https://elixir.bootlin.com/qemu/latest/source/util/module.c#L66
void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    ModuleTypeList *l;

    e = g_malloc0(sizeof(*e));
    e->init = fn;
    e->type = type;

    l = find_type(type);

    QTAILQ_INSERT_TAIL(l, e, node);
}
```

从代码里面的定义可以看出来，type_init 后面的参数是一个函数，调用 type_init 就相当于调用 module_init，在这里函数就是 kvm_type_init，类型就是 MODULE_INIT_QOM. 是不是感觉和驱动有点儿像？

module_init 最终要调用 register_module_init. 属于 MODULE_INIT_QOM 这种类型的，有一个 Module 列表 ModuleTypeList，列表里面是一项一项的 ModuleEntry. KVM 就是其中一项，并且会初始化每一项的 init 函数为参数表示的函数 fn，也即 KVM 这个 module 的 init 函数就是 kvm_type_init.

当然，MODULE_INIT_QOM 这种类型会有很多很多的 module，从后面的代码可以看到，所有调用 type_init 的地方都注册了一个 MODULE_INIT_QOM 类型的 Module. 了解了 Module 的注册机制， 继续回到 qemu_init 函数中 module_call_init 的调用.

```c
// https://elixir.bootlin.com/qemu/latest/source/util/module.c#L93
void module_call_init(module_init_type type)
{
    ModuleTypeList *l;
    ModuleEntry *e;

    if (modules_init_done[type]) {
        return;
    }

    l = find_type(type);

    QTAILQ_FOREACH(e, l, node) {
        e->init();
    }

    modules_init_done[type] = true;
}
```

在 module_call_init 中，会找到 MODULE_INIT_QOM 这种类型对应的 ModuleTypeList，找出列表中所有的 ModuleEntry，然后调用每个 ModuleEntry 的 init 函数. 这里需要注意的是，在 module_call_init 调用的这一步，所有 Module 的 init 函数都已经被调用过了. 后面会看到很多的 Module，当看到它们的时候，就需要意识到，它的 init 函数在这里被调用过了. 这里还是以对于 kvm 这个 module 为例子，看看它的 init 函数都做了哪些事情, 就会发现，其实它调用的是 [kvm_type_init](https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L3097).

```c
// https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L3089
static const TypeInfo kvm_accel_type = {
    .name = TYPE_KVM_ACCEL,
    .parent = TYPE_ACCEL,
    .instance_init = kvm_accel_instance_init,
    .class_init = kvm_accel_class_init,
    .instance_size = sizeof(KVMState),
};

static void kvm_type_init(void)
{
    type_register_static(&kvm_accel_type);
}

// https://elixir.bootlin.com/qemu/v5.0.0/source/qom/object.c#L89
static void type_table_add(TypeImpl *ti)
{
    assert(!enumerating_types);
    g_hash_table_insert(type_table_get(), (void *)ti->name, ti);
}

static TypeImpl *type_table_lookup(const char *name)
{
    return g_hash_table_lookup(type_table_get(), name);
}

static TypeImpl *type_new(const TypeInfo *info)
{
    TypeImpl *ti = g_malloc0(sizeof(*ti));
    int i;

    g_assert(info->name != NULL);

    if (type_table_lookup(info->name) != NULL) {
        fprintf(stderr, "Registering `%s' which already exists\n", info->name);
        abort();
    }

    ti->name = g_strdup(info->name);
    ti->parent = g_strdup(info->parent);

    ti->class_size = info->class_size;
    ti->instance_size = info->instance_size;

    ti->class_init = info->class_init;
    ti->class_base_init = info->class_base_init;
    ti->class_data = info->class_data;

    ti->instance_init = info->instance_init;
    ti->instance_post_init = info->instance_post_init;
    ti->instance_finalize = info->instance_finalize;

    ti->abstract = info->abstract;

    for (i = 0; info->interfaces && info->interfaces[i].type; i++) {
        ti->interfaces[i].typename = g_strdup(info->interfaces[i].type);
    }
    ti->num_interfaces = i;

    return ti;
}

static TypeImpl *type_register_internal(const TypeInfo *info)
{
    TypeImpl *ti;
    ti = type_new(info);

    type_table_add(ti);
    return ti;
}

TypeImpl *type_register(const TypeInfo *info)
{
    assert(info->parent);
    return type_register_internal(info);
}

TypeImpl *type_register_static(const TypeInfo *info)
{
    return type_register(info);
}
```

每一个 Module 既然要模拟某种设备，那应该定义一种类型 TypeImpl 来表示这些设备，这其实是一种面向对象编程的思路，只不过这里用的是纯 C 语言的实现，所以需要变相实现一下类和对象.

kvm_type_init 会注册 kvm_accel_type，定义上面的代码，就可以认为这样动态定义了一个类。这个类的名字是 TYPE_KVM_ACCEL，这个类有父类 TYPE_ACCEL，这个类的初始化应该调用函数 kvm_accel_class_init（看，这里已经直接叫类 class 了）. 如果用这个类声明一个对象，对象的大小应该是 instance_size. 是不是有点儿 Java 语言反射的意思，根据一些名称的定义，一个类就定义好了.

这里的调用链为：kvm_type_init->type_register_static->type_register->type_register_internal.

在 type_register_internal 中，会根据 kvm_accel_type 这个 TypeInfo，创建一个 TypeImpl 来表示这个新注册的类，也就是说，TypeImpl 才是我们想要声明的那个 class. 在 qemu 里面，有一个全局的哈希表 type_table，用来存放所有定义的类. 在 type_new 里面，先从全局表里面根据名字找这个类. 如果找到，说明这个类曾经被注册过，就报错；如果没有找到，说明这是一个新的类，则将 TypeInfo 里面信息填到 TypeImpl 里面. type_table_add 会将这个类注册到全局的表里面. 到这里，我们注意，class_init 还没有被调用，也即这个类现在还处于纸面的状态.

这点更加像 Java 的反射机制了. 在 Java 里面，对于一个类，首先写代码的时候要写一个 class xxx 的定义，编译好就放在.class 文件中，这也是出于纸面的状态. 然后，Java 会有一个 Class 对象，用于读取和表示这个纸面上的 class xxx，可以生成真正的对象.

相同的过程在后面的代码中也可以看到，class_init 会生成 XXXClass，就相当于 Java 里面的 Class 对象，TypeImpl 还会有一个 instance_init 函数，相当于构造函数，用于根据 XXXClass 生成 Object，这就相当于 Java 反射里面最终创建的对象. 和构造函数对应的还有 instance_finalize，相当于析构函数.

这一套反射机制放在 qom 文件夹下面，全称 QEMU Object Model，也即用 C 实现了一套面向对象的反射机制.

说完了初始化 Module，还回到 qemu_init 函数接着分析.

## 2. 解析 qemu 的命令行
接下来, 开始解析 qemu 的命令行了. qemu 的命令行解析，就是下面这样一长串.
```c
// https://elixir.bootlin.com/qemu/latest/source/softmmu/vl.c#L2871
 	qemu_add_opts(&qemu_drive_opts);
    ...
```

为什么有这么多的 opts 呢？这是因为，上面创建vm的参数都是简单的参数，实际运行中创建的 kvm 参数会复杂 N 倍. 这里贴一个开源云平台软件 OpenStack 创建出来的 KVM 的参数，如下所示, 挺复杂的，不需要全部看懂，只需要看懂一部分就行了:
```bash
# qemu-system-x86_64
-enable-kvm
-name instance-00000024
-machine pc-i440fx-trusty,accel=kvm,usb=off
-cpu SandyBridge,+erms,+smep,+fsgsbase,+pdpe1gb,+rdrand,+f16c,+osxsave,+dca,+pcid,+pdcm,+xtpr,+tm2,+est,+smx,+vmx,+ds_cpl,+monitor,+dtes64,+pbe,+tm,+ht,+ss,+acpi,+ds,+vme
-m 2048
-smp 1,sockets=1,cores=1,threads=1
......
-rtc base=utc,driftfix=slew
-drive file=/var/lib/nova/instances/1f8e6f7e-5a70-4780-89c1-464dc0e7f308/disk,if=none,id=drive-virtio-disk0,format=qcow2,cache=none
-device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1
-netdev tap,fd=32,id=hostnet0,vhost=on,vhostfd=37
-device virtio-net-pci,netdev=hostnet0,id=net0,mac=fa:16:3e:d1:2d:99,bus=pci.0,addr=0x3
-chardev file,id=charserial0,path=/var/lib/nova/instances/1f8e6f7e-5a70-4780-89c1-464dc0e7f308/console.log
-vnc 0.0.0.0:12
-device cirrus-vga,id=video0,bus=pci.0,addr=0x2
```

- -enable-kvm：表示启用硬件辅助虚拟化
- -name instance-00000024：表示虚拟机的名称
- -machine pc-i440fx-trusty,accel=kvm,usb=off：machine是什么呢？其实就是计算机体系结构.

	qemu 会模拟多种体系结构，常用的有普通 PC 机，也即 x86 的 32 位或者 64 位的体系结构、Mac 电脑 PowerPC 的体系结构、Sun 的体系结构、MIPS 的体系结构，精简指令集. 如果使用 KVM hardware-assisted virtualization，也即 BIOS 中 VD-T 是打开的，则参数中 accel=kvm;如果不使用 hardware-assisted virtualization，用的是纯模拟，则有参数 accel = tcg，-no-kvm.
-cpu SandyBridge,+erms,+smep,+fsgsbase,+pdpe1gb,+rdrand,+f16c,+osxsave,+dca,+pcid,+pdcm,+xtpr,+tm2,+est,+smx,+vmx,+ds_cpl,+monitor,+dtes64,+pbe,+tm,+ht,+ss,+acpi,+ds,+vme：表示设置 CPU，SandyBridge 是 Intel 处理器，后面的加号都是添加的 CPU 的参数，这些参数会显示在 /proc/cpuinfo 里面
-m 2048：表示内存
-smp 1,sockets=1,cores=1,threads=1：SMP 是对称多处理器，和 NUMA 对应. qemu 仿真了一个具有 1 个 vcpu，一个 socket，一个 core，一个 threads 的处理器.

	socket 就是主板上插 cpu 的槽的数目，也即常说的“路”，core 就是平时说的“核”，即双核、4 核等。thread 就是每个 core 的硬件线程数，即超线程. 举个具体的例子，某个服务器是：2 路 4 核超线程（一般默认为 2 个线程），通过 cat /proc/cpuinfo，看到的是 2*4*2=16 个 processor，很多人也习惯当成 16 核了
-rtc base=utc,driftfix=slew：表示系统时间由参数 -rtc 指定
-device cirrus-vga,id=video0,bus=pci.0,addr=0x2：表示显示器用参数 -vga 设置，默认为 cirrus，它模拟了 CL-GD5446PCI VGA card
- 有关网卡，使用 -net 参数和 -device

	- 从 HOST 角度：-netdev tap,fd=32,id=hostnet0,vhost=on,vhostfd=37
	- 从 GUEST 角度：-device virtio-net-pci,netdev=hostnet0,id=net0,mac=fa:16:3e:d1:2d:99,bus=pci.0,addr=0x3
- 有关硬盘，使用 -hda -hdb，或者使用 -drive 和 -device

	- 从 HOST 角度：-drive file=/var/lib/nova/instances/1f8e6f7e-5a70-4780-89c1-464dc0e7f308/disk,if=none,id=drive-virtio-disk0,format=qcow2,cache=none
	- 从 GUEST 角度：-device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1-vnc
0.0.0.0:12：设置 VNC

在 qemu_init 函数中，接下来的 for 循环和大量的 switch case 语句，就是对于这些参数的解析，就不一一解析，后面真的用到这些参数的时候，再仔细看.

## 3. 初始化 machine
回到 qemu_init 函数，接下来是[初始化 machine](https://elixir.bootlin.com/qemu/latest/source/softmmu/vl.c#L2871).

```c
// https://elixir.bootlin.com/qemu/latest/source/softmmu/vl.c#L2871
    machine_class = select_machine();
    ...
    current_machine = MACHINE(object_new_with_class(OBJECT_CLASS(machine_class))); // https://elixir.bootlin.com/qemu/latest/source/softmmu/vl.c#L3876
```

这里面的 machine_class 是什么呢？这还得从 machine 参数`-machine pc-i440fx-trusty,accel=kvm,usb=off`说起.


这里的 pc-i440fx 是 x86 机器默认的体系结构, 在 hw/i386/pc_piix.c 中，它定义了对应的 machine_class.

```c
// https://elixir.bootlin.com/qemu/v5.0.0/source/include/hw/i386/pc.h#L281
#define DEFINE_PC_MACHINE(suffix, namestr, initfn, optsfn) \
    static void pc_machine_##suffix##_class_init(ObjectClass *oc, void *data) \
    { \
        MachineClass *mc = MACHINE_CLASS(oc); \
        optsfn(mc); \
        mc->init = initfn; \
    } \
    static const TypeInfo pc_machine_type_##suffix = { \
        .name       = namestr TYPE_MACHINE_SUFFIX, \
        .parent     = TYPE_PC_MACHINE, \
        .class_init = pc_machine_##suffix##_class_init, \
    }; \
    static void pc_machine_init_##suffix(void) \
    { \
        type_register(&pc_machine_type_##suffix); \
    } \
    type_init(pc_machine_init_##suffix)

// https://elixir.bootlin.com/qemu/v5.0.0/source/hw/i386/pc_piix.c#L398
#define DEFINE_I440FX_MACHINE(suffix, name, compatfn, optionfn) \
    static void pc_init_##suffix(MachineState *machine) \
    { \
        void (*compat)(MachineState *m) = (compatfn); \
        if (compat) { \
            compat(machine); \
        } \
        pc_init1(machine, TYPE_I440FX_PCI_HOST_BRIDGE, \
                 TYPE_I440FX_PCI_DEVICE); \
    } \
    DEFINE_PC_MACHINE(suffix, name, pc_init_##suffix, optionfn)

static void pc_i440fx_machine_options(MachineClass *m)
{
    PCMachineClass *pcmc = PC_MACHINE_CLASS(m);
    pcmc->default_nic_model = "e1000";

    m->family = "pc_piix";
    m->desc = "Standard PC (i440FX + PIIX, 1996)";
    m->default_machine_opts = "firmware=bios-256k.bin";
    m->default_display = "std";
    machine_class_allow_dynamic_sysbus_dev(m, TYPE_RAMFB_DEVICE);
}

static void pc_i440fx_5_0_machine_options(MachineClass *m)
{
    PCMachineClass *pcmc = PC_MACHINE_CLASS(m);
    pc_i440fx_machine_options(m);
    m->alias = "pc";
    m->is_default = true;
    pcmc->default_cpu_version = 1;
}

DEFINE_I440FX_MACHINE(v5_0, "pc-i440fx-5.0", NULL,
                      pc_i440fx_5_0_machine_options);
...
```

为了定义 machine_class，这里有一系列的宏定义. 入口是 DEFINE_I440FX_MACHINE. 这个宏有几个参数，v5_0 是后缀，"pc-i440fx-5.0"是名字，pc_i440fx_5_0_machine_options 是一个函数，用于定义 machine_class 相关的选项.

先不看 pc_i440fx_5_0_machine_options，先来看 DEFINE_I440FX_MACHINE.

DEFINE_I440FX_MACHINE里面定义了一个 pc_init_##suffix，也就是 pc_init_v5_0. 这里面转而调用 pc_init1. 注意这里这个函数只是定义了一下，没有被调用.

接下来，DEFINE_I440FX_MACHINE 里面又定义了 DEFINE_PC_MACHINE. 它有四个参数，除了 DEFINE_I440FX_MACHINE 传进来的三个参数以外，多了一个 initfn，也即初始化函数，指向刚才定义的 pc_init_##suffix.

在 DEFINE_PC_MACHINE 中，定义了一个函数 pc_machine_##suffix##class_init. 从函数的名字 class_init 可以看出，这是 machine_class 从纸面上的 class 初始化为 Class 对象的方法. 在这个函数里面，可以看到，它创建了一个 MachineClass 对象，这个就是 Class 对象. MachineClass 对象的 init 函数指向上面定义的 pc_init##suffix，说明这个函数是 machine 这种类型初始化的一个函数，后面会被调用.

接着，看 DEFINE_PC_MACHINE. 它定义了一个 pc_machine_type_##suffix 的 TypeInfo. 这是用于生成纸面上的 class 的原材料，果真后面调用了 type_init.

看到了 type_init，应该能够想到，既然它定义了一个纸面上的 class，那上面的那句 module_call_init，会和上面解析的 type_init 是一样的，在全局的表里面注册了一个全局的名字是"pc-i440fx-5.0"的纸面上的 class，也即 TypeImpl.

现在全局表中有这个纸面上的 class 了.回到 [select_machine](https://elixir.bootlin.com/qemu/latest/source/softmmu/vl.c#L2414).

在 select_machine 中，有两种方式可以生成 MachineClass. 一种方式是 find_default_machine，找一个默认的；另一种方式是 machine_parse，通过解析参数生成 MachineClass. 无论哪种方式，都会调用 [object_class_get_list](https://elixir.bootlin.com/qemu/latest/source/qom/object.c#L1087) 获得一个 MachineClass 的列表，然后在里面找.

```c
// https://elixir.bootlin.com/qemu/latest/source/qom/object.c#L1009
static void object_class_foreach_tramp(gpointer key, gpointer value,
                                       gpointer opaque)
{
    OCFData *data = opaque;
    TypeImpl *type = value;
    ObjectClass *k;

    type_initialize(type);
    k = type->class;

    if (!data->include_abstract && type->abstract) {
        return;
    }

    if (data->implements_type && 
        !object_class_dynamic_cast(k, data->implements_type)) {
        return;
    }

    data->fn(k, data->opaque);
}

// https://elixir.bootlin.com/qemu/latest/source/qom/object.c#L1087
GSList *object_class_get_list(const char *implements_type,
                              bool include_abstract)
{
    GSList *list = NULL;

    object_class_foreach(object_class_get_list_tramp,
                         implements_type, include_abstract, &list);
    return list;
}
```

在全局表 type_table_get() 中，对于每一项 TypeImpl，都执行 object_class_foreach_tramp.


在 object_class_foreach_tramp 中，会调用将 type_initialize，这里面会调用 class_init 将纸面上的 class 也即 TypeImpl 变为 ObjectClass，ObjectClass 是所有 Class 类的祖先，MachineClass 是它的子类.

因为在 machine 的命令行里面，我们指定了名字为"pc-i440fx-5.0"，就肯定能够找到我们注册过了的 TypeImpl，并调用它的 class_init 函数.

因而 pc_machine_##suffix##class_init 会被调用，在这里面，pc_i440fx_machine_options 才真正被调用初始化 MachineClass，并且将 MachineClass 的 init 函数设置为 pc_init##suffix. 也即，当 select_machine 执行完毕后，就有一个 MachineClass 了.

接着，回到 [object_new_with_class](https://elixir.bootlin.com/qemu/latest/source/qom/object.c#L690). 这就很好理解了，MachineClass 是一个 Class 类，接下来应该通过它生成一个 Instance，也即对象，这就是 object_new_with_class 的作用.

object_new_with_class 中，TypeImpl 的 instance_init 会被调用，创建一个对象. current_machine 就是这个对象，它的类型是 MachineState.

至此，绕了这么大一圈，有关体系结构的对象才创建完毕，接下来很多的设备的初始化，包括 CPU 和内存的初始化，都是围绕着体系结构的对象来的，后面会常常看到 current_machine.

## 4. 初始化块设备
接下来初始化的是块设备，调用的是 [configure_blockdev](https://elixir.bootlin.com/qemu/latest/source/softmmu/vl.c#L1028).

```c
// https://elixir.bootlin.com/qemu/latest/source/softmmu/vl.c#L4138
configure_blockdev(&bdo_queue, machine_class, snapshot);
```

## 5. 初始化计算虚拟化的加速模式
接下来初始化的是计算虚拟化的加速模式，也即要不要使用 KVM. 根据参数中的配置是启用 KVM. 这里调用的是 [configure_accelerators](https://elixir.bootlin.com/qemu/latest/source/softmmu/vl.c#L2719).

在 configure_accelerators 中，看命令行参数里面的 accel，发现是 kvm，则调用 [accel_find](https://elixir.bootlin.com/qemu/latest/source/softmmu/vl.c#L2693) 根据名字，得到相应的纸面上的 class，并初始化为 Class 类.

MachineClass 是计算机体系结构的 Class 类，同理，[AccelClass](https://elixir.bootlin.com/qemu/latest/source/include/sysemu/accel.h#L34) 就是加速器的 Class 类，先通过 object_new_with_class，将 AccelClass 这个 Class 类实例化为 AccelState，类似对于体系结构的实例MachineState, 然后调用 accel_init_machine进行初始化.

在 accel_find 中，会根据名字 kvm，找到纸面上的 class，也即 kvm_accel_type. 然后configure_accelerators -> do_configure_accelerator -> accel_init_machine.init_machine()，里面调用 [kvm_accel_type](https://elixir.bootlin.com/qemu/latest/source/accel/kvm/kvm-all.c#L3089) 的 class_init 方法，也即 [kvm_accel_class_init](https://elixir.bootlin.com/qemu/latest/source/accel/kvm/kvm-all.c#L3068).

在 kvm_accel_class_init 中，创建 AccelClass，将 init_machine 设置为 kvm_init. 在 accel_init_machine 中其实就调用了这个 init_machine 函数，也即调用 [kvm_init 方法](https://elixir.bootlin.com/qemu/latest/source/accel/kvm/kvm-all.c#L1865).

这里面的操作就从用户态到内核态的 KVM 了. 就像前面原理讲过的一样，用户态使用内核态 KVM 的能力，需要[打开一个文件 /dev/kvm](https://elixir.bootlin.com/qemu/latest/source/accel/kvm/kvm-all.c#L1903)，这是一个字符设备文件, 打开一个字符设备文件的过程这里不再赘述, 见[io/linux实现].

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L3929
static struct file_operations kvm_chardev_ops = {
	.unlocked_ioctl = kvm_dev_ioctl,
	.llseek		= noop_llseek,
	KVM_COMPAT(kvm_dev_ioctl),
};

static struct miscdevice kvm_dev = {
	KVM_MINOR,
	"kvm",
	&kvm_chardev_ops,
};

// https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L3929
static long kvm_dev_ioctl(struct file *filp,
			  unsigned int ioctl, unsigned long arg)
{
	long r = -EINVAL;

	switch (ioctl) {
	case KVM_GET_API_VERSION:
		if (arg)
			goto out;
		r = KVM_API_VERSION;
		break;
	case KVM_CREATE_VM:
		r = kvm_dev_ioctl_create_vm(arg);
		break;
	case KVM_CHECK_EXTENSION:
		r = kvm_vm_ioctl_check_extension_generic(NULL, arg);
		break;
	case KVM_GET_VCPU_MMAP_SIZE:
		if (arg)
			goto out;
		r = PAGE_SIZE;     /* struct kvm_run */
#ifdef CONFIG_X86
		r += PAGE_SIZE;    /* pio data page */
#endif
#ifdef CONFIG_KVM_MMIO
		r += PAGE_SIZE;    /* coalesced mmio ring page */
#endif
		break;
	case KVM_TRACE_ENABLE:
	case KVM_TRACE_PAUSE:
	case KVM_TRACE_DISABLE:
		r = -EOPNOTSUPP;
		break;
	default:
		return kvm_arch_dev_ioctl(filp, ioctl, arg);
	}
out:
	return r;
}
```

KVM 这个字符设备文件定义了一个字符设备文件的操作函数 kvm_chardev_ops，这里面只定义了 ioctl 的操作. 接下来，用户态就通过 ioctl 系统调用，调用到 kvm_dev_ioctl 这个函数.

kvm_dev_ioctl里可以看到，在用户态 qemu 中，调用 KVM_GET_API_VERSION 查看版本号，内核就有相应的分支，返回版本号，如果能够匹配上，则可调用 KVM_CREATE_VM 创建虚拟机. 创建虚拟机，需要调用 [kvm_dev_ioctl_create_vm](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L3843).

在 kvm_dev_ioctl_create_vm 中，首先调用 kvm_create_vm 创建一个 [struct kvm](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/kvm_host.h#L445) 结构. 这个结构在内核里面代表一个虚拟机. 从其结构的定义里，可以看到，这里面有 vcpu，有 mm_struct 结构. 这个结构本来用来管理进程的内存的. 虚拟机也是一个进程，所以虚拟机的用户进程空间也是用它来表示. 虚拟机里面的操作系统以及应用的进程空间不归它管.

在 kvm_dev_ioctl_create_vm 中，第二件事情就是[创建一个文件描述符，和 struct file 关联起来，这个 struct file 的 file_operations 会被设置为 kvm_vm_fops](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L3861).

kvm_dev_ioctl_create_vm 结束之后，对于一台虚拟机而言，只是在内核中有一个数据结构，对于相应的资源还没有分配，所以还需要接着看.

## 6. 初始化网络设备
接下来，调用 [net_init_clients](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L1542) 进行网络设备的初始化. 可以解析 net 参数，也会在 net_init_clients 中解析 netdev 参数. 这属于网络虚拟化的部分，先暂时放一下.

## 7.CPU 虚拟化
接下来，要调用 [machine_run_board_init](https://elixir.bootlin.com/qemu/latest/source/hw/core/machine.c#L1091). 这里面调用了 MachineClass 的 init 函数, 上面是以`pc-i440fx-5.0`距离, 也就是v5_0. 盼啊盼才到了它，这才调用了 pc_init1.

在 pc_init1 里面，需要重点关注两件重要的事情，一个的 CPU 的虚拟化，主要调用 [x86_cpus_init](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/i386/x86.c#L133)；另外就是内存的虚拟化，主要调用 [pc_memory_init](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/i386/pc.c#L936).

在 x86_cpus_init 中，对于每一个 CPU，都调用 [x86_new_cpu](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/i386/x86.c#L119)，在这里，看到了 object_new，这又是一个从 TypeImpl 到 Class 类再到对象的一个过程. 这个时候，就要看 CPU 的类是怎么组织的了. 在上面的参数里面，CPU 的配置是这样的:`
-cpu SandyBridge,+erms,+smep,+fsgsbase,+pdpe1gb,+rdrand,+f16c,+osxsave,+dca,+pcid,+pdcm,+xtpr,+tm2,+est,+smx,+vmx,+ds_cpl,+monitor,+dtes64,+pbe,+tm,+ht,+ss,+acpi,+ds,+vme`.

在这里要知道，SandyBridge 是 CPU 的一种类型. 在 hw/i386/pc.c 中，能看到这种 CPU 的定义:[`{ "SandyBridge" "-" TYPE_X86_CPU, "min-xlevel", "0x8000000a" }`](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/i386/pc.c#L230).


接下来，就来看"SandyBridge"，也即 TYPE_X86_CPU 这种 CPU 的类，是一个什么样的结构.

```c
// https://elixir.bootlin.com/qemu/v5.0.0/source/hw/core/qdev.c#L1263
static const TypeInfo device_type_info = {
    .name = TYPE_DEVICE,
    .parent = TYPE_OBJECT,
    .instance_size = sizeof(DeviceState),
    .instance_init = device_initfn,
    .instance_post_init = device_post_init,
    .instance_finalize = device_finalize,
    .class_base_init = device_class_base_init,
    .class_init = device_class_init,
    .abstract = true,
    .class_size = sizeof(DeviceClass),
    .interfaces = (InterfaceInfo[]) {
        { TYPE_VMSTATE_IF },
        { TYPE_RESETTABLE_INTERFACE },
        { }
    }
};

// https://elixir.bootlin.com/qemu/v5.0.0/source/hw/core/cpu.c#L440
static const TypeInfo cpu_type_info = {
    .name = TYPE_CPU,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(CPUState),
    .instance_init = cpu_common_initfn,
    .instance_finalize = cpu_common_finalize,
    .abstract = true,
    .class_size = sizeof(CPUClass),
    .class_init = cpu_class_init,
};

// https://elixir.bootlin.com/qemu/v5.0.0/source/target/i386/cpu.c#L7293
static const TypeInfo x86_cpu_type_info = {
    .name = TYPE_X86_CPU,
    .parent = TYPE_CPU,
    .instance_size = sizeof(X86CPU),
    .instance_init = x86_cpu_initfn,
    .abstract = true,
    .class_size = sizeof(X86CPUClass),
    .class_init = x86_cpu_common_class_init,
};
```

CPU 这种类的定义是有多层继承关系的. TYPE_X86_CPU 的父类是 TYPE_CPU，TYPE_CPU 的父类是 TYPE_DEVICE，TYPE_DEVICE 的父类是 TYPE_OBJECT, 到头了.

这里面每一层都有 class_init，用于从 TypeImpl 生产 xxxClass，也有 instance_init 将 xxxClass 初始化为实例.

在 TYPE_X86_CPU 这一层的 class_init 中，也即 [x86_cpu_common_class_init](https://elixir.bootlin.com/qemu/v5.0.0/source/target/i386/cpu.c#L7231) 中，设置了 DeviceClass 的 realize 函数为 x86_cpu_realizefn. 这个函数很重要，马上就能用到.

在 TYPE_DEVICE 这一层的 instance_init 函数 device_initfn，会为这个设备添加一个属性"realized"，要设置这个属性，需要用函数 [device_set_realized](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/core/qdev.c#L852).

回到 [x86_new_cpu](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/i386/x86.c#L119) 函数，它里面就是通过 object_property_set_bool 设置这个属性为 true，所以 device_set_realized 函数会被调用. 在 device_set_realized 中，DeviceClass 的 realize 函数 [x86_cpu_realizefn](https://elixir.bootlin.com/qemu/v5.0.0/source/target/i386/cpu.c#L6472) 会被调用. x86_cpu_realizefn里面的 [qemu_init_vcpu](https://elixir.bootlin.com/qemu/v5.0.0/source/cpus.c#L2058) 会调用 [qemu_kvm_start_vcpu](https://elixir.bootlin.com/qemu/v5.0.0/source/cpus.c#L1998).

在qemu_kvm_start_vcpu里面，为这个 vcpu 创建一个线程，也即虚拟机里面的一个 vcpu 对应物理机上的一个线程，然后这个线程被调度到某个物理 CPU 上. 来看这个 vcpu 的线程执行函数[qemu_kvm_cpu_thread_fn](https://elixir.bootlin.com/qemu/v5.0.0/source/cpus.c#L1218).

在 qemu_kvm_cpu_thread_fn 中，先是 [kvm_init_vcpu](https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L383) 初始化这个 vcpu.

kvm_init_vcpu -> [kvm_get_vcpu](https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L365)，在kvm_get_vcpu中会调用 kvm_vm_ioctl(s, KVM_CREATE_VCPU, (void *)vcpu_id)，在内核里面创建一个 vcpu.

在上面创建 KVM_CREATE_VM 的时候，就已经创建了一个 struct file，它的 file_operations 被设置为 kvm_vm_fops，这个内核文件也是可以响应 ioctl 的. 如果视角切换到内核 KVM，在 kvm_vm_ioctl 函数中，有对于 KVM_CREATE_VCPU 的处理，调用的是 [kvm_vm_ioctl_create_vcpu](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L3022).


在 kvm_vm_ioctl_create_vcpu 中，[kvm_arch_vcpu_create](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/x86.c#L9416) 调用 [kvm_x86_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmx.c#L7846) 的 vcpu_create 函数即[svm_create_vcpu](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/svm/svm.c#L1156)来创建 CPU.

然后，[create_vcpu_fd](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L2994) 又创建了一个 struct file，它的 file_operations 指向 [kvm_vcpu_fops](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L2983). 从这里可以看出，KVM 的内核模块是一个文件，可以通过 ioctl 进行操作. 基于这个内核模块创建的 VM 也是一个文件，也可以通过 ioctl 进行操作. 在这个 VM 上创建的 vcpu 同样是一个文件，同样可以通过 ioctl 进行操作.

回过头来看，kvm_x86_ops 的 vcpu_create 函数, kernel中存在两种kvm_x86_ops: [vmx_x86_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmx.c#L7846)和svm_x86_ops, 分别对应vmx/svm(vmx：英特尔CPU虚拟化技术; svm：AMD的CPU虚拟化技术), 这里看vmx_x86_ops.

vmx_create_vcpu 创建用于表示 vcpu 的结构 [struct vcpu_vmx](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmx.h#L201)，并填写里面的内容. 例如 guest_msrs，学系统调用的时候提过 msr 寄存器，虚拟机也需要有这样的寄存器.

enable_ept 是和内存虚拟化相关的，EPT 全称 Extended Page Table，顾名思义，是优化内存虚拟化的，先忽略.

最最重要的就是 loaded_vmcs 了. VMCS 是什么呢？

它的全称是 Virtual Machine Control Structure. 它是来干什么呢？学进程调度的时候学过，为了支持进程在 CPU 上的切换，CPU 硬件要求有一个 TSS 结构，用于保存进程运行时的所有寄存器的状态，进程切换的时候，需要根据 TSS 恢复寄存器.

虚拟机也是一个进程，也需要切换，而且切换更加的复杂，可能是两个虚拟机之间切换，也可能是虚拟机切换给内核，虚拟机因为里面还有另一个操作系统，要保存的信息比普通的进程多得多. 那就需要有一个结构来保存虚拟机运行的上下文，VMCS 就是是 Intel 实现 CPU 虚拟化，记录 vCPU 状态的一个关键数据结构.

VMCS 数据结构主要包含以下信息:
- Guest-state area，即 vCPU 的状态信息，包括 vCPU 的基本运行环境，例如寄存器等
- Host-state area，是物理 CPU 的状态信息, 物理 CPU 和 vCPU 之间也会来回切换，所以，VMCS 中既要记录 vCPU 的状态，也要记录物理 CPU 的状态
- VM-execution control fields，对 vCPU 的运行行为进行控制. 例如，发生中断怎么办，是否使用 EPT（Extended Page Table）功能等.

接下来，对于 VMCS，有两个重要的操作.

VM-Entry，称为从根模式切换到非根模式，也即切换到 guest 上，这个时候 CPU 上运行的是虚拟机. VM-Exit 称为 CPU 从非根模式切换到根模式，也即从 guest 切换到宿主机. 例如，当要执行一些虚拟机没有权限的敏感指令时.

![](/misc/img/virt/1ec7600be619221dfac03e6ade67f7dc.png)

为了维护这两个动作，VMCS 里面还有几项内容：
- VM-exit control fields，对 VM Exit(退出vmx non-root operation) 的行为进行控制. 比如，VM Exit 的时候对 vCPU 来说需要保存哪些 MSR 寄存器，对于主机 CPU 来说需要恢复哪些 MSR 寄存器
- VM-entry control fields，对 VM Entry(进入vmx non-root operation) 的行为进行控制. 比如，需要保存和恢复哪些 MSR 寄存器等
- VM-exit information fields，记录下发生 VM Exit 发生的原因及一些必要的信息，方便对 VM Exit 事件进行处理.

至此，内核准备完毕.

再回到 qemu 的 kvm_init_vcpu 函数，这里面除了创建内核中的 vcpu 结构之外，还通过 mmap 将内核的 vcpu 结构，映射到 qemu 中 CPUState 的 kvm_run 中，为什么能用 mmap 呢，因为 vcpu 也是一个文件. 再回到这个 vcpu 的线程函数 [qemu_kvm_cpu_thread_fn](https://elixir.bootlin.com/qemu/v5.0.0/source/cpus.c#L1218)，它在执行 kvm_init_vcpu 创建 vcpu 之后，接下来是一个 do-while 循环，也即一直运行，并且通过调用 [kvm_cpu_exec](https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L2307)，运行这个虚拟机.

在 kvm_cpu_exec 中，能看到一个循环，在循环中，kvm_vcpu_ioctl(KVM_RUN) 运行这个虚拟机，这个时候 CPU 进入 VM-Entry，也即进入客户机模式. 如果一直是客户机的操作系统占用这个 CPU，则会一直停留在这一行运行，一旦这个调用返回了，就说明 CPU 进入 VM-Exit 退出客户机模式，将 CPU 交还给宿主机. 在循环中，会对退出的原因 exit_reason 进行分析处理，因为有了 I/O，还有了中断等，做相应的处理. 处理完毕之后，再次循环，再次通过 VM-Entry，进入客户机模式. 如此循环，直到虚拟机正常或者异常退出.

来看 kvm_vcpu_ioctl(KVM_RUN) 在内核做了哪些事情. 因为vcpu 在内核也是一个文件，也是通过 ioctl 进行用户态和内核态通信的，在内核中，调用的是 [kvm_vcpu_ioctl](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L3120).

kvm_vcpu_ioctl根据`case KVM_RUN`调用[kvm_arch_vcpu_ioctl_run(https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/x86.c#L8823)-> [vcpu_run](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/x86.c#L8649)，vcpu_run里面也是一个无限循环.

在这个循环中，除了调用 [vcpu_enter_guest](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/x86.c#L8309) 进入客户机模式运行之外，还有对于信号的响应 signal_pending，也即一台虚拟机是可以被 kill 掉的，还有对于调度的响应，这台虚拟机可以被从当前的物理 CPU 上赶下来，换成别的虚拟机或者其他进程. 这里重点看 vcpu_enter_guest.

在 vcpu_enter_guest 中，会调用 vmx_x86_ops.run 即 [vmx_vcpu_run](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmx.c#L6657) 函数，进入客户机模式.

[在 vmx_vcpu_run 中，出现了汇编语言的代码](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmenter.S#L103), 比较难看懂，但是没有关系呀，里面有注释呀，可以沿着注释来看:
1. 首先是 Store host registers，要从宿主机模式变为客户机模式了，所以原来宿主机运行时候的寄存器要保存下来
1. 接下来是 Load guest registers，将原来客户机运行的时候的寄存器加载进来
1. 接下来是 [Enter guest mode](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmenter.S#L157)，[调用 vmlaunch(cpu指令)进入客户机模型运行](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmenter.S#L157)，或者 vmresume 恢复客户机模型运行
1. 如果客户机因为某种原因退出，Save guest registers, load host registers，也即保存客户机运行的时候的寄存器，就加载宿主机运行的时候的寄存器
1. 最后将 exit_reason 保存在 vmx 结构中.

kernel加载宿主机寄存器，进入宿主机模式运行，并且会记录退出虚拟机模式的原因. 大部分的原因是等待 I/O，因而宿主机调用 [kvm_handle_io](https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L2386) 进行处理.

至此，CPU 虚拟化就解析完了, 可以看到，CPU 的虚拟化是用户态的 qemu 和内核态的 KVM 共同配合完成的, 它们二者通过 ioctl 进行通信.

![](/misc/img/virt/c43639f7024848aa3e828bcfc10ca467.png)

## vm的内存管理
有了虚拟机，内存就变成了四类：
- 虚拟机里面的虚拟内存（Guest OS Virtual Memory，GVA），这是虚拟机里面的进程看到的内存空间
- 虚拟机里面的物理内存（Guest OS Physical Memory，GPA），这是虚拟机里面的操作系统看到的内存，它认为这是物理内存
- 物理机的虚拟内存（Host Virtual Memory，HVA），这是物理机上的 qemu 进程看到的内存空间
- 物理机的物理内存（Host Physical Memory，HPA），这是物理机上的操作系统看到的内存

从 GVA 到 GPA，到 HVA，再到 HPA，这样几经转手，计算机的性能就会变得很差.

### 内存管理
参考:
- [*QEMU学习笔记——内存](https://www.binss.me/blog/qemu-note-of-memory/)

由于 CPU 和内存是紧密结合的，因而内存虚拟化的初始化过程，和 CPU 虚拟化的初始化是一起完成的.

CPU 虚拟化初始化的时候，会调用 kvm_init 函数，它里面打开了"/dev/kvm"这个字符文件，并且通过 ioctl 调用到内核 kvm 的 KVM_CREATE_VM 操作，除了这些 CPU 相关的调用，其实还有内存相关的.

```c
// https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L2099
    kvm_memory_listener_register(s, &s->memory_listener,
                                 &address_space_memory, 0);
    memory_listener_register(&kvm_io_listener,
                             &address_space_io);
    memory_listener_register(&kvm_coalesced_pio_listener,
                             &address_space_io);

// https://elixir.bootlin.com/qemu/v5.0.0/source/include/exec/memory.h#L660
/**
 * AddressSpace: describes a mapping of addresses to #MemoryRegion objects
 */
struct AddressSpace {
    /* private: */
    struct rcu_head rcu;
    char *name;
    MemoryRegion *root;

    /* Accessed via RCU.  */
    struct FlatView *current_map;

    int ioeventfd_nb;
    struct MemoryRegionIoeventfd *ioeventfds;
    QTAILQ_HEAD(, MemoryListener) listeners;
    QTAILQ_ENTRY(AddressSpace) address_spaces_link;
};

// https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L1237
void kvm_memory_listener_register(KVMState *s, KVMMemoryListener *kml,
                                  AddressSpace *as, int as_id)
{
    int i;

    qemu_mutex_init(&kml->slots_lock);
    kml->slots = g_malloc0(s->nr_slots * sizeof(KVMSlot));
    kml->as_id = as_id;

    for (i = 0; i < s->nr_slots; i++) {
        kml->slots[i].slot = i;
    }

    kml->listener.region_add = kvm_region_add;
    kml->listener.region_del = kvm_region_del;
    kml->listener.log_start = kvm_log_start;
    kml->listener.log_stop = kvm_log_stop;
    kml->listener.log_sync = kvm_log_sync;
    kml->listener.log_clear = kvm_log_clear;
    kml->listener.priority = 10;

    memory_listener_register(&kml->listener, as);

    for (i = 0; i < s->nr_as; ++i) {
        if (!s->as[i].as) {
            s->as[i].as = as;
            s->as[i].ml = kml;
            break;
        }
    }
}
```

这里面有两个地址空间 AddressSpace，一个是系统内存的地址空间 address_space_memory，一个用于 I/O 的地址空间 address_space_io. 这里重点看 address_space_memory.

对于一个地址空间，会有多个内存区域 MemoryRegion 组成树形结构. 这里面，root 是这棵树的根. 另外，还有一个 MemoryListener 链表，当内存区域发生变化的时候，需要做一些动作，使得用户态和内核态能够协同，就是由这些 MemoryListener 完成的.

在 kvm_init 这个时候，还没有内存区域加入进来，root 还是空的，但是可以先注册 MemoryListener，这里注册的是 [KVMMemoryListener](https://elixir.bootlin.com/qemu/v5.0.0/source/include/sysemu/kvm_int.h#L28).

在这个 KVMMemoryListener 中是这样配置的：当添加一个 MemoryRegion 的时候，region_add 会被调用, 下面会用到. 

接下来，在 qemu 启动的 qemu_init 函数中，会调用 [cpu_exec_init_all](https://elixir.bootlin.com/qemu/v5.0.0/source/exec.c#L3412)->[memory_map_init](https://elixir.bootlin.com/qemu/v5.0.0/source/exec.c#L2963).

```c
// https://elixir.bootlin.com/qemu/v5.0.0/source/exec.c#L2963
static void memory_map_init(void)
{
    system_memory = g_malloc(sizeof(*system_memory));

    memory_region_init(system_memory, NULL, "system", UINT64_MAX);
    address_space_init(&address_space_memory, system_memory, "memory");

    system_io = g_malloc(sizeof(*system_io));
    memory_region_init_io(system_io, NULL, &unassigned_io_ops, NULL, "io",
                          65536);
    address_space_init(&address_space_io, system_io, "I/O");
}

// https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L2767

void address_space_init(AddressSpace *as, MemoryRegion *root, const char *name)
{
    memory_region_ref(root);
    as->root = root;
    as->current_map = NULL;
    as->ioeventfd_nb = 0;
    as->ioeventfds = NULL;
    QTAILQ_INIT(&as->listeners);
    QTAILQ_INSERT_TAIL(&address_spaces, as, address_spaces_link);
    as->name = g_strdup(name ? name : "anonymous");
    address_space_update_topology(as);
    address_space_update_ioeventfds(as);
}
```
在这里，对于系统内存区域 system_memory 和用于 I/O 的内存区域 system_io，都进行了初始化，并且关联到了相应的地址空间 AddressSpace.

对于系统内存地址空间 address_space_memory，需要把它里面内存区域的根 root 设置为 system_memory. 另外，在这里，还调用了 [address_space_update_topology](https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L1046).

address_space_update_topology里面会生成 AddressSpace 的 flatview, flatview 是什么意思呢？

可以看到，在 AddressSpace 里面，除了树形结构的 MemoryRegion 之外，还有一个 flatview 结构，其实这个结构就是把这样一个树形的内存结构变成平的内存结构. 因为树形内存结构比较容易管理，但是平的内存结构，比较方便和内核里面通信，来请求物理内存. 虽然操作系统内核里面也是用树形结构来表示内存区域的，但是用户态向内核申请内存的时候，会按照平的、连续的模式进行申请. 这里，qemu 在用户态，所以要做这样一个转换.

在 [address_space_set_flatview](https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L1001) 中，将老的 flatview 和新的 flatview 进行比较. 如果不同，说明内存结构发生了变化，会调用[address_space_update_topology_pass](https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L890)->[MEMORY_LISTENER_UPDATE_REGION](https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L155)->[MEMORY_LISTENER_CALL](https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L130).

MEMORY_LISTENER_CALL里面调用所有的 listener. 但是，这个逻辑这里不会执行的. 这是因为这里内存处于初始化的阶段，全局的 flat_views 里面肯定找不到. 因而 [generate_memory_topology](https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L703) 第一次生成了 FlatView，然后才调用了 address_space_set_flatview. 这里面，老的 flatview 和新的 flatview 一定是一样的.

但是，请记住这个逻辑，到这里为止还没解析 qemu 有关内存的参数，所以这里添加的 MemoryRegion 虽然是一个根，但是是空的，是为了管理使用的，后面真的添加内存的时候，这个逻辑还会调用到.

再回到 qemu 启动的 qemu_init 函数中. 接下来的初始化过程会调用 pc_init1. 在这里面，对于 CPU 虚拟化，会调用 pc_init1. 另外，pc_init1 还会调用 [pc_memory_init](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/i386/pc.c#L936)，进行内存的虚拟化，这里解析这一部分.

在 pc_memory_init 中，因为已经知道了虚拟机要申请的内存 machine->ram_size，为了兼容过去的版本，分成两个 MemoryRegion 进行管理，一个是 ram_below_4g，一个是 ram_above_4g. 对于这两个 MemoryRegion，都会初始化一个 alias，也即别名，意思是说，两个 MemoryRegion 其实都指向分配到的内存，只不过分成两个部分，起两个别名指向不同的区域.

> memory_region_allocate_system_memory deleted in bd457782b3b0a313f3991038eb55bc44369c72c6 for "x86/pc: use memdev for RAM"

这两部分 MemoryRegion 都会调用 [memory_region_add_subregion](https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L2399)，将这两部分作为子的内存区域添加到 system_memory 这棵树上.

接下来的调用链为：memory_region_add_subregion->[memory_region_add_subregion_common](https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L2389)->[memory_region_update_container_subregions](https://elixir.bootlin.com/qemu/v5.0.0/source/memory.c#L2369).

在 memory_region_update_container_subregions 中，会将子区域放到链表中，然后调用 memory_region_transaction_commit. 在memory_region_transaction_commit里面，会调用 address_space_set_flatview. 因为内存区域变了，flatview 也会变，就像上面分析过的一样，listener 会被调用.

因为添加了一个 MemoryRegion，region_add 也即 [kvm_region_add](https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L1116).

kvm_region_add 调用的是 [kvm_set_phys_mem](https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L1026)，这里面分配一个用于放这块内存的 KVMSlot 结构，就像一个内存条一样，当然这是在用户态模拟出来的内存条，放在 KVMState 结构里面. 这个结构是创建虚拟机的时候创建的. 接下来，[kvm_set_user_memory_region](https://elixir.bootlin.com/qemu/v5.0.0/source/accel/kvm/kvm-all.c#L296) 就会将用户态模拟出来的内存条，和内核中的 KVM 模块关联起来.

终于，在kvm_set_user_memory_region里，又看到了可以和内核通信的 kvm_vm_ioctl. 来看内核收到 KVM_SET_USER_MEMORY_REGION 会做哪些事情.

接下来的调用链为：[kvm_vm_ioctl_set_memory_region](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1341)->[kvm_set_memory_region](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1329)->[__kvm_set_memory_region](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1211).

在用户态每个 KVMState 有多个 KVMSlot，在内核里面，同样每个 struct kvm 也有多个 struct [kvm_memory_slot](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/kvm_host.h#L341)，两者是对应起来的.

并且，[__kvm_set_memory_region](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1211)->[id_to_memslot](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1248) 函数可以根据用户态的 slot 号得到内核态的 slot 结构. 如果传进来的参数是 KVM_MR_CREATE，表示要创建一个新的内存条，就会调用 kvm_arch_create_memslot 来创建 kvm_memory_slot 的成员 kvm_arch_memory_slot.

接下来就是创建 kvm_memslots 结构，填充这个结构，然后通过 install_new_memslots 将这个新的内存条，添加到 struct kvm 结构中.

至此，用户态的内存结构和内核态的内存结构算是对应了起来.

### 页面分配和映射
上面对于内存的管理，还只是停留在元数据的管理. 对于内存的分配与映射，还没有涉及，接下来，就来看看，页面是如何进行分配和映射的. 上面说了，内存映射对于虚拟机来讲是一件非常麻烦的事情，从 GVA 到 GPA 到 HVA 到 HPA，性能很差，为了解决这个问题，有两种主要的思路.

### 1. 影子页表, **不推荐**
第一种方式就是软件的方式，影子页表  （Shadow Page Table）.

内存映射要通过页表来管理，页表地址应该放在 cr3 寄存器里面. 本来的过程是，客户机要通过 cr3 找到客户机的页表，实现从 GVA 到 GPA 的转换，然后在宿主机上，要通过 cr3 找到宿主机的页表，实现从 HVA 到 HPA 的转换.

为了实现客户机虚拟地址空间到宿主机物理地址空间的直接映射. 客户机中每个进程都有自己的虚拟地址空间，所以 KVM 需要为客户机中的每个进程页表都要维护一套相应的影子页表. 在客户机访问内存时，使用的不是客户机的原来的页表，而是这个页表对应的影子页表，从而实现了从客户机虚拟地址到宿主机物理地址的直接转换. 而且，在 TLB 和 CPU 缓存上缓存的是来自影子页表中客户机虚拟地址和宿主机物理地址之间的映射，也因此提高了缓存的效率.

内存的访问和更新通常是非常频繁的， 要维护影子页表中对应关系会非常复杂, 开销也较大. 同时影子页表的引入也意味着 KVM 需要为每个客户机的页表都要维护一套相应的影子页表，内存占用比较大，而且客户机页表和和影子页表也需要进行实时同步.

### 2. 扩展页表于
ref:
- [**内存虚拟化之基本原理**](https://www.codenong.com/cs106434119/)

第二种方式，就是硬件的方式，Intel 的 EPT（Extent Page Table，扩展页表）技术. EPT页表存放在VMM内核空间，VMM维护.

EPT转换过程: 通过客户机CR3寄存器将GVA转成GPA, 然后通过查询EPT来实现GPA转成HPA. EPT的控制权在VMM中, 只有当CPU工作在非根模式时才参与内存地址的转换.

使用EPT后， 客户机在读写CR3和执行INVLPG指令时不会导致VM Exit, 而且客户页表结构自身导致的页故障也不会导致VM Exit. 所以通过引入硬件上EPT的支持, 简化了内存虚拟化的实现复杂度, 同时也提高了内存地址转换的效率.

EPT 在原有客户机页表对客户机虚拟地址到客户机物理地址映射的基础上，又引入了 EPT 页表来实现客户机物理地址到宿主机物理地址的另一次映射. 客户机运行时，客户机页表被载入 CR3，而 EPT 页表被载入专门的 EPT 页表指针寄存器 EPTP.

有了 EPT，在客户机物理地址到宿主机物理地址转换的过程中，缺页会产生 EPT 缺页异常. KVM 首先根据引起异常的客户机物理地址，映射到对应的宿主机虚拟地址，然后为此虚拟地址分配新的物理页，最后 KVM 再更新 EPT 页表，建立起引起异常的客户机物理地址到宿主机物理地址之间的映射.

KVM 只需为每个客户机维护一套 EPT 页表，也大大减少了内存的开销.

这里，重点看第二种方式. 因为使用了 EPT 之后，客户机里面的页表映射，也即从 GVA 到 GPA 的转换，还是用传统的方式，和在linux kernel内存管理的没有什么区别. 而 EPT 重点帮我们解决的就是从 GPA 到 HPA 的转换问题. 因为要经过两次页表，所以 EPT 又称为 tdp（two dimentional paging）.

EPT 的页表结构也是分为四层，EPT Pointer （EPTP）指向 PML4 的首地址.
![](/misc/img/virt/02e4740398bc3685f366351260ae7230.jpg)

管理物理页面的 Page 结构和kernel的内存管理是一样的. EPT 页表也需要存放在一个页中，这些页要用 kvm_mmu_page 这个结构来管理. 当一个虚拟机运行，进入客户机模式的时候，它会调用 [vcpu_enter_guest](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/x86.c#L8309) 函数，它里面会调用 [kvm_mmu_reload](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu.h#L76)->[kvm_mmu_load](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L5150)

> set_cr3 deleted in 727a7e27cf88a261c5a0f14f4f9ee4d767352766 for "KVM: x86: rename set_cr3 callback and related flags to load_mmu_pgd"

kvm_mmu_load里构建的是页表的根部，也即顶级页表，并且通过[kvm_mmu_load_pgd](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu.h#L98)->`kvm_x86_ops.load_mmu_pgd`->[vmx_load_mmu_pgd](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmx.c#L3098)设置 cr3 来刷新 TLB. [mmu_alloc_roots](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L3817) 会调用 mmu_alloc_direct_roots，因为用的是 EPT 模式，而非影子表. 在 [mmu_alloc_direct_roots](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L3697) 中，[mmu_alloc_root](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L3679)->[kvm_mmu_get_page](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L2472) 会分配一个 [kvm_mmu_page](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/include/asm/kvm_host.h#L325)，来存放顶级页表项.

接下来，当虚拟机真的要访问内存的时候，会发现有的页表没有建立，有的物理页没有分配，这都会触发缺页异常，在 KVM 里面会发送 VM-Exit，从客户机模式转换为宿主机模式，来修复这个缺失的页表或者物理页.

前面讲过，[虚拟机退出客户机模式有很多种原因](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmx.c#L5672)，例如接收到中断、接收到 I/O 等，EPT 的缺页异常也是一种类型，称为 EXIT_REASON_EPT_VIOLATION，对应的处理函数是 [handle_ept_violation](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/vmx/vmx.c#L5273).

在 handle_ept_violation 里面，从 VMCS 中得到没有解析成功的 GPA by vmcs_read64，也即客户机的物理地址，然后调用 [kvm_mmu_page_fault](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L5405)，看为什么解析不成功. kvm_mmu_page_fault -> [kvm_mmu_do_page_fault](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu.h#L108) -> `vcpu->arch.mmu->page_fault == kvm_tdp_page_fault`，其实是 [kvm_tdp_page_fault](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L4193) 函数. tdp 的意思就是 EPT.

> Rename tdp_page_fault() to kvm_tdp_page_fault() in 7a02674d154d38da33517855b6d1d4cfc27a9a04 for "KVM: x86/mmu: Avoid retpoline on ->page_fault() with TDP"

既然没有映射，就应该加上映射，kvm_tdp_page_fault 就是干这个事情的.

在 kvm_tdp_page_fault -> [direct_page_fault](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L4095) 这个函数开头，通过 gpa，也即客户机的物理地址得到客户机的页号 gfn. 接下来，要通过调用 [try_async_pf](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L4062) 得到宿主机的物理地址对应的页号，也即真正的物理页的页号，然后通过 [__direct_map](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L3327) 将两者关联起来.

在 try_async_pf 中，要想得到 pfn，也即物理页的页号，会先通过 [kvm_vcpu_gfn_to_memslot](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1615)，根据客户机的物理地址对应的页号找到内存条，然后调用 [__gfn_to_pfn_memslot](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1929)，根据内存条找到 pfn.

在 __gfn_to_pfn_memslot 中，会调用 [__gfn_to_hva_many](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1658)，从客户机物理地址对应的页号，得到宿主机虚拟地址 hva，然后从宿主机虚拟地址到宿主机物理地址，调用的是 [hva_to_pfn](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1881). hva_to_pfn 会调用 [hva_to_pfn_fast](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1744)/[hva_to_pfn_slow](https://elixir.bootlin.com/linux/v5.8-rc4/source/virt/kvm/kvm_main.c#L1772). 最终都会得到一个物理页面，然后再调用 page_to_pfn 将物理页面转换成为物理页号.

> 为该 HVA 分配一个物理页，有 hva_to_pfn_fast 和 hva_to_pfn_slow 两种， hva_to_pfn_fast 实际上是调用 get_user_page_fast_only ，会尝试去 pin 该 page，即确保该地址所在的物理页在内存中. 如果失败，退化到 hva_to_pfn_slow ，会先去拿 mm->mmap_sem 的锁然后调用 get_user_page_fast_only 来 pin.

> 无论是hva_to_pfn_fast/hva_to_pfn_slow, 最终都会调用 get_user_page_fast_only-> [__gup_longterm_unlocked](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/gup.c#L2716)->[get_user_pages_unlocked](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/gup.c#L2065)->[__get_user_pages_locked](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/gup.c#L1272)->[__get_user_pages](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/gup.c#L1031)-> [faultin_page](https://elixir.bootlin.com/linux/v5.8-rc4/source/mm/gup.c#L862)，在 faultin_page 中会调用 handle_mm_fault来为虚拟机的物理内存空间分配真正的物理页面.

至此，try_async_pf 得到了物理页面，并且转换为对应的物理页号.

接下来，[__direct_map](https://elixir.bootlin.com/linux/v5.8-rc4/source/arch/x86/kvm/mmu/mmu.c#L3327) 会关联客户机物理页号和宿主机物理页号.

__direct_map 首先判断页表的根是否存在，当然存在，刚才初始化了. 接下来是 for_each_shadow_entry 一个循环.

每一个循环中，先是会判断需要映射的 level，是否正是当前循环的这个 it.level. 如果是，则说明是叶子节点，直接映射真正的物理页面 pfn，然后退出. 接着是非叶子节点的情形，判断如果这一项指向的页表项不存在，就要建立页表项，通过 kvm_mmu_get_page 得到保存页表项的页面，然后将这一项指向下一级的页表页面.

至此，内存映射就结束了.

### vm内存管理总结
![](/misc/img/virt/0186c533b7ef706df880dfd775c2449b.jpg)

## 存储虚拟化
存储使用完全虚拟化时，qemu 模拟的设备是一个翻译官的角色, 因为这个时候虚拟机里面的操作系统，意识不到自己是运行在虚拟机里面的，因此这种需要每个指令都翻译的方式，实在是太慢了.

因此有了半虚拟化. 即虚拟机里面的操作系统不是一个通用的操作系统，它知道自己是运行在虚拟机里面的，使用的硬盘设备和网络设备都是虚拟的，应该加载特殊的驱动才能运行. 这些特殊的驱动往往要通过虚拟机里面和外面配合工作的模式，来加速对于物理存储和网络设备的使用.

### virtio 的基本原理
virtio，即虚拟化 I/O 设备. virtio 负责对于虚拟机提供统一的接口. 也就是说，在虚拟机里面的操作系统加载的驱动，以后都统一加载 virtio 就可以了, 简化了驱动开发.

virtio 是对半虚拟化 hypervisor 中的一组通用模拟设备的抽象. Virtio设备本质上也是一个PCI设备.

virtio 的架构可以分为四层:
1. 首先，在虚拟机里面的 virtio 前端，针对不同类型的设备有不同的驱动程序，但是接口都是统一的. 例如，硬盘就是 virtio_blk，网络就是 virtio_net.

1. 其次，在宿主机的 qemu 里面，实现 virtio 后端的逻辑，主要就是操作硬件的设备. 例如在宿主机的 qemu 进程中，当收到客户机的写入请求的时候，调用文件系统的 write 函数，写入宿主机的 VFS 文件系统，最终写到物理硬盘设备上的 qcow2 文件. 再如向内核协议栈发送一个网络包完成虚拟机对于网络的操作.

1. 在 virtio 的前端和后端之间，有一个通信层，里面包含 virtio 层和 virtio-ring 层. virtio 这一层实现的是虚拟队列接口，算是前后端通信的桥梁. 而 virtio-ring 则是该桥梁的具体实现, 它实现了两个环形缓冲区，分别用于保存前端驱动程序和后端处理程序执行的信息.

![virtio 的架构](/misc/img/virt/2e9ef612f7b80ec9fcd91e200f4946f3.png)
![](/misc/img/virt/1f0c3043a11d6ea1a802f7d0f3b0b34b.png)

**vhost是一种 virtio 高性能的后端驱动实现, 是virtio的趋势**, 详情见[这里](/virt/vhost.md).

virtio 使用 virtqueue 进行前端和后端的高速通信. 不同类型的设备队列数目不同. virtio-net 使用两个队列，一个用于接收，另一个用于发送；而 virtio-blk 仅使用一个队列.

如果客户机要向宿主机发送数据，客户机会将数据的 buffer 添加到 virtqueue 中，然后通过写入寄存器通知宿主机. 这样宿主机就可以从 virtqueue 中收到的 buffer 里面的数据.

了解了 virtio 的基本原理，接下来，以硬盘写入为例，具体看一下存储虚拟化的过程.

```c
// https://elixir.bootlin.com/qemu/v5.0.0/source/hw/virtio/virtio.c#L3807
static const TypeInfo virtio_device_info = {
    .name = TYPE_VIRTIO_DEVICE,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(VirtIODevice),
    .class_init = virtio_device_class_init,
    .instance_finalize = virtio_device_instance_finalize,
    .abstract = true,
    .class_size = sizeof(VirtioDeviceClass),
};

static void virtio_register_types(void)
{
    type_register_static(&virtio_device_info);
}

type_init(virtio_register_types)

// https://elixir.bootlin.com/qemu/v5.0.0/source/hw/block/virtio-blk.c#L1316
static const TypeInfo virtio_blk_info = {
    .name = TYPE_VIRTIO_BLK,
    .parent = TYPE_VIRTIO_DEVICE,
    .instance_size = sizeof(VirtIOBlock),
    .instance_init = virtio_blk_instance_init,
    .class_init = virtio_blk_class_init,
};

static void virtio_register_types(void)
{
    type_register_static(&virtio_blk_info);
}

type_init(virtio_register_types)
```

Virtio Block Device 这种类的定义是有多层继承关系的. TYPE_VIRTIO_BLK 的父类是 TYPE_VIRTIO_DEVICE，TYPE_VIRTIO_DEVICE 的父类是 TYPE_DEVICE，TYPE_DEVICE 的父类是 TYPE_OBJECT, 到头了.

type_init 用于注册这种类. 这里面每一层都有 class_init，用于从 TypeImpl 生产 xxxClass. 还有 instance_init，可以将 xxxClass 初始化为实例.

在 TYPE_VIRTIO_BLK 层的 class_init 函数 virtio_blk_class_init 中，定义了 DeviceClass 的 realize 函数为 [virtio_blk_device_realize](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/block/virtio-blk.c#L1122)，这一点在虚拟化CPU那节也有类似的结构.

在 virtio_blk_device_realize 函数中，先是通过 [virtio_init](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/virtio/virtio.c#L3238) 初始化 VirtIODevice 结构.

从 virtio_init 中可以看出，VirtIODevice 结构里面有一个 VirtQueue 数组，这就是 virtio 前端和后端互相传数据的队列，最多 VIRTIO_QUEUE_MAX 个.

回到 virtio_blk_device_realize 函数. 接下来，根据配置的队列数目 num_queues，对于每个队列都调用 [virtio_add_queue](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/virtio/virtio.c#L2395) 来初始化队列.

在每个 VirtQueue 中，都有一个 vring，用来维护这个队列里面的数据；另外还有一个函数 virtio_blk_handle_output，用于处理数据写入，这个函数后面会用到.

至此，VirtIODevice，VirtQueue，vring 之间的关系如下图所示. 这是在 qemu 里面的对应关系，需记好，后面还能看到类似的结构.

![](/misc/img/virt/e18dae0a5951392c4a8e8630e53a616d.jpg)

### qemu 启动过程中的存储虚拟化
对于硬盘的虚拟化，qemu 的启动参数里面有关的是下面两行:
```bash
-drive file=/var/lib/nova/instances/1f8e6f7e-5a70-4780-89c1-464dc0e7f308/disk,if=none,id=drive-virtio-disk0,format=qcow2,cache=none
-device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1
```

其中，第一行指定了宿主机硬盘上的一个文件，文件的格式是 qcow2，这个格式这里不准备解析它，只要明白，对于宿主机上的一个文件，可以被 qemu 模拟称为客户机上的一块硬盘就可以了.

而第二行说明了，使用的驱动是 virtio-blk 驱动.

在 qemu 启动的 qemu_init 函数里面，初始化块设备，是通过 [configure_blockdev](https://elixir.bootlin.com/qemu/v5.0.0/source/softmmu/vl.c#L1028) 调用开始的.

在 configure_blockdev 中，能看到对于 drive 这个参数的解析，并且初始化这个设备要调用 [drive_init_func](https://elixir.bootlin.com/qemu/v5.0.0/source/softmmu/vl.c#L985) 函数，它里面会调用 drive_new 创建一个设备.

在 [drive_new](https://elixir.bootlin.com/qemu/v5.0.0/source/blockdev.c#L760) 里面，会解析 qemu 的启动参数. 对于 virtio 来讲，会解析 device 参数，把 driver 设置为 virtio-blk-pci；还会解析 file 参数，就是指向那个宿主机上的文件.

接下来，drive_new 会调用 blockdev_init，根据参数进行初始化，最后会创建一个 DriveInfo 来管理这个设备.

重点来看 [blockdev_init](https://elixir.bootlin.com/qemu/v5.0.0/source/blockdev.c#L460). 在这里面会发现，如果 file 不为空，则应该调用 [blk_new_open](https://elixir.bootlin.com/qemu/v5.0.0/source/block/block-backend.c#L371) 打开宿主机上的硬盘文件，返回的结果是 BlockBackend，对应上面讲原理的时候的 virtio 的后端.

接下来的调用链为：[bdrv_open](https://elixir.bootlin.com/qemu/v5.0.0/source/block.c#L3381)->[bdrv_open_inherit](https://elixir.bootlin.com/qemu/v5.0.0/source/block.c#L3112)->[bdrv_open_common](https://elixir.bootlin.com/qemu/v5.0.0/source/block.c#L1621).

在 bdrv_open_common 中，根据硬盘文件的格式，得到 BlockDriver. 因为虚拟机的硬盘文件格式有很多种，qcow2 是一种，raw 是一种，vmdk 是一种，各有优缺点，启动虚拟机的时候，可以自由选择. 对于不同的格式，打开的方式不一样，拿 qcow2 来解析, 它的 BlockDriver 实现是[bdrv_qcow2](https://elixir.bootlin.com/qemu/v5.0.0/source/block/qcow2.c#L5531).

根据上面的定义，对于 qcow2 来讲，bdrv_open 调用的是 qcow2_open.

在 qcow2_open 中，会通过 [qemu_coroutine_enter](https://elixir.bootlin.com/qemu/v5.0.0/source/util/qemu-coroutine.c#L168) 进入一个协程 coroutine. 什么叫协程呢？可以简单地将它理解为用户态自己实现的线程.

学线程的时候学过，如果一个程序想实现并发，可以创建多个线程，但是线程是一个内核的概念，创建的每一个线程内核都能看到，内核的调度也是以线程为单位的. 这对于普通的进程没有什么问题，但是对于 qemu 这种虚拟机，如果在用户态和内核态切换来切换去，由于还涉及虚拟机的状态，代价比较大.

但是，qemu 的设备也是需要多线程能力的，怎么办呢？此时就需要在用户态实现一个类似线程的东西，也就是协程，用于实现并发，并且不被内核看到，调度全部在用户态完成.

从后面的读写过程可以看出，协程在后端经常使用. 这里打开一个 qcow2 文件就是使用一个协程，创建一个协程和创建一个线程很像，也需要指定一个函数来执行，[qcow2_open_entry](https://elixir.bootlin.com/qemu/v5.0.0/source/block/qcow2.c#L1790) 就是协程的函数.

可以看到，qcow2_open_entry 函数前面有一个 coroutine_fn，说明它是一个协程函数. 在 qcow2_do_open 中，qcow2_do_open 根据 qcow2 的格式打开硬盘文件. 这个格式[官网](https://github.com/qemu/qemu/blob/master/docs/interop/qcow2.txt)就有，这里就不解析了.

### 前端设备驱动 virtio_blk
虚拟机里面的进程写入一个文件，当然要通过文件系统. 整个过程和在无虚拟化写文件的过程没有区别. 只是到了设备驱动层，看到的就不是普通的硬盘驱动了，而是 virtio 的驱动.

```c
// https://elixir.bootlin.com/linux/latest/source/drivers/block/virtio_blk.c#L971
static struct virtio_driver virtio_blk = {
	.feature_table			= features,
	.feature_table_size		= ARRAY_SIZE(features),
	.feature_table_legacy		= features_legacy,
	.feature_table_size_legacy	= ARRAY_SIZE(features_legacy),
	.driver.name			= KBUILD_MODNAME,
	.driver.owner			= THIS_MODULE,
	.id_table			= id_table,
	.probe				= virtblk_probe,
	.remove				= virtblk_remove,
	.config_changed			= virtblk_config_changed,
#ifdef CONFIG_PM_SLEEP
	.freeze				= virtblk_freeze,
	.restore			= virtblk_restore,
#endif
};

static int __init init(void)
{
	int error;

	virtblk_wq = alloc_workqueue("virtio-blk", 0, 0);
	if (!virtblk_wq)
		return -ENOMEM;

	major = register_blkdev(0, "virtblk");
	if (major < 0) {
		error = major;
		goto out_destroy_workqueue;
	}

	error = register_virtio_driver(&virtio_blk);
	if (error)
		goto out_unregister_blkdev;
	return 0;

out_unregister_blkdev:
	unregister_blkdev(major, "virtblk");
out_destroy_workqueue:
	destroy_workqueue(virtblk_wq);
	return error;
}

static void __exit fini(void)
{
	unregister_virtio_driver(&virtio_blk);
	unregister_blkdev(major, "virtblk");
	destroy_workqueue(virtblk_wq);
}
module_init(init);
module_exit(fini);

MODULE_DEVICE_TABLE(virtio, id_table);
MODULE_DESCRIPTION("Virtio block driver");
MODULE_LICENSE("GPL");
```

virtio 的驱动程序代码在 Linux 操作系统的源代码里面，文件名叫 [drivers/block/virtio_blk.c](https://elixir.bootlin.com/linux/latest/source/drivers/block/virtio_blk.c).

从这里的代码中，能看到非常熟悉的结构. 它会创建一个 workqueue，注册一个块设备，并获得一个主设备号，然后注册一个驱动函数 virtio_blk. 当一个设备驱动作为一个内核模块被初始化的时候，probe 函数会被调用，因而来看一下 [virtblk_probe](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/block/virtio_blk.c#L684).

在 virtblk_probe 中，首先看到的是 struct request_queue，这是每一个块设备都有的一个队列. 还记得吗？它有两个函数，一个是 make_request_fn 函数，用于生成 request；另一个是 request_fn 函数，用于处理 request.

这个 request_queue 的初始化过程在 [blk_mq_init_queue](https://elixir.bootlin.com/linux/v5.8-rc4/source/block/blk-mq.c#L2906) 中. blk_mq_init_queue -> blk_mq_init_queue_data -> [blk_mq_init_allocated_queue](https://elixir.bootlin.com/linux/v5.8-rc4/source/block/blk-mq.c#L3057). 也就是说，一旦上层有写入请求，就通过 blk_mq_make_request 这个函数，将请求放入 request_queue 队列中.

另外，在 virtblk_probe 中，会初始化一个 gendisk. 每一个块设备都有这样一个结构. 在 virtblk_probe 中，还有一件重要的事情就是，init_vq 会来初始化 virtqueue.

按照上面的原理来说，virtqueue 是一个介于客户机前端和 qemu 后端的一个结构，用于在这两端之间传递数据. 这里建立的 struct virtqueue 是客户机前端对于队列的管理的数据结构，在客户机的 linux 内核中通过 kmalloc_array 进行分配. 而队列的实体需要通过函数 [virtio_find_vqs](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/virtio_config.h#L192) 查找或者生成，所以这里还把 callback 函数指定为 [virtblk_done](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/block/virtio_blk.c#L159). 当 buffer 使用发生变化的时候，就需要调用这个 callback 函数进行通知.

根据 virtio_config_ops 的定义，virtio_find_vqs在这里是[virtio_pci_config_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_pci_modern.c#L463), 它会调用 vp_modern_find_vqs. 

在 [vp_modern_find_vqs](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_pci_modern.c#L403) 中，vp_find_vqs 会调用 vp_find_vqs_intx.

在 vp_find_vqs_intx 中，通过 request_irq 注册一个中断处理函数 vp_interrupt，当设备的配置信息发生改变，会产生一个中断，当设备向队列中写入信息时，也会产生一个中断，称为 vq 中断，中断处理函数需要调用相应的队列的回调函数. 然后，根据队列的数目，依次调用 vp_setup_vq，完成 virtqueue、vring 的分配和初始化.

在 vring_create_virtqueue 中，会调用 vring_alloc_queue，来创建队列所需要的内存空间，然后调用 vring_init 初始化结构 struct vring，来管理队列的内存空间，调用 __vring_new_virtqueue，来创建 [struct vring_virtqueue](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_ring.c#L87).

vring_virtqueue结构的一开始，是 [struct virtqueue](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/virtio.h#L27)，它也是 struct virtqueue 的一个扩展, 包含了 [struct vring](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/uapi/linux/virtio_ring.h#L152).

至此发现，虚拟机里面的 virtio 的前端是这样的结构：struct virtio_device 里面有一个 struct vring_virtqueue，在 struct vring_virtqueue 里面有一个 struct vring.

### 中间 virtio 队列的管理
参考:
- [virtio之vring](https://www.cnblogs.com/yi-mu-xi/p/12544695.html)

qemu 初始化的时候，virtio 的后端有数据结构 VirtIODevice，VirtQueue 和 vring 一模一样，前端和后端对应起来，都应该指向刚才创建的那一段内存.
 
现在的问题是，刚才分配的内存在客户机的内核里面，如何告知 qemu 来访问这段内存呢？别忘了，qemu 模拟出来的 virtio block device 只是一个 PCI 设备. 对于客户机来讲，这是一个外部设备，可以通过给外部设备发送指令的方式告知外部设备，这就是代码中 vp_iowrite16 的作用. 它会调用专门给外部设备发送指令的函数 iowrite，告诉外部的 PCI 设备.
 
告知的有三个地址 virtqueue_get_desc_addr、virtqueue_get_avail_addr，virtqueue_get_used_addr. 从客户机角度来看，这里面的地址都是物理地址，也即 GPA（Guest Physical Address）. 因为只有物理地址才是客户机和 qemu 程序都认可的地址，本来客户机的物理内存也是 qemu 模拟出来的.
 
在 qemu 中，对 PCI 总线添加一个设备的时候，会调用 [virtio_pci_device_plugged](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio-pci.c#L1532).

在virtio_pci_device_plugged里面，对于这个加载的设备进行 I/O 操作，会映射到读写某一块内存空间，对应的操作为 [virtio_pci_config_ops](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio-pci.c#L482)，也即写入这块内存空间，这就相当于对于这个 PCI 设备进行某种配置.
 
对 PCI 设备进行配置的时候，会有这样的调用链：[virtio_pci_config_write](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio-pci.c#L448)->[virtio_ioport_write](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio-pci.c#L295)->[virtio_queue_set_addr](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio.c#L2222). 设置 virtio 的 queue 的地址是一项很重要的操作.

从这里我们可以看出，qemu 后端的 VirtIODevice 的 VirtQueue 的 vring 的地址，被设置成了刚才给队列分配的内存的 GPA.

![](/misc/img/virt/2572f8b1e75b9eaab6560866fcb31fd0.jpg)

接着，来看一下这个队列的格式.

![](/misc/img/virt/49414d5acc81933b66410bbba102b0db.jpg)
```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/include/uapi/linux/virtio_ring.h#L97
/* Virtio ring descriptors: 16 bytes.  These can chain together via "next". */
struct vring_desc {
	/* Address (guest-physical). */
	__virtio64 addr;
	/* Length. */
	__virtio32 len;
	/* The flags as indicated above. */
	__virtio16 flags;
	/* We chain unused descriptors via this, too */
	__virtio16 next;
};

struct vring_avail {
	__virtio16 flags;
	__virtio16 idx;
	__virtio16 ring[];
};

/* u32 is used here for ids for padding reasons. */
struct vring_used_elem {
	/* Index of start of used descriptor chain. */
	__virtio32 id;
	/* Total length of the descriptor chain which was used (written to) */
	__virtio32 len;
};

typedef struct vring_used_elem __attribute__((aligned(VRING_USED_ALIGN_SIZE)))
	vring_used_elem_t;

struct vring_used {
	__virtio16 flags;
	__virtio16 idx;
	vring_used_elem_t ring[];
};

/*
 * The ring element addresses are passed between components with different
 * alignments assumptions. Thus, we might need to decrease the compiler-selected
 * alignment, and so must use a typedef to make sure the aligned attribute
 * actually takes hold:
 *
 * https://gcc.gnu.org/onlinedocs//gcc/Common-Type-Attributes.html#Common-Type-Attributes
 *
 * When used on a struct, or struct member, the aligned attribute can only
 * increase the alignment; in order to decrease it, the packed attribute must
 * be specified as well. When used as part of a typedef, the aligned attribute
 * can both increase and decrease alignment, and specifying the packed
 * attribute generates a warning.
 */
typedef struct vring_desc __attribute__((aligned(VRING_DESC_ALIGN_SIZE)))
	vring_desc_t;
typedef struct vring_avail __attribute__((aligned(VRING_AVAIL_ALIGN_SIZE)))
	vring_avail_t;
typedef struct vring_used __attribute__((aligned(VRING_USED_ALIGN_SIZE)))
	vring_used_t;

struct vring {
	unsigned int num;

	vring_desc_t *desc;

	vring_avail_t *avail;

	vring_used_t *used;
};
```

vring 包含三个成员：
- vring_desc 指向分配的内存块，用于存放客户机和 qemu 之间传输的数据
- avail->ring[]是发送端维护的环形队列，指向需要接收端处理的 vring_desc
- used->ring[]是接收端维护的环形队列，指向自己已经处理过了的 vring_desc

### 数据写入的流程
接下来，来看，真的写入一个数据的时候，会发生什么.

按照上面 virtio 驱动初始化的时候的逻辑，[blk_mq_make_request](https://elixir.bootlin.com/linux/v5.8-rc4/source/block/blk-mq.c#L2023) 会被调用. 这个函数比较复杂，会分成多个分支，但是最终都会调用到 request_queue 的  queue_rq 函数即[virtio_mq_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/block/virtio_blk.c#L673) 的[virtio_queue_rq](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/block/virtio_blk.c#L201).

在 virtio_queue_rq 中，会将请求写入的数据，通过 [virtblk_add_req](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/block/virtio_blk.c#L92) 放入 struct virtqueue. 因此，接下来的调用链为：virtblk_add_req->[virtqueue_add_sgs](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_ring.c#L1724)->[virtqueue_add](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_ring.c#L1693)->[](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_ring.c#L1091).

> virtio1.1(最新, 2020.7) 关键的最大改动点就是引入了packed queue，也就是将virtio1.0中的desc ring，avail ring，used ring三个ring打包成一个desc ring了, 这样元数据读取的软件实现可以减少 cache miss，硬件实现可以只用一个 PCI transaction 解决. 相对应的，将virtio 1.0这种实现方式称之为split ring. 

**下面算法描述是旧代码的描述**.

在 virtqueue_add_packed 函数中，能看到，free_head 指向的整个内存块空闲链表的起始位置，用 head 变量记住这个起始位置. 接下来，i 也指向这个起始位置，然后是一个 for 循环，将数据放到内存块里面，放的过程中，next 不断指向下一个空闲位置，这样空闲的内存块被不断的占用. 等所有的写入都结束了，i 就会指向这次存放的内存块的下一个空闲位置，然后 free_head 就指向 i，因为前面的都填满了.

至此，从 head 到 i 之间的内存块，就是这次写入的全部数据. 于是，在 vring 的 avail 变量中，在 ring[]数组中分配新的一项，在 avail 的位置，avail 的计算是 avail_idx_shadow & (vq->vring.num - 1)，其中，avail_idx_shadow 是上一次的 avail 的位置. 这里如果超过了 ring[]数组的下标，则重新跳到起始位置，就说明是一个环. 这次分配的新的 avail 的位置就存放新写入的从 head 到 i 之间的内存块. 然后是 avail_idx_shadow++，这说明这一块内存可以被接收方读取了.

接下来，回到 virtio_queue_rq，调用 [virtqueue_notify](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_ring.c#L1841) 通知接收方. 而 virtqueue_notify 会调用 vp->notify即[vp_notify](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_pci_common.c#L41), vp由[vring_create_virtqueue创建](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_pci_modern.c#L342).

然后，写入一个 I/O 会触发 VM exit. 在解析 CPU 的时候看到过这个逻辑在[kvm_cpu_exec](https://elixir.bootlin.com/qemu/latest/source/accel/kvm/kvm-all.c#L2307)的`case KVM_EXIT_IO`分支由[kvm_handle_io](https://elixir.bootlin.com/qemu/latest/source/accel/kvm/kvm-all.c#L2133)处理.

kvm_handle_io写入的也是一个 I/O 的内存空间，同样会触发 virtio_ioport_write，这次会调用 [address_space_rw](https://elixir.bootlin.com/qemu/latest/source/exec.c#L3274)->[address_space_write](https://elixir.bootlin.com/qemu/latest/source/exec.c#L3258)->[flatview_write](https://elixir.bootlin.com/qemu/latest/source/exec.c#L3167)->[flatview_write_continue](https://elixir.bootlin.com/qemu/latest/source/exec.c#L3118)->[memory_region_dispatch_write](https://elixir.bootlin.com/qemu/latest/source/memory.c#L1455)->[memory_region_write_accessor](https://elixir.bootlin.com/qemu/latest/source/memory.c#L467)的`mr->ops->write`, 此时ops是[notify_ops](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio-pci.c#L1422)/[notify_pio_ops](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio-pci.c#L1431)->[virtio_queue_notify](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio.c#L2352).

> notify_pio_ops/notify_ops的区别在[这里](https://lists.gnu.org/archive/html/qemu-devel/2015-08/msg02411.html). I/O作为CPU和外设交流的一个渠道，主要分为两种，一种是Port I/O即pio，一种是MMIO(Memory mapping I/O), MMIO更普遍.

virtio_queue_notify 会调用 VirtQueue 的 handle_output 函数，前面已经设置过这个函数了，是 [virtio_blk_handle_output](https://elixir.bootlin.com/qemu/latest/source/hw/block/virtio-blk.c#L806). 接下来的调用链为：virtio_blk_handle_output->[virtio_blk_handle_output_do](https://elixir.bootlin.com/qemu/latest/source/hw/block/virtio-blk.c#L801)->[virtio_blk_handle_vq](https://elixir.bootlin.com/qemu/latest/source/hw/block/virtio-blk.c#L763)

在 virtio_blk_handle_vq 中，有一个 while 循环，在循环中调用函数 virtio_blk_get_request 从 vq 中取出请求，然后调用 [virtio_blk_handle_request](https://elixir.bootlin.com/qemu/latest/source/hw/block/virtio-blk.c#L614) 处理从 vq 中取出的请求. 先来看 [virtio_blk_get_request](https://elixir.bootlin.com/qemu/latest/source/hw/block/virtio-blk.c#L614).

可以看到，virtio_blk_get_request 会调用 [virtqueue_pop](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio.c#L1684). 在virtqueue_pop里面，能看到对于 vring 的操作，也即[从这里面将客户机里面写入的数据读取出来，放到 VirtIOBlockReq 结构中](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio.c#L1549). 接下来，就要调用 virtio_blk_handle_request 处理这些数据.

所以接下来的调用链为：virtio_blk_handle_request->[virtio_blk_submit_multireq](https://elixir.bootlin.com/qemu/latest/source/hw/block/virtio-blk.c#L447)->[submit_requests](https://elixir.bootlin.com/qemu/latest/source/hw/block/virtio-blk.c#L384).

在 submit_requests 中，看到了 BlockBackend. 这是在 qemu 启动的时候，打开 qcow2 文件的时候生成的，现在可以用它来写入文件了，调用的是 [blk_aio_pwritev](https://elixir.bootlin.com/qemu/latest/source/block/block-backend.c#L1507).

在 blk_aio_pwritev 中，看到，又是创建了一个协程来进行写入. 写入完毕之后调用 [virtio_blk_rw_complete](https://elixir.bootlin.com/qemu/latest/source/hw/block/virtio-blk.c#L115)即blk_aio_pwritev传入的BlockCompletionFunc->[virtio_blk_req_complete](https://elixir.bootlin.com/qemu/latest/source/hw/block/virtio-blk.c#L75).

在 virtio_blk_req_complete 中，先是调用 virtqueue_push，更新 vring 中 used 变量，表示这部分已经写入完毕，空间可以回收利用了. 但是，这部分的改变仅仅改变了 qemu 后端的 vring，我们还需要通知客户机中 virtio 前端的 vring 的值，因而要调用 virtio_notify. virtio_notify 会调用 virtio_irq 发送一个中断. 还记得前面注册过一个中断处理函数 vp_interrupt 吗？它就是干这个事情的.

就像前面说的一样 vp_interrupt 这个中断处理函数，一是处理配置变化，二是处理 I/O 结束. 第二种的调用链为：vp_interrupt->vp_vring_interrupt->vring_interrupt.

在 vring_interrupt 中，会调用 callback 函数，这个也是在前面注册过的，是 virtblk_done. 接下来的调用链为：virtblk_done->virtqueue_get_buf->virtqueue_get_buf_ctx.

在 virtqueue_get_buf_ctx 中，可以看到，virtio 前端的 vring 中的 last_used_idx 加一，说明这块数据 qemu 后端已经消费完毕. 可以通过 detach_buf 将其放入空闲队列中，留给以后的写入请求使用.

至此，整个存储虚拟化的写入流程才全部完成.

### 存储虚拟化总结
存储虚拟化的场景下，整个写入的过程:
1. 在虚拟机里面，应用层调用 write 系统调用写入文件
1. write 系统调用进入虚拟机里面的内核，经过 VFS，通用块设备层，I/O 调度层，到达块设备驱动
1. 虚拟机里面的块设备驱动是 virtio_blk，它和通用的块设备驱动一样，有一个 request  queue，另外有一个函数 make_request_fn 会被设置为 blk_mq_make_request，这个函数用于将请求放入队列.
1. 虚拟机里面的块设备驱动是 virtio_blk 会注册一个中断处理函数 vp_interrupt. 当 qemu 写入完成之后，它会通知虚拟机里面的块设备驱动
1. blk_mq_make_request 最终调用 virtqueue_add，将请求添加到传输队列 virtqueue 中，然后调用 virtqueue_notify 通知 qemu
1. 在 qemu 中，本来虚拟机正处于 KVM_RUN 的状态，也即处于客户机状态
1. qemu 收到通知后，通过 VM exit 指令退出客户机状态，进入宿主机状态，根据退出原因，得知有 I/O 需要处理
1. qemu 调用 virtio_blk_handle_output，最终调用 virtio_blk_handle_vq
1. virtio_blk_handle_vq 里面有一个循环，在循环中，virtio_blk_get_request 函数从传输队列中拿出请求，然后调用 virtio_blk_handle_request 处理请求
1. virtio_blk_handle_request 会调用 blk_aio_pwritev，通过 BlockBackend 驱动写入 qcow2 文件
1. 写入完毕之后，virtio_blk_req_complete 会调用 virtio_notify 通知虚拟机里面的驱动. 数据写入完成，刚才注册的中断处理函数 vp_interrupt 会收到这个通知.

![](/misc/img/virt/79ad143a3149ea36bc80219940d7d00c.jpg)

## 网络虚拟化
Virtio网络设备是一种虚拟的以太网卡，支持多队列的网络包收发.

virtio标准将其对于队列的抽象称为Virtqueue. Vring即是对Virtqueue的具体实现. 一个Virtqueue由一个Available Ring和Used Ring组成, 前者用于前端向后端发送数据，而后者反之. 而在virtio网络中的TX/RX Queue均由一个Virtqueue实现. 所有的I/O通信架构都有数据平面与控制平面之分.
### 解析初始化过程
还是从 Virtio Network Device 这个设备的初始化入手.
```c
// https://elixir.bootlin.com/qemu/latest/source/hw/net/virtio-net.c#L3259
static const TypeInfo virtio_net_info = {
    .name = TYPE_VIRTIO_NET,
    .parent = TYPE_VIRTIO_DEVICE,
    .instance_size = sizeof(VirtIONet),
    .instance_init = virtio_net_instance_init,
    .class_init = virtio_net_class_init,
};

static void virtio_register_types(void)
{
    type_register_static(&virtio_net_info);
}

type_init(virtio_register_types)
```

Virtio Network Device 这种类的定义是有多层继承关系的，TYPE_VIRTIO_NET 的父类是 TYPE_VIRTIO_DEVICE，TYPE_VIRTIO_DEVICE 的父类是 TYPE_DEVICE，TYPE_DEVICE 的父类是 TYPE_OBJECT，继承关系到头了.

type_init 用于注册这种类. 这里面每一层都有 class_init，用于从 TypeImpl 生成 xxxClass，也有 instance_init，会将 xxxClass 初始化为实例.

TYPE_VIRTIO_NET 层的 class_init 函数 virtio_net_class_init，定义了 DeviceClass 的 realize 函数为 [virtio_net_device_realize](https://elixir.bootlin.com/qemu/latest/source/hw/net/virtio-net.c#L2932)，这一点和存储块设备是一样的.

virtio_net_device_realize里面创建了一个 VirtIODevice，这一点和存储虚拟化也是一样的. 再用virtio_init 用来初始化这个设备. VirtIODevice 结构里面有一个 VirtQueue 数组，这就是 virtio 前端和后端互相传数据的队列，最多有 VIRTIO_QUEUE_MAX 个. 留意一下会发现，这里面有这样的语句 n->max_queues * 2 + 1 > VIRTIO_QUEUE_MAX. 为什么要乘以 2 呢？这是因为，对于网络设备来讲，应该分发送队列和接收队列两个方向，所以乘以 2.

接下来，调用 virtio_net_add_queue 来初始化队列，可以看出来，这里面就有发送 tx_vq 和接收 rx_vq 两个队列.

每个 VirtQueue 中，都有一个 vring 用来维护这个队列里面的数据；另外还有函数 virtio_net_handle_rx 用于处理网络包的接收；函数 virtio_net_handle_tx_bh 用于网络包的发送，这两函数后面会用到.

接下来，[qemu_new_nic](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L277) 会创建一个虚拟机里面的网卡.

### qemu 的启动过程中的网络虚拟化
初始化过程解析完毕以后，接下来从 qemu 的启动过程看起, 对于网卡的虚拟化，qemu 的启动参数里面有关的是下面两行：
```txt
-netdev tap,fd=32,id=hostnet0,vhost=on,vhostfd=37
-device virtio-net-pci,netdev=hostnet0,id=net0,mac=fa:16:3e:d1:2d:99,bus=pci.0,addr=0x3
```

qemu 的 qemu_init 函数会调用 [net_init_clients](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L1542) 进行网络设备的初始化，可以解析 net 参数，也可以在 net_init_clients 中解析 netdev/nic 参数.

net_init_clients 会解析参数netdev 会调用 [net_init_netdev](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L1474)->[net_client_init](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L1108)->[net_client_init1](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L968). net_client_init1 会[根据不同的 driver 类型](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L939)，调用不同的初始化函数.

由于配置的 driver 的类型是 tap，因而这里会调用 [net_init_tap](https://elixir.bootlin.com/qemu/latest/source/net/tap.c#L758)->[net_tap_init](https://elixir.bootlin.com/qemu/latest/source/net/tap.c#L609)->[tap_open](https://elixir.bootlin.com/qemu/latest/source/net/tap-linux.c#L41).

在 tap_open 中，打开一个文件"/dev/net/tun"，然后通过 ioctl 操作这个文件. 这是 Linux 内核的一项机制，和 KVM 机制很像. 其实这就是一种通过打开这个字符设备文件，然后通过 ioctl 操作这个文件和内核打交道，来使用内核的能力.

![](/misc/img/virt/243e93913b18c3ab00be5676bef334d3.png)

为什么需要使用内核的机制呢？因为网络包需要从虚拟机里面发送到虚拟机外面，发送到宿主机上的时候，必须是一个正常的网络包才能被转发. 要形成一个网络包，那就需要经过复杂的协议栈.

客户机会将网络包发送给 qemu. qemu 自己没有网络协议栈，现去实现一个也不可能，太复杂了. 于是，它就要借助内核的力量.

qemu 会将客户机发送给它的网络包，然后转换成为文件流，写入"/dev/net/tun"字符设备. 就像写一个文件一样。内核中 TUN/TAP 字符设备驱动会收到这个写入的文件流，然后交给 TUN/TAP 的虚拟网卡驱动. 这个驱动会将文件流再次转成网络包，交给 TCP/IP 栈，最终从虚拟 TAP 网卡 tap0 发出来，成为标准的网络包. 后面会看到这个过程.

现在到内核里面，看一看打开"/dev/net/tun"字符设备后，内核会发生什么事情. 内核的实现在 [drivers/net/tun.c](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/tun.c) 文件中. 这是一个字符设备驱动程序，应该符合字符设备的格式.

这里面注册了一个 tun_miscdev 字符设备，从它的定义可以看出，这就是"/dev/net/tun"字符设备.

```c
// https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/tun.c
static const struct file_operations tun_fops = {
	.owner	= THIS_MODULE,
	.llseek = no_llseek,
	.read_iter  = tun_chr_read_iter,
	.write_iter = tun_chr_write_iter,
	.poll	= tun_chr_poll,
	.unlocked_ioctl	= tun_chr_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = tun_chr_compat_ioctl,
#endif
	.open	= tun_chr_open,
	.release = tun_chr_close,
	.fasync = tun_chr_fasync,
#ifdef CONFIG_PROC_FS
	.show_fdinfo = tun_chr_show_fdinfo,
#endif
};

static struct miscdevice tun_miscdev = {
	.minor = TUN_MINOR,
	.name = "tun",
	.nodename = "net/tun",
	.fops = &tun_fops,
};
```

qemu 的 tap_open 函数会打开这个字符设备 PATH_NET_TUN. 该过程到了驱动这一层，调用的是 tun_chr_open.

在 tun_chr_open 的参数里面，有一个 struct file，这是代表什么文件呢？它代表的就是打开的字符设备文件"/dev/net/tun"，因而往这个字符设备文件中写数据，就会通过这个 struct file 写入. 这个 struct file 里面的 file_operations，按照字符设备打开的规则，指向的就是 tun_fops.

另外，还需要在 tun_chr_open 创建一个结构 struct tun_file，并且将 struct file 的 private_data 指向它.

在 struct tun_file 中，有一个成员 struct tun_struct，它里面有一个 struct net_device，这个用来表示宿主机上的 tuntap 网络设备. 在 struct tun_file 中，还有 struct socket 和 struct sock，因为要用到内核的网络协议栈，所以就需要这两个结构.

所以，按照 struct tun_file 的注释说的，这是一个很重要的数据结构. "/dev/net/tun"对应的 struct file 的 private_data 指向它，因而可以接收 qemu 发过来的数据. 除此之外，它还可以通过 struct sock 来操作内核协议栈，然后将网络包从宿主机上的 tuntap 网络设备发出去，宿主机上的 tuntap 网络设备对应的 struct net_device 也归它管.

在 qemu 的 tap_open 函数中，打开这个字符设备文件之后，接下来要做的事情是，通过 ioctl 来设置宿主机的网卡 TUNSETIFF.

接下来，ioctl 到了内核里面，会调用 tun_chr_ioctl->[__tun_chr_ioctl](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/tun.c#L3004).

在 __tun_chr_ioctl 中，首先通过 copy_from_user 把配置从用户态拷贝到内核态，调用 tun_set_iff 设置 tuntap 网络设备，然后调用 copy_to_user 将配置结果返回.

tun_set_iff 创建了 struct tun_struct 和 struct net_device，并且将这个 tuntap 网络设备通过 register_netdevice 注册到内核中. 这样，就能在宿主机上通过 ip addr 看到这个网卡了.

![](/misc/img/virt/9826223c7375bec19bd13588f3875ffd.png)

至此宿主机上的内核的数据结构就完成了.

### 关联前端设备驱动和后端设备驱动
接下来, 来解析在客户机中发送一个网络包的时候，会发生哪些事情.

虚拟机里面的进程发送一个网络包，通过文件系统和 Socket 调用网络协议栈，到达网络设备层. 只不过这个不是普通的网络设备，而是 virtio_net 的驱动.

virtio_net 的驱动程序代码在 Linux 操作系统的源代码里面，文件名为 [drivers/net/virtio_net.c](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c).

在 virtio_net 的驱动程序的初始化代码中，需要注册一个驱动函数 [virtio_net_driver](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c#L3269).

当一个设备驱动作为一个内核模块被初始化的时候，probe 函数会被调用，因而来看一下 [virtnet_probe](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c#L2965).

在 virtnet_probe 中，会创建 [struct net_device](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/netdevice.h#L1843)，并且[通过 register_netdev 注册这个网络设备](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c#L3129)，这样在客户机里面，就能看到这个网卡了. 在 virtnet_probe 中，还有一件重要的事情就是，[init_vqs](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c#L3108) 会初始化发送和接收的 virtqueue.

按照前面的 virtio 原理，virtqueue 是一个介于客户机前端和 qemu 后端的一个结构，用于在这两端之间传递数据，对于网络设备来讲有发送和接收两个方向的队列.

这里建立的 struct virtqueue 是客户机前端对于队列的管理的数据结构. [队列的实体需要通过函数 virtnet_find_vqs 查找或者生成，它还会指定接收队列的 callback 函数为 skb_recv_done](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c#L2726)，发送队列的 callback 函数为 skb_xmit_done. 那当 buffer 使用发生变化的时候，就可以调用这个 callback 函数进行通知.

virtnet_find_vqs里的 find_vqs 是在 struct virtnet_info 里的 struct virtio_device 里的 [struct virtio_config_ops](https://elixir.bootlin.com/linux/v5.8-rc4/source/include/linux/virtio_config.h#L70) *config 里面定义的. 根据 virtio_config_ops 的定义，find_vqs 会调用 [vp_modern_find_vqs](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_pci_modern.c#L447)，到这一步和块设备是一样的了.

在 vp_modern_find_vqs 中，[vp_find_vqs](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_pci_common.c#L392) 会调用 [vp_find_vqs_intx](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_pci_common.c#L353). 在 vp_find_vqs_intx 中，通过 request_irq 注册一个中断处理函数 [vp_interrupt](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_pci_common.c#L82). 当设备向队列中写入信息时，会产生一个中断，也就是 vq 中断. 中断处理函数需要调用相应的队列的回调函数，然后根据队列的数目，依次调用 vp_setup_vq 完成 virtqueue、vring 的分配和初始化.

同样，这些数据结构会和 virtio 后端的 VirtIODevice、VirtQueue、vring 对应起来，都应该指向刚才创建的那一段内存.

客户机同样会通过调用专门给外部设备发送指令的函数 iowrite 告诉外部的 pci 设备，这些共享内存的地址.

至此前端设备驱动和后端设备驱动之间的两个收发队列就关联好了，这两个队列的格式和块设备是一样的.

### 发送网络包过程
接下来看当真的发送一个网络包的时候，会发生什么.

当网络包经过客户机的协议栈到达 virtio_net 驱动的时候，按照 net_device_ops 的定义，[virtnet_netdev](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c#L2561)的[start_xmit](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c#L1574) 会被调用.

接下来的调用链为：start_xmit->[xmit_skb](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c#L1527)-> [virtqueue_add_outbuf](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_ring.c#L1758)->[virtqueue_add](virtqueue_add)->[virtqueue_add_packed](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/virtio/virtio_ring.c#L1091)，将网络包放入队列中. 最后[start_xmit调用 virtqueue_notify 通知接收方](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/virtio_net.c#L1638).

写入一个 I/O 会使得 qemu 触发 VM exit，这个逻辑就是解析 CPU 的时候看到过的[kvm_cpu_exec](https://elixir.bootlin.com/qemu/latest/source/accel/kvm/kvm-all.c#L2307)的`case KVM_EXIT_IO`分支由[kvm_handle_io](https://elixir.bootlin.com/qemu/latest/source/accel/kvm/kvm-all.c#L2133)处理的那个流程. 最终会调用 VirtQueue 的 handle_output 函数. 前面已经设置过这个函数了，其实就是 [virtio_net_handle_tx_bh](https://elixir.bootlin.com/qemu/latest/source/hw/net/virtio-net.c#L2253).

virtio_net_handle_tx_bh 通过 [qemu_bh_schedule](https://elixir.bootlin.com/qemu/latest/source/util/async.c#L179) 回调了`q->tx_bh`，而[在 virtio_net_add_queue 中调用 qemu_bh_new，并把函数设置为 [virtio_net_tx_bh](https://elixir.bootlin.com/qemu/latest/source/hw/net/virtio-net.c#L2298). [virtio_net_tx_bh](https://elixir.bootlin.com/qemu/latest/source/hw/net/virtio-net.c#L2298) 函数调用发送函数 [virtio_net_flush_tx](https://elixir.bootlin.com/qemu/latest/source/hw/net/virtio-net.c#L2127).

virtio_net_flush_tx 会调用 [virtqueue_pop](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio.c#L1684) -> [virtqueue_packed_pop](https://elixir.bootlin.com/qemu/latest/source/hw/virtio/virtio.c#L1549). virtqueue_packed_pop里面，能看到对于 vring 的操作，也即从这里面将客户机里面写入的数据读取出来. 然后，virtio_net_flush_tx调用 [qemu_sendv_packet_async](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L738) 发送网络包.


接下来的调用链为：qemu_sendv_packet_async->[qemu_net_queue_send_iov](https://elixir.bootlin.com/qemu/latest/source/net/queue.c#L210)->[qemu_net_queue_flush](https://elixir.bootlin.com/qemu/latest/source/net/queue.c#L251)->[qemu_net_queue_deliver](https://elixir.bootlin.com/qemu/latest/source/net/queue.c#L151).


在 qemu_net_queue_deliver 中，会调用 NetQueue 的 deliver 函数. 在前面[qemu_new_nic](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L277)->[qemu_net_client_setup](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L234)->[qemu_new_net_queue](https://elixir.bootlin.com/qemu/latest/source/net/queue.c#L63) 会把 deliver 函数设置为 [qemu_deliver_packet_iov](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L256). qemu_deliver_packet_iov会调用 [nc->info->receive_iov](https://elixir.bootlin.com/qemu/latest/source/net/net.c#L726).

因为nc->info是[NetClientInfo net_tap_info](https://elixir.bootlin.com/qemu/latest/source/net/tap.c#L348), 因此nc->info->receive_iov即[tap_receive_iov](https://elixir.bootlin.com/qemu/latest/source/net/tap.c#L348). 它会调用 [tap_write_packet](https://elixir.bootlin.com/qemu/latest/source/net/tap.c#L348)->writev写入这个字符设备 by syscall [write(s->fd)](https://elixir.bootlin.com/qemu/latest/source/util/osdep.c#L532). 在内核的字符设备驱动中，[struct file_operations tun_fops](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/tun.c#L3448).write_iter即[tun_chr_write_iter](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/tun.c#L1989) 会被调用.

当使用 writev() 系统调用向 tun/tap 设备的字符设备文件写入数据时，tun_chr_write_iter 函数将被调用. 它会使用 [tun_get_user](https://elixir.bootlin.com/linux/v5.8-rc4/source/drivers/net/tun.c#L1708)，从用户区接收数据，将数据存入 skb 中，然后调用关键的函数 [netif_rx_ni(skb)(https://elixir.bootlin.com/linux/v5.8-rc4/source/net/core/dev.c#L4821)，将 skb 送给 tcp/ip 协议栈处理，最终完成虚拟网卡的数据接收.

至此，从虚拟机内部到宿主机的网络传输过程才算结束.

### 网络虚拟化总结
网络虚拟化场景下网络包的发送过程:
1. 在虚拟机里面的用户态，应用程序通过 write 系统调用写入 socket
1. 写入的内容经过 VFS 层，内核协议栈，到达虚拟机里面的内核的网络设备驱动，也即 virtio_net
1. virtio_net 网络设备有一个操作结构 struct net_device_ops，里面定义了发送一个网络包调用的函数为 start_xmit
1. 在 virtio_net 的前端驱动和 qemu 中的后端驱动之间，有两个队列 virtqueue，一个用于发送，一个用于接收. 然后，需要在 start_xmit 中调用 virtqueue_add，将网络包放入发送队列，然后调用 virtqueue_notify 通知 qemu.
1. qemu 本来处于 KVM_RUN 的状态，收到通知后，通过 VM exit 指令退出客户机模式，进入宿主机模式. 发送网络包的时候，virtio_net_handle_tx_bh 函数会被调用
1. 接下来是一个 for 循环，需要在循环中调用 virtqueue_pop，从传输队列中获取要发送的数据，然后调用 qemu_sendv_packet_async 进行发送
1. qemu 会调用 writev 向字符设备文件写入，进入宿主机的内核
1. 在宿主机内核中字符设备文件的 file_operations 里面的 write_iter 会被调用，也即会调用 tun_chr_write_iter.
1. 在 tun_chr_write_iter 函数中，tun_get_user 将要发送的网络包从 qemu 拷贝到宿主机内核里面来，然后调用 netif_rx_ni 开始调用宿主机内核协议栈进行处理
1. 宿主机内核协议栈处理完毕之后，会发送给 tap 虚拟网卡，完成从虚拟机里面到宿主机的整个发送过程

![网络虚拟化场景下网络包的发送过程](/misc/img/virt/e329505cfcd367612f8ae47054ec8e44.jpg)

## VMX
VT-x引入了一种新的处理器操作叫做 VMX(Virtual Machine Extension). VT-x也称为硬件支持的全虚拟化方案.

它提供了两种处理器的工作环境VMX root operation和VMX non-root operation. 由叫VMCS(virtual-machine
control data structure) 的数据结构实现两种环境之间的切换. VMM运行在VMX root operation, vm运行在VMX non-root operation, 这两种操作模式都支持ring 0~3.

进入VMX非根操作模式被称为`VM Entry`; 从非根操作模式退出, 被称为`VM Exit`.

VMX的根操作模式与非VMX模式下最初的处理器执行模式基本一样, 只是它现在支持了新的VMX相关的指令集以及一些对相关控制寄存器的操作. VMX的非根操作模式是
一个相对受限的执行环境, 为了适应虚拟化而专门做了一定的修改. 在客户机中执行的一些特殊的敏感指令或者一些异常会触发`VM Exit`退到hypervisor中, 从而运行在VMX根模式. 这让hypervisor保持了对处理器资源的控制.

vmm与guest的交互: 软件通过执行VMXON指令进入VMX操作模式下. 在VMX模式下通过VMLAUNCH和VMRESUME指令进入客户机执行模式, 即VMX非根模式. 当在非根模式下触发VM Exit时， 处理器执行控制权再次回到宿主机的虚拟机监控器上. 最后虚拟机监控可以执行VMXOFF指令退出VMX执行模式.

对VMCS的访问是通过VMCS指针来操作的, VMCS指针是一个指向VMCS结构的64位的地址, 使用VMPTRST和VMPTRLD指令对VMCS指针进行读写, 使用MREAD、 VMWRITE和VMCLEAR等指令对VMCS实现配置.

对于一个逻辑处理器, 它可以维护多个VMCS数据结构, 但是在任何时刻只有一个VMCS在当前真正生效. 多个VMCS之间也是可以相互切换的, VMPTRLD指令就让某个
VMCS在当前生效, 而其他VMCS就自然成为不是当前生效的. 一个虚拟机监控器会为一个虚拟客户机上的每一个逻辑处理器维护一个VMCS数据结构.

由于发生一次VM Exit的代价是比较高的（ 可能会消耗成百上千个CPU执行周期， 而平时很多指令是几个CPU执行周期就能完成）. **所以对于VM Exit的分析是
虚拟化中性能分析和调优的一个关键点**.

> 引发vm exit的敏感指令和一些异常可参考intel官方手册的[INSTRUCTIONS THAT CAUSE VM EXITS](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3c-part-3-manual.pdf)或KVM实战的"2.1.1 CPU虚拟化".

> 查看kvm exit status: kvm_stat

## VPID
Intel在内存虚拟化效率方面还引入了VPID（Virtual-processor identifier） 特性, 在硬件级对TLB资源管理进行了优化. 在没有VPID之前,  不同客户机的逻辑CPU在切换执行时需要刷新TLB, 而TLB的刷新会让内存访问的效率下降. VPID技术通过在硬件上为TLB增加一个标志,  可以识别不同的虚拟处理器的地址空间, 所以系统可以区分虚拟机监控器和不同虚拟机上不同处理器的TLB, 在逻辑CPU切换执行时就不会刷新TLB, 
而只需要使用对应的TLB即可. 当CPU运行在非根模式下, 且虚拟机执行控制寄存器的`enable VPID`比特位被置为1时, 当前的VPID的值是VMCS中
的VPID执行控制域的值, 其值是非0的. VPID的值在3种情况下为0， 第1种是在非虚拟化环境中执行时; 第2种是在根模式下执行时;  第3种情况是在非根模式下执行但`enable VPID`控制位被置0时.

## io虚拟化
ref:
- [四种主要网络 IO 虚拟化模型](https://xie.infoq.cn/article/489f746948c1ff8c0662c329f)

方法:
1. 设备模拟(软件实现)

    在VMM中模拟一个传统的io设备, 比如qemu模拟的网卡或磁盘

    优点: 兼容性好, 不需要额外驱动
    缺点: 
       1. 性能较差
       2. 模拟设备的功能特性支持不够多
1. virtio(软件实现)

    在VMM和guest间定义一种全新的适应虚拟化环境的交互接口.

    优点: 性能有所提升
    缺点: 
       1. 兼容性差一点: 依赖guest安装特定驱动
       2. io压力大时, 后端驱动占用cpu较高
1. 设备直接分配(硬件实现)

    将一个物理设备直接分配给guest, 此时io请求的链路很少或基本不需要VMM参与, 因此性能最佳.

    优点: 性能非常好
    缺点:
        1. 需要硬件设备的特性支持
        2. 一个设备只能分配给一个guest
        3. 很难支持动态迁移

    intel的实现是VT-d(Virtualization Technology For Directed I/O）特性, 一般在系统BIOS中可以看到相关的参数设置. Intel VT-d为虚拟机监控器提供了几个重要的能力： I/O设备分配、 DMA重定向、 中断重定向、 中断投递等.

1. 设备共享分配(硬件实现)

    是设备直接分配的扩展, 比如SR-IOV(Single Root I/O Virtualization and Sharing, 需设备和bios VT-d同时支持). 一个设备能模拟成多个设备并独立地分配给guest使用.

    实现了SR-IOV规范的设备, 有一个功能完整的PCI-e设备成为物理功能（Physical Function， PF）. 在使能了SR-IOV之后， PF就会派生出若干个虚拟功能（Virtual Function， VF）. VF看起来依然是一个PCI-e设备, 它拥有最小化的资源配置, 有用独立的资源,  可以作为独立的设备直接分配给客户机使用.

    > intel 82599系统网卡支持SR-IOV, 最多可分出63个VF.

    优点: 性能非常好
    缺点:
        1. 需要硬件设备的特性支持
        2. 很难支持动态迁移