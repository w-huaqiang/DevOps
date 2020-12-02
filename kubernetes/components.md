
## 1.整体架构
<img src="../image/k8s-architecture.png" alt="瀑布模式" width="400" >

kubernetes一共包含以下几个核心组件
- etcd 集群数据库，保存了整个集群的信息，包括资源配置，资源状态等
- apiserver 提供了资源的统一入口，并提供认证、授权、访问控制、API注册等机制
- controller manager 集群大脑，包含多种controller，如replicate controller, namespace controller等。该组件负责维护集群状态，比如规章检测，自动扩展，滚动升级等
- scheduler 负责集群资源调度，按照预定的调度策略将pod调度到相应的主机上
- kubelet work node上的agent，负责维护pod生命周期，同时也负责CVI、CNI的管理
- kube-proxy work node agent，负责Service提供内部负载均衡
- container runtime 容器运行时，并非k8s的组件，但是k8s必须依赖其工作。该组件可以选择不同种类，如docker、rkt、containerd

除了以上核心组件以外，若让一个集群更好服务，一般还包含以下推荐Add-ons组件

- coredns 为集群提供dns，服务发现等服务，一般为比选
- ingress controller 为服务资源提供外网入口，目前比较常用为nginx ingress、treafik ingress等
- metrics server 资源监控信息组件，若未安装此组件将无法使用`kubectl top`等命令，hpa资源也无法创建
- Dashboard 提供集群简单的GUI界面
- cert-manager 证书管理组件，集群内部可能有大量证书创建等需求，此组件可以大大减少证书颁发创建相关的工作量
- istio  servicemesh工具，此组件能让集群达到更高服务等级，比如具备蓝绿发布，AB测试等等。同时解决集群自身kube-proxy无法基于请求的负载均衡。
- linked2 servicemesh工具，功能和istio类似。
- helm k8s应用服务安装部署解决方案，将资源app抽象为chart，helm2需要安装tiller组件，helm3不需要安装
- prometheus 一种k8s解决方案，包含一个时许数据库的server,和监控端的exporter
- grafana 监控展示界面，可对接prometheus,influxDB等多种数据源

## 2. master node
<img src="../image/k8s-master.png" alt="瀑布模式" width="400" >

master node一般包含组件为:
- apiserver
- controller manager
- scheduler
- etcd

> 当然 etcd也可以放在集群之外，etcd若为集群方式为了防止脑裂,其节点数必须为奇数

## 3. work node
<img src="../image/k8s-node.png" alt="瀑布模式" width="400" >

work node组件一般包含组件为：
- kubelet
- kube-proxy
- docker（或者containerd，rkt等)