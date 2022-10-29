# cloud
ref:
- [**重识云原生**](https://blog.csdn.net/junbaozi/article/details/123101731)
- [【重识云原生】第2.5节——商用云主机方案](https://www.jianshu.com/p/3d89e59c982b)

## 裸金属
### 华为HCS裸金属
ref:
- [HCS裸金属服务介绍](https://www.huoban.com/news/post/713.html)

华为HCS裸金属服务基于开源社区OpenStack的Ironic组件能力，并通过华为自研[增强实现裸金属服务器](https://bbs.huaweicloud.com/blogs/380724)的发放功能.

裸金属服务通过PXE技术从服务器自动下载并加载操作系统，调用IPMI带外管理接口实现裸金属服务器的上电、下电、重启等操作，通过调用Nova组件的接口实现计算资源管理，调用Neutron组件的接口实现网络的发放和配置，调用Cinder组件的接口为裸金属服务器提供基于远端存储的云硬盘. 通过Cloud-init从metadata服务等数据源获取数据并对裸金属服务器进行配置，包括：主机名、用户名密码等。

华为HCS创建裸金属服务器时，调用Nova接口启动实例，调用时指定创建的规格、镜像等信息。Nova调度器通过规格匹配已注册的裸金属节点信息，选择合适的计算节点。Nova将调用信息传递到Ironic管理程序，Ironic管理程序通过IPMI接口启动裸金属服务器，裸金属服务器开始PXE MiniOS，MiniOS是个迷你型操作系统，运行时只加载到内存中。MiniOS加载后会运行Ironic Python Agent，简称IPA，IPA通过心跳与Ironic管理程序通讯，并根据选择的镜像从Glance下载镜像文件，写到裸金属服务器的硬盘上，然后从硬盘重启裸金属服务器，自动化完成客户操作系统的安装.