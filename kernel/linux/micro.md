# micro
对于具有同样限定符的函数(比如postcore_initcall), 调用顺序取决于编译顺序.

## `__cold`
将函数标记为较少使用, 此时编译器可以考虑其长度优化, 而不是速度优化

```c
// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/compiler_types.h#L104
#if !defined(CONFIG_CC_IS_GCC) || (CONFIG_FUNCTION_ALIGNMENT == 0)
#define __cold				__attribute__((__cold__))
#else
#define __cold
#endi
```

## `__init`
```c
// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/init.h#L52
#define __init		__section(".init.text") __cold  __latent_entropy __noinitretpoline
```

## `__section`
要求编译器将这段代码放在一个独立的section中, 其名称为`section`, 目的是在一个ELF节中集中存放所有的初始化代码, 以便在初始化结束后, 释放整个节

```c
// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/compiler_attributes.h#L334
#define __section(section)              __attribute__((__section__(section)))
```

## postcore_initcall
```c
// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/init.h#L302
#define postcore_initcall(fn)		__define_initcall(fn, 2)

// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/init.h#L282
#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)

// https://elixir.bootlin.com/linux/v6.6.14/source/include/linux/init.h#L279
#define ___define_initcall(fn, id, __sec)			\
	__unique_initcall(fn, id, __sec, __initcall_id(fn))
```

[`postcore_initcall(pcibus_class_init)`](https://elixir.bootlin.com/linux/v6.6.14/source/drivers/pci/probe.c#L108)是在`.initcall2.init`节定义一个函数指针, 具有唯一的名称, 指向放在`.init.text`节中的`pcibus_class_init`函数

linux初始化过程中调用的所有函数都是用类似的机制实现. 构造好初始化布局后, kernel会依次调用parse_early_param解析`.init.setup`节的内核参数, 调用do_pre_smp_initcalls执行`.initearly.init`节的早期初始化代码, 然后调用`do_initcalls`顺序执行`.initcall#.init`(包括initcallrootfs.init)节的初始化代码. 参考[内核初始化过程中的调用顺序](https://e-mailky.github.io/2016-10-14-linux_kernel_init_seq)