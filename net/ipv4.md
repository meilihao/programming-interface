# ipv4
ipv4(RFC 791)是当今基于标准的internet中的核心协议之一, 由于ip地址耗尽的问题, 正在向ipv6过渡, 但需要熟悉它.

ipv4报头用[iphdr](https://elixir.bootlin.com/linux/v5.10.54/source/include/uapi/linux/ip.h#L86)表示, 它的内容决定了ipv4栈处理数据包的方式, 比如版本不是4(ipv4)或checksum不正确则数据包会被丢弃.