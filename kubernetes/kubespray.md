# content
- [content](#content)
  - [1. 安装k8s集群(部署节点可访问internet)](#1-安装k8s集群部署节点可访问internet)
    - [1.1 准备](#11-准备)
      - [1.1.1 下载kubespray](#111-下载kubespray)
      - [1.1.2 准备本地yum源](#112-准备本地yum源)
      - [1.1.3 更新inventory文件](#113-更新inventory文件)
      - [1.1.4 检查各个节点是否满足需求](#114-检查各个节点是否满足需求)
      - [1.1.5 配置参数文件](#115-配置参数文件)
    - [1.2 安装](#12-安装)
  - [2. 安装k8s集群(离线安装)](#2-安装k8s集群离线安装)
    - [2.1 检查相关资源包是否充足](#21-检查相关资源包是否充足)
      - [2.1.1 yum](#211-yum)
      - [2.1.2 kubespray](#212-kubespray)
      - [2.1.3 pip package](#213-pip-package)
  - [3. 准备离线资源](#3-准备离线资源)
      - [3.1 准备yum](#31-准备yum)
      - [3.2 下载kubespray](#32-下载kubespray)
      - [3.3 下载ansible及相关包](#33-下载ansible及相关包)
      - [备注](#备注)
    - [imges](#imges)
    - [其他备注](#其他备注)
    - [测试vm管理](#测试vm管理)



## 1. 安装k8s集群(部署节点可访问internet)

保证部署节点可以访问internet。由于部署需要访问相关internet下载软件包，故需要保证部署节点可以访问internet(主要下载image,二进制包等,所以待安装k8s的主机还需要自行配置yum源，可以参考[ 准备离线资源](#准备资源)中方式，通过httpd配置本地yum）。

### 1.1 准备

#### 1.1.1 下载kubespray

```bash
# 解压
wget https://github.com/kubernetes-sigs/kubespray/archive/v2.14.1.tar.gz

# 解压
tar -xzvf v2.14.1.tar.gz 
```



#### 1.1.2 准备本地yum源

此处假设待安装k8s没有yum源

1 . [准备yum](#准备yum)

2 . 安装httpd准备yum

```bash
yum install httpd
cp /mnt/centos78 /var/www/html/
```

> 检验是否能够访问:
>
> ```bash
> curl localhost/centos78/
> ```
>
> 注意:
>
> 有时候无法验证能够访问此yum，需要设置selinux
>
> ```bash
> setenforce 0
> ```
>
> 



#### 1.1.3 更新inventory文件

拷贝出新的inventory目录

在`inventory`目录下，包含一个`sample`的目录，可以从此目录拷贝出需要创建集群的目录，比如`pro`

```bash
cd kubespray-2.14.1/

# 安装ansible以及其依赖
pip3 install -r requirements.txt 

# 创建待安装集群目录
cp -r inventory/sample inventory/pro
```



更新`inventory.ini`文件，写入待安装的主机信息

```bash
# 书写主机文件,根据主机不同，需要修改
cat << EOF > inventory/pro/inventory.ini
[all]
node1 ansible_host=3.1.20.120  ansible_password=123456 etcd_member_name=etcd1
node2 ansible_host=3.1.20.121  ansible_password=123456 etcd_member_name=etcd2
node3 ansible_host=3.1.20.122  ansible_password=123456 etcd_member_name=etcd3
node4 ansible_host=3.1.20.123  ansible_password=123456 # ip=10.3.0.4 etcd_member_name=etcd4
node5 ansible_host=3.1.20.124  ansible_password=123456 # ip=10.3.0.5 etcd_member_name=etcd5
node6 ansible_host=3.1.20.125  ansible_password=123456 # ip=10.3.0.6 etcd_member_name=etcd6

# ## configure a bastion host if your nodes are not directly reachable
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube-master]
node1
node2
node3

[etcd]
node1
node2
node3

[kube-node]
node1
node2
node3
node4
node5
node6

[calico-rr]

[k8s-cluster:children]
kube-master
kube-node
calico-rr
EOF
```



#### 1.1.4 检查各个节点是否满足需求

```bash
# 若inventory.ini文件中使用用户名密码，则需要在部署节点安装sshpass插件
yum install sshpass

# 安装docker用于主机缓存image
yum install docker

# 刷掉所有iptables 信息
ansible -i inventory/pro/inventory.ini all -m shell -a "iptables --flush"
ansible -i inventory/pro/inventory.ini all -m shell -a "iptables -t nat --flush"


# 查看selinux 信息------->最好是disabled,不可为 enforcing
ansible -i inventory/pro/inventory.ini all -m shell -a "getenforce"
ansible -i inventory/pro/inventory.ini all -m shell -a "setenforce 0"


# 查看/etc/resolv.conf是否为空
ansible -i inventory/pro/inventory.ini all -m shell -a "cat /etc/resolv.conf"
# 若为空，则需添加nameserver
ansible -i inventory/pro/inventory.ini all -m shell -a "echo 'nameserver 8.8.8.8' >> /etc/resolv.conf"


# 配置yum源，此yum源若使用zdgt提供最小本地yum源
cat << EOF >/root/zdgt.repo
[base]
name=Base
baseurl=http://3.1.20.129/centos78/k8s
gpgcheck=0
EOF

# 同步到各个节点
$ ansible -i inventory/pro/inventory.ini all -m shell -a "rm -rf /etc/yum.repos.d/Cent*"
$ ansible -i inventory/pro/inventory.ini all -m copy -a "src=/root/zdgt.repo dest=/etc/yum.repos.d/zdgt.repo"
```



#### 1.1.5 配置参数文件

更新参数文件，配置相关参数

```yaml
cat << EOF > zdgt.yaml
download_localhost: true
download_keep_remote_cache: true
download_force_cache: true
download_run_once: true

yum_repo: http://3.1.20.129/centos78/docker/
docker_rh_repo_base_url: "{{ yum_repo }}/docker-ce/"
docker_rh_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"

#extras_rh_repo_base_url: "{{ yum_repo }}/extras"
#extras_rh_repo_gpgkey: "{{ yum_repo }}/extras/gpg"

# preinstall
# this package should be in yum repository
common_required_pkgs:
  - "{{ (ansible_distribution == 'openSUSE Tumbleweed') | ternary('openssl-1_1', 'openssl') }}"
  - curl
  - rsync
  - socat
  - unzip
  - e2fsprogs
  - xfsprogs
  - ebtables
  - sshpass

##### 此处使用国内缓存镜像，如果更新镜像，可以参考本项目镜像更新部分
kube_image_repo: "registry.cn-qingdao.aliyuncs.com/huaqiangk8s"
containerd_version: "1.3.7"
containerd_versioned_pkg:
  'latest': "{{ containerd_package }}"
  '1.2.4': "{{ containerd_package }}-1.2.4-3.1.el7"
  '1.2.5': "{{ containerd_package }}-1.2.5-3.1.el7"
  '1.2.6': "{{ containerd_package }}-1.2.6-3.3.el7"
  '1.2.10': "{{ containerd_package }}-1.2.10-3.2.el7"
  '1.2.12': "{{ containerd_package }}-1.2.12-3.1.el7"
  '1.2.13': "{{ containerd_package }}-1.2.13-3.2.el7"
  '1.3.7': "{{ containerd_package }}-1.3.7-3.1.el7"
  'stable': "{{ containerd_package }}-1.2.13-3.2.el7"
  'edge': "{{ containerd_package }}-1.2.13-3.2.el7"
docker_cli_version: "19.03"
docker_version: "19.03"
docker_versioned_pkg:
  'latest': docker-ce
  '17.03': docker-ce-17.03.3.ce-1.el7
  '17.09': docker-ce-17.09.1.ce-1.el7.centos
  '17.12': docker-ce-17.12.1.ce-1.el7.centos
  '18.03': docker-ce-18.03.1.ce-1.el7.centos
  '18.06': docker-ce-18.06.3.ce-3.el7
  '18.09': docker-ce-18.09.9-3.el7
  '19.03': docker-ce-19.03.13-3.el7
  'stable': docker-ce-19.03.12-3.el7
  'edge': docker-ce-19.03.12-3.el7

docker_cli_versioned_pkg:
  'latest': docker-ce-cli
  '18.09': docker-ce-cli-18.09.9-3.el7
  '19.03': docker-ce-cli-19.03.13-3.el7

docker_selinux_versioned_pkg:
  'latest': docker-ce-selinux-17.03.3.ce-1.el7
  '17.03': docker-ce-selinux-17.03.3.ce-1.el7
  'stable': docker-ce-selinux-17.03.3.ce-1.el7
  'edge': docker-ce-selinux-17.03.3.ce-1.el7

# docker_insecure_registries:
#   - mirror.registry.io
#   - 172.19.16.11
EOF
```

### 1.2 安装

```bash
ansible-playbook -i inventory/pro/inventory.ini --become --become-user=root -e "@zdgt.yaml" cluster.yml 
```



## 2. 安装k8s集群(离线安装)

### 2.1 检查相关资源包是否充足

#### 2.1.1 yum

#### 2.1.2 kubespray

#### 2.1.3 pip package

## 3. 准备离线资源

#### 3.1 准备yum

(若k8s节点无法访问internet, 则安装k8s前需要准备此资源，

**1. 操作系统源**

```bash
# 安装yumdownloader
yum install yum-utils createrepo wget rsync

# 当前需要提前准备rpm包如下，若需增加，直接增加即可
for i in curl rsync socat unzip e2fsprogs xfsprogs libselinux-python device-mapper-libs ebtables nss libselinux-python sshpass container-selinux conntrack python3 python3-pip yum-utils python-kitchen python-chardet libxml2-python libseccomp audit-libs-python
do
yumdownloader --resolve --destdir /mnt/centos78/k8s/ $i
done

# 创建repo
cd /mnt/centos78/k8s/ && createrepo .

```



**2. docker**

```bash
cat << EOF > /etc/yum.repos.d/docker-ce.repo 
[docker-ce]
name=Docker-CE Repository
baseurl=https://download.docker.com/linux/centos/7/x86_64/stable
enabled=1
gpgcheck=1
keepcache=0
gpgkey=https://download.docker.com/linux/centos/gpg
EOF

# sync相关yum源
reposync -n --repoid="docker-ce" -p /mnt/centos78/docker
# 下载gpg文件
cd /mnt/centos78/docker/docker-ce $$ wget https://download.docker.com/linux/centos/gpg

# 创建repo
cd /mnt/centos78/docker/docker-ce && createrepo .
```

#### 3.2 下载kubespray

```bash
# 解压
wget https://github.com/kubernetes-sigs/kubespray/archive/v2.14.1.tar.gz

# 解压
tar -xzvf v2.14.1.tar.gz 
```



#### 3.3 下载ansible及相关包

```bash
# 安装python3
yum install python3

# 下载pip包
cd kubespray-2.14.1/
pip3 install --download /mnt/zhenpackage -r requirements.txt

```

> pip离线安装
>
> ```bash
> pip install --no-index --find-links=/mnt/zhenpackage -r requirements.txt
> ```



#### 备注

> 提示：
>
> - yum中需要包含container-selinux,否则需要添加extras
> - 若使用部署节点为download node，则部署节点安装rsync，docker
> - 各个节点均需安装sshpass
> - 各个节点需关闭iptables
> - 部署节点需要访问 https://storage.googleapis.com/kubernetes-release，国内网络环境有时会失败





### imges

```bash
# update github script
k8s.gcr.io/k8s-dns-node-cache:1.15.13
k8s.gcr.io/kube-apiserver:v1.18.9
k8s.gcr.io/kube-controller-manager:v1.18.9
k8s.gcr.io/kube-scheduler:v1.18.9
k8s.gcr.io/kube-proxy:v1.18.9
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7
k8s.gcr.io/cluster-proportional-autoscaler-amd64:1.8.1


# kube_image_repo 境内源
registry.cn-qingdao.aliyuncs.com/huaqiangk8s
```



参数文件

```yam
download_localhost: true
download_keep_remote_cache: true
download_force_cache: true
download_run_once: true

yum_repo: http://3.1.20.129/docker/
docker_rh_repo_base_url: "{{ yum_repo }}/docker-ce/"
docker_rh_repo_gpgkey: "{{ yum_repo }}/gpg"

#extras_rh_repo_base_url: "{{ yum_repo }}/extras"
#extras_rh_repo_gpgkey: "{{ yum_repo }}/extras/gpg"

# preinstall
# this package should be in yum repository
common_required_pkgs:
  - "{{ (ansible_distribution == 'openSUSE Tumbleweed') | ternary('openssl-1_1', 'openssl') }}"
  - curl
  - rsync
  - socat
  - unzip
  - e2fsprogs
  - xfsprogs
  - ebtables
  - sshpass


kube_image_repo: "registry.cn-qingdao.aliyuncs.com/huaqiangk8s"
containerd_version: "1.3.7"
containerd_versioned_pkg:
  'latest': "{{ containerd_package }}"
  '1.2.4': "{{ containerd_package }}-1.2.4-3.1.el7"
  '1.2.5': "{{ containerd_package }}-1.2.5-3.1.el7"
  '1.2.6': "{{ containerd_package }}-1.2.6-3.3.el7"
  '1.2.10': "{{ containerd_package }}-1.2.10-3.2.el7"
  '1.2.12': "{{ containerd_package }}-1.2.12-3.1.el7"
  '1.2.13': "{{ containerd_package }}-1.2.13-3.2.el7"
  '1.3.7': "{{ containerd_package }}-1.3.7-3.1.el7"
  'stable': "{{ containerd_package }}-1.2.13-3.2.el7"
  'edge': "{{ containerd_package }}-1.2.13-3.2.el7"
docker_cli_version: "19.03"
docker_version: "19.03"
docker_versioned_pkg:
  'latest': docker-ce
  '17.03': docker-ce-17.03.3.ce-1.el7
  '17.09': docker-ce-17.09.1.ce-1.el7.centos
  '17.12': docker-ce-17.12.1.ce-1.el7.centos
  '18.03': docker-ce-18.03.1.ce-1.el7.centos
  '18.06': docker-ce-18.06.3.ce-3.el7
  '18.09': docker-ce-18.09.9-3.el7
  '19.03': docker-ce-19.03.13-3.el7
  'stable': docker-ce-19.03.12-3.el7
  'edge': docker-ce-19.03.12-3.el7

docker_cli_versioned_pkg:
  'latest': docker-ce-cli
  '18.09': docker-ce-cli-18.09.9-3.el7
  '19.03': docker-ce-cli-19.03.13-3.el7

docker_selinux_versioned_pkg:
  'latest': docker-ce-selinux-17.03.3.ce-1.el7
  '17.03': docker-ce-selinux-17.03.3.ce-1.el7
  'stable': docker-ce-selinux-17.03.3.ce-1.el7
  'edge': docker-ce-selinux-17.03.3.ce-1.el7



# docker_insecure_registries:
#   - mirror.registry.io
#   - 172.19.16.11
```











```
ansible-playbook -i inventory/pro/inventory.ini --become --become-user=root -e "@zdgt.yaml" cluster.yml 
```







### 其他备注

### 测试vm管理

```bash
#重新配置vm,将测试的vm taint，然后新建
$ for i in {1..6}; do terraform taint vsphere_virtual_machine.vm$i; done
$ terraform apply
```

