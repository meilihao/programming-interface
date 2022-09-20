# nvme
ref:
- [Rust Linux 驱动程序能够实现与 C 驱动程序相当的性能 - with pdf](https://www.oschina.net/news/210339/rust-linux-drivers)

controller: nvme的字符设备, 对应为/dev/nvmeX, 是我们下发控制命令的设备
namespace: nvme的块设备, 对应为/dev/nvmeXnX, 是我们下发IO的设备