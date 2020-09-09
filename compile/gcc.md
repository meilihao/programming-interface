# gcc
gcc(gnu compiler collection)一个编译器, 是整个gnu工程的一个组成部分, 其他部分还有glibc, gdb, binutils(二进制工具链).

binutils(二进制工具链`sudo dpkg -L binutils|xargs ls -l | grep "/usr/bin"`):
- as : 汇编器
- ld : 连接器
- objdump : object分析工具
- strip : 去除ojb中的符号信息
- strings : 显示文件中的可打印字符串
- size : 显示obj或归档文件中的section的大小
- objcopy : 实现obj格式的转换
- nm : 显示obj中的符号信息
- ar : 归档文件(archives)的打包工具, 用来生成静态或动态链接库
- readefl : 显示elf格式obj文件的信息 

查看扩展后的header:`printf "#include <sys/syscall.h>\nSYS_read" | gcc -E -`; 查看其架构对应32位的扩展结果:`printf "#include <sys/syscall.h>\nSYS_read" | gcc -m32 -E -`.

## 源码
${gcc_source}/gcc/config下包含了gcc对目标处理器的支持情况.

### tools
```
# cat << EOF > test.c
int global_int = 0;
int main(int argc, char *argv[]){
    int i;
    static int static_sum = 0;
    int array[10]={0, 1, 2, 3, 4, 5, 6, 7, 8, 9};

    for(i=global_int; i<10;i++){
        int j=i*2;
        static_sum=static_sum+j+array[i];
        if (static_sum>1000){
            goto Label_RET;
        }
    }

    Label_RET:
        return static_sum;      
}
EOF
# gcc -fdump-tree-all -fdump-rtl-all test.c # 输出gcc处理过程中的主要调试信息与工作流程
# gcc -fdump-tree-cfg-all test.c # 获取更详细的cfg信息
# gcc -fdump-tree-cfg-graph  test.c # 仅生成cfg dot
# gcc -fdump-tree-all-graph  test.c # 生成所有相关的dot
# gcc -fdump-tree-original-graph  test.c
# gcc -fdump-tree-original-raw  test.c # ast dump
# gcc -fdump-tree-original-raw-graph  test.c # 同上区别???
# gcc -fdump-tree-gimple  test.c # gimple dump
# gcc -fdump-tree-cfg  test.c # cfg dump
# gcc -fdump-rtl-expand  test.c # rtl dump
# gcc -S test.c || objdump -d a.out # assembly dumps
# dot -Tpng test.c.012t.cfg.dot -o test.c.012t.cfg.png # need `apt install graphviz`, 将文本格式描述的图转成真正的图. 文件名称中的cfg是指Control Flow Graph.
# ll
...
# 各种调试文件格式为: test.c.xxx[t/r].name, xxx为一个编号, t表示该处理过程是基于tree的GIMPLE处理过程, r表示该处理过程是基于RTL的处理过程, 
-rw-rw-r-- 1 chen chen 3.1K 9月   6 12:49 test.c.231t.optimized.dot
-rw-rw-r-- 1 chen chen  11K 9月   6 15:46 test.c.233r.expand
...
```

## 扩展

### weak alias
指定函数的weak属性. 编译glibc时将"函数别名"属性保存在weak symbol中, 这种用法可以在其他地方定义真实的函数. 编译器会自动识别出哪个是真实的定义.

weak_alias(name1,name2) : 为标号name1定义一个弱化名name2. 仅当name2没有在任何地方定义时，连接器就会用name1解析name2相关的符号.