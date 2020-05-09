# makefile

## 规则
```makefile
target ... : prerequisites ...
    command
    ...
    ...
```
target也就是一个目标文件，可以是object file，也可以是执行文件, 还可以是一个标签（label）. prerequisites就是，要生成那个target所需要的文件或是目标. command也就是make需要执行的命令（任意的shell命令）. 这是一个文件的依赖关系，也就是说，target这一个或多个的目标文件依赖于prerequisites中的文件，其生成规则定义在 command中. 如果prerequisites中有一个以上的文件比target文件要新，那么command所定义的命令就会被执行. 这就是makefile的规则, 也就是makefile中最核心的内容.

## FAQ
### 了解make时执行了哪些命令
```bash
$ make "V="
```