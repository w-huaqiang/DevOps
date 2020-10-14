# kubespray 



### 测试vm管理

```bash
#重新配置vm,将测试的vm taint，然后新建
$ for i in {1..6}; do terraform taint vsphere_virtual_machine.vm$i; done
$ terraform apply
```



### 配置node

```bash
# 配置yum源
ansible -i inventory/pro/inventory.ini all -m shell -a "rm -rf /etc/yum.repos.d/Cent*"
ansible -i inventory/pro/inventory.ini all -m copy -a "src=/root/centos-local.repo dest=/etc/yum.repos.d/centos-local.repo"

# 配置dnsserver
ansible -i inventory/pro/inventory.ini all -m shell -a "echo 'nameserver 6.6.6.6' >> /etc/resolv.conf"


```

> 提示：
>
> - yum中需要包含container-selinux,否则需要添加extras
> - 若使用部署节点为download node，则部署节点安装rsync，docker
> - sshpass
> - 各个节点需关闭iptables



### yum

操作系统源

```bash
for i in curl rsync socat unzip e2fsprogs xfsprogs libselinux-python device-mapper-libs ebtables nss libselinux-python sshpass container-selinux conntrack python3 python3-pip yum-utils python-kitchen python-chardet libxml2-python libseccomp audit-libs-python
do
yumdownloader --resolve --destdir /mnt/centos7/k8s/ $i
done

```





docker

```bash
$ cat /etc/yum.repos.d/docker-ce.repo 
[docker-ce]
name=Docker-CE Repository
baseurl=https://download.docker.com/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=1
keepcache=0
gpgkey=https://download.docker.com/linux/centos/gpg

# sync相关yum源
$ reposync -n --repoid="docker-ce" -p /var/www/html/docker
```







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



### 下载依赖的package

```
pip install --download /mnt/zhenpackage -r requirements.txt
pip install --no-index --find-links=/mnt/zhenpackage -r requirements.txt
```



## 安装

### 准备

#### 更新inventory文件

拷贝出新的inventory目录

在`inventory`目录下，包含一个`sample`的目录，可以从此目录拷贝出需要创建集群的目录，比如`pro`

```bash
$ cp -r inventory/sample inventory/pro
```



书写待安装的主机信息

```bash
# 书写主机文件
$ cat inventory/pro/inventory.ini
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
```

#### 检查各个节点是否满足需求

```bash
# 刷掉所有iptables 信息
$ ansible -i inventory/pro/inventory.ini all -m shell -a "iptables --flush"
$ ansible -i inventory/pro/inventory.ini all -m shell -a "iptables -t nat --flush"


# 查看selinux 信息------->最好是disabled,不可为 enforcing
$ ansible -i inventory/pro/inventory.ini all -m shell -a "getenforce"


# 查看/etc/resolv.conf是否为空
$ ansible -i inventory/pro/inventory.ini all -m shell -a "cat /etc/resolv.conf"
# 若为空，则需添加nameserver
$ ansible -i inventory/pro/inventory.ini all -m shell -a "echo 'nameserver 6.6.6.6' >> /etc/resolv.conf"


# 配置yum源，此yum源若使用zdgt提供最小本地yum源，配置参考如上yum配置,则
$ cat /root/zdgt.repo
[base]
name=Base
baseurl=http://3.1.20.129/zdgt/k8s
gpgcheck=0
# 同步到各个节点
$ ansible -i inventory/pro/inventory.ini all -m shell -a "rm -rf /etc/yum.repos.d/Cent*"
$ ansible -i inventory/pro/inventory.ini all -m copy -a "src=/root/zdgt.repo dest=/etc/yum.repos.d/zdgt.repo"
```

