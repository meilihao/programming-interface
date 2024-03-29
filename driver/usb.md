# usb
ref:
- [Linux驱动：USB设备驱动看这一篇就够了](https://zhuanlan.zhihu.com/p/558716468)

linux提供了主机侧和设备侧视角的usb驱动框架.

从主机侧看, 在linux驱动中, 处于最底层的是usb主机控制器硬件, 其上运行的是usb主机控制器驱动, 再上面是usb core, 最后是usb设备驱动层(插入的u盘, 鼠标, usb转串口等设备驱动). 因此需实现的usb驱动包括主机控制器驱动和设备驱动. usb主机控制器驱动控制插入其中的usb设备, 而usb设备驱动控制该设备如何作为从设备与主机通信.

从设备侧看, 在linux驱动中, 处于最底层的是usb设备控制器, 其上是UDC驱动, 再上面是Gadget Function API和Gadget Function驱动程序. 因此需实现的usb驱动包括usb设备控制器(UDC)驱动和Gaddget Function驱动两类. UDC驱动程序直接访问硬件, 控制USB设备和主机间的底层通信, 向上层提供与硬件相关操作的回调函数. Gaddget Function API是UDC驱动回调函数的包装. Gaddget Function驱动程序具体控制usb设备的实现, 使设备表现出`网络连接`, `打印机`或`usb mass storage`等特性. Gaddget Function API把下层的UDC驱动和上层Gaddget Function驱动隔离开, 使得在linux中编写usb设备侧驱动程序时能把功能的实现和底层通信分离.

usb采用树形拓扑结构, 主机侧和设备侧的usb控制器分别被称为主机控制器(host controller)和usb设备控制器(udc), 每条总线上只有一个主机控制器, 负责协调主机和设备间的通信, 而设备不能主动向主机发送任何信息.

usb核心负责usb驱动管理和协议处理的主要工作: 向上为设备驱动提供编程接口, 向下为usb主机控制器驱动提供编程接口; 维护整个系统的usb设备信息; 完成设备热插拔控制, 总线数据传输控制等.

在usb设备的逻辑组织中, 包含设备, 配置, 接口和端口4个层次:
1. 一个usb设备通常有一个或多个配置

	每个usb设备都提供不同级别的配置信息, 不同配置使设备表现出不同的功能组合(在探测/连接期间需从其中选定一个)
1. 一个配置通常有一个或多个接口

	在usb协议中, 接口由多个端点组成, 代表一个基本的功能, 是usb驱动程序控制的对象, 一个功能复杂的usb设备可具有多个接口. 一个配置中的所有接口可同时有效, 并可被不同的驱动程序连接. 每个接口可以有备用接口, 以提供不同质量的服务参数.
1. 一个接口通常有零个或多个端口

	端点是usb通信的最基本形式, 主机只能通过端点与设备进行通信, 以使用设备的功能. 在usb系统中每个端点都有唯一的地址, 它由设备地址和端点号给出的. 每个端点有一定的属性, 其中包括传输方式, 总线访问频率, 带宽, 端点号和数据包的最大容量等. 一个usb端点只能在一个方向上承载数据, 从主机到设备(称输出端点)或从设备到主机(称输入端点). 端点0通常为控制端点, 用于设备初始化参数等. 只要设备连接到usb上并且上电, 端点0即可被访问, 端点1,2等一般用作数据端点, 用于主机与设备间通信.

在linux中, usb设备用usb_device来描述, usb设备描述符是usb_device_descriptor. usb配置用usb_host_config描述, usb配置描述符用usb_config_ddescriptor. usb接口用usb_interface描述, usb接口描述符是usb_interface_descriptor. usb端点用usb_host_endpoint描述, usb端点描述符是usb_endpoint_descriptor. 字符串描述符是usb_string_descriptor, 用于在其他描述符中为某些字段提供字符串索引, 它们可被用于检索描述性字符串, 可以以多种语言形式提供.

`lsusb -v`可输出上述描述符.

```c
// https://elixir.bootlin.com/linux/v6.6.22/source/include/uapi/linux/usb/ch9.h#L286
/* USB_DT_DEVICE: Device descriptor */
struct usb_device_descriptor {
	__u8  bLength; // 描述符长度
	__u8  bDescriptorType; // 描述符类型编号

	__le16 bcdUSB; // usb版本号
	__u8  bDeviceClass; // usb分配的设备类code
	__u8  bDeviceSubClass; // usb分配的子类code
	__u8  bDeviceProtocol; // usb分配的协议code
	__u8  bMaxPacketSize0; // endpoint0最大包大小
	__le16 idVendor; // 厂商编号
	__le16 idProduct; // 产品编号
	__le16 bcdDevice; // 设备出厂编号
	__u8  iManufacturer; // 描述厂商字符串的索引
	__u8  iProduct; // 描述产品字符串的索引
	__u8  iSerialNumber; // 描述设备序列号字符串的索引
	__u8  bNumConfigurations; // 可能的配置数量
} __attribute__ ((packed));

// https://elixir.bootlin.com/linux/v6.6.22/source/include/uapi/linux/usb/ch9.h#L346
/* USB_DT_CONFIG: Configuration descriptor information.
 *
 * USB_DT_OTHER_SPEED_CONFIG is the same descriptor, except that the
 * descriptor type is different.  Highspeed-capable devices can look
 * different depending on what speed they're currently running.  Only
 * devices with a USB_DT_DEVICE_QUALIFIER have any OTHER_SPEED_CONFIG
 * descriptors.
 */
struct usb_config_descriptor {
	__u8  bLength;
	__u8  bDescriptorType;

	__le16 wTotalLength; // 配置所返回的所有数据的大小
	__u8  bNumInterfaces; // 配置所支持的接口数
	__u8  bConfigurationValue; //Set_Configuration命令需要的参数值
	__u8  iConfiguration; // 描述该配置的字符串索引值
	__u8  bmAttributes; // 供电模式的选择
	__u8  bMaxPower; // 设备从总线提取的最大电流
} __attribute__ ((packed));

// https://elixir.bootlin.com/linux/v6.6.22/source/include/uapi/linux/usb/ch9.h#L389
/* USB_DT_INTERFACE: Interface descriptor */
struct usb_interface_descriptor {
	__u8  bLength;
	__u8  bDescriptorType;

	__u8  bInterfaceNumber; // 接口的编号
	__u8  bAlternateSetting; // 备用的接口描述符编号
	__u8  bNumEndpoints; // 该接口使用的端点数, 不包括端点0
	__u8  bInterfaceClass; // 接口类型
	__u8  bInterfaceSubClass; // 接口子类型
	__u8  bInterfaceProtocol; // 接口所遵循的协议
	__u8  iInterface; // 描述接口的字符串索引值
} __attribute__ ((packed));

// https://elixir.bootlin.com/linux/v6.6.22/source/include/uapi/linux/usb/ch9.h#L407
/* USB_DT_ENDPOINT: Endpoint descriptor */
struct usb_endpoint_descriptor {
	__u8  bLength;
	__u8  bDescriptorType;

	__u8  bEndpointAddress; // 端点地址: 0~3位是端点号, 第7位是方向(0,输出; 1, 输入)
	__u8  bmAttributes; // 端点属性: bit[0-1]的值是00表示控制, 01表示同步, 02表示批量, 03表示中断
	__le16 wMaxPacketSize; // 本端点接收或发送的最大信息包的大小
	__u8  bInterval; // 轮询数据传送端点的时间间隔. 对于批量传送的端点以及控制传送的端点, 忽略; 对于同步传送的端点, 为1; 对于中断传送的端点, 值是1~255

	/* NOTE:  these two are _only_ in audio endpoints. */
	/* use USB_DT_ENDPOINT*_SIZE in bLength, not sizeof. */
	__u8  bRefresh;
	__u8  bSynchAddress;
} __attribute__ ((packed));

// https://elixir.bootlin.com/linux/v6.6.22/source/include/uapi/linux/usb/ch9.h#L372
/* USB_DT_STRING: String descriptor */
struct usb_string_descriptor {
	__u8  bLength;
	__u8  bDescriptorType;

	union {
		__le16 legacy_padding;
		__DECLARE_FLEX_ARRAY(__le16, wData);	/* UTF-16LE encoded */
	};
} __attribute__ ((packed));
```

## usb主机控制器驱动
usb主机控制器驱动规格有:
- OHCI(Open Host Controller Interface)

	为非pc系统上以及带有SiS和ALi芯片组的PC主板上的usb芯片提供支持
- UHCI(Universal Host Controller Interface)

	为大多数其他PC主板(包括intel和Via)上的usb芯片提供支持
- EHCI(Enhanced Host Controller Interface)

	由usb2.0 规范提出, 兼容OHCI和UHCI. 由于UHCI的硬件线路比OHCI简单, 成本较低, 但需要较复杂的驱动程序, cpu负荷稍重
- xHCI(eXtensible Host Controller Interface)

	是intel开发的一个usb主机控制器接口, 主要面向usb 3.0, 但也支持usb 2.0及以下的设备

linux用usb_hcd描述usb主机控制器驱动, 及其驱动hc_driver.

usb_hcd相关操作:
- [usb_create_hcd](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/core/hcd.c#L2647): 创建HCD
- [usb_add_hcd](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/core/hcd.c#L2790): 添加HCD
- [usb_remove_hcd](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/core/hcd.c#L3004): 移除HCD

```c
// https://elixir.bootlin.com/linux/v6.6.22/source/include/linux/usb/hcd.h#L68
struct usb_hcd {

	/*
	 * housekeeping
	 */
	struct usb_bus		self;		/* hcd is-a bus */
	struct kref		kref;		/* reference counter */

	const char		*product_desc;	/* product/vendor string */
	int			speed;		/* Speed for this roothub.
						 * May be different from
						 * hcd->driver->flags & HCD_MASK
						 */
	char			irq_descr[24];	/* driver + bus # */

	struct timer_list	rh_timer;	/* drives root-hub polling */
	struct urb		*status_urb;	/* the current status urb */
#ifdef CONFIG_PM
	struct work_struct	wakeup_work;	/* for remote wakeup */
#endif
	struct work_struct	died_work;	/* for when the device dies */

	/*
	 * hardware info/state
	 */
	const struct hc_driver	*driver;	/* hw-specific hooks */

	/*
	 * OTG and some Host controllers need software interaction with phys;
	 * other external phys should be software-transparent
	 */
	struct usb_phy		*usb_phy;
	struct usb_phy_roothub	*phy_roothub;

	/* Flags that need to be manipulated atomically because they can
	 * change while the host controller is running.  Always use
	 * set_bit() or clear_bit() to change their values.
	 */
	unsigned long		flags;
#define HCD_FLAG_HW_ACCESSIBLE		0	/* at full power */
#define HCD_FLAG_POLL_RH		2	/* poll for rh status? */
#define HCD_FLAG_POLL_PENDING		3	/* status has changed? */
#define HCD_FLAG_WAKEUP_PENDING		4	/* root hub is resuming? */
#define HCD_FLAG_RH_RUNNING		5	/* root hub is running? */
#define HCD_FLAG_DEAD			6	/* controller has died? */
#define HCD_FLAG_INTF_AUTHORIZED	7	/* authorize interfaces? */
#define HCD_FLAG_DEFER_RH_REGISTER	8	/* Defer roothub registration */

	/* The flags can be tested using these macros; they are likely to
	 * be slightly faster than test_bit().
	 */
#define HCD_HW_ACCESSIBLE(hcd)	((hcd)->flags & (1U << HCD_FLAG_HW_ACCESSIBLE))
#define HCD_POLL_RH(hcd)	((hcd)->flags & (1U << HCD_FLAG_POLL_RH))
#define HCD_POLL_PENDING(hcd)	((hcd)->flags & (1U << HCD_FLAG_POLL_PENDING))
#define HCD_WAKEUP_PENDING(hcd)	((hcd)->flags & (1U << HCD_FLAG_WAKEUP_PENDING))
#define HCD_RH_RUNNING(hcd)	((hcd)->flags & (1U << HCD_FLAG_RH_RUNNING))
#define HCD_DEAD(hcd)		((hcd)->flags & (1U << HCD_FLAG_DEAD))
#define HCD_DEFER_RH_REGISTER(hcd) ((hcd)->flags & (1U << HCD_FLAG_DEFER_RH_REGISTER))

	/*
	 * Specifies if interfaces are authorized by default
	 * or they require explicit user space authorization; this bit is
	 * settable through /sys/class/usb_host/X/interface_authorized_default
	 */
#define HCD_INTF_AUTHORIZED(hcd) \
	((hcd)->flags & (1U << HCD_FLAG_INTF_AUTHORIZED))

	/*
	 * Specifies if devices are authorized by default
	 * or they require explicit user space authorization; this bit is
	 * settable through /sys/class/usb_host/X/authorized_default
	 */
	enum usb_dev_authorize_policy dev_policy;

	/* Flags that get set only during HCD registration or removal. */
	unsigned		rh_registered:1;/* is root hub registered? */
	unsigned		rh_pollable:1;	/* may we poll the root hub? */
	unsigned		msix_enabled:1;	/* driver has MSI-X enabled? */
	unsigned		msi_enabled:1;	/* driver has MSI enabled? */
	/*
	 * do not manage the PHY state in the HCD core, instead let the driver
	 * handle this (for example if the PHY can only be turned on after a
	 * specific event)
	 */
	unsigned		skip_phy_initialization:1;

	/* The next flag is a stopgap, to be removed when all the HCDs
	 * support the new root-hub polling mechanism. */
	unsigned		uses_new_polling:1;
	unsigned		has_tt:1;	/* Integrated TT in root hub */
	unsigned		amd_resume_bug:1; /* AMD remote wakeup quirk */
	unsigned		can_do_streams:1; /* HC supports streams */
	unsigned		tpl_support:1; /* OTG & EH TPL support */
	unsigned		cant_recv_wakeups:1;
			/* wakeup requests from downstream aren't received */

	unsigned int		irq;		/* irq allocated */
	void __iomem		*regs;		/* device memory/io */
	resource_size_t		rsrc_start;	/* memory/io resource start */
	resource_size_t		rsrc_len;	/* memory/io resource length */
	unsigned		power_budget;	/* in mA, 0 = no limit */

	struct giveback_urb_bh  high_prio_bh;
	struct giveback_urb_bh  low_prio_bh;

	/* bandwidth_mutex should be taken before adding or removing
	 * any new bus bandwidth constraints:
	 *   1. Before adding a configuration for a new device.
	 *   2. Before removing the configuration to put the device into
	 *      the addressed state.
	 *   3. Before selecting a different configuration.
	 *   4. Before selecting an alternate interface setting.
	 *
	 * bandwidth_mutex should be dropped after a successful control message
	 * to the device, or resetting the bandwidth after a failed attempt.
	 */
	struct mutex		*address0_mutex;
	struct mutex		*bandwidth_mutex;
	struct usb_hcd		*shared_hcd;
	struct usb_hcd		*primary_hcd;


#define HCD_BUFFER_POOLS	4
	struct dma_pool		*pool[HCD_BUFFER_POOLS];

	int			state;
#	define	__ACTIVE		0x01
#	define	__SUSPEND		0x04
#	define	__TRANSIENT		0x80

#	define	HC_STATE_HALT		0
#	define	HC_STATE_RUNNING	(__ACTIVE)
#	define	HC_STATE_QUIESCING	(__SUSPEND|__TRANSIENT|__ACTIVE)
#	define	HC_STATE_RESUMING	(__SUSPEND|__TRANSIENT)
#	define	HC_STATE_SUSPENDED	(__SUSPEND)

#define	HC_IS_RUNNING(state) ((state) & __ACTIVE)
#define	HC_IS_SUSPENDED(state) ((state) & __SUSPEND)

	/* memory pool for HCs having local memory, or %NULL */
	struct gen_pool         *localmem_pool;

	/* more shared queuing code would be good; it should support
	 * smarter scheduling, handle transaction translators, etc;
	 * input size of periodic table to an interrupt scheduler.
	 * (ohci 32, uhci 1024, ehci 256/512/1024).
	 */

	/* The HC driver's private data is stored at the end of
	 * this structure.
	 */
	unsigned long hcd_priv[]
			__attribute__ ((aligned(sizeof(s64))));
};

// https://elixir.bootlin.com/linux/v6.6.22/source/include/linux/usb/hcd.h#L68
struct hc_driver {
	const char	*description;	/* "ehci-hcd" etc */
	const char	*product_desc;	/* product/vendor string */
	size_t		hcd_priv_size;	/* size of private data */

	/* irq handler */
	irqreturn_t	(*irq) (struct usb_hcd *hcd);

	int	flags;
#define	HCD_MEMORY	0x0001		/* HC regs use memory (else I/O) */
#define	HCD_DMA		0x0002		/* HC uses DMA */
#define	HCD_SHARED	0x0004		/* Two (or more) usb_hcds share HW */
#define	HCD_USB11	0x0010		/* USB 1.1 */
#define	HCD_USB2	0x0020		/* USB 2.0 */
#define	HCD_USB3	0x0040		/* USB 3.0 */
#define	HCD_USB31	0x0050		/* USB 3.1 */
#define	HCD_USB32	0x0060		/* USB 3.2 */
#define	HCD_MASK	0x0070
#define	HCD_BH		0x0100		/* URB complete in BH context */

	/* called to init HCD and root hub */
	int	(*reset) (struct usb_hcd *hcd);
	int	(*start) (struct usb_hcd *hcd);

	/* NOTE:  these suspend/resume calls relate to the HC as
	 * a whole, not just the root hub; they're for PCI bus glue.
	 */
	/* called after suspending the hub, before entering D3 etc */
	int	(*pci_suspend)(struct usb_hcd *hcd, bool do_wakeup);

	/* called after entering D0 (etc), before resuming the hub */
	int	(*pci_resume)(struct usb_hcd *hcd, pm_message_t state);

	/* called just before hibernate final D3 state, allows host to poweroff parts */
	int	(*pci_poweroff_late)(struct usb_hcd *hcd, bool do_wakeup);

	/* cleanly make HCD stop writing memory and doing I/O */
	void	(*stop) (struct usb_hcd *hcd);

	/* shutdown HCD */
	void	(*shutdown) (struct usb_hcd *hcd);

	/* return current frame number */
	int	(*get_frame_number) (struct usb_hcd *hcd);

	/* manage i/o requests, device state */
	int	(*urb_enqueue)(struct usb_hcd *hcd,
				struct urb *urb, gfp_t mem_flags); // 上层通过usb_submit_urb()提交一个usb请求后, 经usb_hcd_submit_urb()最终到达usb_hcd.driver的urb_enqueue
	int	(*urb_dequeue)(struct usb_hcd *hcd,
				struct urb *urb, int status);

	/*
	 * (optional) these hooks allow an HCD to override the default DMA
	 * mapping and unmapping routines.  In general, they shouldn't be
	 * necessary unless the host controller has special DMA requirements,
	 * such as alignment constraints.  If these are not specified, the
	 * general usb_hcd_(un)?map_urb_for_dma functions will be used instead
	 * (and it may be a good idea to call these functions in your HCD
	 * implementation)
	 */
	int	(*map_urb_for_dma)(struct usb_hcd *hcd, struct urb *urb,
				   gfp_t mem_flags);
	void    (*unmap_urb_for_dma)(struct usb_hcd *hcd, struct urb *urb);

	/* hw synch, freeing endpoint resources that urb_dequeue can't */
	void	(*endpoint_disable)(struct usb_hcd *hcd,
			struct usb_host_endpoint *ep);

	/* (optional) reset any endpoint state such as sequence number
	   and current window */
	void	(*endpoint_reset)(struct usb_hcd *hcd,
			struct usb_host_endpoint *ep);

	/* root hub support */
	int	(*hub_status_data) (struct usb_hcd *hcd, char *buf);
	int	(*hub_control) (struct usb_hcd *hcd,
				u16 typeReq, u16 wValue, u16 wIndex,
				char *buf, u16 wLength);
	int	(*bus_suspend)(struct usb_hcd *);
	int	(*bus_resume)(struct usb_hcd *);
	int	(*start_port_reset)(struct usb_hcd *, unsigned port_num);
	unsigned long	(*get_resuming_ports)(struct usb_hcd *);

		/* force handover of high-speed port to full-speed companion */
	void	(*relinquish_port)(struct usb_hcd *, int);
		/* has a port been handed over to a companion? */
	int	(*port_handed_over)(struct usb_hcd *, int);

		/* CLEAR_TT_BUFFER completion callback */
	void	(*clear_tt_buffer_complete)(struct usb_hcd *,
				struct usb_host_endpoint *);

	/* xHCI specific functions */
		/* Called by usb_alloc_dev to alloc HC device structures */
	int	(*alloc_dev)(struct usb_hcd *, struct usb_device *);
		/* Called by usb_disconnect to free HC device structures */
	void	(*free_dev)(struct usb_hcd *, struct usb_device *);
	/* Change a group of bulk endpoints to support multiple stream IDs */
	int	(*alloc_streams)(struct usb_hcd *hcd, struct usb_device *udev,
		struct usb_host_endpoint **eps, unsigned int num_eps,
		unsigned int num_streams, gfp_t mem_flags);
	/* Reverts a group of bulk endpoints back to not using stream IDs.
	 * Can fail if we run out of memory.
	 */
	int	(*free_streams)(struct usb_hcd *hcd, struct usb_device *udev,
		struct usb_host_endpoint **eps, unsigned int num_eps,
		gfp_t mem_flags);

	/* Bandwidth computation functions */
	/* Note that add_endpoint() can only be called once per endpoint before
	 * check_bandwidth() or reset_bandwidth() must be called.
	 * drop_endpoint() can only be called once per endpoint also.
	 * A call to xhci_drop_endpoint() followed by a call to
	 * xhci_add_endpoint() will add the endpoint to the schedule with
	 * possibly new parameters denoted by a different endpoint descriptor
	 * in usb_host_endpoint.  A call to xhci_add_endpoint() followed by a
	 * call to xhci_drop_endpoint() is not allowed.
	 */
		/* Allocate endpoint resources and add them to a new schedule */
	int	(*add_endpoint)(struct usb_hcd *, struct usb_device *,
				struct usb_host_endpoint *);
		/* Drop an endpoint from a new schedule */
	int	(*drop_endpoint)(struct usb_hcd *, struct usb_device *,
				 struct usb_host_endpoint *);
		/* Check that a new hardware configuration, set using
		 * endpoint_enable and endpoint_disable, does not exceed bus
		 * bandwidth.  This must be called before any set configuration
		 * or set interface requests are sent to the device.
		 */
	int	(*check_bandwidth)(struct usb_hcd *, struct usb_device *);
		/* Reset the device schedule to the last known good schedule,
		 * which was set from a previous successful call to
		 * check_bandwidth().  This reverts any add_endpoint() and
		 * drop_endpoint() calls since that last successful call.
		 * Used for when a check_bandwidth() call fails due to resource
		 * or bandwidth constraints.
		 */
	void	(*reset_bandwidth)(struct usb_hcd *, struct usb_device *);
		/* Returns the hardware-chosen device address */
	int	(*address_device)(struct usb_hcd *, struct usb_device *udev);
		/* prepares the hardware to send commands to the device */
	int	(*enable_device)(struct usb_hcd *, struct usb_device *udev);
		/* Notifies the HCD after a hub descriptor is fetched.
		 * Will block.
		 */
	int	(*update_hub_device)(struct usb_hcd *, struct usb_device *hdev,
			struct usb_tt *tt, gfp_t mem_flags);
	int	(*reset_device)(struct usb_hcd *, struct usb_device *);
		/* Notifies the HCD after a device is connected and its
		 * address is set
		 */
	int	(*update_device)(struct usb_hcd *, struct usb_device *);
	int	(*set_usb2_hw_lpm)(struct usb_hcd *, struct usb_device *, int);
	/* USB 3.0 Link Power Management */
		/* Returns the USB3 hub-encoded value for the U1/U2 timeout. */
	int	(*enable_usb3_lpm_timeout)(struct usb_hcd *,
			struct usb_device *, enum usb3_link_state state);
		/* The xHCI host controller can still fail the command to
		 * disable the LPM timeouts, so this can return an error code.
		 */
	int	(*disable_usb3_lpm_timeout)(struct usb_hcd *,
			struct usb_device *, enum usb3_link_state state);
	int	(*find_raw_port_number)(struct usb_hcd *, int);
	/* Call for power on/off the port if necessary */
	int	(*port_power)(struct usb_hcd *hcd, int portnum, bool enable);
	/* Call for SINGLE_STEP_SET_FEATURE Test for USB2 EH certification */
#define EHSET_TEST_SINGLE_STEP_SET_FEATURE 0x06
	int	(*submit_single_step_set_feature)(struct usb_hcd *,
			struct urb *, int);
};
```

ehci hcd驱动是hcd驱动的实例, 它定义了ehci_hcd, 通常作为usb_hcd的hcd_priv来使用.

[hcd_to_ehci()](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/host/ehci.h#L266)和[ehci_to_hcd](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/host/ehci.h#L270)可实现usb_hcd和ehci_hcd的互换.

ehci的操作, 具体见[ehci_hc_driver](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/host/ehci-hcd.c#L1240):
- [ehci_init](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/host/ehci-hcd.c#L454): 初始化
- [ehci_run](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/host/ehci-hcd.c#L573): 开启
- [ehci_stop](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/host/ehci-hcd.c#L420): 停止
- [ehci_reset](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/host/ehci-hcd.c#L231): 复位

ehci-hcd.c实现了绝大部分ECHI主机驱动的工作.

### chipidea usb主机驱动
chipidea usb ip在嵌入式系统中广泛应用, 代码在[drivers/usb/chipidea](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/chipidea).

当core.c的ci_hdrc_probe()被执行后(一个platform_device与ci_hdrc_driver(platform_driver)匹配上), 调用[ci_hdrc_host_init()](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/chipidea/host.c#L467)完成hc_driver的初始化并赋值一系列与chipidea平台相关的私有数据.

## usb设备驱动
从主机角度, usb设备驱动是值如何访问usb设备, 而不是usb设备内部本身运行的固件程序.

linux实现了几类通用的usb驱动:
1. 音频设备类
1. 通信设备类
1. HID(人机接口)设备类
1. 显示设备类
1. 海量存储设备类
1. 电源设备类
1. 打印设备类
1. 集线器设备类

一般的通用usb设备(u盘, usb键鼠等)都不需要再编写驱动. 特定厂商或芯片的驱动, 也可参考已在内核的驱动模板.

linux为各类usb分配了相应的主设备号, ACM USB是166(/ddev/ttyACM<n>) , usb打印是180(/dev/lp<n>, 次设备号是0~15), usb串口是188(dev/ttyUSB<n>), 具体见[https://www.lanana.org]的设备列表.

在debugfs下, /sys/kernel/debug/usb/devices包含了usb的设备信息. 插入一个u盘后, 即可看到相关信息. USBView工具可显示USB信息.

在sysfs中, 也有usb信息, 但仅限于接口级别, 接口命名规则是: `根集线器-集线器端口号(-集线器端口号-...):配置.接口`, 见`tree /sys/bus/usb`.

usb_driver用于描述usb设备驱动. 编写新usb驱动时, 主要工作是实现probe()(插入)和disconnect()(拔出).

usb_driver的id_table描述了这个usb驱动支持ddusb设备列表, 它指向一个usb_device_id数组. 当usb核心检测到某个设备的属性和某个驱动的usb_device_id匹配时, probe()被执行.

## usb请求块(USB Request Block, URB)
URB是usb设备驱动中用来描述与usb设备通信所用的基本载体和核心数据结构, 类似网络设备驱动的sk_buff.

usb设备中每个端点都处理一个URB队列

URB处理流程:
1. [usb_alloc_urb](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/core/urb.c#L71):被一个USB设备驱动创建
2. 初始化, 被安排给一个特定usb设备的特定端口

	对于中断URB, 使用usb_fill_int_urb()来初始化URB
	对于批量URB, 使用usb_fill_bulk_urb()来初始化
	对于控制URB, 使用usb_fill_control_urb()来初始化
3. 被usb设备驱动提交给usb核心

	在提交urb到usb核心后, 直到完成函数被调用前, 不要访问urb中的任何成员

	usb_submit_urb()在原子上下文和进程上下文中都可以被调用, mem_flags需根据调用环境设置:
	-  GFP_ATOMIC：在中断处理函数、底半部、tasklet、定时器处理函数以及urb完成函数中，在调用者持有自旋锁或者读写锁时以及当驱动将current->state修改为非 TASK_ RUNNING时，应使用此标志
    - GFP_NOIO：在存储设备的块I/O和错误处理路径中, 应使用此标志
    - GFP_KERNEL：如果没有任何理由使用GFP_ATOMIC和GFP_NOIO，就使用GFP_ KERNE
                        
4. 由usb核心提交给指定的usb主机控制器驱动
5. 被usb主机控制器处理, 进行一次到usb设备的传送

在完成回调中存在3种情况, 可用urb->status判断:
1. urb 被成功发送给设备，并且设备返回正确的确认。如果urb->status 为0，意味着对于一个输出urb，数据被成功发送；对于一个输入urb，请求的数据被成功收到
2. 如果发送数据到设备或从设备接收数据时发生了错误，urb->status 将记录错误值
3. urb 被从USB 核心“去除连接”，这发生在驱动通过usb_unlink_urb()或usb_kill_urb()函数取消urb或urb 虽已提交但USB 设备被拔出的情况下


当urb 生命结束时（处理完成或被解除链接），通过urb 结构体的status 成员可以获知其原因:
- 0 表示传输成功
- -ENOENT 表示被usb_kill_urb()杀死
- -ECONNRESET 表示被usb_unlink_urb()杀死
- -EPROTO 表示传输中发生了bitstuff 错误或者硬件未能及时收到响应数据包
- -ENODEV 表示USB 设备已被移除
- -EXDEV 表示等时传输仅完成了一部分
- etc

### 简单的批量与控制urb
有时usb驱动只是从usb设备上接收或向usb设备发送一些简单的数据, 此时没必要走整个的urb处理流程, 可用:
1. usb_bulk_msg

	创建一个usb批量urb并将它发送到设备, 是同步的, 一直等待URB完成才返回
1. usb_control_msg()

	与usb_bulk_msg类似, 但仅给驱动发送和结束USB控制信息而不是批量信息, 也是同步

它们不能在中断上下文和持有自旋锁的情况下使用, 且无法被其他函数取消.

## probe
主要工作:
1. 探测设备的端点地址, 缓冲区大小, 初始化任何可能用于控制usb设备的数据结构
1. 把已初始化的数据结构的指针保存到接口设备中

	usb_set_intfdata(), 与之相反的是usb_get_intfdata().
1. 注册usb设备

	usb_register_dev()

## usb骨架程序
[/drivers/usb/usb-skeleton.c](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/usb/usb-skeleton.c)提供了一个最基础的usb驱动程序即usb骨架程序.

## usb键盘驱动
[usb_kbd_driver](https://elixir.bootlin.com/linux/v6.6.22/source/drivers/hid/usbhid/usbkbd.c#L391)

## USB UDC和Gadget驱动
ref:
- [USB Gadget 驱动程序框架](https://cloud.tencent.com/developer/article/2315297)
- [Android USB之复合设备(gadget)详解 ](https://www.cnblogs.com/blogs-of-lxl/p/16815102.html)

USB UDC驱动指作为**其他usb主机控制器外设**的usb硬件设备上的底层硬件控制器的驱动, 该硬件和驱动负责将一个usb设备依附于一个usb主机控制器上.

比如将Android的手机连到pc上, 手机中的底层usb控制器运行的是UDC驱动, 而手机要变成U盘, 则需要FileStorage驱动即Function驱动.

UDC驱动在drivers/usb/gadget目录, 其下的udc目录包含了很多对应Soc平台的UDC驱动, 而function目录包含了一些Gadget功能:
- Ethernet over USB: 模拟以太网网口
- File-Backed Storage Gadget: 最常见的u盘功能实现
- Serial Gadget: 包含Generic Serial实现和CDC ACM规范实现
- Gadget MIDI: 暴露ALSA MIDI接口
- USB Video Class Gadget: 将linux系统作为另一个系统usb视频采集源
- GadgetFS
- etc

核心数据结构:
- usb_gadget: usb设备控制器
- usb_gadget_ops: UDC操作
- usb_ep: 端点
- usb_ep_ops: 端点操作

在具体的UDC驱动中, 需要封装usb_gadget和每个端点usb_ep, 实现usb_gadget的usb_gadget_ops和usb_ep_ops, 完成usb_request后就可注册UDC(by usb_add_gadget_udc)了.

usb_function通过usb_function_register()来注册.

在Gadget驱动中, 用usb_request来描述一次传输请求, 类似usb主机侧的URB.

Gadget常用api, 供Gadget Function驱动调用, 从而便于它们操作端点:
1. usb_ep_enable/usb_ep_disable
1. alloc_ep_req/usb_alloc_ep_req/usb_free_ep_req
1. usb_ep_queue/usb_ep_dequeue : 提交或取消usb_request
1. usb_ep_fifo_status/usb_ep_fifo_flush: 端点fifo管理
1. usb_ep_autoconfig: 端点自动配置

### Chipidea USB UDC驱动
`drivers/usb/chipidea/udc.c`