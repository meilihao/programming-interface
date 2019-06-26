# compile

## 扩展
1. ld的库搜索顺序
    ```
    $ ld --verbose | grep SEARCH
    ```
1. 如何确定编译的目标target
    ```
    $ ./config.guess # 大部分需编译的软件都会携带该脚本, 比如gcc
    ```