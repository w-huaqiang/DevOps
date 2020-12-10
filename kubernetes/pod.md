# comment
- [comment](#comment)
  - [1. 什么是pod](#1-什么是pod)
    - [1.1 init container](#11-init-container)
    - [1.2 sidcer container](#12-sidcer-container)
  - [2. pod使用](#2-pod使用)
    - [2.1 静态pod](#21-静态pod)

## 1. 什么是pod

> 每当我们去了解一个新的东西的时候，总喜欢给它下一个最初印象的定义，然后随着对它不断的了解来更新这种定义
> 所以开始我们可以理解: pod就是一个虚拟机。

Pod是kubernetes创建和部署的最小单元。一个pod封装一组运行环境（和docker中讲到的容器类似)，这些环境包括内存，CPU，存储资源，网络资源。

Pod对运行环境的封装是通过容器运行时来完成的。kubernetes支持很多容器运行时，docker是最著名的。

Pod共享一组Linux名字空间、控制组（cgroups）和一些其他的隔离。其本质也是通过容器的技术实现，也就是说pod是一组容器（一个或者多个容器）。

Pod 被设计成支持形成内聚服务单元的多个协作过程（形式为容器）。 Pod 中的容器被自动安排到集群中的同一物理机或虚拟机上，并可以一起进行调度。 容器之间可以共享资源和依赖、彼此通信、协调何时以及何种方式终止自身。

例如，你可能有一个容器，为共享卷中的文件提供 Web 服务器支持，以及一个单独的 “sidecar（挂斗）”容器负责从远端更新这些文件，如下图所示：

<img src="../image/pod.svg" alt="瀑布模式" width="400" >

pod的编排文件如：


### 1.1 init container
有时候当我们启用一个pod的时候，总喜欢让其完成某些行为之后再启动我们的应用，比如pod启动之前完成相关目录创建，某些文件的下载等，这个时候可以考虑使用init container。

在pod的资源文件中，当我们设置init container，那么在pod启动之前会进入initing阶段，在initing阶段会执行init container指定的操作，当init container完成之后再启动pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: lumpy-koala
  name: lumpy-koala
spec:
  initContainers:
  - image: busybox
    name: install
    command:
    - touch
    - "/workdir/calm.txt"
    volumeMounts:
    - name: workdir
      mountPath: /workdir
  containers:
  - image: nginx
    name: lumpy-koala
    command:
    - sh
    - -c
    - "[ -f /workdir/calm.txt ] && sleep 36000 || exit"
    volumeMounts:
    - name: workdir
      mountPath: /workdir
  volumes:
  - name: workdir
    emptyDir: {}
```
以上pod为在启动pod之前init container先创建`/workdir/calm.txt`文件。

### 1.2 sidcer container
sidcar 模式其实就是上文提到的多容器的pod的一种应用方式，比如我们已经构建好了一个container用于启动一个服务，但是后来我们又想监控这个服务的某些状态，做法可能有两种方式
1. 更新这个container，将监控信息加入进去
2. 服务的container不变，在pod层面加入一个监控的container

采用第一种方式将应用的逻辑和运维的逻辑放在一起容器造成镜像的臃肿，不利于解耦，比如监控需求的不断改变可能会影响到业务。那么完全可以采用第二种方式将运维层面的逻辑单独处理，然后用一个pod来处理

目前比较流行的云原生“service mesh“处理模式都是第二种方式.

## 2. pod使用

k8s是一套容器编排平台，所以其核心在编排。我们很少会直接使用pod。因为pod是一个灵活的运行实体，其生命周期比较短暂，而且出现故障后不会自愈，没法方便的实现扩容缩容。

比如,如果Pod运行的Node故障，或者是调度器本身故障，这个Pod就会被删除。同样的，如果Pod所在Node缺少资源或者Pod处于维护状态，Pod也会被驱逐。

kubernetes会使用更高层级的抽象资源来编排pod，使我们只需关注应用角度的最终状态，通过定义高层次的抽象资源的template来指定要创建什么样的pod。

主要的编排资源有
- deployment
- statefulset
- daemonset
- cron job/job

### 2.1 静态pod
静态 Pod（Static Pod） 直接由特定节点上的 kubelet 守护进程管理， 不需要API 服务器看到它们。 尽管大多数 Pod 都是通过控制面（例如，Deployment） 来管理的，对于静态 Pod 而言，kubelet 直接监控每个 Pod，并在其失效时重启之。

kubelet 自动尝试为每个静态 Pod 在 Kubernetes API 服务器上创建一个 镜像 Pod。 这意味着在节点上运行的 Pod 在 API 服务器上是可见的，但不可以通过 API 服务器来控制。
