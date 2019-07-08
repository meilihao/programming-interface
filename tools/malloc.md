# malloc
在多线程环境使用tcmalloc和jemalloc效果非常明显.
当线程数量固定，不会频繁创建退出的时候， 可以使用jemalloc；反之使用tcmalloc可能是更好的选择.