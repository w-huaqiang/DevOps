
## 1. 声明式API

ApiServer是k8s的入口，对集群的任何操作都是和集群API交互，k8s中API是声明式的，即均为对API 资源的目标描述，然后由controller来达到我们描述的资源状态。
所以声明式API + 控制器模式是k8s设计的核心。

理解声明式API可以从声明式编程开始，声明式编程是一种编程范式，与命令式编程对立，它描述目标性质，让计算机明白目标，而非流程。声明式编程不用告诉计算机问题领域。而命令式编程需要让算法来明确指导下一步该怎么做。


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
  creationTimestamp: "2020-12-01T04:04:22Z"
  generation: 6
  labels:
    app: my-nginx
  managedFields:
  ...
spec:
  paused: true
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: my-nginx
  strategy:
  ...
status:
  availableReplicas: 2
```

如上例可以看到资源总是包含五个部分
- apiVersion api的组和版本
- kind 资源类型
- metadata 元数据信息
- spec 资源详细描述
- status 资源当前状态信息

## 2. 资源

对k8s的使用，其实就是对资源的描述，所以当我们理解了k8s包含了哪些内部resource(资源)，如何定义这些resource，那么就学会了使用k8s。而对资源的描述，永远都是apiVersion、kind、metadata、spec、status五项，其中status一般不需要人工维护。

当前k8s的资源包含

- 集群资源
  - **node** 集群的work node节点
  - **namespace** 可以将集群划分为多个逻辑的隔离区域，一般使用namespace标记

- 计算资源
  - **pod** k8s计算资源最小调度单位，类似container，但是可以包含多个container
  - **deployment** 计算资源，编排pod，可以产生多个相同的pod副本
  - **statefulSet** 计算资源，编排pod,可以产生多个不同的pod副本，一般用于有状态应用集群
  - **job/cronjob** 计算资源, 用来编排pod，一般做批处理，或者临时处理使用
  - **daemonSet** 计算资源，编排pod，会在每个work node上启动，一般用于安装agent

- 存储资源
  - **PersistentVolume** 存储卷定义,一般管理员创建
  - **PersistentVolumeClaim** 存储卷描述，每个pvc对应一个pv，可以挂给pod使用
  - **StoraceClass** 存储类，可以用来动态分配pv，使用csi/flexVolume两种方式管理，需要格外的provider管理

- 配置资源
  - **configmap** 配置资源定义，一般存储配置文件，但是大小不能超过1M
  - **secret** 加密配置定义，一般存储用户名密码、证书等

- service资源
  - **service/svc** 服务资源定义，一般对应一组计算资源，暴露一个服务IP
  - **ingress** 对外暴露借口，指定访问的url和对应的svc