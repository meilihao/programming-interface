# I2C
ref:
- [Linux I2C核心, 总线与设备驱动](<<Linux设备驱动开发详解#15>>)

I2C由3部分构成:
1. core

	提供了总线驱动和设备驱动的注册, 注销, 通信方法(即Algorithm)上层的与具体适配器无关的代码以及探测设备, 检测设备地址的上层代码
1. 总线驱动

	对I2C硬件体系结构中的适配器断的实现, 适配器可由cpu控制, 甚至直接集成到cpu内部

	主要包含I2C适配器数据结构i2c_adapter, I2C适配器的Algorithm数据结构i2c_algorithm和控制I2C适配器产生通信信号的函数。

	通过I2C总线驱动的代码， 可以控制I2C适配器以主控方式产生开始位, 停止位, 读写周期, 以及以从设备方式被读写, 产生ACK等
1. 设备驱动

	对I2C硬件体系结构中设备断的实现, 设备一般挂载在受cpu控制的I2C适配器上, 通过I2C适配器与cpu交互数据

	主要数据结构是i2c_driver和i2c_client.


sysfs位置: /sys/bus/i2c/, 以适配器地址和芯片地址的形式输出.
源码在`drivers/i2c`:
- i2c-core*.c

	实现i2c的核心功能
- i2c-dev.c
	
	实现了I2C适配器设备文件dd功能, 它并不是针对特定设备而设计的. 只提供了通用的read(), write()和ioctl()等接口. 应用层可借用这些接口访问挂接在适配器上的I2C设备的存储空间或寄存器, 并控制i2c设备的工作方式
- busses

	包含了一些i2c主机控制器的驱动
- algos

	实现了一些i2c总线适配器的通信方法



```c
// https://elixir.bootlin.com/linux/v6.6.22/source/include/linux/i2c.h#L541
/**
 * struct i2c_algorithm - represent I2C transfer method
 * @master_xfer: Issue a set of i2c transactions to the given I2C adapter
 *   defined by the msgs array, with num messages available to transfer via
 *   the adapter specified by adap.
 * @master_xfer_atomic: same as @master_xfer. Yet, only using atomic context
 *   so e.g. PMICs can be accessed very late before shutdown. Optional.
 * @smbus_xfer: Issue smbus transactions to the given I2C adapter. If this
 *   is not present, then the bus layer will try and convert the SMBus calls
 *   into I2C transfers instead.
 * @smbus_xfer_atomic: same as @smbus_xfer. Yet, only using atomic context
 *   so e.g. PMICs can be accessed very late before shutdown. Optional.
 * @functionality: Return the flags that this algorithm/adapter pair supports
 *   from the ``I2C_FUNC_*`` flags.
 * @reg_slave: Register given client to I2C slave mode of this adapter
 * @unreg_slave: Unregister given client from I2C slave mode of this adapter
 *
 * The following structs are for those who like to implement new bus drivers:
 * i2c_algorithm is the interface to a class of hardware solutions which can
 * be addressed using the same bus algorithms - i.e. bit-banging or the PCF8584
 * to name two of the most common.
 *
 * The return codes from the ``master_xfer{_atomic}`` fields should indicate the
 * type of error code that occurred during the transfer, as documented in the
 * Kernel Documentation file Documentation/i2c/fault-codes.rst. Otherwise, the
 * number of messages executed should be returned.
 */
struct i2c_algorithm {
	/*
	 * If an adapter algorithm can't do I2C-level access, set master_xfer
	 * to NULL. If an adapter algorithm can do SMBus access, set
	 * smbus_xfer. If set to NULL, the SMBus protocol is simulated
	 * using common I2C messages.
	 *
	 * master_xfer should return the number of messages successfully
	 * processed, or a negative value on error
	 */
	int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs,
			   int num); // i2c传输函数, i2c主机驱动的大部分工作集中在这里
	int (*master_xfer_atomic)(struct i2c_adapter *adap,
				   struct i2c_msg *msgs, int num);
	int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr,
			  unsigned short flags, char read_write,
			  u8 command, int size, union i2c_smbus_data *data); // SMBus传输函数
	int (*smbus_xfer_atomic)(struct i2c_adapter *adap, u16 addr,
				 unsigned short flags, char read_write,
				 u8 command, int size, union i2c_smbus_data *data);

	/* To determine what the adapter supports */
	u32 (*functionality)(struct i2c_adapter *adap);

#if IS_ENABLED(CONFIG_I2C_SLAVE)
	int (*reg_slave)(struct i2c_client *client);
	int (*unreg_slave)(struct i2c_client *client);
#endif
};
```

i2c_adapter对应于物理上的一个适配器, 而i2c_algorithm对应一套通信方法. 一个i2c适配器需要i2c_algorithm提供的通信函数来控制适配器产生特定的访问周期.

i2c_algorithm.master_xfer用于产生i2c访问周期需要的信号, 以i2c_msg(即I2C消息)为单位.

i2c_driver对应一套驱动方法. i2c_client对应于真实的物理设备. i2c_driver和i2c_client是一对多的关系.

i2c_adapter于i2c_client的关系与I2C硬件体系中适配器和设备的关系一致, 即i2c_client依附于i2c_adapter. i2c_adapter与i2c_client是一对多的关系.

## 命令
在i2c总线子系统中, 命令抽象由i2c_msg提供