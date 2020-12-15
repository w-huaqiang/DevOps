## comment
- [comment](#comment)
- [1. Deployment](#1-deployment)
  - [1.1 replicaSet](#11-replicaset)
  - [1.2 Pod](#12-pod)
- [2. 更新Deployment](#2-更新deployment)

## 1. Deployment

deployment定义一个应用声明，来描述这个应用当前处于什么版本，几个副本，如何升级等信息。其主要功能为：
- 定义pod template版本(通过replicaSet)
- 定义期望的副本数量
- 执行滚动升级
- 应用的扩容与缩容
- 暂停和resume应用的变化

比较典型的用例包括:
1. 创建 Deployment 以将 ReplicaSet 上线。 ReplicaSet 在后台创建 Pods。 检查 ReplicaSet 的上线状态，查看其是否成功。
2. 通过更新 Deployment 的 PodTemplateSpec，声明 Pod 的新状态 。 新的 ReplicaSet 会被创建，Deployment 以受控速率将 Pod 从旧 ReplicaSet 迁移到新 ReplicaSet。 每个新的 ReplicaSet 都会更新 Deployment 的修订版本。
3. 如果 Deployment 的当前状态不稳定，回滚到较早的 Deployment 版本。 每次回滚都会更新 Deployment 的修订版本。
4. 扩大 Deployment 规模以承担更多负载。
5. 暂停 Deployment 以应用对 PodTemplateSpec 所作的多项修改， 然后恢复其执行以启动新的上线版本。
6. 使用 Deployment 状态 来判定上线过程是否出现停滞。
7. 清理较旧的不再需要的 ReplicaSet 。

deployment定义如下

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80


```
`replicas` 定义期望的pod数量
`selector.matchLabels`应与`template.metadata.labels`一致，否则创建不成功

`template` 字段包含以下子字段：
Pod 被使用 labels 字段打上 `app: nginx` 标签。
Pod 模板指示 Pods 运行一个 nginx 容器， 该容器运行版本为 1.14.2 的 nginx Docker Hub镜像。并使用 name 字段将其命名为 `nginx`。

创建
```bash
kubectl apply -f deployment.yaml --record
```
> 使用record可以将执行命令写入deployment注解`kubernetes.io/change-cause`中。

查看当前创建基本信息
```bash
[root@node1 crd]# kubectl get deploy/nginx-deployment -o wide
NAME               READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES         SELECTOR
nginx-deployment   1/3     3            1           3m40s   nginx        nginx:1.14.2   app=nginx
```
`NAME` 列出了集群中 Deployment 的名称。
`READY` 显示应用程序的可用的 副本 数。显示的模式是“就绪个数/期望个数”。
`UP-TO-DATE` 显示为了打到期望状态已经更新的副本数。
`AVAILABLE` 显示应用可供用户使用的副本数。
`AGE `显示应用程序运行的时间。
`CONTAINERS` pod中container的名字
`IMAGES` container使用的image
`SELECTOR` 显示标签选择器

查看当前deployment正在上线的状态
```bash
[root@node1 crd]# kubectl rollout status deploy/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 of 3 updated replicas are available...
Waiting for deployment "nginx-deployment" rollout to finish: 2 of 3 updated replicas are available...

```

### 1.1 replicaSet

deployment控制pod副本是通过replicaSet实现的，而不是直接编排pod。
查看当前namespace下的replicaSet
```bash
[root@node1 crd]# kubectl get rs|grep nginx-deployment
nginx-deployment-6b474476c4                 3         3         2       10m
```

`nginx-deployment-6b474476c4`的命名规则为
`deployment名称 - pod template的HASH`

查看该replicaSet可以看到Ownerreference是deployment nginx-deployment
```bash
[root@node1 crd]# kubectl get rs/nginx-deployment-6b474476c4  -o custom-columns=OWNER:.metadata.ownerReferences
OWNER
[map[apiVersion:apps/v1 blockOwnerDeletion:true controller:true kind:Deployment name:nginx-deployment  uid:0ffc30e1-cec6-4966-bd60-57547236e4df]]
```

### 1.2 Pod
```bash
[root@node1 crd]# kubectl get pod|grep nginx-deployment
nginx-deployment-6b474476c4-g67xp   2/2     Running   0          17m
nginx-deployment-6b474476c4-mqbt7   2/2     Running   0          17m
nginx-deployment-6b474476c4-tskjr   2/2     Running   0          17m
```
`pod`的名称规则为`replicaSet name` - 随机字符

> 注意：此处READ 2/2是我这个ns里面有istio auto inject

## 2. 更新Deployment

在k8s中，应用的更新一般就是其对应容器镜像的更新，可以直接对deployment设置deployment的更新

```bash
[root@node1 crd]# kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record
deployment.apps/nginx-deployment image updated
```
查看更新状态
```bash
[root@node1 crd]# kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
```

此时我们可以看到更新的版本中有两条信息
```bash
[root@node1 crd]# kubectl rollout history deployment/nginx-deployment
deployment.apps/nginx-deployment 
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=deploy.yaml --record=true
2         kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record=true
```