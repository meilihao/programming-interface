# dbus
参考:
- [dbus概念](https://www.freedesktop.org/wiki/IntroductionToDBus/)

D-bus组成:
1. dbus-daemon，一个dbus的后台守护程序，用于多个应用之间消息的转发
1. libdbus.so，dbus的功能接口，当你的程序需要使用dbus时，其实就是调用libdbus.so里面的接口
1. 高层封装，如dbus-glib和QT D-bus，这些其实都对D-bus的再封装，让开发者使用起来更方便

从D-bus官网下载到源码，其实只包含上面所说的１和２两部分，libdbus.so里面的接口也就是官网说的low-level API.

## dbus数据流转
运行一个dbus-daemon就是创建了一条通信的总线Bus. 当一个application连接到这条Bus上面时，就产生了Connection.

每个application里面会有不同的Object. 这里Object的概念，可以简单地理解为C++里面一个类的实例. 从D-bus的概念上说，通信双方是Object，不是application，一个application是可以包含很多个Object的.

而一个Object里面又会有不同的Interface，这个Interface我把它理解为Object里面的一个类的成员. 这些Interface其实是通信方式的集合.

这里又牵扯出来一个通信方式，D-bus里面支持的通信方式有两种，一种叫signal，一种叫method. signal简单地讲，其实就是广播，就是一对多的通信方式，可以从app1向其他所有的app发消息，但其他的app是不会对signal进行回复的. method则是一对一的通信，一问一答. 这种方式有点像rpc，app1调用app2的method并传递参数给这个method，获取到这个method返回的结果.

dbus连接概念:
- address是用来标识dbus-daemon的. 当一个dbus-daemon运行以后，其他的app该怎么连接到这个dbus-daemon，靠的就是address. address的格式要求像这样：`unix:path=/var/run/dbus/system_bus_socket`
- bus name是用来标识application的. 当一个app1连接上dbus-daemon以后，相当于有了一个Connection，但其他的app2、app3怎么找到app1，靠的就是bus name. 这个bus name标识了app1的Connection，也就相当于标识了app1. bus name由两种，一种是以冒号开头的唯一标识，像`:34-907`这样；另一种是通用的标识，是方便人看的，像`com.mycompany.TextEditor`.
- path用于标识Object. 当app1的Object1要跟app2的Object2通信时，Object1要和Object2通信时，就要告诉dbus-daemon，Object2的path. path的格式像这样:`/com/mycompany/TextFileManager`，已“/”开头.
- 每个Interface都会有自己的名字，也就是interface name，我们通过这个interface name就可以找到这个interface. interface name像这样: `org.freedesktop.Hal.Manager`
- Signal和Method也有自己的名字，这个名字无特别的格式要求.