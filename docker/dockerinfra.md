# content

## 1. docker 架构
<img src="../image/docker-infra.jpg" alt="瀑布模式" width="400" >

如上图所示，Docker 是一个客户端-服务器（C/S）架构程序。Docker 客户端只需要向 Docker 服务器或者守护进程发出请求，服务器或者守护进程将完成所有工作并返回结果。我们日常使用的docker命令，其实就是docker客户端程序。
从图中可以看出，docker主要包含三个部分
- **docker 服务端**: 所有核心工作均在服务端完成，包括镜像处理，网络处理，存储处理，运行时工作等等。
- **docker 客户端**: docker就是个客户端，不仅可以链接本地docker daemon也可以链接其他主机的docker daemon。
- **镜像仓库** : docker damon在启动容器时会优先看本地是否包含镜像，若没有则需要去镜像仓库拉去，默认的镜像仓库为docker hub.也可以设置私有镜像仓库.

## 2. 什么是容器
