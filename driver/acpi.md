# acpi
ref:
- [ACPI Specification](https://uefi.org/specifications)
- [从虚拟化看ACPI](https://cloud.tencent.com/developer/article/1087501)
- [Linux启动时如何定位BIOS提供的ACPI表](https://blog.csdn.net/lindahui2008/article/details/81813845)

高级配置与电源接口（Advanced Configuration and Power Interface），简称ACPI，1997年由Intel、Microsoft、Toshiba 所共同制定提供操作系统应用程序管理所有电源管理接口.

ACPI本质来说，是一个pci设备, 通过`cat /proc/ioports`, 可见CPU可以使用io指令访问对应的地址，就可以控制ACPI设备.

整个ACPI表以RSDP（Root System Descriptor Pointer Table，根系统描述符指针表）为入口点，存放了R(X)DST的地址, 每个非叶子节点都会包含指向其他子表的指针，各个表都会有一个表头，在该表头中包含了相应的Signature，用于标识该表，有点类似与该表的ID，除此之外，在表头中还会包含Checksum、Revision、OEM ID等信息。所以查找ACPI表的关键就是在内存中定位到RSDP表. RSDT（根系统说明表）是32位地址，XSDT是64位地址，其功能一样.

对于基于Legacy BIOS的系统而言，RSDP表所在的物理地址并不固定，要么位于EBDA（Extended BIOS Data Area）（位于物理地址0x40E）的前1KB范围内；要么位于0x000E0000 到0x000FFFFF的物理地址范围内. Linux kernel在启动的时候，会去这两个物理地址范围，通过遍历物理地址空间的方法寻找RSDP表，即通过寻找RSDP表的Signature（RSD PTR）来定位RSDP的位置，并通过该表的length和checksum来确保找到的表是正确的.

在内核启动的时候会调用acpi_boot_table_init()函数，该函数会调用到acpi_find_root_pointer()函数用于定位RSDP表，其函数的调用顺序如下：
arch/x86/kernel/head64.S -> arch/x86/kernel/head64.c:x86_64_start_kernel() -> arch/x86/kernel/head64.c:x86_64_start_reservations -> init/main.c:start_kernel() -> arch/x86/kernel/setup.c:setup_arch() -> acpi_boot_table_init() -> acpi_table_init() -> acpi_initialize_tables() -> acpi_os_get_root_pointer() -> acpi_find_root_pointer()

对于基于UEFI的系统而言，RSDP Table位于EFI_SYSTEM_TABLE里的EFI Configuration Table，所以Linux内核可以直接到该表里面查找RSDP表(Search RSDP的GUID)，然后定位所有的ACPI表.

在UEFI系统下， 在EFI System Table

注意XDST第一个Entry一定是FADT (Fixed ACPI Description Table，固定ACPI描述表) ，主要放了一些硬件信息和DSDT的地址，其Signature是“FACP”.

> XSDT：Extended System Description Table，ACPI3.0中取代RSDP，指向了FADT表

> acpi tables 可见 `/sys/firmware/acpi/tables`

> seabios 1.16.2也仅支持acpi 1.0