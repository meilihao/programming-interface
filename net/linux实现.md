# net的linux实现
在TCP/IP模型, Linux内核以及网卡驱动主要实现驱动(链路层)、协议栈(网络层和传输层)这三层上的功能.

在Linux的源码中,网络设备驱动对应的逻辑位于driver/net/ethernet,其中Intel系列网卡的驱动在driver/net/ethernetintel目录下,协议栈模块代码位于kerel和net目录下.

内核和网络设备驱动是通过中断的方式来处理数据包的. 2.4以后的Linux内核版本采用的下半部实现方式是软中断, 由ksoftirad内核线程全权处理. 硬中断是通过给CPU物理引1脚施加电压变化实现的,而软中断是通过给内存中的一个变量赋予二进制值以标记有软中断发生.

kernel不涉及L4之上的各层, 这些层的任务由用户空间应用来处理. 同时kernel不涉及物理层(L1).

kernel涉及的L2,L3,L4分别对应OSI 7层model的数据链路层,网络层,传输层. 从本质上讲, linux网络栈的任务就是将接收到的数据包从L2(网络设备驱动程序)传递给L3(网络层, 通常是ipv4/ipv6). 接下来, 如果数据包的目的地是当前设备, 它就将数据包传递到L4(传输层);如果需要转发, 就将数据包还给L2进行传输. 对于本地生成的出站数据包将从L4依次传递给L3, L2, 再由网络设备驱动程序进行传输.

决定数据包在网络栈中传输过程的因素并非只有路由子系统的查找结果. ebpf/nftables/netfilter钩子也可以, IPsec子系统也可以, ipv4/ipv6的ttl.

网络设备驱动的体系结构:
1. 网络协议接口层

  网络协议接口层向网络层协议提供统一的数据包收发接口，不论上层协议是ARP还是IP，都通过dev_queue_xmit()函数发送数据，并通过netif_rx()函数接收数据. 这一层的存在使得上层协议独立于具体的设备.

2. 网络设备接口层

  网络设备接口层向协议接口层提供的用于描述具体网络设备属性和操作的结构体net_device，该结构体是设备驱动功能层各函数的容器. 网络设备接口层从宏观上规划了具体操作硬件的设备驱动功能层的结构.

3. 设备驱动功能层

  设备驱动功能层的各函数是网络设备接口层net_device数据结构的具体成员，是驱使网络设备硬件完成相应动作的程序，它通过hard_start_xmit()函数启动发送操作，并通过网络设备上的中断触发接收操作

4. 网络设备与媒介层

  网络设备与媒介层完成数据包发送和接收的物理实体，包括网络适配器和具体的传输媒介，网络适配器被设备驱动功能层中的函数在物理上驱动. 在linux上网络设备和媒介都可以是虚拟的.

通常, 网络设备驱动以中断方式接收数据包, 而poll_controller()采用纯轮询方式, 另一种则是NAPI(New API). NAPI接收流程: 接收中断来临->关闭接收中断->以轮询方式接收所有数据包直到收空->开启接收中断->接收中断来临....


linux网络栈处理期间可能有以下行为:
1. 根据协议规则(IPsec, NAT等), 可能需要对数据包进行修改
1. 数据包可能被丢弃
1. 数据包可能导致设备发送错误消息
1. 可能对数据包进行分段
1. 可能需要重组数据包
1. 需要计算数据包的checksum

应用层和内核互通的机制，就是通过 Socket 系统调用. socket属于操作系统的概念，而非网络协议分层的概念. 只不过操作系统选择对于网络协议的实现模式是，二到四层的处理代码在内核里面，七层的处理代码让应用自己去做，两者需要跨内核态和用户态通信，就需要一个系统调用完成这个衔接，这就是 Socket.

> rsockets is a protocol over RDMA that supports a socket-level API for applications.

![](/misc/img/net/98a4496fff94eb02d1b1b8ae88f8dc28.jpeg)

从本质上来讲，所谓的建立连接，其实是为了在客户端和服务端维护连接，而建立一定的数据结构来维护双方交互的状态，并用这样的数据结构来保证面向连接的特性. TCP 无法左右中间的任何通路，也没有什么虚拟的连接，中间的通路根本意识不到两端使用了 TCP 还是 UDP.

所谓的连接，就是两端数据结构状态的协同，两边的状态能够对得上。符合 TCP 协议的规则，就认为连接存在；两面状态对不上，连接就算断了.

流量控制和拥塞控制其实就是根据收到的对端的网络包，调整两端数据结构的状态. TCP 协议的设计理论上认为，这样调整了数据结构的状态，就能进行流量控制和拥塞控制了，其实在通路上是不是真的做到了，谁也管不着.

所谓的可靠，也是两端的数据结构做的事情. 不丢失其实是数据结构在“点名”，顺序到达其实是数据结构在“排序”，面向数据流其实是数据结构将零散的包，按照顺序捏成一个流发给应用层.

总而言之，“连接”两个字让人误以为功夫在通路，其实功夫在两端.

无论是用 socket 操作 TCP，还是 UDP，我们首先都要调用`nt socket(int domain, int type, int protocol)`.

socket 函数用于创建一个 socket 的文件描述符，唯一标识一个 socket. 把它叫作文件描述符，是因为在内核会创建类似文件系统的数据结构，并且后续的操作都有用到它.

socket 函数有三个参数:
- domain：表示使用什么 IP 层协议

    AF_INET 表示 IPv4，AF_INET6 表示 IPv6

- type：表示 socket 类型

    SOCK_STREAM，顾名思义就是 TCP 面向流的，SOCK_DGRAM 就是 UDP 面向数据报的，SOCK_RAW 可以直接操作 IP 层，或者非 TCP 和 UDP 的协议. 例如 ICMP

- protocol 表示的协议，包括 IPPROTO_TCP、IPPTOTO_UDP

通信结束后，还要像关闭文件一样，关闭 socket.

在 Linux 操作系统下，对套接字、套接字的属性、套接字传输的数据格式还有管理套接字连接状态的数据结构分别做了一系列抽象定义.

每个程序使用的套接字都有一个 struct socket 数据结构与 struct sock 数据结构的实例.

Linux 内核在套接字层定义了包含套接字通用属性的数据结构，分别是 struct socket 与 struct sock，它们独立于具体协议；而具体的协议族与协议实例继承了通用套接字的属性，加入协议相关属性，就形成了管理协议本身套接字的结构.

```c
// https://elixir.bootlin.com/linux/v6.6.22/source/include/linux/skbuff.h#L842
/* 套接字缓冲区, 用于在linux网络子系统的各层间传递数据
   分配sk_buff:
   - alloc_skb
   - dev_alloc_skb: 以GFP_ATOMIC优先级进行skb的分配, 原因是该函数经常在设备驱动的接收中断里被调用
   释放:
   - kfree_skb
   - dev_kfree_skb: 用于非中断上下文
   - dev_kfree_skb_irq: 用于中断上下文
   - dev_kfree_skb_any: 用于中断/非中断上下文, 它其实就是做了一个简单上下文判断, 然后调用kfree_skb或dev_kfree_skb_irq

   > 内核内部使用kfree_skb, 网络设备驱动中则最好使用其他3个

   变更:
   - skb_put: 在缓冲区尾部添加数据. 通常在设备驱动接收数据处理中会调用该函数.
   - skb_push: 在缓冲区开头添加数据
   - skb_pull: 与skb_push对应, 在缓冲区开头移除数据
   - skb_reseve: 对于一个空缓冲区, 该函数可调整缓冲区的头部

*/
/**
 * DOC: Basic sk_buff geometry
 *
 * struct sk_buff itself is a metadata structure and does not hold any packet
 * data. All the data is held in associated buffers.
 *
 * &sk_buff.head points to the main "head" buffer. The head buffer is divided
 * into two parts:
 *
 *  - data buffer, containing headers and sometimes payload;
 *    this is the part of the skb operated on by the common helpers
 *    such as skb_put() or skb_pull();
 *  - shared info (struct skb_shared_info) which holds an array of pointers
 *    to read-only data in the (page, offset, length) format.
 *
 * Optionally &skb_shared_info.frag_list may point to another skb.
 *
 * Basic diagram may look like this::
 *
 *                                  ---------------
 *                                 | sk_buff       |
 *                                  ---------------
 *     ,---------------------------  + head
 *    /          ,-----------------  + data
 *   /          /      ,-----------  + tail
 *  |          |      |            , + end
 *  |          |      |           |
 *  v          v      v           v
 *   -----------------------------------------------
 *  | headroom | data |  tailroom | skb_shared_info |
 *   -----------------------------------------------
 *                                 + [page frag]
 *                                 + [page frag]
 *                                 + [page frag]
 *                                 + [page frag]       ---------
 *                                 + frag_list    --> | sk_buff |
 *                                                     ---------
 *
 */

/**
 *  struct sk_buff - socket buffer
 *  @next: Next buffer in list
 *  @prev: Previous buffer in list
 *  @tstamp: Time we arrived/left
 *  @skb_mstamp_ns: (aka @tstamp) earliest departure time; start point
 *    for retransmit timer
 *  @rbnode: RB tree node, alternative to next/prev for netem/tcp
 *  @list: queue head
 *  @ll_node: anchor in an llist (eg socket defer_list)
 *  @sk: Socket we are owned by
 *  @ip_defrag_offset: (aka @sk) alternate use of @sk, used in
 *    fragmentation management
 *  @dev: Device we arrived on/are leaving by
 *  @dev_scratch: (aka @dev) alternate use of @dev when @dev would be %NULL
 *  @cb: Control buffer. Free for use by every layer. Put private vars here
 *  @_skb_refdst: destination entry (with norefcount bit)
 *  @sp: the security path, used for xfrm
 *  @len: Length of actual data
 *  @data_len: Data length
 *  @mac_len: Length of link layer header
 *  @hdr_len: writable header length of cloned skb
 *  @csum: Checksum (must include start/offset pair)
 *  @csum_start: Offset from skb->head where checksumming should start
 *  @csum_offset: Offset from csum_start where checksum should be stored
 *  @priority: Packet queueing priority
 *  @ignore_df: allow local fragmentation
 *  @cloned: Head may be cloned (check refcnt to be sure)
 *  @ip_summed: Driver fed us an IP checksum
 *  @nohdr: Payload reference only, must not modify header
 *  @pkt_type: Packet class
 *  @fclone: skbuff clone status
 *  @ipvs_property: skbuff is owned by ipvs
 *  @inner_protocol_type: whether the inner protocol is
 *    ENCAP_TYPE_ETHER or ENCAP_TYPE_IPPROTO
 *  @remcsum_offload: remote checksum offload is enabled
 *  @offload_fwd_mark: Packet was L2-forwarded in hardware
 *  @offload_l3_fwd_mark: Packet was L3-forwarded in hardware
 *  @tc_skip_classify: do not classify packet. set by IFB device
 *  @tc_at_ingress: used within tc_classify to distinguish in/egress
 *  @redirected: packet was redirected by packet classifier
 *  @from_ingress: packet was redirected from the ingress path
 *  @nf_skip_egress: packet shall skip nf egress - see netfilter_netdev.h
 *  @peeked: this packet has been seen already, so stats have been
 *    done for it, don't do them again
 *  @nf_trace: netfilter packet trace flag
 *  @protocol: Packet protocol from driver
 *  @destructor: Destruct function
 *  @tcp_tsorted_anchor: list structure for TCP (tp->tsorted_sent_queue)
 *  @_sk_redir: socket redirection information for skmsg
 *  @_nfct: Associated connection, if any (with nfctinfo bits)
 *  @nf_bridge: Saved data about a bridged frame - see br_netfilter.c
 *  @skb_iif: ifindex of device we arrived on
 *  @tc_index: Traffic control index
 *  @hash: the packet hash
 *  @queue_mapping: Queue mapping for multiqueue devices
 *  @head_frag: skb was allocated from page fragments,
 *    not allocated by kmalloc() or vmalloc().
 *  @pfmemalloc: skbuff was allocated from PFMEMALLOC reserves
 *  @pp_recycle: mark the packet for recycling instead of freeing (implies
 *    page_pool support on driver)
 *  @active_extensions: active extensions (skb_ext_id types)
 *  @ndisc_nodetype: router type (from link layer)
 *  @ooo_okay: allow the mapping of a socket to a queue to be changed
 *  @l4_hash: indicate hash is a canonical 4-tuple hash over transport
 *    ports.
 *  @sw_hash: indicates hash was computed in software stack
 *  @wifi_acked_valid: wifi_acked was set
 *  @wifi_acked: whether frame was acked on wifi or not
 *  @no_fcs:  Request NIC to treat last 4 bytes as Ethernet FCS
 *  @encapsulation: indicates the inner headers in the skbuff are valid
 *  @encap_hdr_csum: software checksum is needed
 *  @csum_valid: checksum is already valid
 *  @csum_not_inet: use CRC32c to resolve CHECKSUM_PARTIAL
 *  @csum_complete_sw: checksum was completed by software
 *  @csum_level: indicates the number of consecutive checksums found in
 *    the packet minus one that have been verified as
 *    CHECKSUM_UNNECESSARY (max 3)
 *  @dst_pending_confirm: need to confirm neighbour
 *  @decrypted: Decrypted SKB
 *  @slow_gro: state present at GRO time, slower prepare step required
 *  @mono_delivery_time: When set, skb->tstamp has the
 *    delivery_time in mono clock base (i.e. EDT).  Otherwise, the
 *    skb->tstamp has the (rcv) timestamp at ingress and
 *    delivery_time at egress.
 *  @napi_id: id of the NAPI struct this skb came from
 *  @sender_cpu: (aka @napi_id) source CPU in XPS
 *  @alloc_cpu: CPU which did the skb allocation.
 *  @secmark: security marking
 *  @mark: Generic packet mark
 *  @reserved_tailroom: (aka @mark) number of bytes of free space available
 *    at the tail of an sk_buff
 *  @vlan_all: vlan fields (proto & tci)
 *  @vlan_proto: vlan encapsulation protocol
 *  @vlan_tci: vlan tag control information
 *  @inner_protocol: Protocol (encapsulation)
 *  @inner_ipproto: (aka @inner_protocol) stores ipproto when
 *    skb->inner_protocol_type == ENCAP_TYPE_IPPROTO;
 *  @inner_transport_header: Inner transport layer header (encapsulation)
 *  @inner_network_header: Network layer header (encapsulation)
 *  @inner_mac_header: Link layer header (encapsulation)
 *  @transport_header: Transport layer header
 *  @network_header: Network layer header
 *  @mac_header: Link layer header
 *  @kcov_handle: KCOV remote handle for remote coverage collection
 *  @tail: Tail pointer
 *  @end: End pointer
 *  @head: Head of buffer
 *  @data: Data head pointer
 *  @truesize: Buffer size
 *  @users: User count - see {datagram,tcp}.c
 *  @extensions: allocated extensions, valid if active_extensions is nonzero
 */

struct sk_buff {
  union {
    struct {
      /* These two members must be first to match sk_buff_head. */
      struct sk_buff    *next;
      struct sk_buff    *prev;

      union {
        struct net_device *dev;
        /* Some protocols might use this space to store information,
         * while device pointer would be NULL.
         * UDP receive path is one user.
         */
        unsigned long   dev_scratch;
      };
    };
    struct rb_node    rbnode; /* used in netem, ip4 defrag, and tcp stack */
    struct list_head  list;
    struct llist_node ll_node;
  };

  union {
    struct sock   *sk;
    int     ip_defrag_offset;
  };

  union {
    ktime_t   tstamp;
    u64   skb_mstamp_ns; /* earliest departure time */
  };
  /*
   * This is the control buffer. It is free to use for every
   * layer. Please put your private variables there. If you
   * want to keep them across layers you have to do a skb_clone()
   * first. This is owned by whoever has the skb queued ATM.
   */
  char      cb[48] __aligned(8);

  union {
    struct {
      unsigned long _skb_refdst;
      void    (*destructor)(struct sk_buff *skb);
    };
    struct list_head  tcp_tsorted_anchor;
#ifdef CONFIG_NET_SOCK_MSG
    unsigned long   _sk_redir;
#endif
  };

#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
  unsigned long    _nfct;
#endif
  unsigned int    len,
        data_len;
  __u16     mac_len,
        hdr_len;

  /* Following fields are _not_ copied in __copy_skb_header()
   * Note that queue_mapping is here mostly to fill a hole.
   */
  __u16     queue_mapping;

/* if you move cloned around you also must adapt those constants */
#ifdef __BIG_ENDIAN_BITFIELD
#define CLONED_MASK (1 << 7)
#else
#define CLONED_MASK 1
#endif
#define CLONED_OFFSET   offsetof(struct sk_buff, __cloned_offset)

  /* private: */
  __u8      __cloned_offset[0];
  /* public: */
  __u8      cloned:1,
        nohdr:1,
        fclone:2,
        peeked:1,
        head_frag:1,
        pfmemalloc:1,
        pp_recycle:1; /* page_pool recycle indicator */
#ifdef CONFIG_SKB_EXTENSIONS
  __u8      active_extensions;
#endif

  /* Fields enclosed in headers group are copied
   * using a single memcpy() in __copy_skb_header()
   */
  struct_group(headers,

  /* private: */
  __u8      __pkt_type_offset[0];
  /* public: */
  __u8      pkt_type:3; /* see PKT_TYPE_MAX */
  __u8      ignore_df:1;
  __u8      dst_pending_confirm:1;
  __u8      ip_summed:2;
  __u8      ooo_okay:1;

  /* private: */
  __u8      __mono_tc_offset[0];
  /* public: */
  __u8      mono_delivery_time:1; /* See SKB_MONO_DELIVERY_TIME_MASK */
#ifdef CONFIG_NET_XGRESS
  __u8      tc_at_ingress:1;  /* See TC_AT_INGRESS_MASK */
  __u8      tc_skip_classify:1;
#endif
  __u8      remcsum_offload:1;
  __u8      csum_complete_sw:1;
  __u8      csum_level:2;
  __u8      inner_protocol_type:1;

  __u8      l4_hash:1;
  __u8      sw_hash:1;
#ifdef CONFIG_WIRELESS
  __u8      wifi_acked_valid:1;
  __u8      wifi_acked:1;
#endif
  __u8      no_fcs:1;
  /* Indicates the inner headers are valid in the skbuff. */
  __u8      encapsulation:1;
  __u8      encap_hdr_csum:1;
  __u8      csum_valid:1;
#ifdef CONFIG_IPV6_NDISC_NODETYPE
  __u8      ndisc_nodetype:2;
#endif

#if IS_ENABLED(CONFIG_IP_VS)
  __u8      ipvs_property:1;
#endif
#if IS_ENABLED(CONFIG_NETFILTER_XT_TARGET_TRACE) || IS_ENABLED(CONFIG_NF_TABLES)
  __u8      nf_trace:1;
#endif
#ifdef CONFIG_NET_SWITCHDEV
  __u8      offload_fwd_mark:1;
  __u8      offload_l3_fwd_mark:1;
#endif
  __u8      redirected:1;
#ifdef CONFIG_NET_REDIRECT
  __u8      from_ingress:1;
#endif
#ifdef CONFIG_NETFILTER_SKIP_EGRESS
  __u8      nf_skip_egress:1;
#endif
#ifdef CONFIG_TLS_DEVICE
  __u8      decrypted:1;
#endif
  __u8      slow_gro:1;
#if IS_ENABLED(CONFIG_IP_SCTP)
  __u8      csum_not_inet:1;
#endif

#if defined(CONFIG_NET_SCHED) || defined(CONFIG_NET_XGRESS)
  __u16     tc_index; /* traffic control index */
#endif

  u16     alloc_cpu;

  union {
    __wsum    csum;
    struct {
      __u16 csum_start;
      __u16 csum_offset;
    };
  };
  __u32     priority;
  int     skb_iif;
  __u32     hash;
  union {
    u32   vlan_all;
    struct {
      __be16  vlan_proto;
      __u16 vlan_tci;
    };
  };
#if defined(CONFIG_NET_RX_BUSY_POLL) || defined(CONFIG_XPS)
  union {
    unsigned int  napi_id;
    unsigned int  sender_cpu;
  };
#endif
#ifdef CONFIG_NETWORK_SECMARK
  __u32   secmark;
#endif

  union {
    __u32   mark;
    __u32   reserved_tailroom;
  };

  union {
    __be16    inner_protocol;
    __u8    inner_ipproto;
  };

  __u16     inner_transport_header;
  __u16     inner_network_header;
  __u16     inner_mac_header;

  __be16      protocol;
  __u16     transport_header;
  __u16     network_header;
  __u16     mac_header;

#ifdef CONFIG_KCOV
  u64     kcov_handle;
#endif

  ); /* end headers group */

  /* These elements must be at the end, see alloc_skb() for details.  */
  sk_buff_data_t    tail;
  sk_buff_data_t    end;
  unsigned char   *head,
        *data;
  unsigned int    truesize;
  refcount_t    users;

#ifdef CONFIG_SKB_EXTENSIONS
  /* only useable after checking ->active_extensions != 0 */
  struct skb_ext    *extensions;
#endif
};

// https://elixir.bootlin.com/linux/v6.6.22/source/include/linux/netdevice.h#L2067
// netdev_pri: 获取net_device的私有成员. 该结构体可包含设备的特色属性和操作, 自旋锁, 信号量, 定时器, 统计信息等. 参考[dm9000_start_xmit](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/net/ethernet/davicom/dm9000.c#L1015)
/**
 *  struct net_device - The DEVICE structure.
 *
 *  Actually, this whole structure is a big mistake.  It mixes I/O
 *  data with strictly "high-level" data, and it has to know about
 *  almost every data structure used in the INET module.
 *
 *  @name:  This is the first field of the "visible" part of this structure
 *    (i.e. as seen by users in the "Space.c" file).  It is the name
 *    of the interface.
 *
 *  @name_node: Name hashlist node
 *  @ifalias: SNMP alias
 *  @mem_end: Shared memory end
 *  @mem_start: Shared memory start
 *  @base_addr: Device I/O address
 *  @irq:   Device IRQ number
 *
 *  @state:   Generic network queuing layer state, see netdev_state_t
 *  @dev_list:  The global list of network devices
 *  @napi_list: List entry used for polling NAPI devices
 *  @unreg_list:  List entry  when we are unregistering the
 *      device; see the function unregister_netdev
 *  @close_list:  List entry used when we are closing the device
 *  @ptype_all:     Device-specific packet handlers for all protocols
 *  @ptype_specific: Device-specific, protocol-specific packet handlers
 *
 *  @adj_list:  Directly linked devices, like slaves for bonding
 *  @features:  Currently active device features
 *  @hw_features: User-changeable features
 *
 *  @wanted_features: User-requested features
 *  @vlan_features:   Mask of features inheritable by VLAN devices
 *
 *  @hw_enc_features: Mask of features inherited by encapsulating devices
 *        This field indicates what encapsulation
 *        offloads the hardware is capable of doing,
 *        and drivers will need to set them appropriately.
 *
 *  @mpls_features: Mask of features inheritable by MPLS
 *  @gso_partial_features: value(s) from NETIF_F_GSO\*
 *
 *  @ifindex: interface index
 *  @group:   The group the device belongs to
 *
 *  @stats:   Statistics struct, which was left as a legacy, use
 *      rtnl_link_stats64 instead
 *
 *  @core_stats:  core networking counters,
 *      do not use this in drivers
 *  @carrier_up_count:  Number of times the carrier has been up
 *  @carrier_down_count:  Number of times the carrier has been down
 *
 *  @wireless_handlers: List of functions to handle Wireless Extensions,
 *        instead of ioctl,
 *        see <net/iw_handler.h> for details.
 *  @wireless_data: Instance data managed by the core of wireless extensions
 *
 *  @netdev_ops:  Includes several pointers to callbacks,
 *      if one wants to override the ndo_*() functions
 *  @xdp_metadata_ops:  Includes pointers to XDP metadata callbacks.
 *  @ethtool_ops: Management operations
 *  @l3mdev_ops:  Layer 3 master device operations
 *  @ndisc_ops: Includes callbacks for different IPv6 neighbour
 *      discovery handling. Necessary for e.g. 6LoWPAN.
 *  @xfrmdev_ops: Transformation offload operations
 *  @tlsdev_ops:  Transport Layer Security offload operations
 *  @header_ops:  Includes callbacks for creating,parsing,caching,etc
 *      of Layer 2 headers.
 *
 *  @flags:   Interface flags (a la BSD)
 *  @xdp_features:  XDP capability supported by the device
 *  @priv_flags:  Like 'flags' but invisible to userspace,
 *      see if.h for the definitions
 *  @gflags:  Global flags ( kept as legacy )
 *  @padded:  How much padding added by alloc_netdev()
 *  @operstate: RFC2863 operstate
 *  @link_mode: Mapping policy to operstate
 *  @if_port: Selectable AUI, TP, ...
 *  @dma:   DMA channel
 *  @mtu:   Interface MTU value
 *  @min_mtu: Interface Minimum MTU value
 *  @max_mtu: Interface Maximum MTU value
 *  @type:    Interface hardware type
 *  @hard_header_len: Maximum hardware header length.
 *  @min_header_len:  Minimum hardware header length
 *
 *  @needed_headroom: Extra headroom the hardware may need, but not in all
 *        cases can this be guaranteed
 *  @needed_tailroom: Extra tailroom the hardware may need, but not in all
 *        cases can this be guaranteed. Some cases also use
 *        LL_MAX_HEADER instead to allocate the skb
 *
 *  interface address info:
 *
 *  @perm_addr:   Permanent hw address
 *  @addr_assign_type:  Hw address assignment type
 *  @addr_len:    Hardware address length
 *  @upper_level:   Maximum depth level of upper devices.
 *  @lower_level:   Maximum depth level of lower devices.
 *  @neigh_priv_len:  Used in neigh_alloc()
 *  @dev_id:    Used to differentiate devices that share
 *        the same link layer address
 *  @dev_port:    Used to differentiate devices that share
 *        the same function
 *  @addr_list_lock:  XXX: need comments on this one
 *  @name_assign_type:  network interface name assignment type
 *  @uc_promisc:    Counter that indicates promiscuous mode
 *        has been enabled due to the need to listen to
 *        additional unicast addresses in a device that
 *        does not implement ndo_set_rx_mode()
 *  @uc:      unicast mac addresses
 *  @mc:      multicast mac addresses
 *  @dev_addrs:   list of device hw addresses
 *  @queues_kset:   Group of all Kobjects in the Tx and RX queues
 *  @promiscuity:   Number of times the NIC is told to work in
 *        promiscuous mode; if it becomes 0 the NIC will
 *        exit promiscuous mode
 *  @allmulti:    Counter, enables or disables allmulticast mode
 *
 *  @vlan_info: VLAN info
 *  @dsa_ptr: dsa specific data
 *  @tipc_ptr:  TIPC specific data
 *  @atalk_ptr: AppleTalk link
 *  @ip_ptr:  IPv4 specific data
 *  @ip6_ptr: IPv6 specific data
 *  @ax25_ptr:  AX.25 specific data
 *  @ieee80211_ptr: IEEE 802.11 specific data, assign before registering
 *  @ieee802154_ptr: IEEE 802.15.4 low-rate Wireless Personal Area Network
 *       device struct
 *  @mpls_ptr:  mpls_dev struct pointer
 *  @mctp_ptr:  MCTP specific data
 *
 *  @dev_addr:  Hw address (before bcast,
 *      because most packets are unicast)
 *
 *  @_rx:     Array of RX queues
 *  @num_rx_queues:   Number of RX queues
 *        allocated at register_netdev() time
 *  @real_num_rx_queues:  Number of RX queues currently active in device
 *  @xdp_prog:    XDP sockets filter program pointer
 *  @gro_flush_timeout: timeout for GRO layer in NAPI
 *  @napi_defer_hard_irqs:  If not zero, provides a counter that would
 *        allow to avoid NIC hard IRQ, on busy queues.
 *
 *  @rx_handler:    handler for received packets
 *  @rx_handler_data:   XXX: need comments on this one
 *  @tcx_ingress:   BPF & clsact qdisc specific data for ingress processing
 *  @ingress_queue:   XXX: need comments on this one
 *  @nf_hooks_ingress:  netfilter hooks executed for ingress packets
 *  @broadcast:   hw bcast address
 *
 *  @rx_cpu_rmap: CPU reverse-mapping for RX completion interrupts,
 *      indexed by RX queue number. Assigned by driver.
 *      This must only be set if the ndo_rx_flow_steer
 *      operation is defined
 *  @index_hlist:   Device index hash chain
 *
 *  @_tx:     Array of TX queues
 *  @num_tx_queues:   Number of TX queues allocated at alloc_netdev_mq() time
 *  @real_num_tx_queues:  Number of TX queues currently active in device
 *  @qdisc:     Root qdisc from userspace point of view
 *  @tx_queue_len:    Max frames per queue allowed
 *  @tx_global_lock:  XXX: need comments on this one
 *  @xdp_bulkq:   XDP device bulk queue
 *  @xps_maps:    all CPUs/RXQs maps for XPS device
 *
 *  @xps_maps:  XXX: need comments on this one
 *  @tcx_egress:    BPF & clsact qdisc specific data for egress processing
 *  @nf_hooks_egress: netfilter hooks executed for egress packets
 *  @qdisc_hash:    qdisc hash table
 *  @watchdog_timeo:  Represents the timeout that is used by
 *        the watchdog (see dev_watchdog())
 *  @watchdog_timer:  List of timers
 *
 *  @proto_down_reason: reason a netdev interface is held down
 *  @pcpu_refcnt:   Number of references to this device
 *  @dev_refcnt:    Number of references to this device
 *  @refcnt_tracker:  Tracker directory for tracked references to this device
 *  @todo_list:   Delayed register/unregister
 *  @link_watch_list: XXX: need comments on this one
 *
 *  @reg_state:   Register/unregister state machine
 *  @dismantle:   Device is going to be freed
 *  @rtnl_link_state: This enum represents the phases of creating
 *        a new link
 *
 *  @needs_free_netdev: Should unregister perform free_netdev?
 *  @priv_destructor: Called from unregister
 *  @npinfo:    XXX: need comments on this one
 *  @nd_net:    Network namespace this network device is inside
 *
 *  @ml_priv: Mid-layer private
 *  @ml_priv_type:  Mid-layer private type
 *
 *  @pcpu_stat_type:  Type of device statistics which the core should
 *        allocate/free: none, lstats, tstats, dstats. none
 *        means the driver is handling statistics allocation/
 *        freeing internally.
 *  @lstats:    Loopback statistics: packets, bytes
 *  @tstats:    Tunnel statistics: RX/TX packets, RX/TX bytes
 *  @dstats:    Dummy statistics: RX/TX/drop packets, RX/TX bytes
 *
 *  @garp_port: GARP
 *  @mrp_port:  MRP
 *
 *  @dm_private:  Drop monitor private
 *
 *  @dev:   Class/net/name entry
 *  @sysfs_groups:  Space for optional device, statistics and wireless
 *      sysfs groups
 *
 *  @sysfs_rx_queue_group:  Space for optional per-rx queue attributes
 *  @rtnl_link_ops: Rtnl_link_ops
 *
 *  @gso_max_size:  Maximum size of generic segmentation offload
 *  @tso_max_size:  Device (as in HW) limit on the max TSO request size
 *  @gso_max_segs:  Maximum number of segments that can be passed to the
 *      NIC for GSO
 *  @tso_max_segs:  Device (as in HW) limit on the max TSO segment count
 *  @gso_ipv4_max_size: Maximum size of generic segmentation offload,
 *        for IPv4.
 *
 *  @dcbnl_ops: Data Center Bridging netlink ops
 *  @num_tc:  Number of traffic classes in the net device
 *  @tc_to_txq: XXX: need comments on this one
 *  @prio_tc_map: XXX: need comments on this one
 *
 *  @fcoe_ddp_xid:  Max exchange id for FCoE LRO by ddp
 *
 *  @priomap: XXX: need comments on this one
 *  @phydev:  Physical device may attach itself
 *      for hardware timestamping
 *  @sfp_bus: attached &struct sfp_bus structure.
 *
 *  @qdisc_tx_busylock: lockdep class annotating Qdisc->busylock spinlock
 *
 *  @proto_down:  protocol port state information can be sent to the
 *      switch driver and used to set the phys state of the
 *      switch port.
 *
 *  @wol_enabled: Wake-on-LAN is enabled
 *
 *  @threaded:  napi threaded mode is enabled
 *
 *  @net_notifier_list: List of per-net netdev notifier block
 *        that follow this device when it is moved
 *        to another network namespace.
 *
 *  @macsec_ops:    MACsec offloading ops
 *
 *  @udp_tunnel_nic_info: static structure describing the UDP tunnel
 *        offload capabilities of the device
 *  @udp_tunnel_nic:  UDP tunnel offload state
 *  @xdp_state:   stores info on attached XDP BPF programs
 *
 *  @nested_level:  Used as a parameter of spin_lock_nested() of
 *      dev->addr_list_lock.
 *  @unlink_list: As netif_addr_lock() can be called recursively,
 *      keep a list of interfaces to be deleted.
 *  @gro_max_size:  Maximum size of aggregated packet in generic
 *      receive offload (GRO)
 *  @gro_ipv4_max_size: Maximum size of aggregated packet in generic
 *        receive offload (GRO), for IPv4.
 *  @xdp_zc_max_segs: Maximum number of segments supported by AF_XDP
 *        zero copy driver
 *
 *  @dev_addr_shadow: Copy of @dev_addr to catch direct writes.
 *  @linkwatch_dev_tracker: refcount tracker used by linkwatch.
 *  @watchdog_dev_tracker:  refcount tracker used by watchdog.
 *  @dev_registered_tracker:  tracker for reference held while
 *          registered
 *  @offload_xstats_l3: L3 HW stats for this netdevice.
 *
 *  @devlink_port:  Pointer to related devlink port structure.
 *      Assigned by a driver before netdev registration using
 *      SET_NETDEV_DEVLINK_PORT macro. This pointer is static
 *      during the time netdevice is registered.
 *
 *  FIXME: cleanup struct net_device such that network protocol info
 *  moves out.
 */

struct net_device {
  char      name[IFNAMSIZ]; // name
  struct netdev_name_node *name_node;
  struct dev_ifalias  __rcu *ifalias;
  /*
   *  I/O specific fields
   *  FIXME: Merge these and struct ifmap into one
   */
  unsigned long   mem_end;
  unsigned long   mem_start; // 定义了设备所使用的共享内存的开始和结束(mem_end)地址
  unsigned long   base_addr; // 网络设备I/O基地址

  /*
   *  Some hardware also needs these fields (state,dev_list,
   *  napi_list,unreg_list,close_list) but they are not
   *  part of the usual set specified in Space.c.
   */

  unsigned long   state;

  struct list_head  dev_list;
  struct list_head  napi_list;
  struct list_head  unreg_list;
  struct list_head  close_list;
  struct list_head  ptype_all;
  struct list_head  ptype_specific;

  struct {
    struct list_head upper;
    struct list_head lower;
  } adj_list;

  /* Read-mostly cache-line for fast-path access */
  unsigned int    flags; // 部分标志由内核管理, 其他的在接口初始化时被设置以说明接口的能力和特性, 见`ip -j addr`的输出的flags. UP: 设备被激活并开始发送数据包; BROADCAST: 允许广播, LOOPBACK: 回环; MULTICAST, 允许组播.
  xdp_features_t    xdp_features;
  unsigned long long  priv_flags;
  const struct net_device_ops *netdev_ops; // 设备操作函数集合
  const struct xdp_metadata_ops *xdp_metadata_ops;
  int     ifindex;
  unsigned short    gflags;
  unsigned short    hard_header_len; // 网络设备的硬件头长度. 在以太网设备的初始化函数中被设为ETH_HLEN=4

  /* Note : dev->mtu is often read without holding a lock.
   * Writers usually hold RTNL.
   * It is recommended to use READ_ONCE() to annotate the reads,
   * and to use WRITE_ONCE() to annotate the writes.
   */
  unsigned int    mtu; // MTU(最大传输单元)
  unsigned short    needed_headroom;
  unsigned short    needed_tailroom;

  netdev_features_t features;
  netdev_features_t hw_features;
  netdev_features_t wanted_features;
  netdev_features_t vlan_features;
  netdev_features_t hw_enc_features;
  netdev_features_t mpls_features;
  netdev_features_t gso_partial_features;

  unsigned int    min_mtu;
  unsigned int    max_mtu;
  unsigned short    type; // 接口的硬件类型
  unsigned char   min_header_len;
  unsigned char   name_assign_type;

  int     group;

  struct net_device_stats stats; /* not used by modern drivers */

  struct net_device_core_stats __percpu *core_stats;

  /* Stats to monitor link on/off, flapping */
  atomic_t    carrier_up_count;
  atomic_t    carrier_down_count;

#ifdef CONFIG_WIRELESS_EXT
  const struct iw_handler_def *wireless_handlers;
  struct iw_public_data *wireless_data;
#endif
  const struct ethtool_ops *ethtool_ops; // 对应ethtool工具的操作
#ifdef CONFIG_NET_L3_MASTER_DEV
  const struct l3mdev_ops *l3mdev_ops;
#endif
#if IS_ENABLED(CONFIG_IPV6)
  const struct ndisc_ops *ndisc_ops;
#endif

#ifdef CONFIG_XFRM_OFFLOAD
  const struct xfrmdev_ops *xfrmdev_ops;
#endif

#if IS_ENABLED(CONFIG_TLS_DEVICE)
  const struct tlsdev_ops *tlsdev_ops;
#endif

  const struct header_ops *header_ops; // 对应于硬件头部操作, 主要是完成创建硬件头部和从给定的sk_buff分析出硬件头部等操作

  unsigned char   operstate;
  unsigned char   link_mode;

  unsigned char   if_port; // 指定多端口设备使用哪一个端口, 仅针对多端口设备, 比如IF_PORT_10BASE2(同轴电缆)/IF_PORT_10BASET(双绞线)
  unsigned char   dma; // 分配给设备的DMA通道

  /* Interface address info. */
  unsigned char   perm_addr[MAX_ADDR_LEN];
  unsigned char   addr_assign_type;
  unsigned char   addr_len;
  unsigned char   upper_level;
  unsigned char   lower_level;

  unsigned short    neigh_priv_len;
  unsigned short          dev_id;
  unsigned short          dev_port;
  unsigned short    padded;

  spinlock_t    addr_list_lock;
  int     irq; // 设备使用的中断号

  struct netdev_hw_addr_list  uc;
  struct netdev_hw_addr_list  mc;
  struct netdev_hw_addr_list  dev_addrs;

#ifdef CONFIG_SYSFS
  struct kset   *queues_kset;
#endif
#ifdef CONFIG_LOCKDEP
  struct list_head  unlink_list;
#endif
  unsigned int    promiscuity;
  unsigned int    allmulti;
  bool      uc_promisc;
#ifdef CONFIG_LOCKDEP
  unsigned char   nested_level;
#endif


  /* Protocol-specific pointers */

  struct in_device __rcu  *ip_ptr;
  struct inet6_dev __rcu  *ip6_ptr;
#if IS_ENABLED(CONFIG_VLAN_8021Q)
  struct vlan_info __rcu  *vlan_info;
#endif
#if IS_ENABLED(CONFIG_NET_DSA)
  struct dsa_port   *dsa_ptr;
#endif
#if IS_ENABLED(CONFIG_TIPC)
  struct tipc_bearer __rcu *tipc_ptr;
#endif
#if IS_ENABLED(CONFIG_ATALK)
  void      *atalk_ptr;
#endif
#if IS_ENABLED(CONFIG_AX25)
  void      *ax25_ptr;
#endif
#if IS_ENABLED(CONFIG_CFG80211)
  struct wireless_dev *ieee80211_ptr;
#endif
#if IS_ENABLED(CONFIG_IEEE802154) || IS_ENABLED(CONFIG_6LOWPAN)
  struct wpan_dev   *ieee802154_ptr;
#endif
#if IS_ENABLED(CONFIG_MPLS_ROUTING)
  struct mpls_dev __rcu *mpls_ptr;
#endif
#if IS_ENABLED(CONFIG_MCTP)
  struct mctp_dev __rcu *mctp_ptr;
#endif

/*
 * Cache lines mostly used on receive path (including eth_type_trans())
 */
  /* Interface address info used in eth_type_trans() */
  const unsigned char *dev_addr; // 设备的硬件地址. 如果驱动提供了设置mac地址的接口, 那么就会保存在这里

  struct netdev_rx_queue  *_rx;
  unsigned int    num_rx_queues;
  unsigned int    real_num_rx_queues;

  struct bpf_prog __rcu *xdp_prog;
  unsigned long   gro_flush_timeout;
  int     napi_defer_hard_irqs;
#define GRO_LEGACY_MAX_SIZE 65536u
/* TCP minimal MSS is 8 (TCP_MIN_GSO_SIZE),
 * and shinfo->gso_segs is a 16bit field.
 */
#define GRO_MAX_SIZE    (8 * 65535u)
  unsigned int    gro_max_size;
  unsigned int    gro_ipv4_max_size;
  unsigned int    xdp_zc_max_segs;
  rx_handler_func_t __rcu *rx_handler;
  void __rcu    *rx_handler_data;
#ifdef CONFIG_NET_XGRESS
  struct bpf_mprog_entry __rcu *tcx_ingress;
#endif
  struct netdev_queue __rcu *ingress_queue;
#ifdef CONFIG_NETFILTER_INGRESS
  struct nf_hook_entries __rcu *nf_hooks_ingress;
#endif

  unsigned char   broadcast[MAX_ADDR_LEN];
#ifdef CONFIG_RFS_ACCEL
  struct cpu_rmap   *rx_cpu_rmap;
#endif
  struct hlist_node index_hlist;

/*
 * Cache lines mostly used on transmit path
 */
  struct netdev_queue *_tx ____cacheline_aligned_in_smp;
  unsigned int    num_tx_queues;
  unsigned int    real_num_tx_queues;
  struct Qdisc __rcu  *qdisc;
  unsigned int    tx_queue_len;
  spinlock_t    tx_global_lock;

  struct xdp_dev_bulk_queue __percpu *xdp_bulkq;

#ifdef CONFIG_XPS
  struct xps_dev_maps __rcu *xps_maps[XPS_MAPS_MAX];
#endif
#ifdef CONFIG_NET_XGRESS
  struct bpf_mprog_entry __rcu *tcx_egress;
#endif
#ifdef CONFIG_NETFILTER_EGRESS
  struct nf_hook_entries __rcu *nf_hooks_egress;
#endif

#ifdef CONFIG_NET_SCHED
  DECLARE_HASHTABLE (qdisc_hash, 4);
#endif
  /* These may be needed for future network-power-down code. */
  struct timer_list watchdog_timer;
  int     watchdog_timeo;

  u32                     proto_down_reason;

  struct list_head  todo_list;

#ifdef CONFIG_PCPU_DEV_REFCNT
  int __percpu    *pcpu_refcnt;
#else
  refcount_t    dev_refcnt;
#endif
  struct ref_tracker_dir  refcnt_tracker;

  struct list_head  link_watch_list;

  enum { NETREG_UNINITIALIZED=0,
         NETREG_REGISTERED, /* completed register_netdevice */
         NETREG_UNREGISTERING,  /* called unregister_netdevice */
         NETREG_UNREGISTERED, /* completed unregister todo */
         NETREG_RELEASED,   /* called free_netdev */
         NETREG_DUMMY,    /* dummy device for NAPI poll */
  } reg_state:8;

  bool dismantle;

  enum {
    RTNL_LINK_INITIALIZED,
    RTNL_LINK_INITIALIZING,
  } rtnl_link_state:16;

  bool needs_free_netdev;
  void (*priv_destructor)(struct net_device *dev);

#ifdef CONFIG_NETPOLL
  struct netpoll_info __rcu *npinfo;
#endif

  possible_net_t      nd_net;

  /* mid-layer private */
  void        *ml_priv;
  enum netdev_ml_priv_type  ml_priv_type;

  enum netdev_stat_type   pcpu_stat_type:8;
  union {
    struct pcpu_lstats __percpu   *lstats;
    struct pcpu_sw_netstats __percpu  *tstats;
    struct pcpu_dstats __percpu   *dstats;
  };

#if IS_ENABLED(CONFIG_GARP)
  struct garp_port __rcu  *garp_port;
#endif
#if IS_ENABLED(CONFIG_MRP)
  struct mrp_port __rcu *mrp_port;
#endif
#if IS_ENABLED(CONFIG_NET_DROP_MONITOR)
  struct dm_hw_stat_delta __rcu *dm_private;
#endif
  struct device   dev;
  const struct attribute_group *sysfs_groups[4];
  const struct attribute_group *sysfs_rx_queue_group;

  const struct rtnl_link_ops *rtnl_link_ops;

  /* for setting kernel sock attribute on TCP connection setup */
#define GSO_MAX_SEGS    65535u
#define GSO_LEGACY_MAX_SIZE 65536u
/* TCP minimal MSS is 8 (TCP_MIN_GSO_SIZE),
 * and shinfo->gso_segs is a 16bit field.
 */
#define GSO_MAX_SIZE    (8 * GSO_MAX_SEGS)

  unsigned int    gso_max_size;
#define TSO_LEGACY_MAX_SIZE 65536
#define TSO_MAX_SIZE    UINT_MAX
  unsigned int    tso_max_size;
  u16     gso_max_segs;
#define TSO_MAX_SEGS    U16_MAX
  u16     tso_max_segs;
  unsigned int    gso_ipv4_max_size;

#ifdef CONFIG_DCB
  const struct dcbnl_rtnl_ops *dcbnl_ops;
#endif
  s16     num_tc;
  struct netdev_tc_txq  tc_to_txq[TC_MAX_QUEUE];
  u8      prio_tc_map[TC_BITMASK + 1];

#if IS_ENABLED(CONFIG_FCOE)
  unsigned int    fcoe_ddp_xid;
#endif
#if IS_ENABLED(CONFIG_CGROUP_NET_PRIO)
  struct netprio_map __rcu *priomap;
#endif
  struct phy_device *phydev;
  struct sfp_bus    *sfp_bus;
  struct lock_class_key *qdisc_tx_busylock;
  bool      proto_down;
  unsigned    wol_enabled:1;
  unsigned    threaded:1;

  struct list_head  net_notifier_list;

#if IS_ENABLED(CONFIG_MACSEC)
  /* MACsec management functions */
  const struct macsec_ops *macsec_ops;
#endif
  const struct udp_tunnel_nic_info  *udp_tunnel_nic_info;
  struct udp_tunnel_nic *udp_tunnel_nic;

  /* protected by rtnl_lock */
  struct bpf_xdp_entity xdp_state[__MAX_XDP_MODE];

  u8 dev_addr_shadow[MAX_ADDR_LEN];
  netdevice_tracker linkwatch_dev_tracker;
  netdevice_tracker watchdog_dev_tracker;
  netdevice_tracker dev_registered_tracker;
  struct rtnl_hw_stats64  *offload_xstats_l3;

  struct devlink_port *devlink_port;
};
#define to_net_dev(d) container_of(d, struct net_device, dev)

// https://elixir.bootlin.com/linux/v6.6.22/source/include/linux/netdevice.h#L1410
/*
 * This structure defines the management hooks for network devices.
 * The following hooks can be defined; unless noted otherwise, they are
 * optional and can be filled with a null pointer.
 *
 * int (*ndo_init)(struct net_device *dev);
 *     This function is called once when a network device is registered.
 *     The network device can use this for any late stage initialization
 *     or semantic validation. It can fail with an error code which will
 *     be propagated back to register_netdev.
 *
 * void (*ndo_uninit)(struct net_device *dev);
 *     This function is called when device is unregistered or when registration
 *     fails. It is not called if init fails.
 *
 * int (*ndo_open)(struct net_device *dev);
 *     This function is called when a network device transitions to the up
 *     state.
 *
 * int (*ndo_stop)(struct net_device *dev);
 *     This function is called when a network device transitions to the down
 *     state.
 *
 * netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb,
 *                               struct net_device *dev);
 *  Called when a packet needs to be transmitted.
 *  Returns NETDEV_TX_OK.  Can return NETDEV_TX_BUSY, but you should stop
 *  the queue before that can happen; it's for obsolete devices and weird
 *  corner cases, but the stack really does a non-trivial amount
 *  of useless work if you return NETDEV_TX_BUSY.
 *  Required; cannot be NULL.
 *
 * netdev_features_t (*ndo_features_check)(struct sk_buff *skb,
 *             struct net_device *dev
 *             netdev_features_t features);
 *  Called by core transmit path to determine if device is capable of
 *  performing offload operations on a given packet. This is to give
 *  the device an opportunity to implement any restrictions that cannot
 *  be otherwise expressed by feature flags. The check is called with
 *  the set of features that the stack has calculated and it returns
 *  those the driver believes to be appropriate.
 *
 * u16 (*ndo_select_queue)(struct net_device *dev, struct sk_buff *skb,
 *                         struct net_device *sb_dev);
 *  Called to decide which queue to use when device supports multiple
 *  transmit queues.
 *
 * void (*ndo_change_rx_flags)(struct net_device *dev, int flags);
 *  This function is called to allow device receiver to make
 *  changes to configuration when multicast or promiscuous is enabled.
 *
 * void (*ndo_set_rx_mode)(struct net_device *dev);
 *  This function is called device changes address list filtering.
 *  If driver handles unicast address filtering, it should set
 *  IFF_UNICAST_FLT in its priv_flags.
 *
 * int (*ndo_set_mac_address)(struct net_device *dev, void *addr);
 *  This function  is called when the Media Access Control address
 *  needs to be changed. If this interface is not defined, the
 *  MAC address can not be changed.
 *
 * int (*ndo_validate_addr)(struct net_device *dev);
 *  Test if Media Access Control address is valid for the device.
 *
 * int (*ndo_do_ioctl)(struct net_device *dev, struct ifreq *ifr, int cmd);
 *  Old-style ioctl entry point. This is used internally by the
 *  appletalk and ieee802154 subsystems but is no longer called by
 *  the device ioctl handler.
 *
 * int (*ndo_siocbond)(struct net_device *dev, struct ifreq *ifr, int cmd);
 *  Used by the bonding driver for its device specific ioctls:
 *  SIOCBONDENSLAVE, SIOCBONDRELEASE, SIOCBONDSETHWADDR, SIOCBONDCHANGEACTIVE,
 *  SIOCBONDSLAVEINFOQUERY, and SIOCBONDINFOQUERY
 *
 * * int (*ndo_eth_ioctl)(struct net_device *dev, struct ifreq *ifr, int cmd);
 *  Called for ethernet specific ioctls: SIOCGMIIPHY, SIOCGMIIREG,
 *  SIOCSMIIREG, SIOCSHWTSTAMP and SIOCGHWTSTAMP.
 *
 * int (*ndo_set_config)(struct net_device *dev, struct ifmap *map);
 *  Used to set network devices bus interface parameters. This interface
 *  is retained for legacy reasons; new devices should use the bus
 *  interface (PCI) for low level management.
 *
 * int (*ndo_change_mtu)(struct net_device *dev, int new_mtu);
 *  Called when a user wants to change the Maximum Transfer Unit
 *  of a device.
 *
 * void (*ndo_tx_timeout)(struct net_device *dev, unsigned int txqueue);
 *  Callback used when the transmitter has not made any progress
 *  for dev->watchdog ticks.
 *
 * void (*ndo_get_stats64)(struct net_device *dev,
 *                         struct rtnl_link_stats64 *storage);
 * struct net_device_stats* (*ndo_get_stats)(struct net_device *dev);
 *  Called when a user wants to get the network device usage
 *  statistics. Drivers must do one of the following:
 *  1. Define @ndo_get_stats64 to fill in a zero-initialised
 *     rtnl_link_stats64 structure passed by the caller.
 *  2. Define @ndo_get_stats to update a net_device_stats structure
 *     (which should normally be dev->stats) and return a pointer to
 *     it. The structure may be changed asynchronously only if each
 *     field is written atomically.
 *  3. Update dev->stats asynchronously and atomically, and define
 *     neither operation.
 *
 * bool (*ndo_has_offload_stats)(const struct net_device *dev, int attr_id)
 *  Return true if this device supports offload stats of this attr_id.
 *
 * int (*ndo_get_offload_stats)(int attr_id, const struct net_device *dev,
 *  void *attr_data)
 *  Get statistics for offload operations by attr_id. Write it into the
 *  attr_data pointer.
 *
 * int (*ndo_vlan_rx_add_vid)(struct net_device *dev, __be16 proto, u16 vid);
 *  If device supports VLAN filtering this function is called when a
 *  VLAN id is registered.
 *
 * int (*ndo_vlan_rx_kill_vid)(struct net_device *dev, __be16 proto, u16 vid);
 *  If device supports VLAN filtering this function is called when a
 *  VLAN id is unregistered.
 *
 * void (*ndo_poll_controller)(struct net_device *dev);
 *
 *  SR-IOV management functions.
 * int (*ndo_set_vf_mac)(struct net_device *dev, int vf, u8* mac);
 * int (*ndo_set_vf_vlan)(struct net_device *dev, int vf, u16 vlan,
 *        u8 qos, __be16 proto);
 * int (*ndo_set_vf_rate)(struct net_device *dev, int vf, int min_tx_rate,
 *        int max_tx_rate);
 * int (*ndo_set_vf_spoofchk)(struct net_device *dev, int vf, bool setting);
 * int (*ndo_set_vf_trust)(struct net_device *dev, int vf, bool setting);
 * int (*ndo_get_vf_config)(struct net_device *dev,
 *          int vf, struct ifla_vf_info *ivf);
 * int (*ndo_set_vf_link_state)(struct net_device *dev, int vf, int link_state);
 * int (*ndo_set_vf_port)(struct net_device *dev, int vf,
 *        struct nlattr *port[]);
 *
 *      Enable or disable the VF ability to query its RSS Redirection Table and
 *      Hash Key. This is needed since on some devices VF share this information
 *      with PF and querying it may introduce a theoretical security risk.
 * int (*ndo_set_vf_rss_query_en)(struct net_device *dev, int vf, bool setting);
 * int (*ndo_get_vf_port)(struct net_device *dev, int vf, struct sk_buff *skb);
 * int (*ndo_setup_tc)(struct net_device *dev, enum tc_setup_type type,
 *           void *type_data);
 *  Called to setup any 'tc' scheduler, classifier or action on @dev.
 *  This is always called from the stack with the rtnl lock held and netif
 *  tx queues stopped. This allows the netdevice to perform queue
 *  management safely.
 *
 *  Fiber Channel over Ethernet (FCoE) offload functions.
 * int (*ndo_fcoe_enable)(struct net_device *dev);
 *  Called when the FCoE protocol stack wants to start using LLD for FCoE
 *  so the underlying device can perform whatever needed configuration or
 *  initialization to support acceleration of FCoE traffic.
 *
 * int (*ndo_fcoe_disable)(struct net_device *dev);
 *  Called when the FCoE protocol stack wants to stop using LLD for FCoE
 *  so the underlying device can perform whatever needed clean-ups to
 *  stop supporting acceleration of FCoE traffic.
 *
 * int (*ndo_fcoe_ddp_setup)(struct net_device *dev, u16 xid,
 *           struct scatterlist *sgl, unsigned int sgc);
 *  Called when the FCoE Initiator wants to initialize an I/O that
 *  is a possible candidate for Direct Data Placement (DDP). The LLD can
 *  perform necessary setup and returns 1 to indicate the device is set up
 *  successfully to perform DDP on this I/O, otherwise this returns 0.
 *
 * int (*ndo_fcoe_ddp_done)(struct net_device *dev,  u16 xid);
 *  Called when the FCoE Initiator/Target is done with the DDPed I/O as
 *  indicated by the FC exchange id 'xid', so the underlying device can
 *  clean up and reuse resources for later DDP requests.
 *
 * int (*ndo_fcoe_ddp_target)(struct net_device *dev, u16 xid,
 *            struct scatterlist *sgl, unsigned int sgc);
 *  Called when the FCoE Target wants to initialize an I/O that
 *  is a possible candidate for Direct Data Placement (DDP). The LLD can
 *  perform necessary setup and returns 1 to indicate the device is set up
 *  successfully to perform DDP on this I/O, otherwise this returns 0.
 *
 * int (*ndo_fcoe_get_hbainfo)(struct net_device *dev,
 *             struct netdev_fcoe_hbainfo *hbainfo);
 *  Called when the FCoE Protocol stack wants information on the underlying
 *  device. This information is utilized by the FCoE protocol stack to
 *  register attributes with Fiber Channel management service as per the
 *  FC-GS Fabric Device Management Information(FDMI) specification.
 *
 * int (*ndo_fcoe_get_wwn)(struct net_device *dev, u64 *wwn, int type);
 *  Called when the underlying device wants to override default World Wide
 *  Name (WWN) generation mechanism in FCoE protocol stack to pass its own
 *  World Wide Port Name (WWPN) or World Wide Node Name (WWNN) to the FCoE
 *  protocol stack to use.
 *
 *  RFS acceleration.
 * int (*ndo_rx_flow_steer)(struct net_device *dev, const struct sk_buff *skb,
 *          u16 rxq_index, u32 flow_id);
 *  Set hardware filter for RFS.  rxq_index is the target queue index;
 *  flow_id is a flow ID to be passed to rps_may_expire_flow() later.
 *  Return the filter ID on success, or a negative error code.
 *
 *  Slave management functions (for bridge, bonding, etc).
 * int (*ndo_add_slave)(struct net_device *dev, struct net_device *slave_dev);
 *  Called to make another netdev an underling.
 *
 * int (*ndo_del_slave)(struct net_device *dev, struct net_device *slave_dev);
 *  Called to release previously enslaved netdev.
 *
 * struct net_device *(*ndo_get_xmit_slave)(struct net_device *dev,
 *              struct sk_buff *skb,
 *              bool all_slaves);
 *  Get the xmit slave of master device. If all_slaves is true, function
 *  assume all the slaves can transmit.
 *
 *      Feature/offload setting functions.
 * netdev_features_t (*ndo_fix_features)(struct net_device *dev,
 *    netdev_features_t features);
 *  Adjusts the requested feature flags according to device-specific
 *  constraints, and returns the resulting flags. Must not modify
 *  the device state.
 *
 * int (*ndo_set_features)(struct net_device *dev, netdev_features_t features);
 *  Called to update device configuration to new features. Passed
 *  feature set might be less than what was returned by ndo_fix_features()).
 *  Must return >0 or -errno if it changed dev->features itself.
 *
 * int (*ndo_fdb_add)(struct ndmsg *ndm, struct nlattr *tb[],
 *          struct net_device *dev,
 *          const unsigned char *addr, u16 vid, u16 flags,
 *          struct netlink_ext_ack *extack);
 *  Adds an FDB entry to dev for addr.
 * int (*ndo_fdb_del)(struct ndmsg *ndm, struct nlattr *tb[],
 *          struct net_device *dev,
 *          const unsigned char *addr, u16 vid)
 *  Deletes the FDB entry from dev coresponding to addr.
 * int (*ndo_fdb_del_bulk)(struct ndmsg *ndm, struct nlattr *tb[],
 *         struct net_device *dev,
 *         u16 vid,
 *         struct netlink_ext_ack *extack);
 * int (*ndo_fdb_dump)(struct sk_buff *skb, struct netlink_callback *cb,
 *           struct net_device *dev, struct net_device *filter_dev,
 *           int *idx)
 *  Used to add FDB entries to dump requests. Implementers should add
 *  entries to skb and update idx with the number of entries.
 *
 * int (*ndo_mdb_add)(struct net_device *dev, struct nlattr *tb[],
 *          u16 nlmsg_flags, struct netlink_ext_ack *extack);
 *  Adds an MDB entry to dev.
 * int (*ndo_mdb_del)(struct net_device *dev, struct nlattr *tb[],
 *          struct netlink_ext_ack *extack);
 *  Deletes the MDB entry from dev.
 * int (*ndo_mdb_dump)(struct net_device *dev, struct sk_buff *skb,
 *           struct netlink_callback *cb);
 *  Dumps MDB entries from dev. The first argument (marker) in the netlink
 *  callback is used by core rtnetlink code.
 *
 * int (*ndo_bridge_setlink)(struct net_device *dev, struct nlmsghdr *nlh,
 *           u16 flags, struct netlink_ext_ack *extack)
 * int (*ndo_bridge_getlink)(struct sk_buff *skb, u32 pid, u32 seq,
 *           struct net_device *dev, u32 filter_mask,
 *           int nlflags)
 * int (*ndo_bridge_dellink)(struct net_device *dev, struct nlmsghdr *nlh,
 *           u16 flags);
 *
 * int (*ndo_change_carrier)(struct net_device *dev, bool new_carrier);
 *  Called to change device carrier. Soft-devices (like dummy, team, etc)
 *  which do not represent real hardware may define this to allow their
 *  userspace components to manage their virtual carrier state. Devices
 *  that determine carrier state from physical hardware properties (eg
 *  network cables) or protocol-dependent mechanisms (eg
 *  USB_CDC_NOTIFY_NETWORK_CONNECTION) should NOT implement this function.
 *
 * int (*ndo_get_phys_port_id)(struct net_device *dev,
 *             struct netdev_phys_item_id *ppid);
 *  Called to get ID of physical port of this device. If driver does
 *  not implement this, it is assumed that the hw is not able to have
 *  multiple net devices on single physical port.
 *
 * int (*ndo_get_port_parent_id)(struct net_device *dev,
 *         struct netdev_phys_item_id *ppid)
 *  Called to get the parent ID of the physical port of this device.
 *
 * void* (*ndo_dfwd_add_station)(struct net_device *pdev,
 *         struct net_device *dev)
 *  Called by upper layer devices to accelerate switching or other
 *  station functionality into hardware. 'pdev is the lowerdev
 *  to use for the offload and 'dev' is the net device that will
 *  back the offload. Returns a pointer to the private structure
 *  the upper layer will maintain.
 * void (*ndo_dfwd_del_station)(struct net_device *pdev, void *priv)
 *  Called by upper layer device to delete the station created
 *  by 'ndo_dfwd_add_station'. 'pdev' is the net device backing
 *  the station and priv is the structure returned by the add
 *  operation.
 * int (*ndo_set_tx_maxrate)(struct net_device *dev,
 *           int queue_index, u32 maxrate);
 *  Called when a user wants to set a max-rate limitation of specific
 *  TX queue.
 * int (*ndo_get_iflink)(const struct net_device *dev);
 *  Called to get the iflink value of this device.
 * int (*ndo_fill_metadata_dst)(struct net_device *dev, struct sk_buff *skb);
 *  This function is used to get egress tunnel information for given skb.
 *  This is useful for retrieving outer tunnel header parameters while
 *  sampling packet.
 * void (*ndo_set_rx_headroom)(struct net_device *dev, int needed_headroom);
 *  This function is used to specify the headroom that the skb must
 *  consider when allocation skb during packet reception. Setting
 *  appropriate rx headroom value allows avoiding skb head copy on
 *  forward. Setting a negative value resets the rx headroom to the
 *  default value.
 * int (*ndo_bpf)(struct net_device *dev, struct netdev_bpf *bpf);
 *  This function is used to set or query state related to XDP on the
 *  netdevice and manage BPF offload. See definition of
 *  enum bpf_netdev_command for details.
 * int (*ndo_xdp_xmit)(struct net_device *dev, int n, struct xdp_frame **xdp,
 *      u32 flags);
 *  This function is used to submit @n XDP packets for transmit on a
 *  netdevice. Returns number of frames successfully transmitted, frames
 *  that got dropped are freed/returned via xdp_return_frame().
 *  Returns negative number, means general error invoking ndo, meaning
 *  no frames were xmit'ed and core-caller will free all frames.
 * struct net_device *(*ndo_xdp_get_xmit_slave)(struct net_device *dev,
 *                  struct xdp_buff *xdp);
 *      Get the xmit slave of master device based on the xdp_buff.
 * int (*ndo_xsk_wakeup)(struct net_device *dev, u32 queue_id, u32 flags);
 *      This function is used to wake up the softirq, ksoftirqd or kthread
 *  responsible for sending and/or receiving packets on a specific
 *  queue id bound to an AF_XDP socket. The flags field specifies if
 *  only RX, only Tx, or both should be woken up using the flags
 *  XDP_WAKEUP_RX and XDP_WAKEUP_TX.
 * int (*ndo_tunnel_ctl)(struct net_device *dev, struct ip_tunnel_parm *p,
 *       int cmd);
 *  Add, change, delete or get information on an IPv4 tunnel.
 * struct net_device *(*ndo_get_peer_dev)(struct net_device *dev);
 *  If a device is paired with a peer device, return the peer instance.
 *  The caller must be under RCU read context.
 * int (*ndo_fill_forward_path)(struct net_device_path_ctx *ctx, struct net_device_path *path);
 *     Get the forwarding path to reach the real device from the HW destination address
 * ktime_t (*ndo_get_tstamp)(struct net_device *dev,
 *           const struct skb_shared_hwtstamps *hwtstamps,
 *           bool cycles);
 *  Get hardware timestamp based on normal/adjustable time or free running
 *  cycle counter. This function is required if physical clock supports a
 *  free running cycle counter.
 *
 * int (*ndo_hwtstamp_get)(struct net_device *dev,
 *         struct kernel_hwtstamp_config *kernel_config);
 *  Get the currently configured hardware timestamping parameters for the
 *  NIC device.
 *
 * int (*ndo_hwtstamp_set)(struct net_device *dev,
 *         struct kernel_hwtstamp_config *kernel_config,
 *         struct netlink_ext_ack *extack);
 *  Change the hardware timestamping parameters for NIC device.
 */
struct net_device_ops {
  int     (*ndo_init)(struct net_device *dev);
  void      (*ndo_uninit)(struct net_device *dev);
  int     (*ndo_open)(struct net_device *dev); // 打开设备, 获取设备需要的I/O地址, IRQ, DMA通道等
  int     (*ndo_stop)(struct net_device *dev); // 停止设备
  netdev_tx_t   (*ndo_start_xmit)(struct sk_buff *skb,
              struct net_device *dev); // 启动数据包的发送
  netdev_features_t (*ndo_features_check)(struct sk_buff *skb,
                  struct net_device *dev,
                  netdev_features_t features);
  u16     (*ndo_select_queue)(struct net_device *dev,
                struct sk_buff *skb,
                struct net_device *sb_dev);
  void      (*ndo_change_rx_flags)(struct net_device *dev,
                   int flags);
  void      (*ndo_set_rx_mode)(struct net_device *dev);
  int     (*ndo_set_mac_address)(struct net_device *dev,
                   void *addr); // 用于设置设备的mac地址
  int     (*ndo_validate_addr)(struct net_device *dev);
  int     (*ndo_do_ioctl)(struct net_device *dev,
                  struct ifreq *ifr, int cmd); // 进行I/O控制
  int     (*ndo_eth_ioctl)(struct net_device *dev,
             struct ifreq *ifr, int cmd);
  int     (*ndo_siocbond)(struct net_device *dev,
            struct ifreq *ifr, int cmd);
  int     (*ndo_siocwandev)(struct net_device *dev,
              struct if_settings *ifs);
  int     (*ndo_siocdevprivate)(struct net_device *dev,
                  struct ifreq *ifr,
                  void __user *data, int cmd);
  int     (*ndo_set_config)(struct net_device *dev,
                    struct ifmap *map); // 用于配置接口, 也可用于改变设备的I/O地址和中断号
  int     (*ndo_change_mtu)(struct net_device *dev,
              int new_mtu);
  int     (*ndo_neigh_setup)(struct net_device *dev,
               struct neigh_parms *);
  void      (*ndo_tx_timeout) (struct net_device *dev,
               unsigned int txqueue); // 当数据包发送超时时, 调用该函数, 它采取重新启动数据包发送过程或重新启动硬件等措施来恢复设备到正常状态

  void      (*ndo_get_stats64)(struct net_device *dev,
               struct rtnl_link_stats64 *storage);
  bool      (*ndo_has_offload_stats)(const struct net_device *dev, int attr_id);
  int     (*ndo_get_offload_stats)(int attr_id,
               const struct net_device *dev,
               void *attr_data);
  struct net_device_stats* (*ndo_get_stats)(struct net_device *dev); // 获取网络设备的详细信息, 包括流量统计信息

  int     (*ndo_vlan_rx_add_vid)(struct net_device *dev,
                   __be16 proto, u16 vid);
  int     (*ndo_vlan_rx_kill_vid)(struct net_device *dev,
                    __be16 proto, u16 vid);
#ifdef CONFIG_NET_POLL_CONTROLLER
  void                    (*ndo_poll_controller)(struct net_device *dev);
  int     (*ndo_netpoll_setup)(struct net_device *dev,
                 struct netpoll_info *info);
  void      (*ndo_netpoll_cleanup)(struct net_device *dev);
#endif
  int     (*ndo_set_vf_mac)(struct net_device *dev,
              int queue, u8 *mac);
  int     (*ndo_set_vf_vlan)(struct net_device *dev,
               int queue, u16 vlan,
               u8 qos, __be16 proto);
  int     (*ndo_set_vf_rate)(struct net_device *dev,
               int vf, int min_tx_rate,
               int max_tx_rate);
  int     (*ndo_set_vf_spoofchk)(struct net_device *dev,
                   int vf, bool setting);
  int     (*ndo_set_vf_trust)(struct net_device *dev,
                int vf, bool setting);
  int     (*ndo_get_vf_config)(struct net_device *dev,
                 int vf,
                 struct ifla_vf_info *ivf);
  int     (*ndo_set_vf_link_state)(struct net_device *dev,
               int vf, int link_state);
  int     (*ndo_get_vf_stats)(struct net_device *dev,
                int vf,
                struct ifla_vf_stats
                *vf_stats);
  int     (*ndo_set_vf_port)(struct net_device *dev,
               int vf,
               struct nlattr *port[]);
  int     (*ndo_get_vf_port)(struct net_device *dev,
               int vf, struct sk_buff *skb);
  int     (*ndo_get_vf_guid)(struct net_device *dev,
               int vf,
               struct ifla_vf_guid *node_guid,
               struct ifla_vf_guid *port_guid);
  int     (*ndo_set_vf_guid)(struct net_device *dev,
               int vf, u64 guid,
               int guid_type);
  int     (*ndo_set_vf_rss_query_en)(
               struct net_device *dev,
               int vf, bool setting);
  int     (*ndo_setup_tc)(struct net_device *dev,
            enum tc_setup_type type,
            void *type_data);
#if IS_ENABLED(CONFIG_FCOE)
  int     (*ndo_fcoe_enable)(struct net_device *dev);
  int     (*ndo_fcoe_disable)(struct net_device *dev);
  int     (*ndo_fcoe_ddp_setup)(struct net_device *dev,
                  u16 xid,
                  struct scatterlist *sgl,
                  unsigned int sgc);
  int     (*ndo_fcoe_ddp_done)(struct net_device *dev,
                 u16 xid);
  int     (*ndo_fcoe_ddp_target)(struct net_device *dev,
                   u16 xid,
                   struct scatterlist *sgl,
                   unsigned int sgc);
  int     (*ndo_fcoe_get_hbainfo)(struct net_device *dev,
              struct netdev_fcoe_hbainfo *hbainfo);
#endif

#if IS_ENABLED(CONFIG_LIBFCOE)
#define NETDEV_FCOE_WWNN 0
#define NETDEV_FCOE_WWPN 1
  int     (*ndo_fcoe_get_wwn)(struct net_device *dev,
                u64 *wwn, int type);
#endif

#ifdef CONFIG_RFS_ACCEL
  int     (*ndo_rx_flow_steer)(struct net_device *dev,
                 const struct sk_buff *skb,
                 u16 rxq_index,
                 u32 flow_id);
#endif
  int     (*ndo_add_slave)(struct net_device *dev,
             struct net_device *slave_dev,
             struct netlink_ext_ack *extack);
  int     (*ndo_del_slave)(struct net_device *dev,
             struct net_device *slave_dev);
  struct net_device*  (*ndo_get_xmit_slave)(struct net_device *dev,
                  struct sk_buff *skb,
                  bool all_slaves);
  struct net_device*  (*ndo_sk_get_lower_dev)(struct net_device *dev,
              struct sock *sk);
  netdev_features_t (*ndo_fix_features)(struct net_device *dev,
                netdev_features_t features);
  int     (*ndo_set_features)(struct net_device *dev,
                netdev_features_t features);
  int     (*ndo_neigh_construct)(struct net_device *dev,
                   struct neighbour *n);
  void      (*ndo_neigh_destroy)(struct net_device *dev,
                 struct neighbour *n);

  int     (*ndo_fdb_add)(struct ndmsg *ndm,
                 struct nlattr *tb[],
                 struct net_device *dev,
                 const unsigned char *addr,
                 u16 vid,
                 u16 flags,
                 struct netlink_ext_ack *extack);
  int     (*ndo_fdb_del)(struct ndmsg *ndm,
                 struct nlattr *tb[],
                 struct net_device *dev,
                 const unsigned char *addr,
                 u16 vid, struct netlink_ext_ack *extack);
  int     (*ndo_fdb_del_bulk)(struct ndmsg *ndm,
                struct nlattr *tb[],
                struct net_device *dev,
                u16 vid,
                struct netlink_ext_ack *extack);
  int     (*ndo_fdb_dump)(struct sk_buff *skb,
            struct netlink_callback *cb,
            struct net_device *dev,
            struct net_device *filter_dev,
            int *idx);
  int     (*ndo_fdb_get)(struct sk_buff *skb,
                 struct nlattr *tb[],
                 struct net_device *dev,
                 const unsigned char *addr,
                 u16 vid, u32 portid, u32 seq,
                 struct netlink_ext_ack *extack);
  int     (*ndo_mdb_add)(struct net_device *dev,
                 struct nlattr *tb[],
                 u16 nlmsg_flags,
                 struct netlink_ext_ack *extack);
  int     (*ndo_mdb_del)(struct net_device *dev,
                 struct nlattr *tb[],
                 struct netlink_ext_ack *extack);
  int     (*ndo_mdb_dump)(struct net_device *dev,
            struct sk_buff *skb,
            struct netlink_callback *cb);
  int     (*ndo_bridge_setlink)(struct net_device *dev,
                  struct nlmsghdr *nlh,
                  u16 flags,
                  struct netlink_ext_ack *extack);
  int     (*ndo_bridge_getlink)(struct sk_buff *skb,
                  u32 pid, u32 seq,
                  struct net_device *dev,
                  u32 filter_mask,
                  int nlflags);
  int     (*ndo_bridge_dellink)(struct net_device *dev,
                  struct nlmsghdr *nlh,
                  u16 flags);
  int     (*ndo_change_carrier)(struct net_device *dev,
                  bool new_carrier);
  int     (*ndo_get_phys_port_id)(struct net_device *dev,
              struct netdev_phys_item_id *ppid);
  int     (*ndo_get_port_parent_id)(struct net_device *dev,
                struct netdev_phys_item_id *ppid);
  int     (*ndo_get_phys_port_name)(struct net_device *dev,
                char *name, size_t len);
  void*     (*ndo_dfwd_add_station)(struct net_device *pdev,
              struct net_device *dev);
  void      (*ndo_dfwd_del_station)(struct net_device *pdev,
              void *priv);

  int     (*ndo_set_tx_maxrate)(struct net_device *dev,
                  int queue_index,
                  u32 maxrate);
  int     (*ndo_get_iflink)(const struct net_device *dev);
  int     (*ndo_fill_metadata_dst)(struct net_device *dev,
                   struct sk_buff *skb);
  void      (*ndo_set_rx_headroom)(struct net_device *dev,
                   int needed_headroom);
  int     (*ndo_bpf)(struct net_device *dev,
             struct netdev_bpf *bpf);
  int     (*ndo_xdp_xmit)(struct net_device *dev, int n,
            struct xdp_frame **xdp,
            u32 flags);
  struct net_device * (*ndo_xdp_get_xmit_slave)(struct net_device *dev,
                struct xdp_buff *xdp);
  int     (*ndo_xsk_wakeup)(struct net_device *dev,
              u32 queue_id, u32 flags);
  int     (*ndo_tunnel_ctl)(struct net_device *dev,
              struct ip_tunnel_parm *p, int cmd);
  struct net_device * (*ndo_get_peer_dev)(struct net_device *dev);
  int                     (*ndo_fill_forward_path)(struct net_device_path_ctx *ctx,
                                                         struct net_device_path *path);
  ktime_t     (*ndo_get_tstamp)(struct net_device *dev,
              const struct skb_shared_hwtstamps *hwtstamps,
              bool cycles);
  int     (*ndo_hwtstamp_get)(struct net_device *dev,
                struct kernel_hwtstamp_config *kernel_config);
  int     (*ndo_hwtstamp_set)(struct net_device *dev,
                struct kernel_hwtstamp_config *kernel_config,
                struct netlink_ext_ack *extack);
};


//https://elixir.bootlin.com/linux/v6.5.2/source/include/linux/net.h#L117
/**
 *  struct socket - general BSD socket
 *  @state: socket state (%SS_CONNECTED, etc)
 *  @type: socket type (%SOCK_STREAM, etc)
 *  @flags: socket flags (%SOCK_NOSPACE, etc)
 *  @ops: protocol specific socket operations
 *  @file: File back pointer for gc
 *  @sk: internal networking protocol agnostic socket representation
 *  @wq: wait queue for several uses
 */
struct socket {
  socket_state    state; // 套接字的状态

  short     type; // 套接字的类型。其取值为SOCK_XXXX形式

  unsigned long   flags; // 套接字的设置标志。存放套接字等待缓冲区的状态信息，其值的形式如SOCK_ASYNC_NOSPACE等

  struct file   *file; // 套接字所属的文件描述符
  struct sock   *sk; // 指向存放套接字属性的结构指针
  const struct proto_ops  *ops; // 套接字层的操作函数块

  struct socket_wq  wq;
};

//https://elixir.bootlin.com/linux/v6.5.2/source/include/net/sock.h#L357
/**
  * struct sock - network layer representation of sockets
  * @__sk_common: shared layout with inet_timewait_sock
  * @sk_shutdown: mask of %SEND_SHUTDOWN and/or %RCV_SHUTDOWN
  * @sk_userlocks: %SO_SNDBUF and %SO_RCVBUF settings
  * @sk_lock: synchronizer
  * @sk_kern_sock: True if sock is using kernel lock classes
  * @sk_rcvbuf: size of receive buffer in bytes
  * @sk_wq: sock wait queue and async head
  * @sk_rx_dst: receive input route used by early demux
  * @sk_rx_dst_ifindex: ifindex for @sk_rx_dst
  * @sk_rx_dst_cookie: cookie for @sk_rx_dst
  * @sk_dst_cache: destination cache
  * @sk_dst_pending_confirm: need to confirm neighbour
  * @sk_policy: flow policy
  * @sk_receive_queue: incoming packets
  * @sk_wmem_alloc: transmit queue bytes committed
  * @sk_tsq_flags: TCP Small Queues flags
  * @sk_write_queue: Packet sending queue
  * @sk_omem_alloc: "o" is "option" or "other"
  * @sk_wmem_queued: persistent queue size
  * @sk_forward_alloc: space allocated forward
  * @sk_reserved_mem: space reserved and non-reclaimable for the socket
  * @sk_napi_id: id of the last napi context to receive data for sk
  * @sk_ll_usec: usecs to busypoll when there is no data
  * @sk_allocation: allocation mode
  * @sk_pacing_rate: Pacing rate (if supported by transport/packet scheduler)
  * @sk_pacing_status: Pacing status (requested, handled by sch_fq)
  * @sk_max_pacing_rate: Maximum pacing rate (%SO_MAX_PACING_RATE)
  * @sk_sndbuf: size of send buffer in bytes
  * @__sk_flags_offset: empty field used to determine location of bitfield
  * @sk_padding: unused element for alignment
  * @sk_no_check_tx: %SO_NO_CHECK setting, set checksum in TX packets
  * @sk_no_check_rx: allow zero checksum in RX packets
  * @sk_route_caps: route capabilities (e.g. %NETIF_F_TSO)
  * @sk_gso_disabled: if set, NETIF_F_GSO_MASK is forbidden.
  * @sk_gso_type: GSO type (e.g. %SKB_GSO_TCPV4)
  * @sk_gso_max_size: Maximum GSO segment size to build
  * @sk_gso_max_segs: Maximum number of GSO segments
  * @sk_pacing_shift: scaling factor for TCP Small Queues
  * @sk_lingertime: %SO_LINGER l_linger setting
  * @sk_backlog: always used with the per-socket spinlock held
  * @sk_callback_lock: used with the callbacks in the end of this struct
  * @sk_error_queue: rarely used
  * @sk_prot_creator: sk_prot of original sock creator (see ipv6_setsockopt,
  *       IPV6_ADDRFORM for instance)
  * @sk_err: last error
  * @sk_err_soft: errors that don't cause failure but are the cause of a
  *         persistent failure not just 'timed out'
  * @sk_drops: raw/udp drops counter
  * @sk_ack_backlog: current listen backlog
  * @sk_max_ack_backlog: listen backlog set in listen()
  * @sk_uid: user id of owner
  * @sk_prefer_busy_poll: prefer busypolling over softirq processing
  * @sk_busy_poll_budget: napi processing budget when busypolling
  * @sk_priority: %SO_PRIORITY setting
  * @sk_type: socket type (%SOCK_STREAM, etc)
  * @sk_protocol: which protocol this socket belongs in this network family
  * @sk_peer_lock: lock protecting @sk_peer_pid and @sk_peer_cred
  * @sk_peer_pid: &struct pid for this socket's peer
  * @sk_peer_cred: %SO_PEERCRED setting
  * @sk_rcvlowat: %SO_RCVLOWAT setting
  * @sk_rcvtimeo: %SO_RCVTIMEO setting
  * @sk_sndtimeo: %SO_SNDTIMEO setting
  * @sk_txhash: computed flow hash for use on transmit
  * @sk_txrehash: enable TX hash rethink
  * @sk_filter: socket filtering instructions
  * @sk_timer: sock cleanup timer
  * @sk_stamp: time stamp of last packet received
  * @sk_stamp_seq: lock for accessing sk_stamp on 32 bit architectures only
  * @sk_tsflags: SO_TIMESTAMPING flags
  * @sk_use_task_frag: allow sk_page_frag() to use current->task_frag.
  *        Sockets that can be used under memory reclaim should
  *        set this to false.
  * @sk_bind_phc: SO_TIMESTAMPING bind PHC index of PTP virtual clock
  *               for timestamping
  * @sk_tskey: counter to disambiguate concurrent tstamp requests
  * @sk_zckey: counter to order MSG_ZEROCOPY notifications
  * @sk_socket: Identd and reporting IO signals
  * @sk_user_data: RPC layer private data. Write-protected by @sk_callback_lock.
  * @sk_frag: cached page frag
  * @sk_peek_off: current peek_offset value
  * @sk_send_head: front of stuff to transmit
  * @tcp_rtx_queue: TCP re-transmit queue [union with @sk_send_head]
  * @sk_security: used by security modules
  * @sk_mark: generic packet mark
  * @sk_cgrp_data: cgroup data for this cgroup
  * @sk_memcg: this socket's memory cgroup association
  * @sk_write_pending: a write to stream socket waits to start
  * @sk_wait_pending: number of threads blocked on this socket
  * @sk_state_change: callback to indicate change in the state of the sock
  * @sk_data_ready: callback to indicate there is data to be processed
  * @sk_write_space: callback to indicate there is bf sending space available
  * @sk_error_report: callback to indicate errors (e.g. %MSG_ERRQUEUE)
  * @sk_backlog_rcv: callback to process the backlog
  * @sk_validate_xmit_skb: ptr to an optional validate function
  * @sk_destruct: called at sock freeing time, i.e. when all refcnt == 0
  * @sk_reuseport_cb: reuseport group container
  * @sk_bpf_storage: ptr to cache and control for bpf_sk_storage
  * @sk_rcu: used during RCU grace period
  * @sk_clockid: clockid used by time-based scheduling (SO_TXTIME)
  * @sk_txtime_deadline_mode: set deadline mode for SO_TXTIME
  * @sk_txtime_report_errors: set report errors mode for SO_TXTIME
  * @sk_txtime_unused: unused txtime flags
  * @ns_tracker: tracker for netns reference
  * @sk_bind2_node: bind node in the bhash2 table
  */
struct sock {
  /*
   * Now struct inet_timewait_sock also uses sock_common, so please just
   * don't add nothing before this first member (__sk_common) --acme
   */
  struct sock_common  __sk_common;
#define sk_node     __sk_common.skc_node
#define sk_nulls_node   __sk_common.skc_nulls_node
#define sk_refcnt   __sk_common.skc_refcnt
#define sk_tx_queue_mapping __sk_common.skc_tx_queue_mapping
#ifdef CONFIG_SOCK_RX_QUEUE_MAPPING
#define sk_rx_queue_mapping __sk_common.skc_rx_queue_mapping
#endif

#define sk_dontcopy_begin __sk_common.skc_dontcopy_begin
#define sk_dontcopy_end   __sk_common.skc_dontcopy_end
#define sk_hash     __sk_common.skc_hash
#define sk_portpair   __sk_common.skc_portpair
#define sk_num      __sk_common.skc_num
#define sk_dport    __sk_common.skc_dport
#define sk_addrpair   __sk_common.skc_addrpair
#define sk_daddr    __sk_common.skc_daddr
#define sk_rcv_saddr    __sk_common.skc_rcv_saddr
#define sk_family   __sk_common.skc_family
#define sk_state    __sk_common.skc_state
#define sk_reuse    __sk_common.skc_reuse
#define sk_reuseport    __sk_common.skc_reuseport
#define sk_ipv6only   __sk_common.skc_ipv6only
#define sk_net_refcnt   __sk_common.skc_net_refcnt
#define sk_bound_dev_if   __sk_common.skc_bound_dev_if
#define sk_bind_node    __sk_common.skc_bind_node
#define sk_prot     __sk_common.skc_prot
#define sk_net      __sk_common.skc_net
#define sk_v6_daddr   __sk_common.skc_v6_daddr
#define sk_v6_rcv_saddr __sk_common.skc_v6_rcv_saddr
#define sk_cookie   __sk_common.skc_cookie
#define sk_incoming_cpu   __sk_common.skc_incoming_cpu
#define sk_flags    __sk_common.skc_flags
#define sk_rxhash   __sk_common.skc_rxhash

  /* early demux fields */
  struct dst_entry __rcu  *sk_rx_dst;
  int     sk_rx_dst_ifindex;
  u32     sk_rx_dst_cookie;

  socket_lock_t   sk_lock;
  atomic_t    sk_drops;
  int     sk_rcvlowat;
  struct sk_buff_head sk_error_queue;
  struct sk_buff_head sk_receive_queue;
  /*
   * The backlog queue is special, it is always used with
   * the per-socket spinlock held and requires low latency
   * access. Therefore we special case it's implementation.
   * Note : rmem_alloc is in this structure to fill a hole
   * on 64bit arches, not because its logically part of
   * backlog.
   */
  struct {
    atomic_t  rmem_alloc;
    int   len;
    struct sk_buff  *head;
    struct sk_buff  *tail;
  } sk_backlog;

#define sk_rmem_alloc sk_backlog.rmem_alloc

  int     sk_forward_alloc;
  u32     sk_reserved_mem;
#ifdef CONFIG_NET_RX_BUSY_POLL
  unsigned int    sk_ll_usec;
  /* ===== mostly read cache line ===== */
  unsigned int    sk_napi_id;
#endif
  int     sk_rcvbuf;
  int     sk_wait_pending;

  struct sk_filter __rcu  *sk_filter;
  union {
    struct socket_wq __rcu  *sk_wq;
    /* private: */
    struct socket_wq  *sk_wq_raw;
    /* public: */
  };
#ifdef CONFIG_XFRM
  struct xfrm_policy __rcu *sk_policy[2];
#endif

  struct dst_entry __rcu  *sk_dst_cache;
  atomic_t    sk_omem_alloc;
  int     sk_sndbuf;

  /* ===== cache line for TX ===== */
  int     sk_wmem_queued;
  refcount_t    sk_wmem_alloc;
  unsigned long   sk_tsq_flags;
  union {
    struct sk_buff  *sk_send_head;
    struct rb_root  tcp_rtx_queue;
  };
  struct sk_buff_head sk_write_queue;
  __s32     sk_peek_off;
  int     sk_write_pending;
  __u32     sk_dst_pending_confirm;
  u32     sk_pacing_status; /* see enum sk_pacing */
  long      sk_sndtimeo;
  struct timer_list sk_timer;
  __u32     sk_priority;
  __u32     sk_mark;
  unsigned long   sk_pacing_rate; /* bytes per second */
  unsigned long   sk_max_pacing_rate;
  struct page_frag  sk_frag;
  netdev_features_t sk_route_caps;
  int     sk_gso_type;
  unsigned int    sk_gso_max_size;
  gfp_t     sk_allocation;
  __u32     sk_txhash;

  /*
   * Because of non atomicity rules, all
   * changes are protected by socket lock.
   */
  u8      sk_gso_disabled : 1,
        sk_kern_sock : 1,
        sk_no_check_tx : 1,
        sk_no_check_rx : 1,
        sk_userlocks : 4;
  u8      sk_pacing_shift;
  u16     sk_type;
  u16     sk_protocol;
  u16     sk_gso_max_segs;
  unsigned long         sk_lingertime;
  struct proto    *sk_prot_creator;
  rwlock_t    sk_callback_lock;
  int     sk_err,
        sk_err_soft;
  u32     sk_ack_backlog;
  u32     sk_max_ack_backlog;
  kuid_t      sk_uid;
  u8      sk_txrehash;
#ifdef CONFIG_NET_RX_BUSY_POLL
  u8      sk_prefer_busy_poll;
  u16     sk_busy_poll_budget;
#endif
  spinlock_t    sk_peer_lock;
  int     sk_bind_phc;
  struct pid    *sk_peer_pid;
  const struct cred *sk_peer_cred;

  long      sk_rcvtimeo;
  ktime_t     sk_stamp;
#if BITS_PER_LONG==32
  seqlock_t   sk_stamp_seq;
#endif
  atomic_t    sk_tskey;
  atomic_t    sk_zckey;
  u32     sk_tsflags;
  u8      sk_shutdown;

  u8      sk_clockid;
  u8      sk_txtime_deadline_mode : 1,
        sk_txtime_report_errors : 1,
        sk_txtime_unused : 6;
  bool      sk_use_task_frag;

  struct socket   *sk_socket;
  void      *sk_user_data;
#ifdef CONFIG_SECURITY
  void      *sk_security;
#endif
  struct sock_cgroup_data sk_cgrp_data;
  struct mem_cgroup *sk_memcg;
  void      (*sk_state_change)(struct sock *sk);
  void      (*sk_data_ready)(struct sock *sk);
  void      (*sk_write_space)(struct sock *sk);
  void      (*sk_error_report)(struct sock *sk);
  int     (*sk_backlog_rcv)(struct sock *sk,
              struct sk_buff *skb);
#ifdef CONFIG_SOCK_VALIDATE_XMIT
  struct sk_buff*   (*sk_validate_xmit_skb)(struct sock *sk,
              struct net_device *dev,
              struct sk_buff *skb);
#endif
  void                    (*sk_destruct)(struct sock *sk);
  struct sock_reuseport __rcu *sk_reuseport_cb;
#ifdef CONFIG_BPF_SYSCALL
  struct bpf_local_storage __rcu  *sk_bpf_storage;
#endif
  struct rcu_head   sk_rcu;
  netns_tracker   ns_tracker;
  struct hlist_node sk_bind2_node;
};

//https://elixir.bootlin.com/linux/v6.5.2/source/include/net/sock.h#L1575
struct socket_alloc {
  struct socket socket;
  struct inode vfs_inode;
};

//https://elixir.bootlin.com/linux/v6.5.2/source/net/socket.c#L3172
static int __init sock_init(void)
{
  int err;
  /*
   *      Initialize the network sysctl infrastructure.
   */
  err = net_sysctl_init();
  if (err)
    goto out;

  /*
   *      Initialize skbuff SLAB cache
   */
  skb_init();

  /*
   *      Initialize the protocols module.
   */

  init_inodecache();

  err = register_filesystem(&sock_fs_type);
  if (err)
    goto out;
  sock_mnt = kern_mount(&sock_fs_type);
  if (IS_ERR(sock_mnt)) {
    err = PTR_ERR(sock_mnt);
    goto out_mount;
  }

  /* The real protocol initialization is performed in later initcalls.
   */

#ifdef CONFIG_NETFILTER
  err = netfilter_init();
  if (err)
    goto out;
#endif

  ptp_classifier_init();

out:
  return err;

out_mount:
  unregister_filesystem(&sock_fs_type);
  goto out;
}

//https://elixir.bootlin.com/linux/v6.5.2/source/include/net/protocol.h#L76
/* This is used to register socket interfaces for IP protocols.  */
struct inet_protosw {
  struct list_head list;

        /* These two fields form the lookup key.  */
  unsigned short   type;     /* This is the 2nd argument to socket(2). */
  unsigned short   protocol; /* This is the L4 protocol number.  */

  struct proto   *prot;
  const struct proto_ops *ops;
  
  unsigned char  flags;      /* See INET_PROTOSW_* below.  */
};
```

struct sock 数据结构包含了大量的内核管理套接字的信息，内核把最重要的成员存放在 struct sock_common 数据结构中，struct sock_common 数据结构嵌入在 struct sock 结构中，它是 struct sock 数据结构的第一个成员.

struct sock_common 数据结构是套接字在网络中的最小描述，它包含了内核管理套接字最重要信息的集合。而 struct sock 数据结构中包含了套接字的全部信息与特点，有的特性很少用到，甚至根本就没有用到.

struct sock 数据结构组织在特定协议的哈希链表中，skc_node 是连接哈希链表中成员的哈希节点，skc_hash 是引用的哈希值。接收和发送数据放在数据 struct sock 数据结构的两个等待队列中：sk_receive_queue 和 sk_write_queue。这两个队列中包含的都是 Socket Buffer.

内核使用 struct sock 数据结构实例中的回调函数，获取套接字上某些事件发生的消息或套接字状态发生变化. 其中，使用最频繁的回调函数是 sk_data_ready，用户进程等待数据到达时，就会调用该回调函数.

套接字的文件描述符的文件访问的重定向，对网络协议栈各层是透明的. 而 inode 和 socket 的关联是通过直接分配一个辅助数据结构socket_alloc来实现的.


在 Linux 网络子系统中，Socket Buffer 代表了一个要发送或者处理的报文, 是一个关键的数据结构，因为它贯穿于整个 TCP/IP 协议栈的各层。Linux 内核对网络数据打包处理的全过程中，始终伴随着这个 Socket Buffer。 可以理解为 Socket Buffer 就是网络数据包在内核中的对象实例.

Socket Buffer 主要由两部分组成:
1. 数据包：存放了在网络中实际流通的数据
2. 管理数据结构（struct sk_buff）：当在内核中对数据包进行时，内核还需要一些其他的数据来管理数据包和操作数据包，例如协议之间的交换信息，数据的状态，时间等

struct sk_buff 数据结构中存放了套接字接收 / 发送的数据。在发送数据时，在套接字层创建了 Socket Buffer 缓冲区与管理数据结构，存放来自应用程序的数据。在接收数据包时，Socket Buffer 则在网络设备的驱动程序中创建，存放来自网络的数据。在发送和接受数据的过程中，各层协议的头信息会不断从数据包中插入和去掉，sk_buff 结构中描述协议头信息的地址指针也会被不断地赋值和复位.

套接字层的初始化`sock_init`要为以后各协议初始化 struct sock 数据结构对象、套接字缓冲区 Socket Buffer 对象等做好准备，预留内存空间.

TCP/IP 协议栈处理完输入数据包后，将数据包交给套接字层，放在套接字的接收缓冲区队列（sk_rcv_queue）。然后数据包从套接字层离开内核，送给应用层等待数据包的用户程序。用户程序向外发送的数据包缓存在套接字的传送缓冲区队列（sk_write_queue），从套接字层进入内核地址空间.

在同一个主机中，可以同时在多个协议上打开多个套接字, 在 Linux 内核里有一个叫做 struct inet_protosw 的数据结构来区分当前数据包的目标套接字. 它就是管理和描述 struct proto_ops 和 struct proto 之间的对应关系, struct proto_ops 就是系统调用套接字的操作函数块，而 struct proto 就是跟内核协议相关的套接字操作函数块.

内核使用 struct  inet_protosw 数据结构实现的协议交换表，将应用程序通过 socketcall 系统调用指定的套接字操作，转换成对某个协议实例实现的套接字操作函数的调用.

struct inet_protosw 类型把 INET 套接字的协议族操作集与传输层协议操作集关联起来。该类型的 inetsw_array 数组变量实现了 INET 套接字的协议族操作集与具体的传输层协议关联。由 struct inet_protosw 数据结构类型数组 inetsw_array[]构成的向量表，称为协议交换表，协议交换表满足了套接字支持多协议栈这项功能.

创建套接字应用程序一般要经过后面这 6 个步骤:
1. 创建套接字
2. 将套接字与地址绑定，设置套接字选项
3. 建立套接字之间的连接
4. 监听套接字
5. 接收、发送数据
6. 关闭、释放套接字

在应用程序中执行 socket 函数，socket 产生系统调用中断执行内核的套接字分路函数 sys_socket call，在 sys_socket call 套接字函数分路器中将调用传送到 sys_socket 函数，由 sys_socket 函数调用套接字的通用创建函数 sock_create。sock_create 函数完成通用套接字创建、初始化任务后，再调用特定协议族的套接字创建函数

一个新的 struct socket 数据结构起始由 sock_create 函数创建，该函数直接调用 `__sock_create` 函数，`__sock_create` 函数的任务是为套接字预留需要的内存空间，由 sock_alloc 函数完成这项功能.

sock_alloc 函数不仅会为 struct socket 数据结构实例预留空间，也会为 struct inode 数据结构实例分配需要的内存空间，这样可以使两个数据结构的实例相关联.

当具体的协议与新套接字相连时，其内部状态的管理由协议自身维护.

创建完套接字后，应用程序需要调用 [sys_bind](https://elixir.bootlin.com/linux/v6.5.2/source/net/socket.c#L1801) 函数把套接字和地址绑定起来.

sys_bind 函数首先会查找套接字对应的 socket 实例，调用 sockfd_lookup_light。在绑定之前，将用户空间的地址拷贝到内核空间的缓冲区中，在拷贝过程中会检查用户传入的地址是否正确。等上述的准备工作完成后，就会调用 inet_bind 函数来完成绑定操作.

当应用程序调用 connect 函数发出连接请求时，内核会启动函数 sys_connect.

调用 listen 函数时，应用程序触发内核的 sys_listen 函数，把套接字描述符 fd 对应的套接字设置为监听模式，观察连接请求.

接受一个客户端的连接请求会调用 accept 函数，应用程序触发内核函数 sys_accept，等待接收连接请求。如果允许连接，则重新创建一个代表该连接的套接字，并返回其套接字描述符.

这个新的套接字描述符与最初创建套接字时，设置的套接字地址族与套接字类型、使用的协议一样。原来创建的套接字不与连接关联，它继续在原套接字上侦听，以便接收其他连接请求.

套接字应用中最简单的传送函数是 send, send 函数的作用类似于 write，但 send 函数允许应用程序指定标志，规定如何对待传送数据. 调用 send 函数时，会触发内核的 sys_send 函数，把发送缓冲区的数据发送出去.

sys_send 函数具体调用流程如下:
1. 应用程序的数据被复制到内核后，sys_send 函数调用 sock_sendmsg，依据协议族类型来执行发送操作
2. 如果是 INET 协议族套接字，sock_sendmsg 将调用 inet_sendmsg 函数
3. 如果采用 TCP 协议，inet_sendmsg 函数将调用 tcp_sendmsg，并按照 TCP 协议规则来发送数据包

send 函数返回发送成功，并不意味着在连接的另一端的进程可以收到数据，这里只能保证发送 send 函数执行成功，发送给网络设备驱动程序的数据没有出错.

接收数据recv 函数与文件读 read 函数类似，recv 函数中可以指定标志来控制如何接收数据，调用 recv 函数时，应用程序会触发内核的 sys_recv 函数，把网络中的数据递交到应用程序。当然，read、recvfrom 函数也会触发 sys_recv 函数.

sys_recv具体流程如下:
1. 为把内核的网络数据转入应用程序的接收缓冲区，sys_recv 函数依次调用 sys_recvfrom、sock_recvfrom 和 `__sock_recvmsg`，并依据协议族类型来执行具体的接收操作
2. 如果是 INET 协议族套接字，`__sock_recvmsg` 将调用 sock_common_recvmsg 函数
3. 如果采用 TCP 协议，sock_common_recvmsg 函数将调用 tcp_recvmsg，按照 TCP 协议规则来接收数据包如果接收方想获取数据包发送端的标识符，应用程序可以调用 sys_recvfrom 函数来获取数据包发送方的源地址

当应用程序调用 shutdown 函数关闭连接时，内核会启动函数 sys_shutdown.
## IPsec
IPsec提供了一种网络层安全解决方案, 它使用ESP和AH协议. 它在ipv6中是强制执行的, 在ipv4中是可选的. 大部分vpn(virtual private network)解决方案都以ipsec为基础.

ipsec支持模式:
- 传输模式
- 隧道模式

## netlink
为完成诸如增删路由, 配置邻接表, 设置IPsec策略和状态等任务, 网络栈必须与用户空间通信, 该通信基于netlink套接字完成.

## 无线
无线栈包含一些常规网络栈没有的独特功能, 比如省电模式. 它还支持一些特殊拓扑结构, 如网状(Mesh)网络, 对等(ad-hoc)网络等. 这些拓扑结构有时要求使用特殊的功能, 比如网状网络mesh使用路由选择协议混合无线网状协议(HWMP, Hybrid Wireless Mesh Protocol), 它运行在L2, 处理的是MAC地址.

## tcp编程
![tcp编程模型](/misc/img/net/997e39e5574252ada22220e4b3646dda.png)

TCP 的服务端要先监听一个端口，一般是先调用 bind 函数，给这个 socket 赋予一个端口和 IP 地址.
```c

int bind(int sockfd, const struct sockaddr *addr,socklen_t addrlen);

struct sockaddr_in {
  __kernel_sa_family_t  sin_family;  /* Address family    */
  __be16    sin_port;  /* Port number      */
  struct in_addr  sin_addr;  /* Internet address    */

  /* Pad to size of `struct sockaddr'. */
  unsigned char    __pad[__SOCK_SIZE__ - sizeof(short int) -
      sizeof(unsigned short int) - sizeof(struct in_addr)];
};

struct in_addr {
  __be32  s_addr;
};
```

其中，sockfd 是创建的 socket 文件描述符. 在 sockaddr_in 结构中，sin_family 设置为 AF_INET，表示 IPv4；sin_port 是端口号；sin_addr 是 IP 地址.

服务端所在的服务器可能有多个网卡、多个地址，可以选择监听在一个地址，也可以监听 0.0.0.0 表示所有的地址都监听. 服务端一般要监听在一个众所周知的端口上，例如，Nginx 一般是 80/443.

客户端要访问服务端，肯定事先要知道服务端的地址. 客户端不需要 bind，是因为随机分配一个端口就可以了，只有你主动去连接别人，别人不会主动连接你，没有人关心客户端监听到了哪里.

如果上面代码中的数据结构，里面的变量名称都有“be”两个字母，代表的意思是“big-endian”. 如果在网络上传输超过 1 Byte 的类型，就要区分大端（Big Endian）和小端（Little Endian）.

TCP/IP 栈是按照大端来设计的，而 x86 机器多按照小端来设计，因而发出去时需要做一个转换.

接下来，服务端要调用 `int listen(int sockfd, int backlog)` 进入 LISTEN 状态，等待客户端进行连接.

连接的建立过程，也即三次握手，是 TCP 层的动作，是在内核完成的，应用层不需要参与.

接着，服务端只需要调用 `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`，等待内核完成了至少一个连接的建立，才返回. 如果没有一个连接完成了三次握手，accept 就一直等待；如果有多个客户端发起连接，并且在内核里面完成了多个三次握手，建立了多个连接，这些连接会被放在一个队列里面. accept 会从队列里面取出一个来进行处理. 如果想进一步处理其他连接，需要调用多次 accept，所以 accept 往往在一个循环里面.

接下来，客户端可以通过 `int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen)` 函数发起连接.

connect()先在参数中指明要连接的 IP 地址和端口号，然后发起三次握手. 内核会给客户端分配一个临时的端口. 一旦握手成功，服务端的 accept 就会返回另一个 socket. 这里需要注意的是，监听的 socket 和真正用来传送数据的 socket，是两个 socket，一个叫作监听 socket，一个叫作已连接 socket. 成功连接建立之后，双方开始通过 read 和 write 函数来读写数据，就像往一个文件流里面写东西一样.

> 在端口不充足的情况下, connect系统调用的cpu消耗会大幅增加.

## udp编程
![udp编程模型](/misc/img/net/283b0e1c21f0277ba5b4b5cbcaca03b2.png)

UDP 是没有连接的，所以不需要三次握手，也就不需要调用 listen 和 connect，但是 UDP 的交互仍然需要 IP 地址和端口号，因而也需要 bind.

对于 UDP 来讲，没有所谓的连接维护，也没有所谓的连接的发起方和接收方，甚至都不存在客户端和服务端的概念，大家就都是客户端，也同时都是服务端. 只要有一个 socket，多台机器就可以任意通信，不存在哪两台机器是属于一个连接的概念. 因此，每一个 UDP 的 socket 都需要 bind. 每次通信时，调用 sendto 和 recvfrom，都要传入 IP 地址和端口.

```c
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
```

## 解析 socket 函数
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501
int __sys_socket(int family, int type, int protocol)
{
	int retval;
	struct socket *sock;
	int flags;

	/* Check the SOCK_* constants for consistency.  */
	BUILD_BUG_ON(SOCK_CLOEXEC != O_CLOEXEC);
	BUILD_BUG_ON((SOCK_MAX | SOCK_TYPE_MASK) != SOCK_TYPE_MASK);
	BUILD_BUG_ON(SOCK_CLOEXEC & SOCK_TYPE_MASK);
	BUILD_BUG_ON(SOCK_NONBLOCK & SOCK_TYPE_MASK);

	flags = type & ~SOCK_TYPE_MASK;
	if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
		return -EINVAL;
	type &= SOCK_TYPE_MASK;

	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

	retval = sock_create(family, type, protocol, &sock);
	if (retval < 0)
		return retval;

	return sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
}

SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
	return __sys_socket(family, type, protocol);
}
```

Socket 系统调用会调用 sock_create 创建一个 struct socket 结构，然后通过 sock_map_fd 和文件描述符对应起来.

在创建 Socket 的时候，有三个参数:
1. family，表示地址族. 不是所有的 Socket 都要通过 IP 进行通信，还有其他的通信方式. 例如，domain sockets 就是通过本地文件进行通信的，不需要 IP 地址. 只不过，通过 IP 地址只是最常用的模式，所以这里着重分析这种模式.

  ```c
  // https://elixir.bootlin.com/glibc/latest/source/bits/socket.h#L109
  /* Protocol families.  */
  #define	PF_UNSPEC	0	/* Unspecified.  */
  #define	PF_LOCAL	1	/* Local to host (pipes and file-domain).  */
  #define	PF_UNIX		PF_LOCAL /* Old BSD name for PF_LOCAL.  */
  #define	PF_FILE		PF_LOCAL /* POSIX name for PF_LOCAL.  */
  #define	PF_INET		2	/* IP protocol family.  */
  ...

  /* Address families.  */
  #define	AF_UNSPEC	PF_UNSPEC
  #define	AF_LOCAL	PF_LOCAL
  #define	AF_UNIX		PF_UNIX
  #define	AF_FILE		PF_FILE
  #define	AF_INET		PF_INET
  ...

  // 或者from kernel, **推荐**
  // https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/socket.h#L175
  /* Supported address families. */
  #define AF_UNSPEC	0
  #define AF_UNIX		1	/* Unix domain sockets 		*/
  #define AF_LOCAL	1	/* POSIX name for AF_UNIX	*/
  #define AF_INET		2	/* Internet IP Protocol 	*/
  #define AF_AX25		3	/* Amateur Radio AX.25 		*/
  #define AF_IPX		4	/* Novell IPX 			*/
  #define AF_APPLETALK	5	/* AppleTalk DDP 		*/
  #define AF_NETROM	6	/* Amateur Radio NET/ROM 	*/
  #define AF_BRIDGE	7	/* Multiprotocol bridge 	*/
  #define AF_ATMPVC	8	/* ATM PVCs			*/
  #define AF_X25		9	/* Reserved for X.25 project 	*/
  #define AF_INET6	10	/* IP version 6			*/
  #define AF_ROSE		11	/* Amateur Radio X.25 PLP	*/
  #define AF_DECnet	12	/* Reserved for DECnet project	*/
  #define AF_NETBEUI	13	/* Reserved for 802.2LLC project*/
  #define AF_SECURITY	14	/* Security callback pseudo AF */
  #define AF_KEY		15      /* PF_KEY key management API */
  #define AF_NETLINK	16
  #define AF_ROUTE	AF_NETLINK /* Alias to emulate 4.4BSD */
  #define AF_PACKET	17	/* Packet family		*/
  #define AF_ASH		18	/* Ash				*/
  #define AF_ECONET	19	/* Acorn Econet			*/
  #define AF_ATMSVC	20	/* ATM SVCs			*/
  #define AF_RDS		21	/* RDS sockets 			*/
  #define AF_SNA		22	/* Linux SNA Project (nutters!) */
  #define AF_IRDA		23	/* IRDA sockets			*/
  #define AF_PPPOX	24	/* PPPoX sockets		*/
  #define AF_WANPIPE	25	/* Wanpipe API Sockets */
  #define AF_LLC		26	/* Linux LLC			*/
  #define AF_IB		27	/* Native InfiniBand address	*/
  #define AF_MPLS		28	/* MPLS */
  #define AF_CAN		29	/* Controller Area Network      */
  #define AF_TIPC		30	/* TIPC sockets			*/
  #define AF_BLUETOOTH	31	/* Bluetooth sockets 		*/
  #define AF_IUCV		32	/* IUCV sockets			*/
  #define AF_RXRPC	33	/* RxRPC sockets 		*/
  #define AF_ISDN		34	/* mISDN sockets 		*/
  #define AF_PHONET	35	/* Phonet sockets		*/
  #define AF_IEEE802154	36	/* IEEE802154 sockets		*/
  #define AF_CAIF		37	/* CAIF sockets			*/
  #define AF_ALG		38	/* Algorithm sockets		*/
  #define AF_NFC		39	/* NFC sockets			*/
  #define AF_VSOCK	40	/* vSockets			*/
  #define AF_KCM		41	/* Kernel Connection Multiplexor*/
  #define AF_QIPCRTR	42	/* Qualcomm IPC Router          */
  #define AF_SMC		43	/* smc sockets: reserve number for
          * PF_SMC protocol family that
          * reuses AF_INET address family
          */
  #define AF_XDP		44	/* XDP sockets			*/

  #define AF_MAX		45	/* For now.. */

  /* Protocol families, same as address families. */
  #define PF_UNSPEC	AF_UNSPEC
  #define PF_UNIX		AF_UNIX
  #define PF_LOCAL	AF_LOCAL
  #define PF_INET		AF_INET
  #define PF_AX25		AF_AX25
  #define PF_IPX		AF_IPX
  #define PF_APPLETALK	AF_APPLETALK
  #define	PF_NETROM	AF_NETROM
  #define PF_BRIDGE	AF_BRIDGE
  #define PF_ATMPVC	AF_ATMPVC
  #define PF_X25		AF_X25
  #define PF_INET6	AF_INET6
  #define PF_ROSE		AF_ROSE
  #define PF_DECnet	AF_DECnet
  #define PF_NETBEUI	AF_NETBEUI
  #define PF_SECURITY	AF_SECURITY
  #define PF_KEY		AF_KEY
  #define PF_NETLINK	AF_NETLINK
  #define PF_ROUTE	AF_ROUTE
  #define PF_PACKET	AF_PACKET
  #define PF_ASH		AF_ASH
  #define PF_ECONET	AF_ECONET
  #define PF_ATMSVC	AF_ATMSVC
  #define PF_RDS		AF_RDS
  #define PF_SNA		AF_SNA
  #define PF_IRDA		AF_IRDA
  #define PF_PPPOX	AF_PPPOX
  #define PF_WANPIPE	AF_WANPIPE
  #define PF_LLC		AF_LLC
  #define PF_IB		AF_IB
  #define PF_MPLS		AF_MPLS
  #define PF_CAN		AF_CAN
  #define PF_TIPC		AF_TIPC
  #define PF_BLUETOOTH	AF_BLUETOOTH
  #define PF_IUCV		AF_IUCV
  #define PF_RXRPC	AF_RXRPC
  #define PF_ISDN		AF_ISDN
  #define PF_PHONET	AF_PHONET
  #define PF_IEEE802154	AF_IEEE802154
  #define PF_CAIF		AF_CAIF
  #define PF_ALG		AF_ALG
  #define PF_NFC		AF_NFC
  #define PF_VSOCK	AF_VSOCK
  #define PF_KCM		AF_KCM
  #define PF_QIPCRTR	AF_QIPCRTR
  #define PF_SMC		AF_SMC
  #define PF_XDP		AF_XDP
  #define PF_MAX		AF_MAX
  ```

> netlink相关的协议在[这里](https://elixir.bootlin.com/linux/v5.8.1/source/include/uapi/linux/netlink.h#L9). netlink[支持实现自定义协议](https://www.cnblogs.com/wenqiang/p/6306727.html), 有公司就基于netlink自定义协议实现了cdp功能.
  
1. type，也即 Socket 的类型. 类型是比较少的
1. protocol，是协议. 协议数目是比较多的，也就是说，多个协议会属于同一种类型. 常用的 Socket 类型有三种，分别是 SOCK_STREAM、SOCK_DGRAM 和 SOCK_RAW

  ```c
  // https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/net.h#L59
  /**
  * enum sock_type - Socket types
  * @SOCK_STREAM: stream (connection) socket
  * @SOCK_DGRAM: datagram (conn.less) socket
  * @SOCK_RAW: raw socket
  * @SOCK_RDM: reliably-delivered message
  * @SOCK_SEQPACKET: sequential packet socket
  * @SOCK_DCCP: Datagram Congestion Control Protocol socket
  * @SOCK_PACKET: linux specific way of getting packets at the dev level.
  *		  For writing rarp and other similar things on the user level.
  *
  * When adding some new socket type please
  * grep ARCH_HAS_SOCKET_TYPE include/asm-* /socket.h, at least MIPS
  * overrides this enum for binary compat reasons.
  */
  enum sock_type {
    SOCK_STREAM	= 1,
    SOCK_DGRAM	= 2,
    SOCK_RAW	= 3,
    SOCK_RDM	= 4,
    SOCK_SEQPACKET	= 5,
    SOCK_DCCP	= 6,
    SOCK_PACKET	= 10,
  };
  ```

  SOCK_STREAM 是面向数据流的，协议 IPPROTO_TCP 属于这种类型. SOCK_DGRAM 是面向数据报的，协议 IPPROTO_UDP 属于这种类型. 如果在内核里面看的话，IPPROTO_ICMP 也属于这种类型. SOCK_RAW 是原始的 IP 包，IPPROTO_IP 属于这种类型.

这里重点看 SOCK_STREAM 类型和 IPPROTO_TCP 协议. 为了管理 family、type、protocol 这三个分类层次，内核会创建对应的数据结构. 接下来，打开 [sock_create](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501) 函数看一下, 它会调用 [__sock_create](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501).

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501
/**
 *	__sock_create - creates a socket
 *	@net: net namespace
 *	@family: protocol family (AF_INET, ...)
 *	@type: communication type (SOCK_STREAM, ...)
 *	@protocol: protocol (0, ...)
 *	@res: new socket
 *	@kern: boolean for kernel space sockets
 *
 *	Creates a new socket and assigns it to @res, passing through LSM.
 *	Returns 0 or an error. On failure @res is set to %NULL. @kern must
 *	be set to true if the socket resides in kernel space.
 *	This function internally uses GFP_KERNEL.
 */

int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
{
	int err;
	struct socket *sock;
	const struct net_proto_family *pf;

	/*
	 *      Check protocol is in range
	 */
	if (family < 0 || family >= NPROTO)
		return -EAFNOSUPPORT;
	if (type < 0 || type >= SOCK_MAX)
		return -EINVAL;

	/* Compatibility.

	   This uglymoron is moved from INET layer to here to avoid
	   deadlock in module load.
	 */
	if (family == PF_INET && type == SOCK_PACKET) {
		pr_info_once("%s uses obsolete (PF_INET,SOCK_PACKET)\n",
			     current->comm);
		family = PF_PACKET;
	}

	err = security_socket_create(family, type, protocol, kern);
	if (err)
		return err;

	/*
	 *	Allocate the socket and allow the family to set things up. if
	 *	the protocol is 0, the family is instructed to select an appropriate
	 *	default.
	 */
	sock = sock_alloc();
	if (!sock) {
		net_warn_ratelimited("socket: no more sockets\n");
		return -ENFILE;	/* Not exactly a match, but its the
				   closest posix thing */
	}

	sock->type = type;

#ifdef CONFIG_MODULES
	/* Attempt to load a protocol module if the find failed.
	 *
	 * 12/09/1996 Marcin: But! this makes REALLY only sense, if the user
	 * requested real, full-featured networking support upon configuration.
	 * Otherwise module support will break!
	 */
	if (rcu_access_pointer(net_families[family]) == NULL)
		request_module("net-pf-%d", family);
#endif

	rcu_read_lock();
	pf = rcu_dereference(net_families[family]);
	err = -EAFNOSUPPORT;
	if (!pf)
		goto out_release;

	/*
	 * We will call the ->create function, that possibly is in a loadable
	 * module, so we have to bump that loadable module refcnt first.
	 */
	if (!try_module_get(pf->owner))
		goto out_release;

	/* Now protected by module ref count */
	rcu_read_unlock();

	err = pf->create(net, sock, protocol, kern);
	if (err < 0)
		goto out_module_put;

	/*
	 * Now to bump the refcnt of the [loadable] module that owns this
	 * socket at sock_release time we decrement its refcnt.
	 */
	if (!try_module_get(sock->ops->owner))
		goto out_module_busy;

	/*
	 * Now that we're done with the ->create function, the [loadable]
	 * module can have its refcnt decremented
	 */
	module_put(pf->owner);
	err = security_socket_post_create(sock, family, type, protocol, kern);
	if (err)
		goto out_sock_release;
	*res = sock;

	return 0;

out_module_busy:
	err = -EAFNOSUPPORT;
out_module_put:
	sock->ops = NULL;
	module_put(pf->owner);
out_sock_release:
	sock_release(sock);
	return err;

out_release:
	rcu_read_unlock();
	goto out_sock_release;
}
EXPORT_SYMBOL(__sock_create);
```

__sock_create先是分配了一个 [struct socket](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/net.h#L112) 结构. 接下来要用到 family 参数. 这里有一个 [net_families](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L173) 数组，可以以 family 参数为下标，找到对应的 [struct net_proto_family](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/net.h#L211).

> net_families的内容由[`sock_register`](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501)而来, 比如ipv4的`[(void)sock_register(&inet_family_ops)](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1974)`. family的值可参考[这里](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/socket.h#L175).

每一个地址族在net_families数组里面都有一项，里面的内容是 net_proto_family. 即每一种地址族都有自己的 net_proto_family，IP 地址族的 net_proto_family 定义如下，里面最重要的就是，create 函数指向 inet_create.

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1111
static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,
	.create = inet_create,
	.owner	= THIS_MODULE,
};
```

回到函数 __sock_create. 接下来, [inet_create](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L248) 会被调用.

在 inet_create 中，先会看到一个循环 list_for_each_entry_rcu. 在这里，第二个参数 type 开始起作用. 因为循环查看的是 inetsw[sock->type]. 这里的 inetsw 也是一个数组，type 作为下标，里面的内容是 struct inet_protosw，是协议，也即 inetsw 数组对于每个类型有一项，这一项里面是属于这个类型的协议.

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L126
/* The inetsw table contains everything that inet_create needs to
 * build a new socket.
 */
static struct list_head inetsw[SOCK_MAX];

// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1946
static int __init inet_init(void)
{
......
/* Register the socket-side information for inet_create. */
	for (r = &inetsw[0]; r < &inetsw[SOCK_MAX]; ++r)
		INIT_LIST_HEAD(r);

	for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
		inet_register_protosw(q);
......
}
```

inetsw 数组是在系统初始化的时候初始化的，就像上面代码里面实现的一样. 首先，一个循环会将 inetsw 数组的每一项，都初始化为一个链表. 一个 type 类型会包含多个 protocol，因而需要一个链表. 接下来一个循环，是将 [inetsw_array](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1120) 注册到 inetsw 数组里面去. inetsw_array 的定义如下，这个数组里面的内容很重要，后面会用到它们.

回到 inet_create 的 list_for_each_entry_rcu 循环中. 到这里就好理解了，这是在 inetsw 数组中，根据 type 找到属于这个类型的列表，然后依次比较列表中的 struct inet_protosw 的 protocol 是不是用户指定的 protocol；如果是，就得到了符合用户指定的 family->type->protocol 的 struct inet_protosw *answer 对象.

接下来，struct socket *sock 的 ops 成员变量，被赋值为 answer 的 ops. 对于 TCP 来讲，就是 inet_stream_ops. 后面任何用户对于这个 socket 的操作，都是通过 inet_stream_ops 进行的.

接下来，创建一个 struct sock *sk 对象. 这里比较让人困惑. socket 和 sock 看起来几乎一样，容易让人混淆，这里需要说明一下，**socket 是用于负责对上给用户提供接口，并且和文件系统关联, 而 sock，负责向下对接内核网络协议栈**.

在 sk_alloc 函数中，struct inet_protosw *answer 结构的 tcp_prot 赋值给了 struct sock *sk 的 sk_prot 成员. [tcp_prot](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L2638) 的定义如下，里面定义了很多的函数，都是 sock 之下内核协议栈的动作.

在 inet_create 函数中，接下来创建一个 struct inet_sock 结构，这个结构一开始就是 struct sock，然后扩展了一些其他的信息，剩下的代码就填充这些信息. 这一幕会经常看到，将一个结构放在另一个结构的开始位置，然后扩展一些成员，通过对于指针的强制类型转换，来访问这些成员.

socket 的创建至此结束.

## 解析 bind 函数
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L1501
/*
 *	Bind a name to a socket. Nothing much to do here since it's
 *	the protocol's responsibility to handle the local address.
 *
 *	We move the socket address to kernel space before we call
 *	the protocol layer (having also checked the address is ok).
 */

int __sys_bind(int fd, struct sockaddr __user *umyaddr, int addrlen)
{
	struct socket *sock;
	struct sockaddr_storage address;
	int err, fput_needed;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
		err = move_addr_to_kernel(umyaddr, addrlen, &address);
		if (!err) {
			err = security_socket_bind(sock,
						   (struct sockaddr *)&address,
						   addrlen);
			if (!err)
				err = sock->ops->bind(sock,
						      (struct sockaddr *)
						      &address, addrlen);
		}
		fput_light(sock->file, fput_needed);
	}
	return err;
}

SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
{
	return __sys_bind(fd, umyaddr, addrlen);
}
```

在 bind 中，sockfd_lookup_light 会根据 fd 文件描述符，找到 struct socket 结构. 然后将 sockaddr 从用户态拷贝到内核态，然后调用 struct socket 结构里面 ops 的 bind 函数. 根据前面创建 socket 的时候的设定，调用的是 [inet_stream_ops](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1015) 的 bind 函数，也即调用 [inet_bind](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L435).

bind 里面会[调用 sk_prot 的 get_port 函数](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L525)，也即 inet_csk_get_port 来检查端口是否冲突，是否可以绑定. 如果允许，则会设置 struct inet_sock 的本方的地址 inet_saddr 和本方的端口 inet_sport，对方的地址 inet_daddr 和对方的端口 inet_dport 都初始化为 0.

bind 的逻辑相对比较简单，就到这里了.

## 解析 listen 函数
```c
// https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1867
/*
 *	Perform a listen. Basically, we allow the protocol to do anything
 *	necessary for a listen, and if that works, we mark the socket as
 *	ready for listening.
 */

int __sys_listen(int fd, int backlog)
{
	struct socket *sock;
	int err, fput_needed;
	int somaxconn;

	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
		somaxconn = READ_ONCE(sock_net(sock->sk)->core.sysctl_somaxconn);
		if ((unsigned int)backlog > somaxconn)
			backlog = somaxconn;

		err = security_socket_listen(sock, backlog);
		if (!err)
			err = READ_ONCE(sock->ops)->listen(sock, backlog);

		fput_light(sock->file, fput_needed);
	}
	return err;
}

SYSCALL_DEFINE2(listen, int, fd, int, backlog)
{
	return __sys_listen(fd, backlog);
}
```

用户态的socket文件描述符fd只是一个整数而已,内核是没有办法直接用的. 在 listen 中需要通过 sockfd_lookup_light()，找到 struct socket 结构. 接着获取系统里的net.core.somaxconn内核参数的值,和用户传入的backlog比较后, 取一个最小值(**该值和半连接队列、全连接队列都有关系**)传入下一步. 最后调用 struct socket 结构里面 ops 的 listen 函数. 根据上面创建 socket 的时候的设定，调用的是 [inet_stream_ops](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L1051) 的 listen 函数，也即调用 [inet_listen](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L229)->[__inet_listen_sk](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L190)

__inet_listen_sk()设置的sk_max_ack_backlog是全连接队列的最大⻓度, 是执行listen()时传入的min(backag,net.core.somaxconn)的那个值. 因此如果遇到了全连接队列溢出的问题, 想加大该队列长度,那么需要同时考虑执行listen函数时传入的backlog和net.core.somaxconn.

同时如果这个 socket 还不在 TCP_LISTEN 状态，__inet_listen_sk()会调用 [inet_csk_listen_start](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/inet_connection_sock.c#L1235) 进入监听状态.

inet_csk_listen_start里面建立了一个新的结构 [inet_connection_sock](https://elixir.bootlin.com/linux/v6.8/source/include/net/inet_connection_sock.h#L82)，这个结构一开始是 struct inet_sock，inet_csk()其实做了一次强制类型转换，扩大了结构. 接着`reqsk_queue_alloc(&icsk->icsk_accept_queue)`初始化了接收队列.

> tcp_sock、inet_connection_sock、 inet_sock、 sock是逐层嵌套的关系, 类似面向对象里的继承. 对于TCP的socket来说, sock对象实际上是一个tcp_sock. 因此TCP中的sock对象随
时可以强制类型转换为tcp_sock、 inet_connecion_sock、 inet_sock、sock来使用.

struct inet_connection_sock 结构比较复杂. 如果打开它，就能看到处于各种状态的队列，各种超时时间、拥塞控制等字眼.

说TCP 是面向连接的，就是客户端和服务端都是有一个结构维护连接的状态，就是指这个结构.

首先，inet_connection_sock中的 icsk_accept_queue 是一个[request_sock_queue](https://elixir.bootlin.com/linux/v6.8/source/include/net/request_sock.h#L175)类型的对象, 是内核用来接收客户端请求的数据结构, 平时说的全连接队列(accept队列)就在这个数据结构里实现的.

>![旧版3.10 icsk_accept_queue](/misc/img/net/icsk_accept_queue.png)
> 新版request_sock_queue删除了半连接队列(syn请求队列)`struct listen_sock *listen_opt`.

对于全连接队列来说,在它上面不需要进行复杂的查找工作,accept处理的时候只是先进先出地接受就好了. 所以全连接队列通过rskq_accept_head和rskq_accept_tail以链表的形式来管理.

连接已满判断:
1. 全连接[sk_acceptq_is_full()](https://elixir.bootlin.com/linux/v6.8/source/include/net/sock.h#L1004)
1. 半连接[inet_csk_reqsk_queue_is_full()](https://elixir.bootlin.com/linux/v6.8/source/include/net/inet_connection_sock.h#L284)

在 TCP 的状态里面，有一个 listen 状态，当调用 listen 函数之后，就会进入这个状态，虽然写程序的时候，一般要等待服务端调用 accept 后，等待在那里的时候，让客户端就发起连接. 其实服务端一旦处于 listen 状态，不用 accept，客户端也能发起连接. 其实 TCP 的状态中，没有一个是否被 accept 的状态，那 accept 函数的作用是什么呢？

在内核中，为每个 Socket 维护两个队列. 一个是已经建立了连接的队列，这时候连接三次握手已经完毕，处于 established 状态；一个是还没有完全建立连接的队列，这个时候三次握手还没完成，处于 syn_rcvd 的状态. 服务端调用 accept 函数，其实是在第一个队列中拿出一个已经完成的连接进行处理. 如果还没有完成就阻塞等待. 这里的 icsk_accept_queue 就是第一个队列.

初始化完之后，将 TCP 的状态设置为 TCP_LISTEN，再次调用 get_port 判断端口是否冲突.

至此，listen 的逻辑就结束了.

## 解析 accept 函数
accept的重点工作就是从已经建立好的全连接队列中取出一个返回给用户进程.

```c
// https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1991
/*
 *	For accept, we attempt to create a new socket, set up the link
 *	with the client, wake up the client, then return the new
 *	connected fd. We collect the address of the connector in kernel
 *	space and move it to user at the very end. This is unclean because
 *	we open the socket then return an error.
 *
 *	1003.1g adds the ability to recvmsg() to query connection pending
 *	status to recvmsg. We need to add that support in a way thats
 *	clean when we restructure accept also.
 */

int __sys_accept4(int fd, struct sockaddr __user *upeer_sockaddr,
		  int __user *upeer_addrlen, int flags)
{
	int ret = -EBADF;
	struct fd f;

	f = fdget(fd);
	if (f.file) {
		ret = __sys_accept4_file(f.file, upeer_sockaddr,
					 upeer_addrlen, flags);
		fdput(f);
	}

	return ret;
}

SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr,
		int __user *, upeer_addrlen, int, flags)
{
	return __sys_accept4(fd, upeer_sockaddr, upeer_addrlen, flags);
}

SYSCALL_DEFINE3(accept, int, fd, struct sockaddr __user *, upeer_sockaddr,
		int __user *, upeer_addrlen)
{
	return __sys_accept4(fd, upeer_sockaddr, upeer_addrlen, 0);
}
```

accept 函数的实现，印证了 socket 的原理中说的那样，原来的 socket 是监听 socket，这里会找到原来的 struct socket，并基于它去创建一个新的 newsock. 这才是连接 socket. 除此之外，还会创建一个新的 struct file 和 fd，并关联到 socket.

这里面还会调用 struct socket 的 sock->ops->accept，也即会调用 inet_stream_ops 的 accept 函数，也即 [inet_accept](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L773).

inet_accept 会调用 struct sock 的 sk1->sk_prot->accept，也即 tcp_prot 的 accept 函数 [inet_csk_accept](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/inet_connection_sock.c#L654).

inet_csk_accept的reqsk_queue_remove()很简单,就是从全连接队列的链表里获取一个头元素返回就行了.

inet_csk_accept 的实现，印证了上面讲的两个队列的逻辑. 如果 icsk_accept_queue 为空，则调用 inet_csk_wait_for_connect 进行等待；等待的时候，调用 schedule_timeout，让出 CPU，并且将进程状态设置为 TASK_INTERRUPTIBLE. 如果再次 CPU 醒来，会接着判断 icsk_accept_queue 是否为空，同时也会调用 signal_pending 看有没有信号可以处理. 一旦 icsk_accept_queue 不为空，就从 inet_csk_wait_for_connect 中返回，在队列中取出一个 struct sock 对象赋值给 newsk.

## 解析 connect 函数
什么情况下，icsk_accept_queue 才不为空呢？当然是三次握手完成才可以.

![](/misc/img/net/ab92c2afb4aafb53143c471293ccb2df.png)

三次握手一般是由客户端调用 connect 发起.

客户端在执行connect函数的时候,把本地socket状态设置成了TCP_SYN_SENT, 选了一个可用的端口, 接着发出SYN握手请求并启动重传定时器.

```c
// https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L2031
/*
 *	Attempt to connect to a socket with the server address.  The address
 *	is in user space so we verify it is OK and move it to kernel space.
 *
 *	For 1003.1g we need to add clean support for a bind to AF_UNSPEC to
 *	break bindings
 *
 *	NOTE: 1003.1g draft 6.3 is broken with respect to AX.25/NetROM and
 *	other SEQPACKET protocols that take time to connect() as it doesn't
 *	include the -EINPROGRESS status for such sockets.
 */

int __sys_connect_file(struct file *file, struct sockaddr_storage *address,
		       int addrlen, int file_flags)
{
	struct socket *sock;
	int err;

	sock = sock_from_file(file);
	if (!sock) {
		err = -ENOTSOCK;
		goto out;
	}

	err =
	    security_socket_connect(sock, (struct sockaddr *)address, addrlen);
	if (err)
		goto out;

	err = READ_ONCE(sock->ops)->connect(sock, (struct sockaddr *)address,
				addrlen, sock->file->f_flags | file_flags);
out:
	return err;
}

int __sys_connect(int fd, struct sockaddr __user *uservaddr, int addrlen)
{
	int ret = -EBADF;
	struct fd f;

	f = fdget(fd);
	if (f.file) {
		struct sockaddr_storage address;

		ret = move_addr_to_kernel(uservaddr, addrlen, &address);
		if (!ret)
			ret = __sys_connect_file(f.file, &address, addrlen, 0);
		fdput(f);
	}

	return ret;
}

SYSCALL_DEFINE3(connect, int, fd, struct sockaddr __user *, uservaddr,
		int, addrlen)
{
	return __sys_connect(fd, uservaddr, addrlen);
}
```

connect()会先根据 fd 文件描述符，找到 [struct socket](https://elixir.bootlin.com/linux/v6.8/source/include/net/sock.h#L341) 结构. 接着，会调用 struct socket 结构里面 ops 的 connect 函数，根据前面创建 socket 的时候的设定，调用 [inet_stream_ops](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L1051) 的 connect 函数，也即调用 [inet_stream_connect](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L743).

在 inet_stream_connect -> [__inet_stream_connect](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L625) 里面，会发现，如果 socket 处于 SS_UNCONNECTED 状态(刚创建完毕的socket的状态)，那就调用 struct sock 的 sk->sk_prot->connect. 对于AF_INET的tcp socket来说, sk->sk_prot=tcp_prot, 因此connect()是[tcp_v4_connect](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L201).

tcp_v4_connect:
```c
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
	...
	tcp_set_state(sk, TCP_SYN_SENT);
	err = inet_hash_connect(tcp_death_row, sk); // 动态选择一个端口
	if (err)
		goto failure;

	sk_set_txhash(sk);

	rt = ip_route_newports(fl4, rt, orig_sport, orig_dport,
			       inet->inet_sport, inet->inet_dport, sk);
	if (IS_ERR(rt)) {
		err = PTR_ERR(rt);
		rt = NULL;
		goto failure;
	}
	...

	err = tcp_connect(sk); // 根据sk中的信息, 构建一个syn报文并发送出去

	...
}
EXPORT_SYMBOL(tcp_v4_connect);
```

[__inet_hash_connect()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/inet_hashtables.c#L994)传入的两个重要参数:
- port_offset = inet_sk_port_offset(sk): 根据要连接的目的ip和端口信息生成随机数
- __inet_check_established: 检查是否和现有ESTABLISH状态的连接冲突的函数

__inet_hash_connect逻辑:
1. `int port = inet_sk(sk)->inet_num;...if (port)`: 判断是否调用过bind
1. `inet_sk_get_local_port_range`: 会读取net.ipv4.ip_local_port_range的可用端口范围, 默认是32768~61000. 意味着端口总可用的数量是61000-32768=28232个.
1. `for (i = 0; i < remaining; i += step, port += step)`: 从某个随机数开始, 把整个可用端口范围遍历一遍, 直到找到可用的端口为止

  整个系统中会维护一个所有使用过的端口的哈希表,它就是hinfo->bhash. 如果在哈希表中没有找到,那么说明这个端口是可用的. 这个时候通过inet_bind_bucket_create()申请一个inet bind_bucket来记录端口已经被使用,并用哈希表的形式都管理了起来

  **如果ip_local_port_range中的端口快被用光了,这时候内核就大概率要把执行更多的循环才能找到可用端口, 这会导致connect系统调用的CPU开销上涨, 此时server负载可能还显示为不高**.

  因为在每次的循环内部需要等待锁以及在哈希表中执行多次的搜索. 这里的锁是自旋锁,是一种非阻塞的锁,如果资源被占用,进程并不会被挂起,而是会占用CPU去不断尝试获取锁.

  解决方法:
  1. 修改内核参数net.ipv4.ip_local_port_range多预留一些端口号
  1. 尽量复用连接, 使用长连接来削减频繁的握手处理
  1. 尽快回收TIME_WAT. 
  
    - 开启tcp_tw_reuse和tcp_tw_recycle, 不推荐
    - 设置最大TIME_WAIT数量: net.ipv4.tcp_max_tw_buckets= 10000

  1. `inet_is_local_reserved_port`: 跳过保留端口 by 是否在net.ipv4.ip_local_reserved_ports中
  1. 查找`是hinfo-=bhash`
  1. 通过 check_established = [__inet_check_established](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/inet_hashtables.c#L539) 继续检查是否可用

    check_estabished作用就是检测现有的TCP连接中是否四元组和要建立的连接四元素完全一致. 如果不完全一致,那么该端又仍然可用. 因此**connect时, 一个端口号可以用于多条连接, 比如client用同一个端口连接server端的两个不同端口. 这与bind的端口同协议下仅使用一次不同**.

    __inet_check_established:
    1. `head = inet_ehash_bucket(hinfo, hash)`找到哈希桶

      head是是所有ESTABLISH状态的socket组成的哈希表, 然后遍历这个哈希表, 使用inet_match来判断是否可用.
    1. `sk_nulls_for_each(sk2, node, &head->chain)`: 遍历是否存在一样的四元组
1. 遍历完所有端口都没找到合适的,就返回-EADDRNOTAVAL=`Cannot assign requested address`

在 tcp_v4_connect 函数中，[ip_route_connect](https://elixir.bootlin.com/linux/v6.8/source/include/net/route.h#L303) 其实是做一个路由的选择. 为什么呢？因为三次握手马上就要发送一个 SYN 包了，这就要凑齐源地址、源端口、目标地址、目标端口. 目标地址和目标端口是服务端的，已经知道源端口是客户端随机分配的，源地址应该用哪一个呢？这时候要选择一条路由，看从哪个网卡出去，就应该填写哪个网卡的 IP 地址. 接下来，在发送 SYN 之前，先将客户端 socket 的状态设置为 TCP_SYN_SENT, 然后初始化 TCP 的 seq num，也即 write_seq，然后调用 [tcp_connect](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_output.c#L4022) 进行发送.

在 tcp_connect 中，有一个新的结构 [struct tcp_sock](https://elixir.bootlin.com/linux/v6.8/source/include/linux/tcp.h#L192)，如果打开它，会发现它是 struct inet_connection_sock 的一个扩展，struct inet_connection_sock 在 struct tcp_sock 开头的位置，通过强制类型转换访问，故伎重演又一次. struct tcp_sock 里面维护了更多的 TCP 的状态，同样是遇到了再分析.

接下来 tcp_init_nondata_skb 初始化一个 SYN 包, `tcp_connect_queue_skb`添加包到发送队列sk_write_queue，tcp_transmit_skb 将 SYN 包发送出去，inet_csk_reset_xmit_timer 设置了一个 timer，如果 SYN 发送不成功，则再次发送.

该定时器的作用是等到一定时间后收不到服务端的反馈的时候来开启重传. 首次超时时间是在[TCP_TIMEOUT_INIT](https://elixir.bootlin.com/linux/v6.8/source/include/net/tcp.h#L150)宏中定义的, 由[`tcp_connect_init`](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_output.c#L3832)的`inet_csk(sk)->icsk_rto = tcp_timeout_init(sk)`设置, 该值在Linux 3.10版本中是1秒,在一些老版本中是3秒.

如果能正常接收到服务端响应的synack,那么客户端的这个定时器会清除. 这段逻辑在tcp_rearm_rto里, 调用顺序为tcp_rcv_state_process -> tcp_rev_synsent_state_
proces -> tcp_ack -> tcp_clean_rtx_queue -> top_rearm_rto. 如果服务端发生了丢包,那么定时器到时后会进入回调函数[tcp_write_timer](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_timer.c#L702)中进行重传. 其实不只是握手,连接状态的超时重传也是在tcp_write_timer完成的.

> tcp_write_timeout会判断是否重试过多, 如果是则退出重试逻辑. **下一次重传的时间是上一次的两倍**. 对于SYN握手包主要的判断依据是net.ipv4.tcp_syn_retries.

发送网络包的过程，放到之后讲解. 这里姑且认为 SYN 已经发送出去了.

回到 __inet_stream_connect 函数，在调用 sk->sk_prot->connect 之后，inet_wait_for_connect 会一直等待客户端收到服务端的 ACK. 而我们知道，服务端在 accept 之后，也是在等待中. 网络包是如何接收的呢？对于解析的详细过程，还是在会在之后讲解，这里为了解析三次握手，简单的看网络包接收到 TCP 层做的部分事情.

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1737
/* thinking of making this const? Don't.
 * early_demux can change based on sysctl.
 */
static struct net_protocol tcp_protocol = {
	.early_demux	=	tcp_v4_early_demux,
	.early_demux_handler =  tcp_v4_early_demux,
	.handler	=	tcp_v4_rcv,
	.err_handler	=	tcp_v4_err,
	.no_policy	=	1,
	.netns_ok	=	1,
	.icmp_strict_tag_validation = 1,
};
```

kernel会通过 struct net_protocol 结构中的 handler 进行接收，调用的函数是 tcp_v4_rcv. 接下来的调用链为 tcp_v4_rcv->[tcp_v4_do_rcv](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L1885)->tcp_rcv_state_process. [tcp_rcv_state_process](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_input.c#L6619)，顾名思义，是用来处理接收一个网络包后引起状态变化的.

```c
int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
	enum skb_drop_reason reason;
	struct sock *rsk;

	...

	if (sk->sk_state == TCP_LISTEN) { // 服务端收到第一步握手SYN或者第三步ACK都会走到这里
		struct sock *nsk = tcp_v4_cookie_check(sk, skb);

		if (!nsk)
			goto discard;
		if (nsk != sk) {
			if (tcp_child_process(sk, nsk, skb)) {
				rsk = nsk;
				goto reset;
			}
			return 0;
		}
	} else
		sock_rps_save_rxhash(sk, skb);

	if (tcp_rcv_state_process(sk, skb)) {
		rsk = sk;
		goto reset;
	}
	return 0;

reset:
	tcp_v4_send_reset(rsk, skb);
 ...
}
EXPORT_SYMBOL(tcp_v4_do_rcv);
```

```c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct inet_connection_sock *icsk = inet_csk(sk);
	const struct tcphdr *th = tcp_hdr(skb);
	struct request_sock *req;
	int queued = 0;
	bool acceptable;
	SKB_DR(reason);

	switch (sk->sk_state) {
  ...

	case TCP_LISTEN:
		if (th->ack)
			return 1;

		if (th->rst) {
			SKB_DR_SET(reason, TCP_RESET);
			goto discard;
		}
		if (th->syn) { // 判断是否为SYN握手包
			if (th->fin) {
				SKB_DR_SET(reason, TCP_FLAGS);
				goto discard;
			}
			/* It is possible that we process SYN packets from backlog,
			 * so we need to make sure to disable BH and RCU right there.
			 */
			rcu_read_lock();
			local_bh_disable();
			acceptable = icsk->icsk_af_ops->conn_request(sk, skb) >= 0;
			local_bh_enable();
			rcu_read_unlock();

			if (!acceptable)
				return 1;
			consume_skb(skb);
			return 0;
		}
		SKB_DR_SET(reason, TCP_FLAGS);
		goto discard;

	case TCP_SYN_SENT:
	...
}
EXPORT_SYMBOL(tcp_rcv_state_process);
```

目前服务端是处于 TCP_LISTEN 状态的，而且发过来的包是 SYN，因而就有了上面的代码，调用 icsk->icsk_af_ops->conn_request 函数. struct inet_connection_sock 对应的操作是 [inet_connection_sock_af_ops](https://elixir.bootlin.com/linux/v6.8/source/include/net/inet_connection_sock.h#L35)的[ipv4_specific](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L2433)，按照下面的定义，其实调用的是 [tcp_v4_conn_request](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L1710).

tcp_v4_conn_request:
```c
int tcp_conn_request(struct request_sock_ops *rsk_ops,
		     const struct tcp_request_sock_ops *af_ops,
		     struct sock *sk, struct sk_buff *skb)
{
	struct tcp_fastopen_cookie foc = { .len = -1 };
	__u32 isn = TCP_SKB_CB(skb)->tcp_tw_isn;
	struct tcp_options_received tmp_opt;
	struct tcp_sock *tp = tcp_sk(sk);
	struct net *net = sock_net(sk);
	struct sock *fastopen_sk = NULL;
	struct request_sock *req;
	bool want_cookie = false;
	struct dst_entry *dst;
	struct flowi fl;
	u8 syncookies;

  ...
	if ((syncookies == 2 || inet_csk_reqsk_queue_is_full(sk)) && !isn) { // inet_csk_reqsk_queue_is_full: 半连接队列是否已满
		want_cookie = tcp_syn_flood_action(sk, rsk_ops->slab_name); // tcp_syn_flood_action是否开启了tcp_syncookies. 如果队列已满且未开启tcp_syncookies, 直接丢弃
		if (!want_cookie)
			goto drop;
	}

	if (sk_acceptq_is_full(sk)) { // 全连接队列是否已满
		NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS);
		goto drop;
	}

	req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie); // 分配request_sock对象
	if (!req)
		goto drop;

	...
	if (fastopen_sk) {
		af_ops->send_synack(fastopen_sk, dst, &fl, req,
				    &foc, TCP_SYNACK_FASTOPEN, skb);
		/* Add the child socket directly into the accept queue */
		if (!inet_csk_reqsk_queue_add(sk, req, fastopen_sk)) {
			reqsk_fastopen_remove(fastopen_sk, req, false);
			bh_unlock_sock(fastopen_sk);
			sock_put(fastopen_sk);
			goto drop_and_free;
		}
		sk->sk_data_ready(sk);
		bh_unlock_sock(fastopen_sk);
		sock_put(fastopen_sk);
	} else {
		tcp_rsk(req)->tfo_listener = false;
		if (!want_cookie) {
			req->timeout = tcp_timeout_init((struct sock *)req);
			inet_csk_reqsk_queue_hash_add(sk, req, req->timeout); // 添加到半连接队列, 并开启计时器. 该计时器作用: 如果某个时间内还没有收到client的第三次握手, 服务端会重传synack包.
		}
		af_ops->send_synack(sk, dst, &fl, req, &foc,
				    !want_cookie ? TCP_SYNACK_NORMAL :
						   TCP_SYNACK_COOKIE,
				    skb);
		if (want_cookie) {
			reqsk_free(req);
			return 0;
		}
	}
	...
}
EXPORT_SYMBOL(tcp_conn_request);
```

SYN Flood攻击就是通过耗光服务端上的半连接队列来使得正常的用户连接请求无法被响应. 不过在现在的Linux内核里只要打开tcp_syncookies,半连接队列满了仍然可以保证正常握手的进行

如果全队列满了, 服务器对握手包的处理还是会丢弃它.

假设服务端侧发生了全/半连接队列溢出而导致的丢包,那么转换到客户端视角来看就是SYN包没有任何响应.

此时客户端会利用发送SYN时开启的一个重传定时器, 如果收不到预期的synack,超时重传的逻辑就会开始执行, 其时间单位都是以秒来计算的, 这意味着,如果有握手重传发生,即使第一次重传就能成功,那接又最快响应也是1秒以后的事情了, 这对接又耗时影响非常大.

tcp_v4_conn_request 会调用 [tcp_conn_request](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_input.c#L7089)，这个函数也比较长，里面调用了 send_synack，但实际调用的是 [tcp_v4_send_synack](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L1163). 具体发送的过程不去管它，看注释能知道，这是收到了 SYN 后，回复一个 SYN-ACK，回复完毕后，服务端处于 TCP_SYN_RECV.

因此服务端响应ack的主要工作是判断接收队列是否满了,满的话可能会丢弃该请求,否则发出synack. 申请request_sock添加到半连接队列中,同时启动定时器.

这个时候，轮到客户端接收网络包了. 都是 TCP 协议栈，所以过程和服务端没有太多区别，还是会走到 tcp_rcv_state_process 函数的，只不过由于客户端目前处于 TCP_SYN_SENT 状态，就进入了下面的代码分支.

```c
static int tcp_rcv_synsent_state_process(struct sock *sk, struct sk_buff *skb,
					 const struct tcphdr *th)
{
	...
  tcp_ack(sk, skb, FLAG_SLOWPATH) // ->  tcp_clean_rtx_queue(): 删除发送队列; 删除定时器等.
  ...

		tcp_finish_connect(sk, skb); // 连接建立完成

		...
		}
		tcp_send_ack(sk);
		return -1;
	}

	...
}

// https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_input.c#L6215
void tcp_finish_connect(struct sock *sk, struct sk_buff *skb)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct inet_connection_sock *icsk = inet_csk(sk);

	tcp_ao_finish_connect(sk, skb);
	tcp_set_state(sk, TCP_ESTABLISHED); // 修改socket状态
	icsk->icsk_ack.lrcvtime = tcp_jiffies32;

	if (skb) {
		icsk->icsk_af_ops->sk_rx_dst_set(sk, skb);
		security_inet_conn_established(sk, skb);
		sk_mark_napi_id(sk, skb);
	}

	tcp_init_transfer(sk, BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB, skb); // 初始化拥塞控制

	/* Prevent spurious tcp_cwnd_restart() on first data
	 * packet.
	 */
	tp->lsndtime = tcp_jiffies32;

	if (sock_flag(sk, SOCK_KEEPOPEN))
		inet_csk_reset_keepalive_timer(sk, keepalive_time_when(tp)); // 保活计时器打开

	if (!tp->rx_opt.snd_wscale)
		__tcp_fast_path_on(tp, tp->snd_wnd);
	else
		tp->pred_flags = 0;
}

// https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_output.c#L4194
void __tcp_send_ack(struct sock *sk, u32 rcv_nxt)
{
	struct sk_buff *buff;

	/* If we have been reset, we may not send again. */
	if (sk->sk_state == TCP_CLOSE)
		return;

	/* We are not putting this on the write queue, so
	 * tcp_transmit_skb() will set the ownership to this
	 * sock.
	 */
	buff = alloc_skb(MAX_TCP_HEADER,
			 sk_gfp_mask(sk, GFP_ATOMIC | __GFP_NOWARN)); // 申请和构建ack包
	...
	/* Send it off, this clears delayed acks for us. */
	__tcp_transmit_skb(sk, buff, 0, (__force gfp_t)0, rcv_nxt); // 发送ack包
}
EXPORT_SYMBOL_GPL(__tcp_send_ack);
```

tcp_rcv_synsent_state_process是客户端响应synack的主要逻辑:
1.  tcp_finish_connect: 客户端将自己的socket状态修改为ESTABLISHED,接着打开TCP的保活计时器
2. [tcp_send_ack](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_output.c#L4236)->__tcp_send_ack: 在__tcp_send_ack中构造ack包,并把它发送出去

客户端响应来自服务端的synack时清除了connect时设置的重传定时器,把当前socket状态设置为ESTABLISHED,开启保活计时器后发出第三次握手的ack确认.

又轮到服务端接收网络包了，还是tcp_v4_do_rcv(), 由于这已经是第三次握手了,半连接队列里会存在第一次握手时留下的半连接信息, 所以tcp_v4_cookie_check()的执行逻辑会不太一样. tcp_v4_cookie_check()会检查syncookie，因为没有syn包没有ack选项，因此忽略， 如果syncookie验证通过则创建新的sock.

> [tcp_v4_hnd_req()]已被tcp_v4_cookie_check()取代 by v4.4-rc1 [tcp/dccp: install syn_recv requests into ehash table](https://github.com/torvalds/linux/commit/079096f103faca2dd87342cca6f23d4b34da8871)

> [v3.19 inet_csk_reqsk_queue_unlink(sk, req, prev)/inet_csk_reqsk_queue_removed(sk, req)](https://elixir.bootlin.com/linux/v3.19/source/net/ipv4/tcp_minisocks.c#L723)->[v4.3 inet_csk_reqsk_queue_drop(sk, req)/inet_csk_reqsk_queue_add(sk, req, child)](https://elixir.bootlin.com/linux/v4.3/source/net/ipv4/tcp_minisocks.c#L560)-> 没找到具体什么时候删除

tcp_v4_cookie_check->[cookie_v4_check](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/syncookies.c#L389)->[tcp_get_cookie_sock](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/syncookies.c#L205)

tcp_get_cookie_sock:
```c
struct sock *tcp_get_cookie_sock(struct sock *sk, struct sk_buff *skb,
				 struct request_sock *req,
				 struct dst_entry *dst)
{
	struct inet_connection_sock *icsk = inet_csk(sk);
	struct sock *child;
	bool own_req;

	child = icsk->icsk_af_ops->syn_recv_sock(sk, skb, req, dst,
						 NULL, &own_req); // 创建子socket
	if (child) {
		refcount_set(&req->rsk_refcnt, 1);
		sock_rps_save_rxhash(child, skb);

		if (rsk_drop_req(req)) {
			reqsk_put(req);
			return child;
		}

		if (inet_csk_reqsk_queue_add(sk, req, child)) // 加入全连接队列
			return child;

		bh_unlock_sock(child);
		sock_put(child);
	}
	__reqsk_free(req);

	return NULL;
}
EXPORT_SYMBOL(tcp_get_cookie_sock);
```

icsk->icsk_af_ops->syn_recv_sock是一个指针,icsk->icsk_af_ops=ipv4_specific, 因此它指向的是[tcp_v4_syn_recv_sock()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L1730).

在第三次握手里又会在tcp_v4_syn_recv_sock()里继续判断一次全连接队列是否满了,如果满了修改一下计数器就丢弃了. 如果队列不满,那么就申请创建新的sock对象. 在用户进程调用accept
的时候,直接把该对象取出来,再包装成一个socke对象就返回了.

注意, **这里即第三次握手失败并不是客户端重试, 而是由服务端来重发Synack. 重试次数由net.ipv4.tcp_synackretries控制. 在实践中, 客户端发送ack后往往是以为连接建立成功了, 并开始发送数据, 其实这时候连接还没有真的建立起来. 它发出去的数据,包括重试将全部被服务端无视, 直到连接真正建立成功后才行**.

再到 tcp_rcv_state_process(), 由于服务端目前处于状态 TCP_SYN_RECV 状态，因而走了`case TCP_SYN_RECV`分支. 当收到这个网络包的时候，服务端也处于 TCP_ESTABLISHED 状态，三次握手结束.

## 总结
![](/misc/img/net/c028381cf45d65d3f148e57408d26bd8.png)

其实socket的系统调用规律就都一样了：
- bind 第一层调用 inet_stream_ops 的 inet_bind 函数，第二层调用 tcp_prot 的 inet_csk_get_port 函数
- listen 第一层调用 inet_stream_ops 的 inet_listen 函数，第二层调用 tcp_prot 的 inet_csk_get_port 函数
- accept 第一层调用 inet_stream_ops 的 inet_accept 函数，第二层调用 tcp_prot 的 inet_csk_accept 函数
- connect 第一层调用 inet_stream_ops 的 inet_stream_connect 函数，第二层调用 tcp_prot 的 tcp_v4_connect 函数

建立一条TCP连接需要消耗多⻓时间, 以上几步操作,可以简单划分为两类:
1. 第一类是内核消耗CPU进行接收、发送或者是处理,包括系统调用、软中断和上下文切换. 它们的耗时基本都是几微秒左右
2. 第二类是网络传输,当包被从一台机器上发出以后,中间要经过各式各样的网线,各种交换机路由器。所以网络传输的耗时相比本机的CPU处理,就要高得多了. 根据网络远近一般在几毫秒到几百毫秒不等.

所以,在正常的TCP连接的建立过程中,一般考虑网络延时即可.

一个RTT指的是包从一台机器到另外一台机器的一个来回的延迟时间, 所以从全局来看, TCP连接建立的网络耗时大约需要三次传输,再加上少许的双方CPU开销,总共大约比1.5倍RTT大一点点. 不过从客户端视角来看,只要ACK包发出了,内核就认为连接建立成功,可以开始发送数据了. 所以如果在客户端打点统计TCP连接建立耗时,只需两次传输耗时-—即1个RTT多一点的时间. 对于服务端视角来看同理,从SYN包收到开始算,到收到ACK,中间也是一次RTT耗时. 不过这些针对的是握手正常的情况,如果握手过程出了问题,可就不是这么回事了.

### 丢包
服务端在第一次握手时,在如下两种情况下可能会丢包:
1. 半连接队列滿, 且tcp_syncookes为0
1. 全连接队列满,且有未完成的半连接请求

在这两种情况下, 从客户端视角来看和网络断了没有区别, 都是发出去的SYN包没有响应, 然后等待定时器到时后重传握手请求. 总的重传次数受net.ipv4.tcp_syn_retries
影响(而不是决定).

sever端在第三次握手时也可能出问题,如果全连接队列满,仍将发生丢包. 不过第三次握手失败时,只有服务端知道(客户端误以为连接已经建立成功). 服务端根据半
连接队列里的握手信息发起synack重试, 重试次数由net.ipv4.top_synack_retries控制.

一旦线上出现了上面这些连接队列溢出导致的问题,服务端将会受到比较严重的影响. 即使第一次重试就能够成功, 但接口响应耗时将直接上涨到秒. 如果重试两三次都没有成功, Nginx很有可能直接就报访问超时失败了.

解决方法:
1. 打开syncookie
  在现代的Linux版本里, 可以通过打开tcp_syncookies来防止过多的请求打满半连接队列, 包括SYN Flood攻击, 来解决服务端因为半连接队列满而发生的丢包.
2. 加大连接队列长度

  全连接队列的长度是min(backlog, net.core.somaxconn), 半连接队列长度有点小复杂还没定位到, 但以前是`min(backlog, somaxconn, tcp_max_syn_backlog)+ 1 再上取整到2的N次幂,且最小不能小于16`.

  如果需要加大全/半连接队列长度,需调节以上的一个或多个参数来达到目的. 只要队列长度合适, 就能很大程度降低握手异常概率的发生. 其中全连接队列在修改完后可以通过`ss -nlt`中输出的Send-Q来确认最终生效长度.

  > Recv-Q表示当前该进程的全连接队列使用情况, 如果Recv-Q已经逼近了Send-Q,那么可能不需要等到丟包也应该准备加大全连接队列了.
1. 尽快调用accept

  这虽然一般不会成为问题, 应用应尽快在握手成功之后通过accept把新连接取走, 不要忙于处理其他业务逻辑而导致全连接队列塞满了
1. 尽早拒绝

  如果加大队列后仍然有非常偶发的队列溢出, 可以暂且容忍. 但如果仍然有较长时间处理不过来, 那就直接报错,不要让容户端超时等待. 比如将Redis、MySQL等服务器的内核参数tcp_abort_on _overtlow设置为1. 如果队列满了,直接发resel指令给客户端, 告诉其不要傻等. 这时候客户端会收到错误`cornection reset by peer`. 拒绝少量用户的访问请求, 比把整个服务都搞崩了还是要强的
1. 尽量减少TCP连接的次数
  
  如果上述方法都未能根治问题, 这时应该思考是否可以用长连接代替短连接, 减少过于频繁的三次握手. 该放法不但能降低握手出问题的可能性, 而且还顺带砍掉了三次握手的各种内存、CPU、时间上的开销,对提升性能也有较大帮助.

判断—台服务器当前是否有半/全连接队列溢出产生丢包的方法:
1. 全队列

  全连接队列溢出都会记录到ListenOverflows这个MIB (Management Information Base,管理信息库), 对应SNMP统计信息中的ListenDrops这一项, 共有两处:
  1. server在响应client的SYN握手包时: [tcp_conn_request]的[`NET_INC_STATS(sock_net(sk), LINUX_MIB_LISTENOVERFLOWS)`](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_input.c#L7122)和后面的`tcp_listendrop()`, 可以看到,全连接队列满了以后调用NET_INC_STATS增加了LINUX_MIB_LISTENOVERFLOWS和LINUX_MIB_LISTENDROPS这两个MIB.
  2. server在响应第三次握手的时候: [tcp_v4_syn_recv_sock](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L1748)会再次判断全连接队列是否溢出. 如果溢出, 同样会增加这两个MIB

  > 在[/net/ipv4/proc.c](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/proc.c#L195)中,LINUX_MIB_LISTENOVERFLOWS和LINUX_MIB_LISTENDROPS都被整合进SNMP统计信息

  在执行`nstat -z -t 1 |grep -i ListenOverflows`/[`netstat -s | grep overflowed`](https://github.com/giftnuss/net-tools/blob/master/statistics.c#L243)时, 就可以判断是否有丢包发生, 因为ListenOverflows只有在全连接队列满的时候才会增加

1. 半队列

  溢出时更新的是LINUX_MIB_LISTENDROPS这个MIB,对应到SNMP就是ListenDrops这个统计项. 在[tcp_conn_request()](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_input.c#L7118).

  但是问题在于，不仅仅只是在半连接队列发生溢出的时候会增加该值,比如 tcp_conn_request()和 [`tcp_req_err->httptcp_req_err`](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_ipv4.c#L476). 所以根据上面的 nstat/netstat 看半连接队列是否溢出是不靠谱的.

  建议是不要纠结怎么看是否丢包了. 直接看tcp_syncookies是不是为1就行. 如果该值是1, 那么[`want_cookie = tcp_syn_flood_action(sk, rsk_ops->slab_name);`](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/tcp_input.c#L7122)就返回真,是根本不会发生半连接溢出丟包的. 如果tcp_syncookies不是1,则建议改成1就行了.

  如果因为各种原因就是不想打开tcp_syncookies,就想吞看是否有因为半连接队列满而导致的SYN丢弃, 除了`nstat -z -t 1 |grep -i ListenDrops`/`netstat -s|grep SYNs`的结果,建议同时查看当前listen端口上的SYN_RECV的数量(`ss -antp|grep SYN_RECV|wc-l`/`netstat -antp |grep SYN_RECV|wc -l`). 如果SYN_RECV状态的连接数量达到算出来的队列长度,那么可以确定有半连接队列溢出.

### 内存占用
tcp连接相关的内核对象
- socket
  socket的创建方式有两种:
  1. 一种是直接调用socket函数

    `__sys_socket_create`->`sock_create`->`sock_create`-> `__sock_create`-> [`sock = sock_alloc();`](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1535), 在sock_alloc函数中,申请了一个struct socket内核对象, 并将其与inode信息关联起来.

    sock_alloc=> new_node_pseudo => alloc_inode (`sock_mnt->mnt_sb->op`=`&sockfs_ops`) => `inode = ops->alloc_inode(sb)`, 因此直接看[sock_alloc_inode](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L304), 它调用alloc_inode_sb从sock_inode_cachep缓存中申请一个struct socket_alloc对象(该对象还包括socket.wq, 因为较小, 忽略)

    > `sock_mnt = kern_mount(&sock_fs_type);`=> 
      `fs_context_for_mount()` => 
      
        => `init_fs_context = fc->fs_type->init_fs_context`
        => init_fs_context(fc)=>
          => `init_pseudo(fc, SOCKFS_MAGIC)`: `fc->fs_private = ctx`+`fc->ops = &pseudo_fs_context_ops`
          => `ctx->ops = &sockfs_ops;`
      `fc_mount(fc)` 
          => vfs_get_tree(): fc->ops->get_tree(fc) => [vfs_get_super()](https://elixir.bootlin.com/linux/v6.8/source/fs/super.c#L1255) : [`s->s_op = ctx->ops ?: &simple_super_operations`](https://elixir.bootlin.com/linux/v6.8/source/fs/libfs.c#L580)
          => vfs_create_mount(): [`mnt->mnt.mnt_sb = fc->root->d_sb`](https://elixir.bootlin.com/linux/v6.8/source/fs/namespace.c#L1111)

  1. 另外一种是调用accept接收

    [`SYSCALL_DEFINE4(accept4, ...)`](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1535)->`__sys_accept4`->`__sys_accept4_file`->`do_accept`

    do_accept调用了`newsock = sock_alloc()`和`newfile = sock_alloc_file()`创建了sock_alloc_inode(该对象中包含了struct inode 和struct socket),struct dentry和struct file 3个内核对象的申请.

    不过tcp_sock对象的创建过程有点不太一样, 服务端内核在第三次握手成功的时候,就已经创建好了tcp_sock, 并且一同放到了全连接队列中. 这样在调用accept函数接收的时候,只需要从全连接队列中取出来直接用就行了,无须再单独申请.

1. tcp_sock

  对于IPV4来说, inet协议族对应的create()是[inet_create](https://elixir.bootlin.com/linux/v6.8/source/net/ipv4/af_inet.c#L251) by inet_family_ops.

  因此__sock create中对pf->create的调用会执行到inet_create(). 在这个函数中, 将会到TCP这个slab缓存中申请一个stuct sock内核对象出来, 其中TCP这个slab缓存是在inet_init [`proto_register(&tcp_prot, 1)`](https://elixir.bootlin.com/linux/v6.8/source/net/core/sock.c#L3925)中初始化的. 即协议栈初始化的时候,会创建一个名为TCP、大小为sizeof(struct tcp_ sock)的slab缓存,并把它记到tcp_prot->slab的字段下.

  > 在TCP slab缓存中实际存放的是struct tcp_sock对象,是struct sock的扩展.
1. dentry

   socket系统调用[`SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)`](https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L1535), 除了__sys_socket_create以外,还调用了一个sock_map_fd, 以此为入口将完成struct dentry的申请.

   内核初始化的时候创建好了一个dentry_cache的slab缓存,所有的struct dentry对象都将在这里进行分配.

   `sock_map_fd`->`sock_alloc_file`->`alloc_file_pseudo`->`d_alloc_pseudo`

   alloc_file_pseudo其实完成了struct dentry和struct file 两个内核对象的申请.
1. flip对象申请 (struct file)

  由alloc_file_pseudo()的alloc_file()申请. 在Linux上,一切皆是文件,正是通过和struct file对象的关联来让sacket看起来也是一个文件. struct file是通过[filp_cachep](https://elixir.bootlin.com/linux/v6.8/source/fs/file_table.c#L469)来进行管理的.

实测TCP内核对象开销:
1. 准备阶段
  1. 客户端

    1. 需要调整如下内核参数:
      1. 调整ip_local_port_range来保证可用端口数大于5万个
      1. 保证tw_reuse和tw_recycle是关闭状态的,否则连接无法进入TIME_WAIT
      1. 调整tcp_max_tw_buckets保证能有5万个TIME_WAT状态供观察

    2. `echo "3" > /proc/sys/vm/drop_caches` # 先清理pagecache、dentries和nodes
    3. `cat /proc/meminfo |grep Slab`记录输出c1

    > 可用slabtop查看slab变化

  1. server
    1. 需要调整如下内核参数:
      1. net.core.somaxconn = 1024 : 避免把连接队列打满,进而导致握手过慢

    2. `echo "3" > /proc/sys/vm/drop_caches` # 先清理pagecache、dentries和nodes
    3. `cat /proc/meminfo |grep Slab` #记录输出s1
1. 实验

  1. 在client端发起请求5000个
  2. 在client的其他terminal观察slabtop

    和实验开始前的数据相比, kmalloc-64、dentry、kmalloc-256、 sock_inode_cache、TCP这5个内核对象都有了明显的增加. 这些其实就是上面提到的socket内部相关的内核对象. 其中的kmalloc-256是filp. krnalloc-64既包括前文提到的socket_wq,也包括记录端口使用关系的哈希表中使用的inet_bind_bucket元素. 每次使用一个端口的时候,就会申请—个inet_bind_bucket以记录该端口被使用过, 所有的inet_bind_bucket以哈希表的形式组织了起来, 以便下次再选择端口的时候查找该哈希表来判断一个端口有没有被使用.

    至于不直接显示filp, tcp_bind_bucket等,而是显示kmalloc-xx,那是因为Linux内部的一个叫slab merging的功能, 该功能可能会将同等大小的stab缓存放到一起. Linux源码中提供了工具`slabinfo -a`可以查看都有哪些slab参与了合并.

    > cd kernel/tools/vm && make slabinfo
  3. 在client查看`cat /proc/meminfo |grep Slab`记录输出c2, . 结果就是(c2-c1)/5000≈3.34KB/一条ESTABLISH状态的空连接
  4. 在server的其他terminal观察slabtop, 同理根据`cat /proc/meminfo |grep Slab`记录输出s2, 算得一条ESTABLISH状态的空连接大约是3.05KB

    大致也是kmalloc-64、dentry、kmalloc-256、sock_inode_cache、TCP这五个对象. 不过和客户端相比, kmalloc-64明显要消耗得少一些. 这是因为服务端不需要tcp_bind_bucket记录端口占用.
  5. 在client 按ctrl+c, 会发出FIN, 然后让tcp连接进入FIN_WAIT2, 查看`cat /proc/meminfo |grep Slab`记录输出c3, 可算的是0.396 KB. 可见在FIN_WAT2状态下,TCP连接的开销要比ESTABLISH状态下小得多.

    可见dentry、tilp、sock_inode_cache、TCP这四个对象都被回收了,只剩下kmalloc-64, 另外多了一个只有0.25KB的tw_sock_TCP. 总之,FIN_WAIT2状态下的TCP连接占用的内存很小. 内核在不需要的时候会尽量回收不再使用的内核对象,以节约内存.
  6. 再在server 按ctrl+c, 会发出FIN, 让client tcp连接进入TIME_WAIT, 查看client `cat /proc/meminfo |grep Slab`记录输出c4, 可算的是0.41 KB, 和client FIN_WAIT2时差不多.

  注意: 实验后slab内存管理还是会适度存在一些浪费, 外加其他slab缓存对象的使用, 所以实际内存占用会比`cat /proc/meminfo |grep Slab`大一些

数据收发对内存的消耗相当复杂,涉及tcp_rmem、tcp_wmem等内核参数限制, 也涉及滑动窗口、流量控制等协议层面的影响, 不做测试.

## 解析 socket 的 Write 操作
socket 对于用户来讲，是一个文件一样的存在，拥有一个文件描述符. 因而对于网络包的发送，可以使用对于 socket 文件的写入系统调用，也就是 write 系统调用. write 系统调用对于一个文件描述符的操作，大致过程都是类似的. 对于每一个打开的文件都有一个 struct file 结构，write 系统调用会最终调用 stuct file 结构指向的 file_operations 操作. 对于 socket 来讲，它的 file_operations 定义如下：
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L149
/*
 *	Socket files have a set of 'special' operations as well as the generic file ones. These don't appear
 *	in the operation structures but are done directly via the socketcall() multiplexor.
 */

static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.sendpage =	sock_sendpage,
	.splice_write = generic_splice_sendpage,
	.splice_read =	sock_splice_read,
	.show_fdinfo =	sock_show_fdinfo,
};
```

按照文件系统的写入流程，调用的是 [sock_write_iter](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L982).

在 sock_write_iter 中，通过 VFS 中的 struct file，将创建好的 socket 结构拿出来，然后调用 sock_sendmsg. 而 sock_sendmsg 会调用 [sock_sendmsg_nosec](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L650).

sock_sendmsg_nosec里调用了 socket 的 ops 的 sendmsg，据 inet_stream_ops 的定义，这里调用的是 [inet_sendmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L807).

这里面，从 socket 结构中，可以得到更底层的 sock 结构，然后调用 sk_prot 的 sendmsg 方法即tcp_sendmsg.

## 解析 tcp_sendmsg 函数
根据 tcp_prot 的定义，实际调用的是 [tcp_sendmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L1436).

tcp_sendmsg->[tcp_sendmsg_locked](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L1185), tcp_sendmsg_locked 的实现还是很复杂的，这里面做了这样几件事情.

msg 是用户要写入的数据，这个数据要拷贝到内核协议栈里面去发送；在内核协议栈里面，入站或出站的网络数据包都是用 struct sk_buff 表示的, 它也被称为skb(套接字缓冲区)，因而第一件事情就是找到一个空闲的内存空间，将用户要写入的数据，拷贝到 struct sk_buff 的管辖范围内. 而第二件事情就是发送 [struct sk_buff](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/skbuff.h#L711).

> 使用skb必须遵循skb api.

在 tcp_sendmsg_locked 中，首先通过强制类型转换，将 sock 结构转换为 struct tcp_sock，这个是维护 TCP 连接状态的重要数据结构.

接下来是 tcp_sendmsg_locked 的第一件事情，把数据拷贝到 struct sk_buff. 先声明一个变量 copied，初始化为 0，这表示拷贝了多少数据. 紧接着是一个循环，[while (msg_data_left(msg))](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L1271)，也即如果用户的数据没有发送完毕，就一直循环. 循环里声明了一个 copy 变量，表示这次拷贝的数值，在循环的最后有 copied += copy，将每次拷贝的数量都加起来. 这里只需要看一次循环做了哪些事情:
1. 第一步，tcp_write_queue_tail 从 TCP 写入队列 sk_write_queue 中拿出最后一个 struct sk_buff，在这个写入队列中排满了要发送的 struct sk_buff，为什么要拿最后一个呢？这里面只有最后一个，可能会因为上次用户给的数据太少，而没有填满.
1. 第二步，tcp_send_mss 会计算 MSS，也即 Max Segment Size. 这是什么呢？这个意思是说，在网络上传输的网络包的大小是有限制的，而这个限制在最底层开始就有. MTU（Maximum Transmission Unit，最大传输单元）是二层的一个定义. 以以太网为例，MTU 为 1500 个 Byte，前面有 6 个 Byte 的目标 MAC 地址，6 个 Byte 的源 MAC 地址，2 个 Byte 的类型，后面有 4 个 Byte 的 CRC 校验，共 1518 个 Byte. 在 IP 层，一个 IP 数据报在以太网中传输，如果它的长度大于该 MTU 值，就要进行分片传输, 即它决定了数据包是否需要分段.

在 TCP 层有个 MSS（Maximum Segment Size，最大分段大小），等于 MTU 减去 IP 头，再减去 TCP 头. 也就是，在不分片的情况下，TCP 里面放的最大内容. 在这里，max 是 struct sk_buff 的最大数据长度，skb->len 是当前已经占用的 skb 的数据长度，相减得到当前 skb 的剩余数据空间.

1. 第三步，如果 copy 小于 0，说明最后一个 struct sk_buff 已经没地方存放了，需要调用 sk_stream_alloc_skb，重新分配 struct sk_buff，然后调用 skb_entail，将新分配的 sk_buff 放到队列尾部.

struct sk_buff 是存储网络包的重要的数据结构，在应用层数据包叫 data，在 TCP 层我们称为 segment，在 IP 层叫 packet，在数据链路层称为 frame. 在 struct sk_buff，首先是一个链表，将 struct sk_buff 结构串起来.

接下来，struct sk_buff 从中的 headers_start 开始，到 headers_end 结束，里面都是各层次的头的位置. 这里面有二层的 mac_header、三层的 network_header 和四层的 transport_header.

最后几项， head 指向分配的内存块起始地址. data 这个指针指向的位置是可变的. 它有可能随着报文所处的层次而变动. 当接收报文时，从网卡驱动开始，通过协议栈层层往上传送数据报，通过增加 skb->data 的值，来逐步剥离协议首部. 而要发送报文时，各协议会创建 sk_buff{}，在经过各下层协议时，通过减少 skb->data 的值来增加协议首部. tail 指向数据的结尾，end 指向分配的内存块的结束地址.

要分配这样一个结构，sk_stream_alloc_skb 会最终调用到 __alloc_skb. 在这个函数里面，除了分配一个 sk_buff 结构之外，还要分配 sk_buff 指向的数据区域. 这段数据区域分为下面这几个部分:
  1. 第一部分是连续的数据区域
  1. 紧接着是第二部分，一个 struct skb_shared_info 结构. 这个结构是对于网络包发送过程的一个优化，因为传输层之上就是应用层了. 按照 TCP 的定义，应用层感受不到下面的网络层的 IP 包是一个个独立的包的存在的. 反正就是一个流，往里写就是了，可能一下子写多了，超过了一个 IP 包的承载能力，就会出现上面 MSS 的定义，拆分成一个个的 Segment 放在一个个的 IP 包里面，也可能一次写一点，一次写一点，这样数据是分散的，在 IP 层还要通过内存拷贝合成一个 IP 包. 为了减少内存拷贝的代价，有的网络设备支持分散聚合（Scatter/Gather）I/O，顾名思义，就是 IP 层没必要通过内存拷贝进行聚合，让散的数据零散的放在原处，在设备层进行聚合. 如果使用这种模式，网络包的数据就不会放在连续的数据区域，而是放在 struct skb_shared_info 结构里面指向的离散数据，skb_shared_info 的成员变量 skb_frag_t frags[MAX_SKB_FRAGS]，会指向一个数组的页面，就不能保证连续了.

![](/misc/img/net/9ad34c3c748978f915027d5085a858b8.png)

1. 于是就有了第四步. 在注释 [/* Where to copy to? */](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L1314) 后面有个 if-else 分支. if 分支就是 skb_add_data_nocache 将数据拷贝到连续的数据区域. else 分支就是 skb_copy_to_page_nocache 将数据拷贝到 struct skb_shared_info 结构指向的不需要连续的页面区域.
1. 第五步，就是要发送网络包了. 第一种情况是积累的数据报数目太多了，因而需要通过调用 __tcp_push_pending_frames 发送网络包. 第二种情况是，这是第一个网络包，需要马上发送，调用 tcp_push_one. 无论 __tcp_push_pending_frames 还是 tcp_push_one，都会调用 [tcp_write_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L2426) 发送网络包.

至此，tcp_sendmsg 解析完了.

## 解析 tcp_write_xmit 函数
接下来看，[tcp_write_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L2426) 是如何发送网络包的.

这里面主要的逻辑是一个循环，用来处理发送队列，只要队列不空，就会发送.

在一个循环中，涉及 TCP 层的很多传输算法，需要一一解析.

第一个概念是 TSO（TCP Segmentation Offload）. 如果发送的网络包非常大，就像上面说的一样，要进行分段. 分段这个事情可以由协议栈代码在内核做，但是缺点是比较费 CPU，另一种方式是延迟到硬件网卡去做，需要网卡支持对大数据包进行自动分段，可以降低 CPU 负载.

在代码中，tcp_init_tso_segs 会调用 tcp_set_skb_tso_segs. 这里面有这样的语句：[DIV_ROUND_UP(skb->len, mss_now)](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L1285). 也就是 sk_buff 的长度除以 mss_now，应该分成几个段. 如果算出来要分成多个段，接下来就是要看，是在这里（协议栈的代码里面）分好，还是等待到了底层网卡再分.

于是，调用函数 tcp_mss_split_point，开始计算切分的 limit。这里面会计算 max_len = mss_now * max_segs，根据现在不切分来计算 limit，所以下一步的判断中，大部分情况下 tso_fragment 不会被调用，等待到了底层网卡来切分.

第二个概念是拥塞窗口的概念（cwnd，congestion window），也就是说为了避免拼命发包，把网络塞满了，定义一个窗口的概念，在这个窗口之内的才能发送，超过这个窗口的就不能发送，来控制发送的频率. 那窗口大小是多少呢？就是遵循下面这个著名的拥塞窗口变化图.

![](/misc/img/net/404a6c5041452c0641ae3cba5319dc1f.png)

一开始的窗口只有一个 mss 大小叫作 slow start（慢启动）. 一开始的增长速度的很快的，翻倍增长, 一旦到达一个临界值 ssthresh，就变成线性增长，就称为拥塞避免. 什么时候算真正拥塞呢？就是出现了丢包. 一旦丢包，一种方法是马上降回到一个 mss，然后重复先翻倍再线性对的过程. 如果觉得太过激进，也可以有第二种方法，就是降到当前 cwnd 的一半，然后进行线性增长.

在代码中，tcp_cwnd_test 会将当前的 snd_cwnd，减去已经在窗口里面尚未发送完毕的网络包，那就是剩下的窗口大小 cwnd_quota，也即就能发送这么多了.

第三个概念就是接收窗口rwnd 的概念（receive window），也叫滑动窗口. 如果说拥塞窗口是为了怕把网络塞满，在出现丢包的时候减少发送速度，那么滑动窗口就是为了怕把接收方塞满，而控制发送速度.

![](/misc/img/net/9791e2f9ff63a9d8f849df7cd55fe965.png)

滑动窗口，其实就是接收方告诉发送方自己的网络包的接收能力，超过这个能力，就收不了了. 因为滑动窗口的存在，将发送方的缓存分成了四个部分:
1. 第一部分：发送了并且已经确认的. 这部分是已经发送完毕的网络包，这部分没有用了，可以回收.
1. 第二部分：发送了但尚未确认的. 这部分，发送方要等待，万一发送不成功，还要重新发送，所以不能删除.
1. 第三部分：没有发送，但是已经等待发送的. 这部分是接收方空闲的能力，可以马上发送，接收方收得了.
1. 第四部分：没有发送，并且暂时还不会发送的. 这部分已经超过了接收方的接收能力，再发送接收方就收不了了.

![](/misc/img/net/b62eea403e665bb196dceba571392531.png)

因为滑动窗口的存在，接收方的缓存也要分成了三个部分:
1. 第一部分：接受并且确认过的任务. 这部分完全接收成功了，可以交给应用层了
1. 第二部分：还没接收，但是马上就能接收的任务. 这部分有的网络包到达了，但是还没确认，不算完全完毕，有的还没有到达，那就是接收方能够接受的最大的网络包数量
1. 第三部分：还没接收，也没法接收的任务. 这部分已经超出接收方能力

在网络包的交互过程中，接收方会将第二部分的大小，作为 AdvertisedWindow 发送给发送方，发送方就可以根据它来调整发送速度了. 在 tcp_snd_wnd_test 函数中，会判断 sk_buff 中的 end_seq 和 tcp_wnd_end(tp) 之间的关系，也即这个 sk_buff 是否在滑动窗口的允许范围之内. 如果不在范围内，说明发送要受限制了，就要把 is_rwnd_limited 设置为 true.

接下来，[tcp_mss_split_point](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L1826) 函数要被调用了.

tcp_mss_split_point里面除了会判断上面讲的，是否会因为超出 mss 而分段，还会判断另一个条件，就是是否在滑动窗口的运行范围之内，如果小于窗口的大小，也需要分段，也即需要调用 tso_fragment.

tcp_write_xmit在一个循环的最后，是调用 [tcp_transmit_skb](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L1251)，真的去发送一个网络包.

tcp_transmit_skb -> [__tcp_transmit_skb](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_output.c#L1078), __tcp_transmit_skb 这个函数比较长，主要做了两件事情，第一件事情就是填充 TCP 头.

tcp header里面有源端口，设置为 inet_sport，有目标端口，设置为 inet_dport；有序列号，设置为 tcb->seq；有确认序列号，设置为 tp->rcv_nxt. 把所有的 flags 设置为 tcb->tcp_flags. 设置选项为 opts. 设置窗口大小为 tp->rcv_wnd.

全部设置完毕之后，就会调用 icsk_af_ops 的 queue_xmit 方法，icsk_af_ops 指向 [ipv4_specific](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L2113)，也即调用的是 ip_queue_xmit 函数.

## 解析 ip_queue_xmit 函数
从 [ip_queue_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/ip.h#L234) 函数开始，就进入 IP 层的发送逻辑了.

ip_queue_xmit -> [__ip_queue_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L451).

在 __ip_queue_xmit 中，也即 IP 层的发送函数里面，有三部分逻辑.

第一部分，选取路由，也即要发送这个包应该从哪个网卡出去. 这件事情主要由 ip_route_output_ports 函数完成.

接下来的调用链为：ip_route_output_ports->ip_route_output_flow->__ip_route_output_key->ip_route_output_key_hash->[ip_route_output_key_hash_rcu](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/route.c#L2502).

ip_route_output_key_hash_rcu 先会调用 fib_lookup. FIB 全称是 Forwarding Information Base，转发信息表, 其实就是咱们常说的路由表.

路由表可以有多个，一般会有一个主表，RT_TABLE_MAIN. 然后 fib_table_lookup 函数在这个表里面进行查找.

路由表是一个什么样的结构呢？路由就是在 Linux 服务器上的路由表里面配置的一条一条规则. 这些规则大概是这样的：想访问某个网段，从某个网卡出去，下一跳是某个 IP. 可以通过 ip route 命令查看路由表.

fib_table_lookup 的代码逻辑比较复杂，好在注释比较清楚. 因为路由表要按照前缀进行查询，希望找到最长匹配的那一个，例如 192.168.2.0/24 和 192.168.0.0/16 都能匹配 192.168.2.100/24, 但是，应该使用 192.168.2.0/24 的这一条.

为了更方面的做这个事情，使用了 Trie 树这种结构. 比如有一系列的字符串：{bcs#, badge#, baby#, back#, badger#, badness#}. 之所以每个字符串都加上 #，是希望不要一个字符串成为另外一个字符串的前缀. 然后我们把它们放在 Trie 树中，如下图所示：
![](/misc/img/net/3f0a99cf1c47afcd0bd740c4b7802511.png)

对于将 IP 地址转成二进制放入 trie 树，也是同样的道理，可以很快进行路由的查询. 找到了路由，就知道了应该从哪个网卡发出去. 然后，ip_route_output_key_hash_rcu 会调用 __mkroute_output，创建一个 struct rtable，表示找到的路由表项. 这个结构是由 [rt_dst_alloc](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/route.c#L1620) 函数分配的.

最终返回 struct rtable 实例，第一部分也就完成了.

第二部分，就是准备 IP 层的头，往里面填充内容.

在ip header里面，服务类型设置为 tos，标识位里面设置是否允许分片 frag_off. 如果不允许，而遇到 MTU 太小过不去的情况，就发送 ICMP 报错. TTL 是这个包的存活时间，为了防止一个 IP 包迷路以后一直存活下去，每经过一个路由器 TTL 都减一，减为零则“死去”. 设置 protocol，指的是更上层的协议，这里是 TCP. 源地址和目标地址由 ip_copy_addrs 设置. 最后，设置 options.

第三部分，就是调用 [ip_local_out](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L119) 发送 IP 包.

ip_local_out -> [__ip_local_out](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L98), 然后里面调用了 nf_hook. 这是什么呢？nf 的意思是 Netfilter，这是 Linux 内核的一个机制，用于在网络发送和转发的关键节点上加上 hook 函数，这些函数可以截获数据包，对数据包进行干预. 一个著名的实现，就是内核模块 ip_tables. 在用户态，还有一个客户端程序 iptables，用命令行来干预内核的规则.

![](/misc/img/net/75c8257049eed99499e802fcc2eacf4d.png)

iptables 有表和链的概念，最终要的是两个表.

filter 表处理过滤功能，主要包含以下三个链:
- INPUT 链：过滤所有目标地址是本机的数据包
- FORWARD 链：过滤所有路过本机的数据包
- OUTPUT 链：过滤所有由本机产生的数据包

nat 表主要处理网络地址转换，可以进行 SNAT（改变源地址）、DNAT（改变目标地址），包含以下三个链:
- PREROUTING 链：可以在数据包到达时改变目标地址
- OUTPUT 链：可以改变本地产生的数据包的目标地址
- POSTROUTING 链：在数据包离开时改变数据包的源地址

![](/misc/img/net/765e5431fe4b17f62b1b5712cc82abda.png)

在这里，网络包马上就要发出去了，因而是 NF_INET_LOCAL_OUT，也即 ouput 链，如果用户曾经在 iptables 里面写过某些规则，就会在 nf_hook 这个函数里面起作用.

ip_local_out 再调用 [dst_output](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/dst.h#L433)，就是真正的发送数据.

这里调用的就是 struct rtable 成员 dst 的 ouput 函数. 在 rt_dst_alloc 中，可以看到，output 函数指向的是 [ip_output](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L421).

在 ip_output 里面，又看到了熟悉的 NF_HOOK. 这一次是 NF_INET_POST_ROUTING，也即 POSTROUTING 链，处理完之后，调用 [ip_finish_output](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L309).

## 解析 ip_finish_output 函数
从 ip_finish_output 函数开始，发送网络包的逻辑由第三层到达第二层. ip_finish_output 最终调用 [ip_finish_output2](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_output.c#L185).

在 ip_finish_output2 中，先找到 struct rtable 路由表里面的下一跳，下一跳一定和本机在同一个局域网中，可以通过二层进行通信，因而通过 ip_neigh_for_gw -> __ipv4_neigh_lookup_noref，查找如何通过二层访问下一跳.

__ipv4_neigh_lookup_noref 是从本地的 [ARP 表](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/arp.c#L151)中查找下一跳的 MAC 地址.

如果在 ARP 表中没有找到相应的项，则调用 [__neigh_create](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/neighbour.c#L668) 进行创建.

__neigh_create 先调用 neigh_alloc，创建一个 struct neighbour 结构，用于维护 MAC 地址和 ARP 相关的信息. 这个名字也很好理解，大家都是在一个局域网里面，可以通过 MAC 地址访问到，当然是邻居了.

在 neigh_alloc 中，先分配一个 struct neighbour 结构并且初始化. 这里面比较重要的有两个成员，一个是 arp_queue，所以上层想通过 ARP 获取 MAC 地址的任务，都放在这个队列里面. 另一个是 timer 定时器，设置成过一段时间就调用 neigh_timer_handler，来处理这些 ARP 任务.

__neigh_create 然后调用了 arp_tbl 的 constructor 函数，也即调用了 [arp_constructor](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/arp.c#L220)，在这里面定义了 ARP 的操作 [arp_hh_ops](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/arp.c#L137).

__neigh_create 最后是将创建的 struct neighbour 结构放入一个哈希表，从里面的代码逻辑比较容易看出，这是一个数组加链表的链式哈希表，先计算出哈希值 hash_val，得到相应的链表，然后循环这个链表找到对应的项，如果找不到就在最后插入一项.

回到 ip_finish_output2，在 __neigh_create 之后，会调用 [neigh_output](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/neighbour.h#L501) 发送网络包.

按照上面对于 struct neighbour 的操作函数 arp_hh_ops 的定义，output 调用的是 [neigh_resolve_output](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/neighbour.c#L1469).

在 neigh_resolve_output 里面，首先 neigh_event_send 触发一个事件，看能否激活 ARP.

在 __neigh_event_send 中，激活 ARP 分两种情况.

第一种情况是马上激活，也即 immediate_probe. 另一种情况是延迟激活则仅仅设置一个 timer, 然后将 ARP 包放在 arp_queue 上. 如果马上激活，就直接调用 neigh_probe；如果延迟激活，则定时器到了就会触发 neigh_timer_handler，在这里面还是会调用 [neigh_probe](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/neighbour.c#L1000). 就来看 neigh_probe 的实现，在这里面会从 arp_queue 中拿出 ARP 包来，然后调用 struct neighbour 的 solicit 操作.

按照上面对于 struct neighbour 的操作函数 arp_hh_ops 的定义，solicit 调用的是 arp_solicit，在这里可以找到对应于 [arp_send_dst](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/arp.c#L298) 的调用，创建并发送一个 arp 包，得到结果放在 struct dst_entry 里面.

回到 neigh_resolve_output 中，当 ARP 发送完毕，就可以调用 [dev_queue_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L4162) 发送二层网络包了.

就像在学硬盘块设备的时候学过，每个块设备都有队列，用于将内核的数据放到队列里面，然后设备驱动从队列里面取出后，将数据根据具体设备的特性发送给设备。网络设备也是类似的，对于发送来说，有一个发送队列 struct netdev_queue *txq. 这里还有另一个变量叫做 struct Qdisc，这个是什么呢？如果在一台 Linux 机器上运行 ip addr，我们能看到对于一个网卡，都有下面的输出.

这里面有个关键字 qdisc pfifo_fast 是什么意思呢？qdisc 全称是 queueing discipline，中文叫排队规则. 内核如果需要通过某个网络接口发送数据包，都需要按照为这个接口配置的 qdisc（排队规则）把数据包加入队列.

最简单的 qdisc 是 pfifo，它不对进入的数据包做任何的处理，数据包采用先入先出的方式通过队列. pfifo_fast 稍微复杂一些，它的队列包括三个波段（band）. 在每个波段里面，使用先进先出规则.

三个波段的优先级也不相同. band 0 的优先级最高，band 2 的最低. 如果 band 0 里面有数据包，系统就不会处理 band 1 里面的数据包，band 1 和 band 2 之间也是一样.

数据包是按照服务类型（Type of Service，TOS）被分配到三个波段里面的. TOS 是 IP 头里面的一个字段，代表了当前的包是高优先级的，还是低优先级的.

pfifo_fast 分为三个先入先出的队列，称为三个 Band. 根据网络包里面的 TOS，看这个包到底应该进入哪个队列. TOS 总共四位，每一位表示的意思不同，总共十六种类型.

![](/misc/img/net/ab6af2f9e1a64868636080a05cfde0d9.png)

通过命令行 tc qdisc show dev eth0，可以输出结果 priomap，也是十六个数字. 在 0 到 2 之间，和 TOS 的十六种类型对应起来. 不同的 TOS 对应不同的队列。其中 Band 0 优先级最高，发送完毕后才轮到 Band 1 发送，最后才是 Band 2.

接下来，[__dev_xmit_skb](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L3734) 开始进行网络包发送.

__dev_xmit_skb 会将请求放入队列，然后调用 __qdisc_run 处理队列中的数据. qdisc_restart 用于数据的发送. 根据注释中的说法，qdisc 的另一个功能是用于控制网络包的发送速度，因而如果超过速度，就需要重新调度，则会调用 __netif_schedule.

__netif_schedule 会调用 __netif_reschedule，发起一个软中断 NET_TX_SOFTIRQ. 学设备驱动程序的时候学过，设备驱动程序处理中断，分两个过程，一个是屏蔽中断的关键处理逻辑，一个是延迟处理逻辑. 当时说工作队列是延迟处理逻辑的处理方案，软中断也是一种方案.

在系统初始化的时候，会定义软中断的处理函数. 例如，NET_TX_SOFTIRQ 的处理函数是 net_tx_action，用于发送网络包. 还有一个 NET_RX_SOFTIRQ 的处理函数是 net_rx_action，用于接收网络包.
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L10616
/*
 *	Initialize the DEV module. At boot time this walks the device list and
 *	unhooks any devices that fail to initialise (normally hardware not
 *	present) and leaves us with a valid list of present and active devices.
 *
 */

/*
 *       This is called single threaded during boot, so no need
 *       to take the rtnl semaphore.
 */
static int __init net_dev_init(void)
...
	open_softirq(NET_TX_SOFTIRQ, net_tx_action);
	open_softirq(NET_RX_SOFTIRQ, net_rx_action);
...
}
```

这里先来解析一下 [net_tx_action](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L4838).

会发现，net_tx_action 还是调用了 qdisc_run，还是会调用 __qdisc_run，然后调用 [qdisc_restart](https://elixir.bootlin.com/linux/v5.8.1/source/net/sched/sch_generic.c#L357) 发送网络包.

再看qdisc_restart 的实现. qdisc_restart 将网络包从 Qdisc 的队列中拿下来，然后调用 [sch_direct_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/sched/sch_generic.c#L285) 进行发送.

在 sch_direct_xmit 中，调用 [dev_hard_start_xmit](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L3562) 进行发送，如果发送不成功，会返回 NETDEV_TX_BUSY. 这说明网络卡很忙，于是就调用 dev_requeue_skb，重新放入队列.

在 dev_hard_start_xmit 中，是一个 while 循环. 每次在队列中取出一个 sk_buff，调用 xmit_one 发送. 接下来的调用链为：xmit_one->netdev_start_xmit->__netdev_start_xmit.

这个时候，已经到了设备驱动层了. kernel里能看到，[drivers/net/ethernet/intel/ixgb/ixgb_main.c 里面有对于这个网卡的操作的定义](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L335).

在ixgb_main.c里面，可以找到对于 ndo_start_xmit 的定义，调用 [ixgb_xmit_frame](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L1478).

在 ixgb_xmit_frame 中，会得到这个网卡对应的适配器，然后将其放入硬件网卡的队列中.

至此，整个发送才算结束.

## 总结时刻
![](/misc/img/net/79cc42f3163d159a66e163c006d9f36f.png)

这个过程分成几个层次:
1. VFS 层：write 系统调用找到 struct file，根据里面的 file_operations 的定义，调用 sock_write_iter 函数. sock_write_iter 函数调用 sock_sendmsg 函数.
1. Socket 层：从 struct file 里面的 private_data 得到 struct socket，根据里面 ops 的定义，调用 inet_sendmsg 函数.
1. Sock 层：从 struct socket 里面的 sk 得到 struct sock，根据里面 sk_prot 的定义，调用 tcp_sendmsg 函数.
1. TCP 层：tcp_sendmsg 函数会调用 tcp_write_xmit 函数，tcp_write_xmit 函数会调用 tcp_transmit_skb，在这里实现了 TCP 层面向连接的逻辑
1. IP 层：扩展 struct sock，得到 struct inet_connection_sock，根据里面 icsk_af_ops 的定义，调用 ip_queue_xmit 函数.
1. IP 层：ip_route_output_ports 函数里面会调用 fib_lookup 查找路由表. FIB 全称是 Forwarding Information Base，转发信息表，也就是路由表
1. 在 IP 层里面要做的另一个事情是填写 IP 层的头
1. 在 IP 层还要做的一件事情就是通过 iptables 规则
1. MAC 层：IP 层调用 ip_finish_output 进行 MAC 层
1. MAC 层需要 ARP 获得 MAC 地址，因而要调用 ___neigh_lookup_noref 查找属于同一个网段的邻居，它会调用 neigh_probe 发送 ARP
1. 有了 MAC 地址，就可以调用 dev_queue_xmit 发送二层网络包了，它会调用 __dev_xmit_skb 会将请求放入队列
1. 设备层：网络包的发送会触发一个软中断 NET_TX_SOFTIRQ 来处理队列中的数据. 这个软中断的处理函数是 net_tx_action
1. 在软中断处理函数中，会将网络包从队列上拿下来，调用网络设备的传输函数 ixgb_xmit_frame，将网络包发到设备的队列上去

## 解析接收网络包的过程
ref:
- [DM9000驱动分析](<<Linux设备驱动开发详解#14.9>>)

  [DM9000](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/net/ethernet/davicom/dm9000.c)是开发板采用的网络芯片, 最新是DM9051

register_netdev()/unregister_netdev(): 注册/删除网络设备. alloc_netdev/alloc_netdev_mqs/alloc_etherdev/alloc_etherdev_mq/alloc_etherdev_mqs可辅助net_device的创建和成员的赋值; free_netdev是执行相反的过程.

网络设备
- 打开:
  1. 使能设备使用的硬件资源, 申请I/O区域, 中断和DMA通道等
  1. 调用netif_start_queue(), 激活设备发送队列
- 关闭
  1. 调用netif_stop_queue(), 停止设备传输包
  1. 释放设备所使用的I/O区域, 中断和DMA通道等

发送数据包时, 会调用驱动提供的hard_start_transmit(), 在设备初始化时, 它是指向设备的xxx_tx()

网络适配器硬件电路可检测链路上是否有载波(通常by中断), 从而判断网络的连接是否正常. netif_carrier_on/netif_carrier_off可改变设备的连接状态. netif_carrier_ok()可判断链路上是否存在载波

netif_running: 判断设备是否正在运行.

### 设备驱动层
网卡作为一个硬件，接收到网络包，应该怎么通知操作系统，这个网络包到达了呢？方法是触发一个中断. 老式网络设备驱动在中断模式下工作, 每接收一个数据包, 就中断一次. 但是这里有个问题，就是网络包的到来，往往是很难预期的. 网络吞吐量比较大的时候(高负载)，网络包的到达会十分频繁. 如果此时非常频繁地去触发中断，效率可想而知.

网络包能不能一起来，这个我们没法儿控制，但是可以有一种机制，就是当一些网络包到来触发了中断，内核处理完这些网络包之后，可以先进入主动轮询 poll 网卡的方式，主动去接收到来的网络包. 如果一直有，就一直处理，等处理告一段落，就返回干其他的事情. 当再有下一批网络包到来的时候，再中断，再轮询 poll. 这样就会大大减少中断的数量，提升网络处理的效率，这种处理方式称为 NAPI. 采用NAPI可提高设备在高负载下的性能. 当前, 几乎所有linux网络设备驱动都支持该技术了. 从kernel 3.11起, 新增了频繁轮询套接字(busy polling on sockets)的功能, 用于那些不惜以提供cpu使用率为代价而尽可能降低延迟的套接字应用程序.

在linux系统中，其网卡驱动大多通过PCI总线与系统相连，同时，内核对于所有PCI总线上的设备是通过PCI子系统来进行管理，通过PCI子系统提供各种PCI设备驱动程序共同的所有通用功能. 因此，作为linux系统中的PCI网卡设备，其加载的流程必然分为两个部分：作为PCI设备向PCI子系统注册；作为网卡设备向网络子系统注册.

以之前发送网络包时的网卡 drivers/net/ethernet/intel/ixgb/ixgb_main.c 为例子，来进行解析.

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L95
static struct pci_driver ixgb_driver = {
	.name     = ixgb_driver_name,
	.id_table = ixgb_pci_tbl,
	.probe    = ixgb_probe,
	.remove   = ixgb_remove,
	.err_handler = &ixgb_err_handler
};


MODULE_AUTHOR("Intel Corporation, <linux.nics@intel.com>");
MODULE_DESCRIPTION("Intel(R) PRO/10GbE Network Driver");
MODULE_LICENSE("GPL v2");
MODULE_VERSION(DRV_VERSION);

#define DEFAULT_MSG_ENABLE (NETIF_MSG_DRV|NETIF_MSG_PROBE|NETIF_MSG_LINK)
static int debug = -1;
module_param(debug, int, 0);
MODULE_PARM_DESC(debug, "Debug level (0=none,...,16=all)");

/**
 * ixgb_init_module - Driver Registration Routine
 *
 * ixgb_init_module is the first routine called when the driver is
 * loaded. All it does is register with the PCI subsystem.
 **/

static int __init
ixgb_init_module(void)
{
	pr_info("%s - version %s\n", ixgb_driver_string, ixgb_driver_version);
	pr_info("%s\n", ixgb_copyright);

	return pci_register_driver(&ixgb_driver);
}

module_init(ixgb_init_module);

// https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/pci.h#L1374
/* pci_register_driver() must be a macro so KBUILD_MODNAME can be expanded */
#define pci_register_driver(driver)		\
	__pci_register_driver(driver, THIS_MODULE, KBUILD_MODNAME)

// https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L1400
/**
 * __pci_register_driver - register a new pci driver
 * @drv: the driver structure to register
 * @owner: owner module of drv
 * @mod_name: module name string
 *
 * Adds the driver structure to the list of registered drivers.
 * Returns a negative value on error, otherwise 0.
 * If no error occurred, the driver remains registered even if
 * no device was claimed during registration.
 */
int __pci_register_driver(struct pci_driver *drv, struct module *owner,
			  const char *mod_name)
{
	/* initialize common driver fields */
	drv->driver.name = drv->name;
	drv->driver.bus = &pci_bus_type;
	drv->driver.owner = owner;
	drv->driver.mod_name = mod_name;
	drv->driver.groups = drv->groups;

	spin_lock_init(&drv->dynids.lock);
	INIT_LIST_HEAD(&drv->dynids.list);

	/* register with core */
	return driver_register(&drv->driver);
}
EXPORT_SYMBOL(__pci_register_driver);
```

每个PCI设备驱动都有一个[pci_driver](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/pci.h#L848)变量，它描述了一个PCI驱动的信息. 每个PCI设备都由一组参数唯一地标识，这些参数保存在结构体pci_driver的[pci_device_id](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/mod_devicetable.h#L38)中.

ixgb_init_module ->pci_register_driver->__pci_register_driver->[driver_register](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/driver.c#L147)->[bus_add_driver](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/bus.c#L594) -> [driver_attach](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L1065) -> [bus_for_each_dev#fn(dev, data)](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/bus.c#L292) ->[__driver_attach](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L1005) -> [device_driver_attach](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L963)->[driver_probe_device](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L683)->[really_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L466)->[`dev->bus->probe(dev)`](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/base/dd.c#L466) , 因为`__pci_register_driver()`中设置了[`drv->driver.bus = &pci_bus_type;`](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L1620)->[pci_device_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L413)->[__pci_device_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L376)->[pci_call_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L332)->[ local_pci_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/pci/pci-driver.c#L287)

在网卡驱动程序初始化的时候，会调用 ixgb_init_module，注册一个驱动 ixgb_driver，并且调用它的 probe 函数 [ixgb_probe](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L363).

在 ixgb_probe 中，会创建一个 [struct net_device](https://elixir.bootlin.com/linux/v5.10.50/source/include/linux/netdevice.h#L1865) 表示这个网络设备，并且 [netif_napi_add](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L6597) 函数为这个网络设备注册一个轮询 poll 函数 [ixgb_clean](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L1757)，将来一旦出现网络包的时候，就是要通过它来轮询了. 当一个网卡被激活的时候，会调用函数 [ixgb_open](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L598)->[ixgb_up](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L176)，在这里面注册一个硬件的中断处理函数[ixgb_intr](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L1725).

> napi_enable/napi_disable: 使能或禁止napi调度

> napi_schedule_prep/napi_schedule/napi_complete: 检查NAPI是否可以调度/调度轮询实例的运行/在NAPI处理完成时调用

如果一个网络包到来，触发了硬件中断，就会调用 ixgb_intr，这里面会调用 [__napi_schedule](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L6278).

__napi_schedule 是处于中断处理的关键部分，在它被调用的时候，中断是暂时关闭的，但是处理网络包是个复杂的过程，需要到延迟处理部分，所以 ____napi_schedule 将当前设备放到 struct softnet_data 结构的 poll_list 里面，说明在延迟处理部分可以接着处理这个 poll_list 里面的网络设备. 然后 ____napi_schedule 触发一个软中断 NET_RX_SOFTIRQ，通过软中断触发中断处理的延迟处理部分，也是常用的手段. 软中断 NET_RX_SOFTIRQ 对应的中断处理函数是 [net_rx_action](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L6729).

在 net_rx_action 中，会得到 [struct softnet_data](https://elixir.bootlin.com/linux/v5.8.1/source/include/linux/netdevice.h#L3095) 结构，这个结构在发送的时候也遇到过. 当时它的 output_queue 用于网络包的发送，这里的 poll_list 用于网络包的接收.

在 net_rx_action 中，接下来是一个循环，在 poll_list 里面取出网络包到达的设备，然后调用 napi_poll 来轮询这些设备，napi_poll 会调用最初设备初始化的时候，注册的 poll 函数，对于 ixgb_driver，对应的函数是 ixgb_clean. ixgb_clean 会调用 [ixgb_clean_rx_irq](https://elixir.bootlin.com/linux/v5.8.1/source/drivers/net/ethernet/intel/ixgb/ixgb_main.c#L1933).

在网络设备的驱动层，有一个用于接收网络包的 rx_ring. 它是一个环，从网卡硬件接收的包会放在这个环里面. 这个环里面的 buffer_info[]是一个数组，存放的是网络包的内容. i 和 j 是这个数组的下标，在 ixgb_clean_rx_irq 里面的 while 循环中，依次处理环里面的数据. 在ixgb_clean_rx_irq里面，看到了 i 和 j 加一之后，如果超过了数组的大小，就跳回下标 0，就说明这是一个环. ixgb_check_copybreak 函数将 buffer_info 里面的内容，拷贝到 struct sk_buff *skb，从而可以作为一个网络包进行后续的处理，然后调用 netif_receive_skb.

## 网络协议栈的二层逻辑
从 netif_receive_skb 函数开始，我们就进入了内核的网络协议栈.

接下来的调用链为：netif_receive_skb->netif_receive_skb_internal->__netif_receive_skb->[__netif_receive_skb_core](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/dev.c#L5074). 在 __netif_receive_skb_core 中，先是处理了二层的一些逻辑, 例如，对于 VLAN 的处理，接下来要想办法交给第三层.

在网络包 struct sk_buff 里面，二层的头里面有一个 protocol，表示里面一层，也即三层是什么协议. deliver_ptype_list_skb 在一个协议列表中逐个匹配. 如果能够匹配到，就返回.

这些协议的注册在网络协议栈初始化的时候， [inet_init 函数调用 dev_add_pack(&ip_packet_type)](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L2055)，添加了 IP 协议. 协议被放在一个链表里面.

假设这个时候的网络包是一个 IP 包，则在这个链表里面一定能够找到 ip_packet_type，在 __netif_receive_skb_core 中会调用 [ip_packet_type 的 func 函数](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1940)即[ip_rcv](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_input.c#L530).

## 网络协议栈的 IP 层
从 ip_rcv 函数开始，处理逻辑就从二层到了三层，IP 层.

在 ip_rcv 中，得到 IP 头，然后又遇到了NF_HOOK，这次因为是接收网络包，第一个 hook 点是 NF_INET_PRE_ROUTING，也就是 iptables 的 PREROUTING 链. 如果里面有规则，则执行规则，然后调用 [ip_rcv_finish](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_input.c#L414).

ip_rcv_finish 得到网络包对应的路由表，然后调用 dst_input，在 dst_input 中，调用的是 struct rtable 的成员的 dst 的 input 函数. 在 rt_dst_alloc 中，可以看到，input 函数指向的是 [ip_local_deliver](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_input.c#L240).

在 ip_local_deliver 函数中，如果 IP 层进行了分段，则进行重新的组合. 接下来就是熟悉的 NF_HOOK。hook 点在 NF_INET_LOCAL_IN，对应 iptables 里面的 INPUT 链。在经过 iptables 规则处理完毕后，会调用 [ip_local_deliver_finish](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/ip_input.c#L226).

在 IP 头中，有一个字段 protocol 用于指定里面一层的协议，在这里应该是 TCP 协议. 于是，从 [inet_protos](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/protocol.c#L27) 数组中，找出 TCP 协议对应的处理函数. 这个数组里面的内容是 [struct net_protocol](https://elixir.bootlin.com/linux/v5.8.1/source/include/net/protocol.h#L37).

```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1946
static int __init inet_init(void)
{
	...
	if (inet_add_protocol(&udp_protocol, IPPROTO_UDP) < 0)
		pr_crit("%s: Cannot add UDP protocol\n", __func__);
	if (inet_add_protocol(&tcp_protocol, IPPROTO_TCP) < 0)
		pr_crit("%s: Cannot add TCP protocol\n", __func__);
	...
}

// https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1737
/* thinking of making this const? Don't.
 * early_demux can change based on sysctl.
 */
static struct net_protocol tcp_protocol = {
	.early_demux	=	tcp_v4_early_demux,
	.early_demux_handler =  tcp_v4_early_demux,
	.handler	=	tcp_v4_rcv,
	.err_handler	=	tcp_v4_err,
	.no_policy	=	1,
	.netns_ok	=	1,
	.icmp_strict_tag_validation = 1,
};

/* thinking of making this const? Don't.
 * early_demux can change based on sysctl.
 */
static struct net_protocol udp_protocol = {
	.early_demux =	udp_v4_early_demux,
	.early_demux_handler =	udp_v4_early_demux,
	.handler =	udp_rcv,
	.err_handler =	udp_err,
	.no_policy =	1,
	.netns_ok =	1,
};
```

在系统初始化的时候，网络协议栈的初始化调用的是 inet_init，它会调用 inet_add_protocol，将 TCP 协议对应的处理函数 tcp_protocol、UDP 协议对应的处理函数 udp_protocol，放到 inet_protos 数组中. 在上面的网络包的接收过程中，会取出 TCP 协议对应的处理函数 tcp_protocol，然后调用 handler 函数，也即 [tcp_v4_rcv](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L1873) 函数.

## 网络协议栈的 TCP 层
从 tcp_v4_rcv 函数开始, 处理逻辑就从 IP 层到了 TCP 层.

在 tcp_v4_rcv 中，得到 TCP 的头之后，就可以开始处理 TCP 层的事情. 因为 TCP 层是分状态的，状态被维护在数据结构 struct sock 里面，因而要根据 IP 地址以及 TCP 头里面的内容，在 tcp_hashinfo 中找到这个包对应的 struct sock，从而得到这个包对应的连接的状态.

接下来，就根据不同的状态做不同的处理，例如，tcp_v4_rcv代码中的 TCP_LISTEN、TCP_NEW_SYN_RECV 状态属于连接建立过程中. 这个在分析三次握手的时候学过了. 再如，TCP_TIME_WAIT 状态是连接结束的时候的状态，这个暂时可以不用看.

接下来，来分析最主流的网络包的接收过程，这里面涉及三个队列：
- backlog 队列
- prequeue 队列
- sk_receive_queue 队列

为什么接收网络包的过程，需要在这三个队列里面倒腾过来、倒腾过去呢？这是因为，同样一个网络包要在三个主体之间交接.

第一个主体是软中断的处理过程. 如果没忘记的话，在执行 tcp_v4_rcv 函数的时候，依然处于软中断的处理逻辑里，所以必然会占用这个软中断.

第二个主体就是用户态进程. 如果用户态触发系统调用 read 读取网络包，也要从队列里面找.

第三个主体就是内核协议栈. 哪怕用户进程没有调用 read，读取网络包，当网络包来的时候，也得有一个地方收着呀.

这时候，就能够了解上面代码中 sock_owned_by_user 的意思了，其实就是说，当前这个 sock 是不是正有一个用户态进程等着读数据呢，如果没有，内核协议栈也调用 tcp_add_backlog，暂存在 backlog 队列中，并且抓紧离开软中断的处理过程.

如果有一个用户态进程等待读取数据呢？就先调用 tcp_prequeue，也即赶紧放入 prequeue 队列，并且离开软中断的处理过程. 在这个函数里面，会看到对于 sysctl_tcp_low_latency 的判断，也即是不是要低时延地处理网络包.

如果把 sysctl_tcp_low_latency 设置为 0，那就要放在 prequeue 队列中暂存，这样不用等待网络包处理完毕，就可以离开软中断的处理过程，但是会造成比较长的时延. 如果把 sysctl_tcp_low_latency 设置为 1，还是调用 [tcp_v4_do_rcv](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_ipv4.c#L1613).

在 tcp_v4_do_rcv 中，分两种情况，一种情况是连接已经建立，处于 TCP_ESTABLISHED 状态，调用 [tcp_rcv_established](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L5588). 另一种情况，就是其他的状态，调用 [tcp_rcv_state_process](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L6183).

![](/misc/img/net/385ff4a348dfd2f64feb0d7ba81e2bc6.png)

在 tcp_rcv_state_process 中，如果对着 TCP 的状态图进行比对，能看到，对于 TCP 所有状态的处理，其中和连接建立相关的状态，已经分析过，所以重点关注连接状态下的工作模式.

在连接状态下，会调用 tcp_rcv_established. 在这个函数里面，会调用 [tcp_data_queue](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L4797)，将其放入 sk_receive_queue 队列进行处理.

在 tcp_data_queue 中，对于收到的网络包，要分情况进行处理.

第一种情况，seq == tp->rcv_nxt，说明来的网络包正是我服务端期望的下一个网络包. 这个时候判断 sock_owned_by_user，也即用户进程也是正在等待读取，这种情况下，就直接 skb_copy_datagram_msg，将网络包拷贝给用户进程就可以了.

如果用户进程没有正在等待读取，或者因为内存原因没有能够拷贝成功，tcp_queue_rcv 里面还是将网络包放入 sk_receive_queue 队列.

接下来，tcp_rcv_nxt_update 将 tp->rcv_nxt 设置为 end_seq，也即当前的网络包接收成功后，更新下一个期待的网络包.

这个时候，还会判断一下另一个队列，out_of_order_queue，也看看乱序队列的情况，看看乱序队列里面的包，会不会因为这个新的网络包的到来，也能放入到 sk_receive_queue 队列中.

例如，客户端发送的网络包序号为 5、6、7、8、9. 在 5 还没有到达的时候，服务端的 rcv_nxt 应该是 5，也即期望下一个网络包是 5. 但是由于中间网络通路的问题，5、6 还没到达服务端，7、8 已经到达了服务端了，这就出现了乱序.

乱序的包不能进入 sk_receive_queue 队列. 因为一旦进入到这个队列，意味着可以发送给用户进程. 然而，按照 TCP 的定义，用户进程应该是按顺序收到包的，没有排好序，就不能给用户进程. 所以，7、8 不能进入 sk_receive_queue 队列，只能暂时放在 out_of_order_queue 乱序队列中.

当 5、6 到达的时候，5、6 先进入 sk_receive_queue 队列. 这个时候再来看 out_of_order_queue 乱序队列中的 7、8，发现能够接上. 于是，7、8 也能进入 sk_receive_queue 队列了. [tcp_ofo_queue](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp_input.c#L4506) 函数就是做这个事情的.

至此第一种情况处理完毕.

第二种情况，end_seq 不大于 rcv_nxt，也即服务端期望网络包 5. 但是，来了一个网络包 3，怎样才会出现这种情况呢？肯定是服务端早就收到了网络包 3，但是 ACK 没有到达客户端，中途丢了，那客户端就认为网络包 3 没有发送成功，于是又发送了一遍，这种情况下，要赶紧给客户端再发送一次 ACK，表示早就收到了.

第三种情况，seq 不小于 rcv_nxt + tcp_receive_window. 这说明客户端发送得太猛了. 本来 seq 肯定应该在接收窗口里面的，这样服务端才来得及处理，结果现在超出了接收窗口，说明客户端一下子把服务端给塞满了.

这种情况下，服务端不能再接收数据包了，只能发送 ACK 了，在 ACK 中会将接收窗口为 0 的情况告知客户端，客户端就知道不能再发送了. 这个时候双方只能交互窗口探测数据包，直到服务端因为用户进程把数据读走了，空出接收窗口，才能在 ACK 里面再次告诉客户端，又有窗口了，又能发送数据包了.

第四种情况，seq 小于 rcv_nxt，但是 end_seq 大于 rcv_nxt，这说明从 seq 到 rcv_nxt 这部分网络包原来的 ACK 客户端没有收到，所以重新发送了一次，从 rcv_nxt 到 end_seq 时新发送的，可以放入 sk_receive_queue 队列.

当前四种情况都排除掉了，说明网络包一定是一个乱序包了. 这里有点儿难理解，还是用上面那个乱序的例子仔细分析一下 rcv_nxt=5.

假设 tcp_receive_window 也是 5，也即超过 10 服务端就接收不了了. 当前来的这个网络包既不在 rcv_nxt 之前（不是 3 这种），也不在 rcv_nxt + tcp_receive_window 之后（不是 11 这种），说明这正在我们期望的接收窗口里面，但是又不是 rcv_nxt（不是我们马上期望的网络包 5），这正是上面的例子中网络包 7、8 的情况.

对于网络包 7、8，只好调用 tcp_data_queue_ofo 进入 out_of_order_queue 乱序队列，但是没有关系，当网络包 5、6 到来的时候，会走第一种情况，把 7、8 拿出来放到 sk_receive_queue 队列中.

至此，网络协议栈的处理过程就结束了.

## Socket 层
当接收的网络包进入各种队列之后，接下来就要等待用户进程去读取它们了.

读取一个 socket，就像读取一个文件一样，读取 socket 的文件描述符，通过 read 系统调用.

read 系统调用对于一个文件描述符的操作，大致过程都是类似的，最终它会调用到用来表示一个打开文件的结构 stuct file 指向的 file_operations 操作. 对于 socket 来讲，它的 file_operations 定义如下：
```c
// https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L149
/*
 *	Socket files have a set of 'special' operations as well as the generic file ones. These don't appear
 *	in the operation structures but are done directly via the socketcall() multiplexor.
 */

static const struct file_operations socket_file_ops = {
	.owner =	THIS_MODULE,
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,
	.unlocked_ioctl = sock_ioctl,
#ifdef CONFIG_COMPAT
	.compat_ioctl = compat_sock_ioctl,
#endif
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.sendpage =	sock_sendpage,
	.splice_write = generic_splice_sendpage,
	.splice_read =	sock_splice_read,
	.show_fdinfo =	sock_show_fdinfo,
};
```

按照文件系统的读取流程，调用的是 [sock_read_iter](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L960).

在 sock_read_iter 中，通过 VFS 中的 struct file，将创建好的 socket 结构拿出来，然后调用 [sock_recvmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L900)，sock_recvmsg 会调用 [sock_recvmsg_nosec](https://elixir.bootlin.com/linux/v5.8.1/source/net/socket.c#L883).

sock_recvmsg_nosec里调用了 socket 的 ops 的 recvmsg，根据 [inet_stream_ops](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L1015) 的定义，这里调用的是 [inet_recvmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/af_inet.c#L835).

在inet_recvmsg里面，从 socket 结构，可以得到更底层的 sock 结构，然后调用 sk_prot 的 recvmsg 方法, 根据 tcp_prot 的定义，调用的是 [tcp_recvmsg](https://elixir.bootlin.com/linux/v5.8.1/source/net/ipv4/tcp.c#L2015).

tcp_recvmsg 这个函数比较长，里面逻辑也很复杂，好在里面有一段注释概括了这里面的逻辑. 注释里面提到了三个队列，receive_queue 队列、prequeue 队列和 backlog 队列. 这里面，需要把前一个队列处理完毕，才处理后一个队列.

tcp_recvmsg 的整个逻辑也是这样执行的：这里面有一个 while 循环，不断地读取网络包.

这里，会先处理 sk_receive_queue 队列. 如果找到了网络包，就跳到 found_ok_skb 这里. 这里会调用 skb_copy_datagram_msg，将网络包拷贝到用户进程中，然后直接进入下一层循环.

直到 sk_receive_queue 队列处理完毕，才到了 sysctl_tcp_low_latency 判断. 如果不需要低时延，则会有 prequeue 队列. 于是，就跳到 do_prequeue 这里，调用 tcp_prequeue_process 进行处理.

如果 sysctl_tcp_low_latency 设置为 1，也即没有 prequeue 队列，或者 prequeue 队列为空，则需要处理 backlog 队列，在 release_sock 函数中处理.

[release_sock](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/sock.c#L3062) 会调用 [__release_sock](https://elixir.bootlin.com/linux/v5.8.1/source/net/core/sock.c#L2534)，这里面会依次处理队列中的网络包.

最后，哪里都没有网络包，只好调用 sk_wait_data，继续等待在哪里，等待网络包的到来.

至此，网络包的接收过程到此结束.

## 总结
![](/misc/img/net/20df32a842495d0f629ca5da53e47152.png)

接收网络包的过程:
1. 硬件网卡接收到网络包之后，通过 DMA 技术，将网络包放入 Ring Buffer
1. 硬件网卡通过中断通知 CPU 新的网络包的到来
1. 网卡驱动程序会注册中断处理函数 ixgb_intr
1. 中断处理函数处理完需要暂时屏蔽中断的核心流程之后，通过软中断 NET_RX_SOFTIRQ 触发接下来的处理过程
1. NET_RX_SOFTIRQ 软中断处理函数 net_rx_action，net_rx_action 会调用 napi_poll，进而调用 ixgb_clean_rx_irq，从 Ring Buffer 中读取数据到内核 struct sk_buff
1. 调用 netif_receive_skb 进入内核网络协议栈，进行一些关于 VLAN 的二层逻辑处理后，调用 ip_rcv 进入三层 IP 层
1. 在 IP 层，会处理 iptables 规则，然后调用 ip_local_deliver，交给更上层 TCP 层
1. 在 TCP 层调用 tcp_v4_rcv，这里面有三个队列需要处理，如果当前的 Socket 不是正在被读；取，则放入 backlog 队列，如果正在被读取，不需要很实时的话，则放入 prequeue 队列，其他情况调用 tcp_v4_do_rcv
1. 在 tcp_v4_do_rcv 中，如果是处于 TCP_ESTABLISHED 状态，调用 tcp_rcv_established，其他的状态，调用 tcp_rcv_state_process
1. 在 tcp_rcv_established 中，调用 tcp_data_queue，如果序列号能够接的上，则放入 sk_receive_queue 队列；如果序列号接不上，则暂时放入 out_of_order_queue 队列，等序列号能够接上的时候，再放入 sk_receive_queue 队列.

至此内核接收网络包的过程到此结束，接下来就是用户态读取网络包的过程，这个过程分成几个层次:
- VFS 层：read 系统调用找到 struct file，根据里面的 file_operations 的定义，调用 sock_read_iter 函数
- sock_read_iter 函数调用 sock_recvmsg 函数
- Socket 层：从 struct file 里面的 private_data 得到 struct socket，根据里面 ops 的定义，调用 inet_recvmsg 函数
- Sock 层：从 struct socket 里面的 sk 得到 struct sock，根据里面 sk_prot 的定义，调用 tcp_recvmsg 函数
- TCP 层：tcp_recvmsg 函数会依次读取 receive_queue 队列、prequeue 队列和 backlog 队列

## core
### [struct net_device](https://elixir.bootlin.com/linux/v5.10.50/source/include/linux/netdevice.h#L1865) 
net_device用与表示一个网络设备.

struct:
- irq : 设备的irq号
- netdev_ops : 网络设备回调函数的ops对象, 包含用于打开和停止设备, 开始传输, 修改网络设备MTU等函数
- ethool_ops : ethool回调函数的ops对象, 它支持通过运行ethtool命令来获取有关设备的信息
- promiscuity : promiscuity计数器大于0, 网络栈就不会丢弃那些目的地并非本地主机的数据包, 这样tcpdump和wireshark等数据分析工具(嗅探器)就能对其加以利用. 嗅探器会在用户空间打开原始socket, 从而捕获此类发往别处的数据包. 每运行一个嗅探器, promiscuity就加1; 反之, 每关闭一个就减1; 当promiscuity为0时该设备就退出混杂模式(promiscuous mode).

### [struct sk_buff](https://elixir.bootlin.com/linux/v5.10.50/source/include/linux/skbuff.h#L713)
使用skb必须遵循skb api:
- skb_pull_inline()/skb_pull() : 向前移动skb->data指针
- skb_transport_header() : 从skb中提取L4报头(传输层报头)
- skb_network_header() : 从skb中提取L3报头(网络层报头)
- skb_mac_header() : 从skb中提取L2报头(MAC报头)

skb包含数据包的报头(L2, L3和L4)和有效载荷, 在数据包沿网络栈传输的过程中, 可能添加或删除报头.

通常调用netdev_alloc_skb()分配skb. 原先分配skb的dev_alloc_skb()已被废弃. 释放skb用kfree_skb()/dev_kfree_skb().

struct:
- pkt_type : 由数据链路层(L2)决定, 即由eth_type_trans()根据目标以太网地址确定的.

  - PACKET_MULTICAST : 组播地址
  - PACKET_BROADCAST : 广播地址
  - PACKET_MULTICAST : 当前主机的地址
- dev : 一个strcut net_device 的实例. 对于入站的数据包, 表示接收它的网络设备; 对于出站数据包则是发送它的网络设备.
- sk : sock对象. 中转数据包的sk为NULL, 表示不是当前主机生成的.

相关methods：
- eth_type_trans() : 在接收路径中使用, 它会根据以太网报头指定的以太网类型ethertype设置skb的protocol. 它还会调用skb_pull_inline(), 将skb->data前移14(ETH_HLEN, 以太网报头的长度),目的是让指针指向当前层的报头.

  当数据包位于网络设备驱动程序接收路径的L2时, skb->data指向L2(以太网)报头; 调用eth_type_trans()后, 数据包即将进入L3, 此时skb->data指向L3(网络层)报头, 而这个报头紧跟在以太网报头后面.
- ip_rcv()/ipv6_rcv() : 对于接收到的每个数据包, 都应由相应的网络层协议处理程序进行处理, 由dev_add_pack()注册. 它们分别处理ipv4/ipv6数据包.

  在ip_rcv()中, 将执行大部分完整性检查. 如果一切ok, 将调用一个NF_INET_PRE_ROUTING钩子回调函数(前提是已注册)对数据包进行处理. 接下来, 如果数据包没有被这个钩子回调函数丢弃, 将调用ip_rcv_finish()在路由选择子系统中进行查找, 查找操作将根据目的地创建一个缓存条目(dst_entry对象).

### socket
用户空间的套接字

### sock
L3的套接字

## FAQ
### 根据主机的ip查找其mac
该工作由邻接子系统完成. 邻居发现在ipv4由arp协议完成; 在ipv6由NDISC协议负责. 它们区别在于: arp依赖于发送广播请求; NDISC依赖于发送ICMPv6请求(属于组播数据包).

### Too many open files
在Linux系统中,限制打开文件数的内核参数包含以下三个:fs.nr_open、nofile和fs.file-max. 想要加大可打开文件数的限制就需要涉及对这三个参数的修改.

但这几个参数里有的是进程级的,有的是系统级的,有的是用户进程级的, 而且这几个参数还有依赖关系,修改的时候,稍有不慎还可能把机器搞出问题.

`SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)`->`sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));`

```c
// https://elixir.bootlin.com/linux/v6.8/source/net/socket.c#L485
static int sock_map_fd(struct socket *sock, int flags)
{
	struct file *newfile;
	int fd = get_unused_fd_flags(flags); // 获取可用fd句柄号, 这只是在找一个可用的数组下标而己: 在这里会判断打开文件数是否超过soft nofile和fs.nr_open
	if (unlikely(fd < 0)) {
		sock_release(sock);
		return fd;
	}

	newfile = sock_alloc_file(sock, flags, NULL); // 创建sock_a11oc_file对象: 在这里会判断打开文件数是否超过 fs.file-max
	if (!IS_ERR(newfile)) {
		fd_install(fd, newfile);
		return fd;
	}

	put_unused_fd(fd);
	return PTR_ERR(newfile);
}

// https://elixir.bootlin.com/linux/v6.8/source/fs/file.c#L562
int get_unused_fd_flags(unsigned flags)
{
	return __get_unused_fd_flags(flags, rlimit(RLIMIT_NOFILE)); // rlimit(RLIMIT_NOFILE)是读取limits.conf中配置的soft nofile(rim_max应hard nofile)
}
EXPORT_SYMBOL(get_unused_fd_flags);
```

![socket的fd和sock_alloc_file](/misc/img/net/socket_file.png)

__get_unused_fd_flags->[alloc_fd](https://elixir.bootlin.com/linux/v6.8/source/fs/file.c#L499), 它判断要分配的句柄号是不是超过了limits.conf中nofile的限制.
因为fd是当前进程相关的,是一个从O开始的整数, 只要确保分配出去的fd编号不超过limits.conf中的nofile,就能保证该进程打开的文件总数不会超过这个数. 如果超限,就报错EMFILE ( Too many open files).

alloc_fd -> [expand_files](https://elixir.bootlin.com/linux/v6.8/source/fs/file.c#L214), 在expand_files中,又见到nr( 就是fd编号) 和fs.nr_open相比较了, 超过这个限制, 返回错误EMFILE.

由上可见,无论是和fs.nr_open, 还是和soft nofile比较, 都是用当前进程的文件描述符序号比较的, 所以这两个参数都是进程级别的.