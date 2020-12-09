# eBPF
参考:
- [Linux超能力BPF技术介绍及学习分享](https://www.tuicool.com/articles/eAfEvia)
- [BPF社区和生态](https://mp.weixin.qq.com/s?__biz=MzI3NzA5MzUxNA==&mid=2664608487&idx=1&sn=6f3ddadb16ffa71557b41907999d5261)
- [Libbpf-tools —— 让 Tracing 工具身轻如燕](https://pingcap.com/blog-cn/libbpf-tools/)

BPF全称是Berkeley Packet Filter, 伯克利包过滤器. 它发明之处是网络过滤神器, tcpdump就是基于此. 它是 Linux 内核提供的基于 BPF 字节码的动态注入技术（常应用于 tcpdump、raw socket 过滤等）. eBPF(extended Berkeley Packet Filter)是针对于 BPF 的扩展增强，丰富了 BPF 指令集，提供了 Map 的 KV 存储结构. 开发者可以利用 bpf() 系统调用，初始化 eBPF 的 Program 和 Map，利用 netlink 消息或者 setsockopt() 系统调用，将 eBPF 字节码注入到特定的内核处理流程中（如 XDP、socket filter 等）.

> 原有的BPF被称为了cBPF(classic BPF, 目前基本已废弃, linux kernel只运行eBPF, 内核会将cBPF转成eBPF再执行). 

eBPF演进成为了一套通用执行引擎，提供可基于系统或程序事件高效安全执行特定代码的通用能力，通用能力的使用者不再局限于内核开发者. 其使用场景不再仅仅是网络分析，可以基于eBPF开发性能分析、系统追踪、网络优化等多种类型的工具和平台.

eBPF 由执行字节码指令、存储对象和辅助函数组成. eBPF字节码指令在内核执行前必须通过 BPF 验证器的验证，同时在启用 BPF JIT 模式的内核中，会直接将字节码指令转成内核可执行的本地指令运行，具有很高的执行效率.

![](/misc/img/net/640.webp)
![eBPF注入位置](/misc/img/net/eeuMjee.webp)

BPF优势:
- 强安全：BPF验证器（verifier）会保证每个程序能够安全运行，它会去检查将要运行到内核空间的程序的每一行是否安全可靠，如果检查不通过，它将拒绝这个程序被加载到内核中去，从而保证内核本身不会崩溃，这是不同于开发内核模块的. 比如以下几种情况是无法通过的BPF验证器的：

    - 没有实际加载BPF程序所需的权限
    - 访问任意内核空间的内存数据
    - 将任意内核空间的内存数据暴露给用户空间

    >  BPF验证机制很像Chrome浏览器对于Javascript脚本的沙盒机制

- 高性能：一旦通过了BPF验证器，那么它就会进入JIT编译阶段，利用Just-In-Time编译器，编译生成的是通用的字节码，它是完全可移植的，可以在x86和ARM等任意球CPU架构上加载这个字节码，这样我们能获得本地编译后的程序运行速度，而且是安全可靠的
- 持续交付：通过JIT编译后，就会把编译后的程序附加到内核中各种系统调用的钩子（hook）上，而且可以在不影响系统运行的情况下，实时在线地替换这些运行在Linux内核中的BPF程序. 举个例子，拿一个处理网络数据包的应用程序来说，在每秒都要处理几十万个数据包的情况下，在一个数据包和下一个数据包之间，加载到网络系统调用hook上的BPF程序是可以自动替换的，可以预见到的结果是，上一个数据包是旧版本的程序在处理，而下一个数据包就会看到新版本的程序了，没有任何的中断. 这就是无缝升级，从而实现持续交付的能力.

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

## BPF应用场景
- cilium : Cilium是首款完全基于eBPF程序实现了kube-proxy的所有功能的K8S CNI网络插件，无需依赖iptables和IPVS.

    ![](/misc/img/net/eBPF_cilium.png)
    ![配合eBPF Map存储后端Pod地址和端口，实现高效查询和更新](/misc/img/net/cilium_pod.png)

## bpf tools
![](https://github.com/iovisor/bcc/blob/master/images/bcc_tracing_tools_2019.png)

## next net acl
参考:
- [eBPF技术实践：高性能ACL](https://www.tuicool.com/articles/NZJjUbi)

随着 eBPF 技术的快速发展，bpfilter 有望取代 iptables/nftables，成为下一代网络 ACL 的解决方案.

XDP（eXpress Data Path）是基于 eBPF 实现的高性能、可编程的数据平面技术. 它能够在网络包进入用户态直接对网络包进行过滤或者处理.

![xdp架构](/misc/img/net/2M36Vbb.webp)

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

## FAQ
### funccount-bpfcc 'go:fmt.*'报"include/linux/compiler_types.h:210:24: note: expanded from macro 'asm_inline'"
见[Missing support for asm_inline in Linux 5.4](https://github.com/iovisor/bcc/issues/2546)

将bcc升级到v0.12.0及以上即可.