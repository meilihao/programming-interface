# seabios
ref:
- [SeaBIOS实现简单分析](https://www.cnblogs.com/gnuemacs/p/14287120.html)
- [Advanced	x86: BIOS and System Management	Mode Internals Reset Vector](https://opensecuritytraining.info/IntroBIOS_files/Day1_XX_Advanced%20x86%20-%20BIOS%20and%20SMM%20Internals%20-%20Reset%20Vector.pdf)

## 源码
BIOS是放在EEPROM中的, x86 PC上电的时候，BIOS被映射在1M内存的末端:
- 如果BIOS大小是64k，则其地址空间为0xF0000-0xFFFFF
- 如果BIOS大小是128k，则其地址空间为0xE0000-0xFFFFF

还有个是VGABIOS, 显示芯片驱动. 大小一般不超过64K, 映射在0xC0000-0xD0000区间.

当前bios是128k, 根据bios在内存的位置反汇编bios: `ndisasm -o 0xe0000 bios.bin > bios.asm`

SeaBIOS里的第一条指令在romlayout.S的reset_vector中, 是`jmpf 0xf000:e05b`.