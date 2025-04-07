# eBPF
参考:
- [Linux超能力BPF技术介绍及学习分享](https://www.tuicool.com/articles/eAfEvia)
- [BPF社区和生态](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664608487&idx=1&sn=6f3ddadb16ffa71557b41907999d5261)
- [Libbpf-tools —— 让 Tracing 工具身轻如燕](https://pingcap.com/blog-cn/libbpf-tools/)
- [kernel bpf examples](https://elixir.bootlin.com/linux/v5.10-rc7/source/samples/bpf)
- [如何将 eBPF 添加到可观察性产品中](https://brendangregg.com/blog/2021-07-03/how-to-add-bpf-observability.html)
- [Linux网络新技术基石 eBPF and XDP](https://mp.weixin.qq.com/s/BOamc7V7lZQa1FTuJMqSIA)
- [BCC (BPF Compiler Collection)](https://tonydeng.github.io/sdn-handbook/linux/bpf/bcc.html)
- [**eBPF 零侵扰分布式追踪的进展和探索**](https://my.oschina.net/u/3681970/blog/11048275)
- [DeepFlow 实战：eBPF 技术如何提升故障排查效率](https://my.oschina.net/u/4489239/blog/11422982)
- [**linux内核观测技术bpf**](https://github.com/DavadDi/bpf_study)
- [基于 eBPF 的云原生可观测性深度实践](https://lrting.top/backend/13662/)
- [eBPF技术实践白皮书](https://developer.aliyun.com/article/1375002)
- [**eunomia-bpf/awesome-ebpf-zh**](https://github.com/eunomia-bpf/awesome-ebpf-zh)
- [**Linux超能力BPF技术介绍及学习分享（技术创作101训练营）**](https://cloud.tencent.com/developer/article/1698426)
- [eCapture](https://www.oschina.net/news/341096/ecapture-v1-0-0)


BPF全称是Berkeley Packet Filter, 伯克利包过滤器. 它发明之处是网络过滤神器, tcpdump就是基于此. 它是 Linux 内核提供的基于 BPF 字节码的动态注入技术（常应用于 tcpdump、raw socket 过滤等）. eBPF(extended Berkeley Packet Filter)是针对于 BPF 的扩展增强，丰富了 BPF 指令集，提供了 Map 的 KV 存储结构. 开发者可以利用 bpf() 系统调用，初始化 eBPF 的 Program 和 Map，利用 netlink 消息或者 setsockopt() 系统调用，将 eBPF 字节码注入到特定的内核处理流程中（如 XDP、socket filter 等）, 允许以安全的方式在各个挂钩点执行字节码. 它用于许多 Linux 内核子系统，最突出的是网络、跟踪和安全（例如沙箱）.

> 原有的BPF被称为了cBPF(classic BPF, 目前基本已废弃, linux kernel只运行eBPF, 内核会将cBPF转成eBPF再执行).

> BPF CO-RE=`Compile Once – Run Everywhere`, 用于解决BPF可移植性问题, 使得一旦bpf程序成功编译并通过内核验证，那么它将在不同的内核版本之间正确工作，而无需为每个特定内核重新编译它.

> 创建BTF(BPF类型格式)是为了替代更通用、更详细的简洁调试信息. BTF是一种节省空间、紧凑但仍具有足够表现力的格式，可以描述C程序的所有类型信息.

> 与运行时的BCC相比，libbpf + BPF CO-RE将内存开销减少了近9倍, 因为libbcc库包含一个庞大的LLVM或Clang库.

eBPF演进成为了一套通用执行引擎，提供可基于系统或程序事件高效安全执行特定代码的通用能力，通用能力的使用者不再局限于内核开发者. 其使用场景不再仅仅是网络分析，可以基于eBPF开发性能分析、系统追踪、网络优化等多种类型的工具和平台.

eBPF 由执行字节码指令、存储对象和辅助函数组成. eBPF字节码指令在内核执行前必须通过 BPF 验证器的验证，同时在启用 BPF JIT 模式的内核中，会直接将字节码指令转成内核可执行的本地指令运行，具有很高的执行效率.

![bpf架构](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21YHTH1k6dfB5BDCT6JX9btibcIvzOISVcgMXJSeM7BXPfovUDC7KfWMg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![](/misc/img/net/640.webp)
![ebpf总体设计](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21ckKpFCZ7s0c5M3nbgw3y68WbLJoficiaGS9STIyxZBRia8cHnOdfqn6Bw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
![eBPF注入位置](/misc/img/net/eeuMjee.webp)

BPF优势:
- 强安全：BPF验证器（verifier）会保证每个程序能够安全运行，它会去检查将要运行到内核空间的程序的每一行是否安全可靠，如果检查不通过，它将拒绝这个程序被加载到内核中去，从而保证内核本身不会崩溃，这是不同于开发内核模块的. 比如以下几种情况是无法通过的BPF验证器的：

    - 没有实际加载BPF程序所需的权限
    - 访问任意内核空间的内存数据
    - 将任意内核空间的内存数据暴露给用户空间

    >  BPF验证机制很像Chrome浏览器对于Javascript脚本的沙盒机制

- 高性能：一旦通过了BPF验证器，那么它就会进入JIT编译阶段，利用Just-In-Time编译器，编译生成的是通用的字节码，它是完全可移植的，可以在x86和ARM等任意球CPU架构上加载这个字节码，这样我们能获得本地编译后的程序运行速度，而且是安全可靠的
- 持续交付：通过JIT编译后，就会把编译后的程序附加到内核中各种系统调用的钩子（hook）上，而且可以在不影响系统运行的情况下，实时在线地替换这些运行在Linux内核中的BPF程序. 举个例子，拿一个处理网络数据包的应用程序来说，在每秒都要处理几十万个数据包的情况下，在一个数据包和下一个数据包之间，加载到网络系统调用hook上的BPF程序是可以自动替换的，可以预见到的结果是，上一个数据包是旧版本的程序在处理，而下一个数据包就会看到新版本的程序了，没有任何的中断. 这就是无缝升级，从而实现持续交付的能力.

## ebpf总体设计
![ebpf总体设计](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21ckKpFCZ7s0c5M3nbgw3y68WbLJoficiaGS9STIyxZBRia8cHnOdfqn6Bw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

包括:
1. eBPF Runtime

    ![eBPF Runtime](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21S0hYUdncRriaWljziaUCKOJ7XOQ1UAswCCYliaFI7icYtrEV0SZPtTRGSw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)


    - 安全保障 ： eBPF的verifier 将拒绝任何不安全的程序并提供沙箱运行环境
    - 持续交付： 程序可以更新在不中断工作负载的情况下
    - 高性能：JIT编译器可以保证运行性能
1. eBPF Hooks

    ![eBPF Hooks](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21ZfPGXWKXeYTfHLLAa2IKDic6hUfPnHVK6GiaBs301SYiaRyNId2y5ZwjA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

    hooks位置: 内核函数 (kprobes)、用户空间函数 (uprobes)、系统调用、fentry/fexit、跟踪点、网络设备 (tc/xdp)、网络路由、TCP 拥塞算法、套接字（数据面）
1. eBPF Maps

    Map 类型:
    - Hash tables, Arrays
    - LRU (Least Recently Used)
    - Ring Buffer
    - Stack Trace
    - LPM (Longest Prefix match)

    作用:
    - 程序状态
    - 程序配置
    - 程序间共享数据
    - 和用户空间共享状态、指标和统计
1. eBPF Helpers

    ![eBPF Helpers](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21DLLOvQrSbBIgPOxYWzeXefgibrJao4M4DxibibSX5txvGkgIlf6rKpicvA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

    比如:
    - 随机数
    - 获取当前时间
    - map访问
    - 获取进程/cgroup 上下文
    - 处理网络数据包和转发
    - 访问套接字数据
    - 执行尾调用
    - 访问进程栈
    - 访问系统调用参数
    - ...

1. eBPF Tail and Function Calls

    ![eBPF Tail and Function Calls](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21bvdQhByhRib1icXPaj8nEG4pVSh5d0TvUygSzfWGdEBibDPbp9kFuRZlw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

    尾调用作用:
    - 将程序链接在一起
    - 将程序拆分为独立的逻辑组件
    - 使 BPF 程序可组合

    函数调用作用:
    - 重用内部的功能程序
    - 减少程序大小（避免内联）
- eBPF JIT Compiler

    ![eBPF JIT Compiler](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21mfgzOuEDWO2oyHkTEVlcFW3b8qyk8ickEuT9kTicEWibuwLBuyL3gNibPA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

    - 确保本地执行性能而不需要了解CPU
    - 将 BPF字节码编译到CPU架构特定指令集



[eBPF可以做什么？](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21mqT3QgJoEibaGNrAPCBaezMtw51MY6RDrjkywQMKeVL9d261588fLwQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

[eBPF 开源 Projects](https://mp.weixin.qq.com/s/BOamc7V7lZQa1FTuJMqSIA)

## BPF超能的核心技能点
### 1. BPF Hooks
BPF Hooks, 即BPF钩子，也就是在内核中，哪些地方可以加载BPF程序，在目前的Linux内核中已经有了近10种的钩子，如下所示：
- kernel functions (kprobes)
- userspace functions (uprobes)
- system calls
- fentry/fexit
- Tracepoints
- network devices (tc/xdp)
- network routes
- TCP congestion algorithms
- sockets (data level)

从文件打开、创建TCP链接、Socket链接到发送系统消息等几乎所有的系统调用，加上用户空间的各种动态信息，都能加载BPF程序，可以说是无所不能. 它们在内核中形成不同的BPF程序类型，在加载时会有类型判断.

### 2. BPF Map
对于BPF程序来说，可以在BPF Map存储数据状态、统计信息和指标信息. BPF程序本身只有指令，不会包含实际数据及其状态. 我们可以在BPF程序创建BPF Map，这个Map像其他编程语言具有的Map数据结构类似，也有很多类型，常用的就是Hash和Array类型，如下所示：
- Hash tables, Arrays
- LRU (Least Recently Used)
- Ring Buffer
- Stack Trace
- LPM (Longest Prefix match)

值得一提的是：
1. BPF Map是可以被用户空间访问并操作的
1. BPF Map是可以与BPF程序分离的，即当创建一个BPF Map的BPF程序运行结束后，该BPF Map还能存在，而不是随着程序一起消亡

基于上面两个特点，意味着我们可以利用BPF Map持久化数据，在不丢失重要数据的同时，更新BPF程序逻辑，实现在不同程序之间共享信息，在收集统计信息或指标等场景下，尤其有用.

### 3. BPF Helper Function
BPF Helper Function, 即BPF辅助函数.

它们是面向开发者的，提供操作BPF程序和BPF Map的工具类函数. 由于内核本身会有不定期更新迭代，如果直接调用内核模块，那天可能就不能用了，因此通过定义和维护BPF辅助函数，由BPF辅助函数来去面对后端的内核函数的变化，对开发者透明，形成稳定API接口.

## BPF能力的限制
BPF技术虽然强大，但是为了保证内核的处理安全和及时响应，内核对于BPF 技术也给予了诸多限制，如下是几个重点限制：
- eBPF 程序不能调用任意的内核参数，只限于内核模块中列出的 BPF Helper 函数，函数支持列表也随着内核的演进在不断增加
- eBPF程序不允许包含无法到达的指令，防止加载无效代码，延迟程序的终止
- eBPF 程序中循环次数限制且必须在有限时间内结束
- eBPF 堆栈大小被限制在 MAXBPFSTACK，截止到内核 Linux 5.8 版本，被设置为 512. 目前没有计划增加这个限制，解决方法是改用 BPF Map，它的大小是无限的.
- eBPF 字节码大小最初被限制为 4096 条指令，截止到内核 Linux 5.8 版本， 当前已将放宽至 100 万指令（ BPF_COMPLEXITY_LIMIT_INSNS），对于无权限的BPF程序，仍然保留4096条限制 ( BPF_MAXINSNS ).

## [BPF 程序类型](https://elixir.bootlin.com/linux/v5.10/source/include/uapi/linux/bpf.h#L170)
> **可通过`man 2 bpf`详细了解**.

主要分为两类:
- tracing

    可方便地了解系统中正在发生的事情
- networking

    可以检查和处理系统中的网络流量

具体分类(按添加到kernel的时间排序):
1. BPF_PROG_TYPE_SOCKET_FILTER

    仅允许出于可观察性目的访问数据包而无法修改
1. BPF_PROG_TYPE_KPROBE

    kprobes是可以动态附加到内核中某些调用点的函数, BPF kprobe程序类型允许将BPF程序用作kprobe处理程序.

    > kprobes在内核中不是稳定的入口点，因此，需要确保kprobe BPF程序与使用的内核版本的兼容性

    编写附加到kprobe的BPF程序时，需要决定是将其作为函数调用中的第一条指令执行还是在调用完成时执行, 即需要在BPF程序的标头中声明此行为.

    例如，如果您要在内核调用exec syscall时检查参数，则可以在调用开始时附加程序, 此时需要在代码段开始处设置`SEC("kprobe/sys_exec")`; 如果要检查调用exec syscall的返回值，则需要在代码段开始处设置`SEC("kretprobe/sys_exec")`
1. BPF_PROG_TYPE_TRACEPOINT

    允许将BPF程序附加到内核提供的跟踪点处理程序. 跟踪点是内核代码库中的静态标记，可注入任意代码以进行跟踪和调试. 它们不如kprobes灵活，因为它们需要事先由内核定义，但是可以保证在将其引入内核后保持稳定. 当要调试系统时，可提供更高的可预测性. 系统中的所有跟踪点都在目录`/sys/kernel/debug/tracing/events`中定义. 一个有趣的事实是BPF声明了自己的跟踪点，因此可以编写检查其他BPF程序行为的BPF程序. BPF跟踪点在`/sys/kernel/debug/tracing/events/bpf`中定义.
1. BPF_PROG_TYPE_XDP

    可以编写在网络数据包到达内核的早期就执行的代码, 由于数据包是在早期执行的，因此对如何处理该数据包具有更高级别的控制.

    XDP程序定义了几个可以控制的操作，这些操作可以决定如何处理数据包. XDP程序返回`XDP_PASS`意味着应该将数据包传递到内核中的下一个子系统; 返回`XDP_DROP`意味着内核应完全忽略此数据包，并且对其不执行任何其他操作; 返回`XDP_TX`意味着应将数据包转发回首先接收到数据包的网络接口卡（NIC）.

    这种控制级别为网络层中许多有趣的程序打开了大门, 比如实现程序以保护网络免受DDoS攻击. XDP已成为BPF的主要组件之一.
1. BPF_PROG_TYPE_PERF_EVENT

    允许将BPF代码附加到Perf事件. Perf是内核中的性能分析器，它能够生成硬件和软件的性能数据. 可以使用它来监控许多事情，从计算机的CPU到系统上运行的任何软件, 当将BPF程序附加到Perf事件时，每次Perf生成可供分析的数据时，都将执行该代码.
1. BPF_PROG_TYPE_CGROUP_SKB

    允许将BPF逻辑附加到cgroup. 它们允许cgroup控制它们所包含的进程内的网络流量. 借助这些程序，可以在将网络数据包传递到cgroup中的进程之前决定如何处理它. 内核尝试传递到同一cgroup中任何进程的任何数据包都将通过每一个过滤器. 同时，可以决定cgroup中的进程通过该接口发送网络数据包时该怎么做. 它们的行为类似于 BPF_PROG_TYPE_SOCKET_FILTER 程序, 主要区别在于 BPF_PROG_TYPE_CGROUP_SKB 程序附加到cgroup内的所有进程，而不是特定进程；此行为适用于在给定cgroup中创建的当前套接字和将来的套接字. 附加到cgroup的BPF程序在容器环境中很有用，在容器环境中，进程组受cgroup约束，并且可以对所有进程应用相同的策略，而不必独立识别每个进程. Cillium广泛使用cgroup套接字程序将其策略应用于组而不是隔离的容器中.
1. BPF_PROG_TYPE_CGROUP_SOCK

    允许在cgroup中的任何进程打开网络套接字时执行代码. 此行为类似于附加到cgroup套接字缓冲区的程序，与其给访问通过网络的数据包的权限，不如控制进程打开新套接字时发生的情况. 这对于为可以打开套接字的程序组提供安全性和访问控制很有用，而不必分别限制每个进程的功能.
1. BPF_PROG_TYPE_SOCK_OPS

    当数据包在内核网络栈中的多个阶段传输时允许在运行时修改套接字连接选项. 它们附加到cgroup，就像 BPF_PROG_TYPE_CGROUP_SOCK 和 BPF_PROG_TYPE_CGROUP_SKB 一样，但是与这些程序类型不同，它们可以在连接的生命周期中多次调用.

    当用这种类型创建一个BPF程序时，bpf函数调用将收到一个名为op的参数，该参数代表内核在套接字连接生命周期内时会执行的操作. 有了这些信息，就可以访问诸如网络IP地址和连接端口之类的数据，并且可以修改连接选项以设置超时并更改给定数据包的往返延迟时间.

    例如，Facebook使用它为同一数据中心内的连接设置短恢复时间目标（RTO）. RTO是指系统或网络连接在这种情况下在出现故障后可以恢复的时间. 该参数还表示系统在遭受无法接受的后果之前可能无法使用多长时间. 以Facebook为例，它假设同一数据中心中的计算机应具有较短的RTO，而Facebook使用BPF程序修改此阈值.
1. BPF_PROG_TYPE_SK_SKB

    允许访问套接字映射和套接字重定向. 套接字映射允许保留对多个套接字的引用. 当拥有这些引用时，可以使用特殊的辅助函数将传入的数据包从套接字重定向到另一个套接字. 当要使用BPF实现负载平衡功能时，这很有用. 通过跟踪多个套接字，可以在它们之间转发网络数据包而无需离开内核空间. 诸如Cillium和Facebook的Katran之类的项目广泛使用了这类程序来进行网络流量控制.
1. BPF_PROG_TYPE_CGROUP_DEVICE

    允许在给定设备上执行cgroup中的操作. cgroupsv1 的第一个实现具有一种机制，允许为特定设备的设置权限. 但是，cgroupv2缺少此功能, 引入了此类程序以提供该功能. 同是使得在需要时可以更灵活地设置这些权限.
1. BPF_PROG_TYPE_SK_MSG

    允许控制msg是否应该被传递到套接字. 当内核创建套接字时，它将套接字存储在上述套接字映射中. 该映射使内核可以快速访问特定的套接字组. 当将套接字消息BPF程序附加到套接字映射时，发送到这些套接字的所有消息将在传递它们之前被程序过滤. 在过滤消息之前，内核会复制消息中的数据，以便开发者可以读取它并决定如何处理它. 这些程序有两个可能的返回值：SK_PASS和SK_DROP. 如果希望内核将消息发送到套接字，则使用第一个消息; 如果希望内核忽略消息而不将消息传递到套接字，则使用后一个消息.
1. BPF_PROG_TYPE_RAW_TRACEPOINT

    内核开发人员添加了一个新的跟踪点程序，以解决访问内核保留的原始格式的跟踪点参数的需求. 这种格式可以访问到有关内核正在执行的task的更多详细信息, 且它的性能开销很小. 大多数时候，开发者会希望在程序中使用常规跟踪点来避免这种性能开销，但是请记住，还可以在需要时使用原始跟踪点来访问原始参数.
1. BPF_PROG_TYPE_CGROUP_SOCK_ADDR

    当用户态程序受特定cgroup控制时，可以操纵它们附加到的IP地址和端口号. 在某些情况下，如果要确保一组特定的用户态程序使用相同的IP地址和端口，则系统使用多个IP地址。当将这些用户态程序放在同一cgroup中时，这些BPF程序可以灵活地操作这些绑定. 这样可以确保这些应用程序的所有传入和传出连接都使用BPF程序提供的IP和端口.
1. BPF_PROG_TYPE_SK_REUSEPORT

    SO_REUSEPORT 是内核中的一个选项，它允许将同一主机中的多个进程绑定到同一端口. 当需要在多个线程之间分配负载时，此选项可在接受的网络连接中提供更高的性能. 这类程序钩子到内核用来决定是否要重用端口的逻辑中. 如果BPF程序返回SK_DROP，则可以防止程序重用同一端口，并且当从这些BPF程序返回SK_PASS时，还可以通知内核遵循其自己的重用例程.
1. BPF_PROG_TYPE_FLOW_DISSECTOR

    分流器是内核的一个组件，从网络数据包到达系统开始到数据包传递到用户态程序的时间内，跟踪经过不同层的网络数据包. 它允许使用不同的分类方法来控制数据包的流向. 内核中的内置解剖器称为Flower classifier，防火墙和其他过滤设备使用它来决定如何处理特定的数据包. 这类程序设计为在流分解器路径中挂钩逻辑. 它们提供内置解剖器无法提供的安全性保证，例如确保程序始终终止，而内置解剖器可能无法保证. 这些BPF程序可以修改网络数据包在内核中遵循的流.
1. 其他程序, 这些程序是用于指定领域的，其用法尚未为社区广泛采用.

    1. 流量分类程序 （Traffic classifier programs）

        BPF_PROG_TYPE_SCHED_CLS 和BPF_PROG_TYPE_SCHED_ACT 是两种BPF程序，可用于分类网络流量并修改套接字缓冲区中数据包的某些属性.
    1. 轻量级隧道程序（Lightweight tunnel programs）

        BPF_PROG_TYPE_LWT_IN，BPF_PROG_TYPE_LWT_OUT，BPF_PROG_TYPE_LWT_XMIT和BPF_PROG_TYPE_LWT_SEG6LOCAL是BPF程序的类型，可用于将代码附加到内核的轻量级隧道基础架构
    1. 红外设备程序（Infrared device programs）

        BPF_PROG_TYPE_LIRC_MODE2 程序允许通过连接将BPF程序附加到红外设备（例如遥控器）来获得乐趣

## BPF 校验器(BPF Verifier)
因为要在kernel里执行bpf, 必须要保证其正确且没有安全问题, 这就依赖于bpf 验证器.

1. 验证程序执行的第一项检查是对VM即将加载的代码的静态分析. 第一次检查的目的是确保程序有预期的结果. 为此，验证程序将使用代码创建有向循环图（DAG）. 验证程序分析的每个指令将成为图中的一个节点，并且每个节点都链接到下一条指令. 验证程序生成此图后，它将执行深度优先搜索（DFS），以确保程序完成并且代码不包含危险路径. 这意味着它将遍历图的每个分支，一直到分支的底部，以确保没有递归循环.

这些是验证器在第一次检查期间可能拒绝您的代码的情形，要求有以下几个方面：

    1. 该程序不包含控制循环. 为确保程序不会陷入无限循环，验证程序会拒绝任何类型的控制循环. 已经提出了在BPF程序中允许循环的建议，但未知是否被社区采用.
    1. 该程序不会尝试执行超过内核允许的最大指令数的指令. 当前可执行的最大指令数为4,096, 此限制是为了防止BPF永远运行.
    1. 程序不包含任何无法访问的指令，例如从未执行过的条件或功能. 这样可以防止在VM中加载无效代码，这也会延迟BPF程序的终止.
    1. 该程序不会尝试越界

1. 验证者执行的第二项检查是BPF程序的空运行. 这意味着验证器将尝试分析程序将要执行的每条指令，以确保它不会执行任何无效的指令. 此执行还将检查所有内存指针是否均已正确访问和取消引用. 最后，空运行向验证程序通知程序中的控制流，以确保无论程序采用哪个控制路径，它都会到达BPF_EXIT指令. 为此，验证程序会跟踪堆栈中所有访问过的分支路径，并在采用新路径之前对其进行评估，以确保它不会多次访问特定路径.

经过这两项检查后，验证器认为程序可以安全执行.

可使用bpf syscall调试(为用户态程序提供与内核中的eBPF进行交互的途径)验证bpf程序的检查, 使用该syscall加载程序时，可以设置几个属性，这些属性将使验证程序打印其操作日志：
```c
// log_level字段告诉验证器是否打印任何日志. 如果将其设置为1，它将打印其日志；如果将其设置为0，它将不打印任何内容. 如果要打印验证程序日志，还需要提供日志缓冲区及其大小. 该缓冲区是一个多行字符串，可通过打印该字符串以检查验证器做出的决定.
union bpf_attr attr = { 
    .prog_type = type,
    .insns = ptr_to_u64(insns),
    .insn_cnt = insn_cnt,
    .license =ptr_to_u64(license),
    .log_buf =ptr_to_u64(bpf_log_buf),
    .log_size = LOG_BUF_SIZE,
    .log_level = 1,
};

bpf(BPF_PROG_LOAD, &attr, sizeof(attr));
```

```c
// cmd是eBPF支持的cmd，分为三类： 操作注入的代码、操作用于通信的map、前两个操作的混合
int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

## BPF类型格式
BPF类型格式（BTP）是元数据结构的集合，可增强BPF程序，映射和功能的调试信息. BTF包含源信息，通过BPFTool之类的工具可以对BPF数据的实现更丰富的展示. 此元数据存储在类型是.BTF的可执行程序中. BTF信息有助于使程序更容易调试，但是它会大大增加二进制文件的大小，因为它需要跟踪程序中声明的所有内容的类型信息.

BPF验证程序还使用此信息来确保程序定义的结构类型正确.

BTF仅用于注释C的类型. LLVM之类的BPF编译器知道如何引入这些信息，因此开发者无需完成将这些信息添加到每个结构中的繁琐任务. 但是，在某些情况下，工具链仍需要一些注释以增强程序.

## BPF 尾部调用
BPF程序可以使用尾部调用来调用其他BPF程序. 这是一个强大的功能，因为它允许通过组合较小的BPF函数来实现更复杂的程序. 5.2之前的内核版本对BPF程序可以生成的机器指令数量有严格的限制(该限制设置为4,096)，以确保程序可以在合理的时间内终止. 但是，随着构建更复杂的BPF程序，用户需要一种方法来扩展由内核施加的指令限制，这就是尾部调用起作用的地方. 从内核版本5.2开始，指令限制增加到一百万条指令. 尾部调用嵌套也受到限制，在这种情况下，最多只能进行32个调用，这意味着可以在一个链中组合多达32个程序，以生成更复杂的解决方案.

当从另一个BPF程序调用BPF程序时，内核会完全重设程序上下文. 记住这一点很重要，因为因此可能需要一种在程序之间共享信息的方法. 每个BPF程序作为其参数接收的上下文对象都不会帮助我们解决此数据共享问题.

## BPF应用场景
- cilium : Cilium是首款完全基于eBPF程序实现了kube-proxy的所有功能的K8S CNI网络插件，无需依赖iptables和IPVS.

    ![](/misc/img/net/eBPF_cilium.png)
    ![配合eBPF Map存储后端Pod地址和端口，实现高效查询和更新](/misc/img/net/cilium_pod.png)

## bpf tools

### bcc
参考:
- [bcc开发](https://www.cnblogs.com/charlieroro/p/13265252.html)

![](https://github.com/iovisor/bcc/blob/master/images/bcc_tracing_tools_2019.png)

Bcc(BPF Compiler Collection)是ebpf的编译工具集合，前端提供python/lua调用，本身通过c语言实现，集成llvm/clang，将ebpf代码注入kernel，提供一些更人性化的函数给用户使用.

> 该框架主要针对涉及应用程序和系统分析/跟踪的用例，其中 eBPF 程序用于收集统计信息或生成事件，用户空间中的对应部分收集数据并以人类可读的形式显示.

> bcc使用`BPF.get_kprobe_functions(b'blk_start_request')`函数校验kernel函数是否存在，其是在/proc/kallsyms中进行检查的，因为/proc/kallsyms保存了Linux内核符号表.

bcc/tools列表:
- argdist.py : 统计指定函数的调用次数、调用所带的参数等等信息，打印直方图
- bashreadline.py : 获取正在运行的 bash 命令所带的参数
- biolatency.py : 统计 block IO 请求的耗时，打印直方图
- biosnoop.py : 打印每次 block IO 请求的详细信息
- biotop.py : 打印每个进程的 block IO 详情
- bitesize.py : 分别打印每个进程的 IO 请求直方图
- bpflist.py : 打印当前系统正在运行哪些 BPF 程序
- btrfsslower.py : 打印 btrfs 慢于某一阈值的 read/write/open/fsync 操作的数量
- cachestat.py : 打印 Linux 页缓存 hit/miss 状况
- cachetop.py : 分别打印每个进程的页缓存状况
- capable.py : 跟踪到内核函数 cap_capable()（安全检查相关）的调用，打印详情
- ujobnew.sh 跟踪内存对象分配事件，打印统计，对研究 GC 很有帮助
- cpudist.py : 统计 task on-CPU time，即任务在被调度走之前在 CPU 上执行的时间
- cpuunclaimed.py : 跟踪 CPU run queues length，打印 idle CPU (yet unclaimed by waiting threads) 百分比
- criticalstat.py : 跟踪涉及内核原子操作的事件，打印调用栈
- dbslower.py : 跟踪 MySQL 或 PostgreSQL 的慢查询
- dbstat.py : 打印 MySQL 或 PostgreSQL 的查询耗时直方图
- dcsnoop.py : 跟踪目录缓存（dcache）查询请求
- dcstat.py : 打印目录缓存（dcache）统计信息
- deadlock.py : 检查运行中的进行可能存在的死锁
- execsnoop.py : 跟踪新进程创建事件
- ext4dist.py : 跟踪 ext4 文件系统的 read/write/open/fsyncs 请求，打印耗时直方图
- ext4slower.py : 跟踪 ext4 慢请求
- filelife.py : 跟踪短寿命文件（跟踪期间创建然后删除）
- fileslower.py : 跟踪较慢的同步读写请求
- filetop.py : 打印文件读写排行榜（top），以及进程详细信息
- funccount.py : 跟踪指定函数的调用次数，支持正则表达式
- funclatency.py : 跟踪指定函数，打印耗时
- funcslower.py : 跟踪唤醒时间（function invocations）较慢的内核和用户函数
- gethostlatency.py : 跟踪 hostname 查询耗时
- hardirqs.py : 跟踪硬中断耗时
- inject.py :
- javacalls.sh
- javaflow.sh
- javagc.sh
- javaobjnew.sh
- javastat.sh
- javathreads.sh
- killsnoop.py : 跟踪 kill()系统调用发出的信号
- llcstat.py : 跟踪缓存引用和缓存命中率事件
- mdflush.py : 跟踪 md driver level 的 flush 事件
- memleak.py : 检查内存泄漏
- mountsnoop.py : 跟踪 mount 和 unmount 系统调用
- mysqld_qslower.py : 跟踪 MySQL 慢查询
- nfsdist.py : 打印 NFS read/write/open/getattr 耗时直方图
- nfsslower.py : 跟踪 NFS read/write/open/getattr 慢操作
- nodegc.sh 跟踪高级语言（Java/Python/Ruby/Node/）的 GC 事件
- offcputime.py : 跟踪被阻塞的进程，打印调用栈、阻塞耗时等信息
- offwaketime.py : 跟踪被阻塞且 off-CPU 的进程
- oomkill.py : 跟踪 Linux out-of-memory (OOM) killer
- opensnoop.py : 跟踪 open()系统调用
- perlcalls.sh
- perlstat.sh
- phpcalls.sh
- phpflow.sh
- phpstat.sh
- pidpersec.py : 跟踪每分钟新创建的进程数量（通过跟踪 fork()）
- profile.py : CPU profiler
- pythoncalls.sh
- pythoonflow.sh
- pythongc.sh
- pythonstat.sh
- reset-trace.sh
- rubycalls.sh
- rubygc.sh
- rubyobjnew.sh
- runqlat.py : 调度器 run queue latency 直方图，每个 task 等待 CPU 的时间
- runqlen.py : 调度器 run queue 使用百分比
- runqslower.py : 跟踪调度延迟很大的进程（等待被执行但是没有空闲 CPU）
- shmsnoop.py : 跟踪 shm*()系统调用
- slabratetop.py : 跟踪内核内存分配缓存（SLAB 或 SLUB）
- sofdsnoop.py : 跟踪 unix socket 文件描述符（FD）
- softirqs.py : 跟踪软中断
- solisten.py : 跟踪内核 TCP listen 事件
- sslsniff.py : 跟踪 OpenSSL/GnuTLS/NSS 的 write/send 和 read/recv 函数
- stackcount.py : 跟踪函数和调用栈
- statsnoop.py : 跟踪 stat()系统调用
- syncsnoop.py : 跟踪 sync()系统调用
- syscount.py : 跟踪各系统调用次数
- tclcalls.sh
- tclflow.sh
- tclobjnew.sh
- tclstat.sh
- tcpaccept.py : 跟踪内核接受 TCP 连接的事件
- tcpconnect.py : 跟踪内核建立 TCP 连接的事件
- tcpconnlat.py : 跟踪建立 TCP 连接比较慢的事件，打印进程、IP、端口等详细信息
- tcpdrop.py : 跟踪内核 drop TCP 包或片（segment）的事件
- tcplife.py : 打印跟踪期间建立和关闭的的 TCP session
- tcpretrans.py : 跟踪 TCP 重传
- tcpstates.py : 跟踪 TCP 状态变化，包括每个状态的时长
- tcpsubnet.py : 根据 destination 打印每个 subnet 的 throughput
- tcptop.py : 根据 host 和 port 打印 throughput
- tcptracer.py : 跟踪进行 TCP connection 操作的内核函数
- tplist.py : 打印内核 tracepoint 和 USDT probes 点，已经它们的参数
- trace.py : 跟踪指定的函数，并按照指定的格式打印函数当时的参数值
- ttysnoop.py : 跟踪指定的 tty 或 pts 设备，将其打印复制一份输出
- vfscount.py : 统计 VFS（虚拟文件系统）调用
- vfsstat.py : 跟踪一些重要的 VFS 函数，打印统计信息
- wakeuptime.py : 打印进程被唤醒的延迟及其调用栈
- xfsdist.py : 打印 XFS read/write/open/fsync 耗时直方图
- xfsslower.py : 打印 XFS 慢请求
- zfsdist.py : 打印 ZFS read/write/open/fsync 耗时直方图
- zfsslower.py : 打印 ZFS 慢请求

### bpftrace
bpftrace 是一种用于 Linux eBPF 的高级跟踪语言，可在最近的 Linux 内核 (4.x) 中使用。bpftrace 使用 LLVM 作为后端将脚本编译为 eBPF 字节码，并利用 BCC 与 Linux eBPF 子系统以及现有的 Linux 跟踪功能进行交互：内核动态跟踪 (kprobes)、用户级动态跟踪 (uprobes) 和跟踪点. bpftrace 语言的灵感来自 awk、C 和前身跟踪器，例如 DTrace 和 SystemTap.

![bpftrace](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21gAw4gv6HhkzJjqmwbBheAxVtsl4uCSMHr9zYyVjiavlFXAYE5zc5gDA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### eBPF Go 库
eBPF Go 库提供了一个通用的 eBPF 库，它将获取 eBPF 字节码的过程与 eBPF 程序的加载和管理解耦。eBPF 程序通常是通过编写高级语言创建的，然后使用 clang/LLVM 编译器编译为 eBPF 字节码.

![eBPF Go 库](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21D4ZF1sFleoI6w86ksriaQv9zGhBa1XcX7B6lqUFPFxYLEnlzs7xic6bg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### libbpf C/C++ 库
libbpf 库是一个基于 C/C++ 的通用 eBPF 库，它有助于解耦从 clang/LLVM 编译器生成的 eBPF 目标文件加载到内核中，并通过提供易于使用的库 API 来抽象与 BPF 系统调用的交互应用程序.

![libbpf C/C++ 库](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21zS9gak7grtSYlDdITV1pYXF2P6BScdWZnR0h2mYAXzbwib7Am6hYA6A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### next net acl
参考:
- [eBPF技术实践：高性能ACL](https://www.tuicool.com/articles/NZJjUbi)
- [从Bcc到xdp原理分析](https://kernel.taobao.org/2019/05/bcc_to_xdp/)

随着 eBPF 技术的快速发展，bpfilter 有望取代 iptables/nftables，成为下一代网络 ACL 的解决方案.

XDP（eXpress Data Path）是基于 eBPF 实现的高性能、可编程的数据平面技术, 是Linux 内核中提供高性能、可编程的网络数据包处理框架. 它能够在网络包进入用户态直接对网络包进行过滤或者处理.

![xdp架构](/misc/img/net/2M36Vbb.webp)
![XDP整体框架](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21yEsfia4WFia9sz6NXOFx163Wt5ymHQjbeQ4pib52eOPkAzIsgRPuzxtCw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

xdp优点:
- 直接接管网卡的RX数据包（类似DPDK用户态驱动）处理
- 通过运行BPF指令快速处理报文
- 和Linux协议栈无缝对接

XDP总体设计:
![XDP总体设计](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ21oiblJrLqn4u6psXmQMOHX0v69bb9nrn2RxYhXlCuYxQvKqG6oRMB4aA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

XDP总体设计包括以下几个部分：
- XDP驱动

    网卡驱动中XDP程序的一个挂载点，每当网卡接收到一个数据包就会执行这个XDP程序；XDP程序可以对数据包进行逐层解析、按规则进行过滤，或者对数据包进行封装或者解封装，修改字段对数据包进行转发等；
- BPF虚拟机

    一个XDP程序首先是由用户编写用受限制的C语言编写的，然后通过clang前端编译生成BPF字节码，字节码加载到内核之后运行在eBPF虚拟机上，虚拟机通过即时编译将XDP字节码编译成底层二进制指令；eBPF虚拟机支持XDP程序的动态加载和卸载
- BPF maps

    存储键值对，作为用户态程序和内核态XDP程序、内核态XDP程序之间的通信媒介，类似于进程间通信的共享内存访问；用户态程序可以在BPF映射中预定义规则，XDP程序匹配映射中的规则对数据包进行过滤等；XDP程序将数据包统计信息存入BPF映射，用户态程序可访问BPF映射获取数据包统计信息；
- BPF程序校验器

    XDP程序肯定是我们自己编写的，那么如何确保XDP程序加载到内核之后不会导致内核崩溃或者带来其他的安全问题呢？程序校验器就是在将XDP字节码加载到内核之前对字节码进行安全检查，比如判断是否有循环，程序长度是否超过限制，程序内存访问是否越界，程序是否包含不可达的指令；
- XDP Action
    XDP用于报文的处理，支持如下action：
    ```c
    enum xdp_action {
        XDP_ABORTED = 0,
        XDP_DROP,
        XDP_PASS,
        XDP_TX,
        XDP_REDIRECT,
    };
    ```

    - XDP_DROP：在驱动层丢弃报文，通常用于实现DDos或防火墙
    - XDP_PASS：允许报文上送到内核网络栈，同时处理该报文的CPU会分配并填充一个skb，将其传递到GRO引擎。之后的处理与没有XDP程序的过程相同。
    - XDP_TX：从当前网卡发送出去。
    - XDP_REDIRECT：从其他网卡发送出去。
    - XDP_ABORTED：表示程序产生了异常，其行为和 XDP_DROP相同，但 XDP_ABORTED 会经过 trace_xdp_exception tracepoint，因此可以通过 tracing 工具来监控这种非正常行为。

- AF_XDP

    AF_XDP 是为高性能数据包处理而优化的地址族，AF_XDP 套接字使 XDP 程序可以将帧重定向到用户空间应用程序中的内存缓冲区。

XDP设计原则
- XDP 专为高性能而设计。它使用已知技术并应用选择性约束来实现性能目标
- XDP 还具有可编程性。无需修改内核即可即时实现新功能
- XDP 不是内核旁路。它是内核协议栈的快速路径
- XDP 不替代TCP/IP 协议栈。与协议栈协同工作
- XDP 不需要任何专门的硬件。它支持网络硬件的少即是多原则

XDP技术优势
- 及时处理

    - 在网络协议栈前处理，由于 XDP 位于整个 Linux 内核网络软件栈的底部，能够非常早地识别并丢弃攻击报文，具有很高的性能。可以改善 iptables 协议栈丢包的性能瓶颈
    - DDIO
    - Packeting steering
    - 轮询式
- 高性能优化

    - 无锁设计
    - 批量I/O操作
    - 不需要分配skbuff
    - 支持网络卸载
    - 支持网卡RSS

- 指令虚拟机

    - 规则优化，编译成精简指令，快速执行
    - 支持热更新，可以动态扩展内核功能
    - 易编程-高级语言也可以间接在内核运行
    - 安全可靠，BPF程序先校验后执行，XDP程序没有循环

- 可扩展模型

    - 支持应用处理（如应用层协议GRO）
    - 支持将BPF程序卸载到网卡
    - BPF程序可以移植到用户空间或其他操作系统

- 可编程性

    - 包检测，BPF程序发现的动作
    - 灵活（无循环）协议头解析
    - 可能由于流查找而有状态
    - 简单的包字段重写（encap/decap）

XDP 有三种工作模式:
- Native XDP

    默认模式，在这种模式中，XDP BPF 程序直接运行在网络驱动的早期接收路径上（early receive path）

- Offloaded XDP

    在这种模式中，XDP BPF程序直接 offload 到网卡

- Generic XDP

    对于还没有实现 native 或 offloaded XDP 的驱动，内核提供了一个 generic XDP 选项，这种设置主要面向的是用内核的 XDP API 来编写和测试程序的开发者，对于在生产环境使用XDP，推荐要么选择native要么选择offloaded模式.

XDP vs DPDK
![XDP vs DPDK](https://mmbiz.qpic.cn/mmbiz_png/cYSwmJQric6mhcZBGEw04CPU7YpgGCQ219cJF9wSlSJUZxLRMu8Zj1MNRWxBvNWVbT97ciaIpCATchL742BA1nicw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

相对于DPDK，XDP：
- 优点

    - 无需第三方代码库和许可
    - 同时支持轮询式和中断式网络
    - 无需分配大页
    - 无需专用的CPU
    - 无需定义新的安全网络模型

- 缺点
    
    注意XDP的性能提升是有代价的，它牺牲了通用型和公平性

    - XDP不提供缓存队列（qdisc），TX设备太慢时直接丢包，因而不要在RX比TX快的设备上使用XDP
    - XDP程序是专用的，不具备网络协议栈的通用性

如何选择？

- 内核延伸项目，不想bypass内核的下一代高性能方案
- 想直接重用内核代码
- 不支持DPDK程序环境

XDP适合场景
- DDoS防御
- 防火墙
- 基于XDP_TX的负载均衡
- 网络统计
- 流量监控
- 栈前过滤/处理
- ...

XDP 位于网卡驱动层, 当数据包经过 DMA 存放到 ring buffer 之后, 分配 skb 之前即可被 XDP 处理. 数据包经过 XDP 之后, 会有 4 种去向：
- XDP_DROP：丢包
	
	网上有比较数据，利用 XDP 技术的丢包速率要比 iptables 高 4 倍左右.
- XDP_PASS：上送协议栈
- XDP_TX：从当前网卡发送出去
- XDP_REDIRECT：从其他网卡发送出去

由于 XDP 位于整个 Linux 内核网络软件栈的底部, 能够非常早地识别并丢弃攻击报文, 具有很高的性能. 这为改善 iptables/nftables 协议栈丢包的性能瓶颈，提供了非常棒的解决方案.

相对于DPDK，XDP具有以下优点:
- 无需第三方代码库和许可
- 同时支持轮询式和中断式网络
- 无需分配大页
- 无需专用的CPU
- 无需定义新的安全网络模型

XDP的使用场景包括:
- DDoS防御
- 防火墙
- 基于XDP_TX的负载均衡
- 网络统计
- 复杂网络采样
- 高速交易平台

ACL 控制平面负责创建 eBPF 的 Program、Map，注入 XDP 处理流程中: 其中 eBPF 的 Program 存放报文匹配、丢包等处理逻辑，eBPF 的 Map 存放 ACL 规则.
![ACL 控制平面架构](/misc/img/net/mmqQ7fA.webp)

## bpf map
bpf map是驻留在kernel中的kv存储. 任何BPF程序都可以访问它们, 在用户态中运行的程序也可以使用文件描述符访问这些映射, 即内核上运行的代码和加载了该代码的程序可以在运行时使用消息传递相互通信.

kernel将kv视为二进制 blobs，它并不关心在map中保留的内容.

### 创建map
方法1, 创建BPF映射的最直接方法是使用bpf syscall:
```c
    union bpf_attr my_map { 
        .map_type = BPF_MAP_TYPE_HASH, 
        .key_size = sizeof(int), 
        .value_size = sizeof(int), 
        .max_entries = 100,
        .map_flags = BPF_F_NO_PREALLOC,
    };

    int fd = bpf(BPF_MAP_CREATE, &my_map, sizeof(my_map));
```

如果调用失败, 则内核返回值-1, 失败的原因可能有三个:
1. 如果其中一个属性无效，则内核将errno变量设置为EINVAL
1. 如果执行该操作的用户没有足够的特权，则内核将errno变量设置为EPERM
1. 如果没有足够的内存来存储映射，则内核将errno变量设置为ENOMEM

方法2, 使用辅助函数创建map.
内核包括一些约定和帮助程序，用于生成和使用BPF映射. 与直接执行系统调用相比，会发现这些约定的出现频率更高，因为它们更具可读性且易于遵循. 请记住，即使直接在内核中运行，这些约定仍会使用bpf syscall来创建映射，如果事先不知道要使用哪种映射，则直接使用syscall会更有用.

```c
    // 功能同上, 即封装了上面直接使用syscall bpf创建map的example
    int fd;
    fd = bpf_create_map(BPF_MAP_TYPE_HASH, sizeof(int), sizeof(int), 100,
            BPF_F_NO_PREALOC);
```

如果知道程序中需要哪种映射，也可以预先定义. 这对于增加程序中使用的映射的可见性有很大帮助:
```c
    struct bpf_map_def SEC("maps") my_map = { 
        .type = BPF_MAP_TYPE_HASH, 
        .key_size = sizeof(int), 
        .value_size = sizeof(int), 
        .max_entries = 100,
        .map_flags =BPF_F_NO_PREALLOC, 
    };
```

以这种方式定义映射: 使用`section` 属性. 在本例中为`SEC("maps")`, 该宏告诉kernel此结构是BPF映射，应该相应地创建它.

在上面示例中，没有与映射关联的文件描述符标识符. 在这种情况下，内核使用名为map_data的全局变量将有关映射的信息存储在程序中. 此变量是一个结构数组，根据在代码中指定每个映射的方式进行排序. 例如，如果先前的映射是代码中指定的第一个映射，则可以从数组的第一个元素获取文件描述符标识符`fd = map_data[0].fd;`.

### 使用map

## FAQ
### funccount-bpfcc 'go:fmt.*'报"include/linux/compiler_types.h:210:24: note: expanded from macro 'asm_inline'"
见[Missing support for asm_inline in Linux 5.4](https://github.com/iovisor/bcc/issues/2546)

将bcc升级到v0.12.0及以上即可.

### ### build bcc
```bash
# # [get bcc code by see here](https://github.com/iovisor/bcc/blob/master/INSTALL.md#libbpf-submodule)
sudo apt install apt-get install clang-11 lldb-11 lld-11 libclang-11-dev luajit libluajit-5.1-dev arping iperf netperf cmake bison flex
cd <bcc_resource>
mkdir bcc/build && cd bcc/build
cmake ..
make
sudo make install # bcc会被安装在/usr/local/share/bcc(可使用`-DCMAKE_INSTALL_PREFIX=/usr`修改安装路径), 默认编译使用的是python2 binding(但也可能是根据/usr/bin/python进行推测)
sudo ldconfig # 刷新`.so` cache
sudo /usr/local/share/bcc/tools/tcpconnect # 执行bcc tools验证bcc. 需要`sudo ldconfig`, 避免执行时报`OSError: libbcc.so.0: cannot open shared object file: No such file or directory`
cmake -DPYTHON_CMD=python3 .. # build python3 binding
pushd src/python/
make
sudo make install
popd
ls -l /usr/bin/python
sudo ln -s -f $(which python3) /usr/bin/python
sudo PYTHONPATH=/usr/local/lib/python3/dist-packages ./tcptop -C 1 3 # 因为安装在`/usr/local/lib/python3`的原因需要添加PYTHONPATH, 应该可通过DCMAKE_INSTALL_PREFIX修正或使用`sudo mv /usr/local/lib/python3/dist-packages/bcc* /usr/lib/python3/dist-packages`修正路径
```
