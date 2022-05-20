# cloud
云计算的模型是以服务为导向的, 根据服务的层次不同, 可分为:
- IaaS：基础设施服务，Infrastructure-as-a-service

    用户掌控os, (虚拟)网络, (虚拟)硬件等, 比如aliyun ecs等.
- PaaS：平台服务，Platform-as-a-service

    云服务商提供给客户开发, 运维应用程序的运行环境, 用户负责维护自己的应用, 不用关系os, 网络及支撑它们的硬件等, 比如Google App Engine等.
- SaaS：软件服务，Software-as-a-service

    云服务商提供给客户可直接使用的软件服务, 比如Google Docs.

![云计算服务层次模型](/misc/img/os/cloud_level.jpeg)

![数据中心操作系统](/misc/img/os/1a8450f1fcda83b75c9ba301ebf9fbe5.jpg)

# 发行版
## openeuler
欧拉宣称，他们做到了五个统一：[统一内核、统一构建、统一 SDK、统一联接和统一开发工具](https://linux.cn/article-14512-1.html)。在社区开发过程中，欧拉把所有场景的所有组件的开发都归到了一个 openEuler 代码仓上，通过这种方式实现了任何场景都基于同一套代码。并且，欧拉操作系统通过 EulerMaker 项目提供了一套完整的构建和裁剪能力，这使得基于同一套代码构建的二进制，在面向不同的场景发布的时候，可以自如地进行构建和裁剪，最终形成适用于不同场景的镜像