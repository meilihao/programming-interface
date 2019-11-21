# /etc/passwd
用户登录信息保存位置

格式(7个以冒号分隔的字段):

`登录名:加密后的登录密码(在[/etc/shadow](shadow.md)里):UID:GID:注释:用户home目录:用户登录shell`

比如:
```
...
chen:x:1000:1000::/home/chen:/usr/bin/fish
...
```

`password`部分被`x`替代表示密码实际存储在`/etc/shadow`.

`login shell`中的`/usr/sbin/nologin`和`/bin/false`:

- `/usr/sbin/nologin`,礼貌地拒绝登录(会显示一条提示信息),但可以使用其他服务,比如ftp.
- `/bin/false`,什么也不做只是返回一个错误状态,然后立即退出.即用户会无法登录,并且不会有任何提示,是最严格的禁止login选项，一切服务都不能用.

> 如果存在`/etc/nologin`文件,则系统**只允许root用户**登录,其他用户全部被拒绝登录,并向他们显示`/etc/nologin`文件的内容
> `/etc/shells`保存了支持的所有shell, 可使用`chsh`修改用户自身的shell.

## 扩展
### passwd与usermod锁用户及用户解锁的区别
passwd -l user 后shadow中user的密码前增加2个`!`
passwd -u user 将去除shadow中user密码前的2个`!`
usermod -L user 后shadow中user的密码前增加1个`!`
usermod -U user 将去除shadow中user密码前的1个`!`
