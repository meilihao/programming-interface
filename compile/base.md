# base

# 内存安全与效率
参考:
- [实例讲解代码之内存安全与效率](https://linux.cn/article-13845-1.html)

对于正在执行的进程process, 内存被划分为三个区域：栈(stack)、堆(heap)和静态区(static area).

> 内存的 静态区 为可执行代码（例如 C 语言函数）、字符串文字（例如“Hello, world!”）和全局变量提供存储空间. 该区域是静态的，因为它的大小从进程执行开始到结束都固定不变。由于静态区相当于进程固定大小的内存占用，因此经验法则是通过避免使用全局数组等方法来使该区域尽可能小。

作为通用 CPU 寄存器的替补，栈 为**代码块（例如函数或循环体）中的局部变量提供暂存器存储**.

```c
void some_func(int a, int b) {
   int n;
   ...
}
```

通过 a 和 b 传递的参数以及局部变量 n 的存储会在栈中，除非编译器可以找到通用寄存器。编译器倾向于优先将通用寄存器用作暂存器，因为 CPU 对这些寄存器的访问速度很快（一个时钟周期）。然而，这些寄存器在台式机、笔记本电脑和手持机器的标准架构上很少（大约 16 个）.

使用栈结构就意味着轻松高效地使用内存。编译器（而非程序员）会编写管理栈的代码，管理过程通过分配和释放所需的暂存器存储来实现；程序员声明函数参数和局部变量，将实现过程交给编译器。此外，完全相同的栈存储可以在连续的函数调用和代码块（如循环）中重复使用。精心设计的模块化代码会将栈存储作为暂存器的首选内存选项，同时优化编译器要尽可能使用通用寄存器而不是栈。

## 栈
在 C 语言中，在块（例如函数或循环体）内定义的任何变量默认都有一个 auto 存储类，这意味着该变量是基于栈的。存储类 register 现在已经过时了，因为 C 编译器会主动尝试尽可能使用 CPU 寄存器。只有在块内定义的变量可能是 register，如果没有可用的 CPU 寄存器，编译器会将其更改为 auto.

## 堆
堆 提供的存储是通过程序员代码显式分配的，堆分配的语法因语言而异. 在 C 中，成功调用库函数 malloc（或其变体 calloc 等）会分配指定数量的字节（在 C++ 和 Java 等语言中，new 运算符具有相同的用途）.

> malloc 函数不会初始化堆分配的存储空间，因此里面是随机值. 相比之下，其变体函数 calloc 会将分配的存储初始化为零. 这两个函数都返回 NULL 来表示分配失败.

编程语言在如何释放堆分配的存储方面有着巨大的差异：
- 在 Java、Go、Lisp 和 Python 等带gc语言中，程序员不会显式释放动态分配的堆存储.
- Rust 编译器会编写堆释放代码. 这是 Rust 在不依赖垃圾回收器的情况下，使堆释放实现自动化的开创性努力，但这也会带来运行时复杂性和开销.
- 在 C（和 C++）中，堆释放是程序员的任务. 程序员调用 malloc 分配堆存储，然后负责相应地调用库函数 free 来释放该存储空间（在 C++ 中，new 运算符分配堆存储，而 delete 和 delete[] 运算符释放此类存储）.C 语言避免了垃圾回收器的成本和复杂性，但也不过是让程序员承担了堆释放的任务.

内存泄漏会创建处于已分配状态但不可访问的堆块，从而会加速碎片化。因此，释放不再需要的堆存储是程序员帮助减少碎片整理需求的一种方式.

检查内存泄露:
```bash
# valgrind --leak-check=full ./leaky # 检查 leaky 程序是否存在内存泄漏
```

## 静态区
C 语言中，以 static 或 extern 修饰的函数和变量驻留在内存中所谓的 静态区 中，因为在程序执行期间该区域大小是固定不变的.

在所有块之外定义的函数或变量默认为 extern；因此，函数和变量要想存储类型为 static ，必须显式指定.

extern 和 static 的区别在于作用域：extern 修饰的函数或变量可以实现跨文件可见（需要声明）。相比之下，static 修饰的函数仅在 定义 该函数的文件中可见，而 static 修饰的变量仅在 定义 该变量的文件（或文件中的块）中可见.

任何 extern 的对象——无论函数或变量——必须 定义 在所有块(代码块)之外。此外，在所有块之外定义的变量默认为 extern.