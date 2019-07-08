# malloc
参考:
- [ptmalloc,tcmalloc和jemalloc内存分配策略研究](https://cloud.tencent.com/developer/article/1173720)
- [内存优化总结:ptmalloc、tcmalloc和jemalloc](http://www.cnhalo.net/2016/06/13/memory-optimize/)

在多线程环境使用tcmalloc和jemalloc效果非常明显.

当线程数量固定，不会频繁创建退出的时候， 可以使用jemalloc；反之使用tcmalloc可能是更好的选择.