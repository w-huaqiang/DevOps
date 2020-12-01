
## 1. 容器探针
kubelet调用的容器探针包含三种Handler
- ExecAction: 在容器内执行命令。如果命令推出时返回码为0则认为诊断成功
- TCPSocketAction: 对容器的IP地址上指定端口执行TCP检查。如果端口打开，则认为诊断成功
- HTTPGetAction: 对容器IP地址上指定端口和路径执行HTTP Get请求。如果响应的状态码大于等于200且小于400，则认为诊断成功

每次探测结果都将获得三种结果：
- Success: 通过
- Failure: 未通过
- Unknown: 诊断执行失败，不做任何操作

## 2. readinessProbe

该类型指容器是否准备好为请求提供服务。如果探测未通过，endpoint controller将会从pod匹配的所在endpois 列表中删除该pod IP。初始延迟之前的就绪状态默认值为Failure。如果容器不提供readiness，则默认状态为Success。

```yaml
# ExecAction
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5

# TCPSocketAction
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

#HTTPGetAction
livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 30
  periodSeconds: 10
``` 
1. initialDelaySeconds: 在init阶段不执行探测操作
2. periodSeconds: 两次执行探测的时间间隔
3. failureThreshold: 探测失败重试次数

> 如果readinessprobe 探测一直不成功，pod将不会被重启，除非设置了livenessprobe


## 3. livenessProbe

该类型指容器是否在正常运行。如果liveness探测失败，则kubelet会杀死容器，并且容器将更具重启策略来决定下一步。如果容器不提供livenessprobe，则默认状态为sucess。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

探针使用和readness基本相同。
> 如果一个pod 程序启动时间很久，而设置了livenessProbe，会不会造成无限重启的循环之中？
> 答案是**肯定的**，如果设置了liveness，pod启动的时间最长为: initialDelaySeconds + failureThreshold × periodSeconds。如果超过了这个时间kubelet将会杀死容器，要是pod程序启动较慢很有可能陷入无限重启的循环中。为了解决这个问题Kubernetes v1.18 stable了一个启动探测器(startupProbe)

## 4. startupProbe

该类型指容器是否已经启动。如果设置了启动探测器，则其他探测器都会被禁用，直到此探测器成功为止。如果启动探测器失败，kubelet将杀死容器。如果容器没有提供启动探测，则默认状态为success。

```yaml
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 1
  periodSeconds: 10

startupProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 30
  periodSeconds: 10
```
> 这里，应用程序将会有最多 5 分钟(30 * 10 = 300s) 的时间来完成它的启动。 一旦启动探测成功一次，存活探测任务就会接管对容器的探测，对容器死锁可以快速响应。 如果启动探测一直没有成功，容器会在 300 秒后被杀死，并且根据 restartPolicy 来设置 Pod 状态。



Probe 有很多配置字段，可以使用这些字段精确的控制存活和就绪检测的行为：

initialDelaySeconds：容器启动后要等待多少秒后存活和就绪探测器才被初始化，默认是 0 秒，最小值是 0。
periodSeconds：执行探测的时间间隔（单位是秒）。默认是 10 秒。最小值是 1。
timeoutSeconds：探测的超时后等待多少秒。默认值是 1 秒。最小值是 1。
successThreshold：探测器在失败后，被视为成功的最小连续成功数。默认值是 1。 存活和启动探测的这个值必须是 1。最小值是 1。
failureThreshold：当探测失败时，Kubernetes 的重试次数。 存活探测情况下的放弃就意味着重新启动容器。 就绪探测情况下的放弃 Pod 会被打上未就绪的标签。默认值是 3。最小值是 1。
HTTP Probes 可以在 httpGet 上配置额外的字段：

host：连接使用的主机名，默认是 Pod 的 IP。也可以在 HTTP 头中设置 “Host” 来代替。
scheme ：用于设置连接主机的方式（HTTP 还是 HTTPS）。默认是 HTTP。
path：访问 HTTP 服务的路径。
httpHeaders：请求中自定义的 HTTP 头。HTTP 头字段允许重复。
port：访问容器的端口号或者端口名。如果数字必须在 1 ～ 65535 之间。