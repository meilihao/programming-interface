# FAQ

## module
### 编译plx驱动报`set_fs ... error: dereferencing pointer to incomplete type`
报错内容:
```
./arch/x86/include/asm/uaccess.h: In function 'set_fs':
./arch/x86/include/asm/uaccess.h:31:9: error: dereferencing pointer to incomplete type
```

原因: [set_fs原先是宏, 现在变成了内联函数, 需要将`#include <asm/uaccess.h>`替换为`#include <linux/uaccess.h>`](https://lkml.org/lkml/2017/9/30/189)

### 5.4.17编译kmod报`implicit declaration of function 'pci_enable_msi_exact'`
[`PCI/msi: remove pci_enable_msi_{exact,range}`](https://patchwork.kernel.org/project/linux-media/patch/1483994260-19797-4-git-send-email-hch@lst.de/)
[`spec-sw: kernel: spec-pci: use pci_alloc_irq_vectors with kernel > 4.11`](https://ohwr.org/project/spec-sw/commit/ed5270404a241da95ff054e56ad7606faf44b57a)

`pci_enable_msi_exact(pdev, 1);`换成`pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_MSI | PCI_IRQ_LEGACY);`

**遇到这类错误搜`xxx deprecated`即可**.

### 5.4.17编译kmod报`implicit declaration of function 'page_cache_release'`
[`page_cache_release() -> put_page()`](https://lkml.org/lkml/2016/3/20/141)

### 5.4.17编译kmod报`error too many arguments to function 'get_user_pages'`
[The prototype of get_user_pages() was changed by the Linux kernel developers in 4.4.168 and later.](https://forums.developer.nvidia.com/t/linux-4-4-172-nvidia-340-107-error-too-many-arguments-to-function-get-user-pages/70822)

```c
// https://bugs.gentoo.org/675310
/usr/src/linux-4.20.2-gentoo/include/linux/mm.h:
long get_user_pages(unsigned long start, unsigned long nr_pages,
                unsigned int gup_flags, struct page **pages,
                struct vm_area_struct **vmas);


---> LTS with special backport takes 7 arguments:

/usr/src/linux-4.4.170-gentoo/include/linux/mm.h:
long get_user_pages(struct task_struct *tsk, struct mm_struct *mm,
            unsigned long start, unsigned long nr_pages,
            unsigned int gup_flags, struct page **pages,
            struct vm_area_struct **vmas);
```

参考[zc.c](https://github.com/cryptodev-linux/cryptodev-linux/blob/master/zc.c)即可

### 5.4.17编译kmod报`implicit declaration of function 'pci_get_bus_and_slot'`
[`[30/30] PCI: remove pci_get_bus_and_slot() function`](https://patchwork.kernel.org/project/linux-arm-msm/patch/1511328675-21981-31-git-send-email-okaya@codeaurora.org/)