# risc-v
参考:
- [riscv各种版本gcc工具链编译与安装](http://www.lujun.org.cn/?p=4257)
- [终于有人把RISC-V讲明白了](http://m.elecfans.com/article/653167.html)
- [教你在QEMU上搭建RISC-V Linux](https://zhuanlan.zhihu.com/p/574159681)
- [RISC-V OpenSBI 快速上手](https://tinylab.org/riscv-opensbi-quickstart/)

![基本指令集类型](https://suda-morris.github.io/blog/assets/img/riscv_instruction_type.6459e601.png)

## 指令集
RISC-V只支持小端格式(little-endian).

## 虚拟内存
ref:
- [玄铁C910用户手册.pdf](https://github.com/T-head-Semi/openc910/blob/main/doc/%E7%8E%84%E9%93%81C910%E7%94%A8%E6%88%B7%E6%89%8B%E5%86%8C.pdf)
- [玄铁C910微架构学习](https://zhuanlan.zhihu.com/p/456409077)
- [RISCV MMU 概述](https://blog.csdn.net/pwl999/article/details/123613069)
- [C910 datasheet](https://img.102.alibaba.com/1627958461165/49652c9412c41cb6f39b36fed1244e6e.pdf)
- [Xuantie_C906_R1S0_User_Manual.pdf](https://dl.linux-sunxi.org/D1/)
- [Add Sv57 page table support by Qinglin Pan](https://lore.kernel.org/linux-riscv/20220127024844.2413385-1-panqinglin2020@iscas.ac.cn/#r)
- [Linux on RISC-V](https://kernel-recipes.org/en/2022/wp-content/uploads/2022/06/fustini_riscv_kr2022-compresse.pdf)

riscv64支持Sv39/Sv48/Sv57/Sv64 这几种模式. 因为 C906 设计的应用场景不需要那么多的内存资源, 目前 C906/C910 只支持 Sv39 模式, 对应 3level mmu 映射.

## 寄存器
![](/misc/img/arch/Kazam_screenshot_00000.png)

## 产品
- [Milk-V Pioneer](https://news.mydrivers.com/1/903/903763.htm)

	[SOPHON SG2042是基于平头哥的玄铁IP](https://www.eefocus.com/article/1439667.html)
- [openEuler RISC-V 成功适配 LicheePi 4A 开发板，推动 RISC-V 生态发展](https://www.openeuler.org/zh/blog/20230506-riscv/20230506-riscv.html)

	LicheePi 4A 是首款性能对标树莓派 4 的 RISC-V 开发板，基于阿里巴巴平头哥 TH1520 芯片，搭载 4 核 2.0GHz C910 内核、4TOPS NPU 和 50GFLOPS GPU，为开发者提供强大的性能，满足各种应用场景需求.