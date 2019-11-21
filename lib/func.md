# func
## FAQ
### `err_exit()`, `errExit()`, `errExitEN()`
err_exit()终止进程使用的是_exit(),而非 exit(). 这一退出方式,略去了对 stdio 缓冲区
的刷新以及对退出处理程序(exit handler)的调用, 因此`err_exit()`尤其适合在子进程里使用.

errExitEN()函数与 errExit()大体相同,区别仅仅在于: errExit()打印与当前
errno 值相对应的错误文本不同,而errExitEN()只会打印与 errnum 参数中给定的错误号.