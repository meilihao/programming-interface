# virt
KVM 在内核里面需要有一个模块，来设置当前 CPU 是 Guest OS 在用，还是 Host OS 在用.

KVM 内核模块通过 /dev/kvm 暴露接口，用户态程序可以通过 ioctl 来访问这个接口.

![](/misc/img/virt/f5ee1a44d7c4890e411c2520507ddc62.png)

Qemu 将 KVM 整合进来，将有关 CPU 指令的部分交由内核模块来做，就是 qemu-kvm (qemu-system-XXX).

qemu 和 kvm 整合之后，CPU 的性能问题解决了. 另外 Qemu 还会模拟其他的硬件，如网络和硬盘. 同样，全虚拟化的方式也会影响这些设备的性能.

于是，qemu 采取半虚拟化的方式，让 Guest OS 加载特殊的驱动来做这件事情.

例如，网络需要加载 virtio_net，存储需要加载 virtio_blk，Guest 需要安装这些半虚拟化驱动，GuestOS 知道自己是虚拟机，所以数据会直接发送给半虚拟化设备，经过特殊处理（例如排队、缓存、批量处理等性能优化方式），最终发送给真正的硬件. 这在一定程度上提高了性能.

> CPU 和内存主要使用硬件辅助虚拟化进行加速，需要配备特殊的硬件才能工作

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

qemu每个模块都会有一个定义 TypeInfo，会通过 type_init 变为全局的 TypeImpl. TypeInfo 以及生成的 TypeImpl 有以下成员：
- name 表示当前类型的名称
- parent 表示父类的名称
- class_init 用于将 TypeImpl 初始化为 MachineClass
- instance_init 用于将 MachineClass 初始化为 MachineState

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

在 pc_cpus_init 中，对于每一个 CPU，都调用 [x86_new_cpu](https://elixir.bootlin.com/qemu/v5.0.0/source/hw/i386/x86.c#L119)，在这里，看到了 object_new，这又是一个从 TypeImpl 到 Class 类再到对象的一个过程. 这个时候，就要看 CPU 的类是怎么组织的了. 在上面的参数里面，CPU 的配置是这样的:`
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
- VM-exit control fields，对 VM Exit 的行为进行控制. 比如，VM Exit 的时候对 vCPU 来说需要保存哪些 MSR 寄存器，对于主机 CPU 来说需要恢复哪些 MSR 寄存器
- VM-entry control fields，对 VM Entry 的行为进行控制. 比如，需要保存和恢复哪些 MSR 寄存器等
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

至此，CPU 虚拟化就解析完了.

![](/misc/img/virt/c43639f7024848aa3e828bcfc10ca467.png)

## 总结
![MachineClass](/misc/img/virt/078dc698ef1b3df93ee9569e55ea2f30.png)

每个模块都会有一个定义 TypeInfo，会通过 type_init 变为全局的 TypeImpl. TypeInfo 以及生成的 TypeImpl 有以下成员：
- name 表示当前类型的名称
- parent 表示父类的名称
- class_init 用于将 TypeImpl 初始化为 MachineClass
- instance_init 用于将 MachineClass 初始化为 MachineState

所以，以后遇到任何一个类型的时候，将父类和子类之间的关系，以及对应的初始化函数都要看好，这样就一目了然了.