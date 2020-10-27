# comment


## 1. 安装
> 1.7.3 为例，从1.6这个版本开始，istio将各个组建放到了一起，命名为`istiod`

istioctl是一个功能比较多的istio命令行工具,通过他可以安装配置集群中的istio

1. 下载
```bash
curl -L https://istio.io/downloadIstio | sh -
```

如果想下载指定版本和指定平台的istio可以指定环境变量,如:
```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.6.8 TARGET_ARCH=x86_64 sh -
```

> 也可以直接去github上找指定版本的istio下载，下载解压后会包含一个istioctl

2. 解压并配置
```bash
cd istio-1.7.3/bin && cp istioctl /usr/bin
```
3. 检查命令

```bash
istioctl x precheck
```

4. 执行安装
```bash
istioctl install --set profile=demo
```

profile对应安装的组建

||default|demo|minimal|remote|
|---|--|--|--|--|
|**Core components**|
|istio-egressgateway||x|||
|istio-ingressgateway|x|x|||
|istiod|x|x|x||


检查各个组建pod是否均已启动
```bash
[root@node1 samples]# kubectl get pod -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-egressgateway-66f8f6d69c-dm5zr    1/1     Running   0          4m58s
istio-ingressgateway-758d8b79bd-9kcv5   1/1     Running   0          4m57s
istiod-7556f7fddf-8f5pf                 1/1     Running   0          5m5s
```

> 注意：非云环境下，没有`loadbalance`，所以如果是数据中心安装的，需要将ingress改成`nodeport`
> ```bash
> [root@node1 samples]# kubectl get svc -n istio-system
> NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
>istio-egressgateway    ClusterIP      10.233.12.105   <none>        80/TCP,443/TCP,15443/TCP                                                     7m18s
>istio-ingressgateway   LoadBalancer   10.233.10.234   <pending>     15021:31345/TCP,80:31339/TCP,443:32046/TCP,31400:31368/TCP,15443:31145/TCP   7m17s
>istiod                 ClusterIP      10.233.0.31     <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP,853/TCP                                7m26s
>[root@node1 samples]# kubectl edit svc/istio-ingressgateway -n istio-system``
> # 将type=LoadBalance 改成type=NodePort
> 

5. 安装相关第三方组建

```bash
 kubectl apply -f samples/addons
```
> 如果有错误，可以再运行一遍命令
> 