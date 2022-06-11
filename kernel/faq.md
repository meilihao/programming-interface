# FAQ

## module
### 编译plx驱动报`set_fs ... error: dereferencing pointer to incomplete type`
报错内容:
```
./arch/x86/include/asm/uaccess.h: In function 'set_fs':
./arch/x86/include/asm/uaccess.h:31:9: error: dereferencing pointer to incomplete type
```

原因: [set_fs原先是宏, 现在变成了内联函数, 需要将`#include <asm/uaccess.h>`替换为`#include <linux/uaccess.h>`](https://lkml.org/lkml/2017/9/30/189)