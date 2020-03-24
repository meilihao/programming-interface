# gcc
查看扩展后的header:`printf "#include <sys/syscall.h>\nSYS_read" | gcc -E -`; 查看其架构对应32位的扩展结果:`printf "#include <sys/syscall.h>\nSYS_read" | gcc -m32 -E -`.

## 扩展

### weak alias
指定函数的weak属性. 编译glibc时将"函数别名"属性保存在weak symbol中, 这种用法可以在其他地方定义真实的函数. 编译器会自动识别出哪个是真实的定义.

weak_alias(name1,name2) : 为标号name1定义一个弱化名name2. 仅当name2没有在任何地方定义时，连接器就会用name1解析name2相关的符号.