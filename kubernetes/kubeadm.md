# comment
- [comment](#comment)
    - [Before you begin](#before-you-begin)
    - [ubuntu](#ubuntu)
    - [kubeadm](#kubeadm)
      - [install](#install)
      - [init](#init)
      - [add-ons](#add-ons)
    - [TLS-bootstrap](#tls-bootstrap)
      - [API Server](#api-server)
      - [kube-controller-manager](#kube-controller-manager)
      - [RBAC 资源](#rbac-资源)
      - [创建随机bootstrap token](#创建随机bootstrap-token)
      - [创建cluster-info](#创建cluster-info)
      - [创建bootstrap token](#创建bootstrap-token)
      - [work 节点](#work-节点)

### Before you begin
- 主机要求
   - Ubuntu 16.04+
   - CentOS 7
   - RHEL 7
   - Fedora 25+

- 大于2GB内存/每个主机
- 大于2CPU/每个主机
- 每台主机完全的网络链接
- 唯一的主机名，MAC地址，product_uuid
```bash
# product_uuid
sudo cat /sys/class/dmi/id/product_uuid

#MAC
ifconfig -a

```
- 相关端口可用（最好不安装其他服务)
- Swap 是关闭的


### ubuntu

重置网络: `sudo vi /etc/network/interfaces`

重启网络: /etc/init.d/networking restart     没生效，老的IP 信息还在，最后reboot

-----》`ip addr flush ens160` 然后 `/etc/init.d/networking restart`即可重置IP

### kubeadm

#### install

- letting iptables see bridged traffic

1. Make sure that the `br_netfilter` module is loaded 

```shell
$ lsmod | grep br_netfilter

#if is nil,To load it
$ sudo modprobe br_netfilter
```

2. Add parameter

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

- 检查端口是否被占用

**master**

| Protocol | Direction | Port Range | Purpose                 | Used by             |
| -------- | --------- | ---------- | ----------------------- | ------------------- |
| TCP      | Inbound   | 6443*      | kubernetes API Server   | All                 |
| TCP      | Inbound   | 2379-2380  | etcd server             | kube-apiserver,etcd |
| TCP      | Inbound   | 10250      | kubelet API             | self,control plane  |
| TCP      | Inbound   | 10251      | kube-scheduler          | self                |
| TCP      | Inboudn   | 10252      | kube-controller-manager | self                |

**worker node**

| Protocol | Direction | Port Range  | Purpose     | Used by            |
| -------- | --------- | ----------- | ----------- | ------------------ |
| TCP      | Inbound   | 10250       | kubelet API | self,Control plane |
| TCP      | Inbound   | 30000-32767 | NodePort    | All                |

- install runtime

1. 卸载旧的版本

```bash
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

2. 准备相关环境和包

```bash
$ sudo apt-get update
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

3. 使用脚本自动安装

```bash
$ curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

4. 非root用户执行docker命令

```bash
sudo usermod -aG docker ec2-user
```

> 注意:
>
>   如果测试不成功，则ec2-user退出重新登录下

- install kubelet kubeadm kubectl

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
gpg --keyserver keyserver.ubuntu.com --recv-keys BA07F4FB
gpg --export --armor BA07F4FB | sudo apt-key add -
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```





#### init

- master

查看需要哪些image

```bash
sudo kubeadm config images list
```



这些image的镜像下载需要`科学上网`,所以可以使用国内的镜像源

```bash
sudo kubeadm init --apiserver-advertise-address=3.1.20.55 \
 --image-repository=registry.cn-qingdao.aliyuncs.com/huaqiangk8s \
 --pod-network-cidr=10.244.0.0/16
```

> 注意：
>
> 根据pre-check的内容修改环境信息，比如管理swap:`swapoff -a`，使用root`sudo`等

执行完后会有一段output提示

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 3.1.20.55:6443 --token 4p3m1n.oh80denl9dzrqx5z \
    --discovery-token-ca-cert-hash sha256:f5f5ad0b946459b0273156182c1109a5c2f7c78a2d20ad02ae8289cf12901989 
```

- node

根据master上面的提示在node节点添加即可

```bash
kubeadm join 3.1.20.55:6443 --token 4p3m1n.oh80denl9dzrqx5z \
    --discovery-token-ca-cert-hash sha256:f5f5ad0b946459b0273156182c1109a5c2f7c78a2d20ad02ae8289cf12901989 
```

> 如果当时没保存，现在如何获取这个信息
>
> 1. 创建token
>
> ```bash
> kubeadm token create
> ```
>
> 2. Get discovery-token-ca-cert-hash
>
> ```bash
> openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
>    openssl dgst -sha256 -hex | sed 's/^.* //'
> ```
>
> 



#### add-ons

安装网络插件

```bash
kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

### TLS-bootstrap


#### API Server

检查apiserver 是否开启

```bash
--client-ca-file=/etc/kubernetes/pki/ca.crt
--enable-bootstrap-token-auth=true
```



#### kube-controller-manager

检查controller-manager是否包含参数

```bash
--controllers=*,bootstrapsigner,tokencleaner
--experimental-cluster-signing-duration=8760h0m0s
--cluster-signing-cert-file=/var/lib/kubernetes/ca.pem
--cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem
```



修改添加添加，注意是二进制安装还是systemd控制的，还是kubelet的static pod，修改相应位置即可



#### RBAC 资源

- 允许kubelet 创建CSR

```bash
cat <<EOF | kubectl create -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: create-csrs-for-bootstrapping
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
EOF
```

- 自动为bootstrap签名

```bash
cat <<EOF | kubectl create -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: auto-approve-csrs-for-group
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
```

- 证书自动renew

```bash
cat <<EOF | kubectl create -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: auto-approve-renewals-for-nodes
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
EOF
```
#### 创建随机bootstrap token

```bash
echo $(openssl rand -hex 3).$(openssl rand -hex 8)
```

output

```
80a6ee.fd219151288b08d8
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  # Name MUST be of form "bootstrap-token-<token id>"
  name: bootstrap-token-07401b
  namespace: kube-system

# Type MUST be 'bootstrap.kubernetes.io/token'
type: bootstrap.kubernetes.io/token
stringData:
  # Human readable description. Optional.
  description: "The default bootstrap token generated by 'kubeadm init'."

  # Token ID and secret. Required.
  token-id: 07401b
  token-secret: f395accd246ae52d

  # Expiration. Optional.
  expiration: 2017-03-10T03:22:11Z

  # Allowed usages.
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"

  # Extra groups to authenticate the token as. Must start with "system:bootstrappers:"
  auth-extra-groups: system:bootstrappers:worker,system:bootstrappers:ingress

```



#### 创建cluster-info

```bash
$ kubectl config set-cluster bootstrap \
  --kubeconfig=bootstrap-kubeconfig-public  \
  --server=https://${KUBERNETES_MASTER}:6443 \
  --certificate-authority=ca.pem \
  --embed-certs=true
```

```bash
kubectl -n kube-public create configmap cluster-info \
  --from-file=kubeconfig=bootstrap-kubeconfig-public
```

- 允许anonymous可以获取

```bash
$ kubectl create role anonymous-for-cluster-info --resource=configmaps --resource-name=cluster-info --namespace=kube-public --verb=get,list,watch
$ kubectl create rolebinding anonymous-for-cluster-info-binding --role=anonymous-for-cluster-info --user=system:anonymous --namespace=kube-public
```

#### 创建bootstrap token

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.pem
    server: https://my.server.example.com:6443
  name: bootstrap
contexts:
- context:
    cluster: bootstrap
    user: kubelet-bootstrap
  name: bootstrap
current-context: bootstrap
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: 07401b.f395accd246ae52d
```



#### work 节点

保证包含 `--bootstrap-kubeconfig="/var/lib/kubelet/bootstrap-kubeconfig" --kubeconfig="/var/lib/kubelet/kubeconfig"`