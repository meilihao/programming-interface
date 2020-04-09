# make
## makefile
make主要功能就是通过makeflie来实现的. 它定义了各种源文件间的依赖关系, 阐明了源文件如何进行编译.

linux下, 通常用Makefile代替makefile, 通过`configure`来生成. 在命令行执行make时, make默认会在当前目录查找Makefile. 如果使用其他文件作为Makefile则需要用`-f <makefile>`参数明确指明.