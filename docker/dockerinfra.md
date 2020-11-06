# content

## 1. docker 架构
<img src="../image/docker-infra.jpg" alt="瀑布模式" width="400" >

如上图所示，Docker 是一个客户端-服务器（C/S）架构程序。Docker 客户端只需要向 Docker 服务器或者守护进程发出请求，服务器或者守护进程将完成所有工作并返回结果。我们日常使用的docker命令，其实就是docker客户端程序。
从图中可以看出，docker主要包含三个部分
- **docker 服务端**: 所有核心工作均在服务端完成，包括镜像处理，网络处理，存储处理，运行时工作等等。
- **docker 客户端**: docker就是个客户端，不仅可以链接本地docker daemon也可以链接其他主机的docker daemon。
- **镜像仓库** : docker damon在启动容器时会优先看本地是否包含镜像，若没有则需要去镜像仓库拉去，默认的镜像仓库为docker hub.也可以设置私有镜像仓库.

## 2. Docker 核心原理
Docker容器本质上是宿主机上的进程。Docker通过namespace实现了资源隔离，通过cgroups实现了资源限制，通过写时复制机制实现文件操作。

### 2.1 namespace 资源隔离
如果在Docker出现之前，被问起如何实现操作系统上的隔离，那么第一个会想到的是`chroot`。`chroot`可以直接修改根挂载点，让我们感觉到可以将资源限制在一个区域。但是这不是真的隔离，我们还想进一步隔离。比如网络隔离，就必须有独立的IP、端口、路由等；同时我们还想有自己的主机名；进程间通信也需要隔离，防止高权限攻击低权限进程；我们还有用户权限的隔离的问题；此外不同隔离之间的PID也应该都有自己的。
Linux内核提供了6种namespace隔离的系统调用。
|namespace|系统调用参数|隔离内容|
|--|---|---|
|UTS|CLONE_NEWUTS|主机名与域名|
|IPC|CLONE_NEWIPC|信号量、消息队列和共享内存|
|PID|CLONE_NEWPID|进程编号|
|Network|CLONE_NEWNET|网络设备、网络栈、端口|
|Mount|CLONE_NEWNS|挂载点（文件系统）|
|User|CLONE_NEWUSER|用户和用户组|

Linux内核实现namespace的主要目的，就是实现轻量级虚拟化服务。在同一个namespace下的进程可以感知彼此的变化，而对外界的进程一无所知。这样就可以让namespace里面的进程产生错觉，仿佛自己置身与一个独立的系统环境中，从而达到隔离的目的。

- UTS(UNIX Time-sharing System) namespace 提供了主机名和域名的隔离，这样每个Docker容器就可以拥有独立的主机名和域名了，在网络上可以被视为独立的节点，而非宿主机上的一个进程。使用docker创建出的容器，默认主机名为`container id`，也可以使用--hostname 参数指定主机名
```bash
[root@3-1-20-1-whq ch]# docker run -d -ti --hostname=myhost --name=demo ubuntu top
34e6219cec56ae3627e0258db7a9027223eda3232de9a54ecbe511226eede652
[root@3-1-20-1-whq ch]# 
[root@3-1-20-1-whq ch]# docker exec -ti demo hostname
myhost
```

- IPC(Inter-Process Communication) namespace,涉及IPC资源包括常见的信号量、消息队列和共享内存。申请了IPC资源就申请了一个全局唯一的32位ID，所以IPC namespace实际上包含了系统IPC标识符以及实现POSIX消息队列的文件系统。在同一个IPC namespace下的进程彼此可见，不同IPC namespace下进程互相不可见。Docker使用IPC namespace实现了容器与宿主机、容器与容器之间IPC隔离。
```bash
# 和宿主机使用同一个namespace
docker run -d --ipc=host data-client
# 可以使不同的容器使用同一个IPC namespace
docker run -d --ipc=container:data-server data-client
```

- PID namespace, 对进程PID重新标号，即两个不同namespace下的进程可以有相同的PID。每个PID namespace 都有自己的计数程序。内核为所有的PID namespace维护了一个树状结构，最顶层被称为root namespace