# acpi
ref:
- [从虚拟化看ACPI](https://cloud.tencent.com/developer/article/1087501)

高级配置与电源接口（Advanced Configuration and Power Interface），简称ACPI，1997年由Intel、Microsoft、Toshiba 所共同制定提供操作系统应用程序管理所有电源管理接口.

ACPI本质来说，是一个pci设备, 通过`cat /proc/ioports`, 可见CPU可以使用io指令访问对应的地址，就可以控制ACPI设备.