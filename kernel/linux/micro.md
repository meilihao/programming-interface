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

## [list_head](https://elixir.bootlin.com/linux/v6.6.16/source/include/linux/list.h)
```c
struct list_head {
	struct list_head *next, *prev;
};

#define LIST_HEAD_INIT(name) { &(name), &(name) } // 初始化list

#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name) // 定义并初始化一个名为name的链表


static inline void list_add(struct list_head *new, struct list_head *head) //在head后插入new
static inline void list_add_tail(struct list_head *new, struct list_head *head)//head前插入new, 由于是循环链表，就是说添加到链表尾部
static inline void list_del(struct list_head *entry) // 删除entry
static inline void list_del_init(struct list_head *entry)//删除节点，并初始化被删除的结点（也就是使被删除的结点的prev和next都指向自己）
static inline int list_empty(const struct list_head *head)//判断链表是否为空
static inline int list_is_last(const struct list_head *list, const struct list_head *head) // list是否为最后一个元素
static inline void list_splice(struct list_head *list, struct list_head *head)//通过两个链表的head，进行连接  
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)    //通过偏移值取type类型结构体的首地址. ptr是list_head的地址, type是目标结构体名, member是目标结构体中list_head对应的字段名
#define list_first_entry(ptr, type, member) \
	list_entry((ptr)->next, type, member) // 获得包含下一个list_head节点的type对象
#define list_for_each(pos, head)                   //遍历链表，循环内不可调用list_del()删除节点
#define list_for_each_entry(pos, head, member)				\
	for (pos = list_first_entry(head, typeof(*pos), member);	\
	     !list_entry_is_head(pos, head, member);			\
	     pos = list_next_entry(pos, member)) // 遍历每个type对象. pos是输出, 表示当前循环中对象的地址, type实际由typeof(*pos)实现
#define list_for_each_entry_safe(pos, n, head, member)			\
	for (pos = list_first_entry(head, typeof(*pos), member),	\
		n = list_next_entry(pos, member);			\
	     !list_entry_is_head(pos, head, member); 			\
	     pos = n, n = list_next_entry(n, member)) // n起到临时存储的作用, 其他同上, 区别在于, 如果循环中删除了当前元素, xxx_safe是安全的
```

list_del删除元素后会重置它的prev和next, 因此不带`_safe`后缀的操作在获得元素后不能删除该元素. 有`_safe`的操作是使用n临时复制当前元素的链表关系, 使用n继续遍历, 所以不存在问题.

list_head有一个类似的hlist_head/hlist_node. hlist_head比list_head少一半空间即表头空间省一半.

> 红黑树见btree.h; 基数树见radix-tree.h

## [bitmap](https://elixir.bootlin.com/linux/v6.6.16/source/include/asm-generic/bitops/instrumented-atomic.h#L16)
ref:
- [Bit Operations](https://www.kernel.org/doc/html/latest/core-api/kernel-api.html?highlight=set_bit#bit-operations)

每个arch定义了自己的[bitops.h](https://elixir.bootlin.com/linux/v6.6.16/A/ident/set_bit), 在`arch/<arch>/include/asm/bitops.h`, 没直接在源码里看到x86的, 应该是构建时生成的.
