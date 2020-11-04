# content
- [content](#content)
    - [1. `kernel: XFS: possible memory allocation deadlock in kmem_alloc(mode: 0x250)`](#1-kernel-xfs-possible-memory-allocation-deadlock-in-kmem_allocmode-0x250)
    - [2. `Nameserver limits were exceeded`](#2-nameserver-limits-were-exceeded)
    - [3. docker私有仓库，本地TLS认证](#3-docker私有仓库本地tls认证)
    - [4. kubelet报错](#4-kubelet报错)
    - [5. oracle是否适合跑在kubernetes上，那么跑在docker上呢？](#5-oracle是否适合跑在kubernetes上那么跑在docker上呢)
    - [6.kubectl edit configmap 格式乱了](#6kubectl-edit-configmap-格式乱了)




### 1. `kernel: XFS: possible memory allocation deadlock in kmem_alloc(mode: 0x250)`

```bash
# 内存死锁,可以使用以下方式尝试解决
echo 1 > /proc/sys/vm/drop_caches  #一般无法解决，需要升级内核
```

```bash
# 在 CentOS 7.× 上启用 ELRepo 仓库
$ rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
$ rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm

# 列出可用的系统内核相关包:
$ yum --disablerepo="*" --enablerepo="elrepo-kernel" list available

# 安装新内核
$ yum --enablerepo=elrepo-kernel install kernel-ml

# 为了让新安装的内核成为默认启动选项，你需要如下修改 GRUB 配置,打开并编辑 /etc/default/grub 并设置 GRUB_DEFAULT=0.意思是 GRUB 初始化页面的第一个内核将作为默认内核.
$ cat /etc/default/grub |grep GRUB_DEFAULT
GRUB_DEFAULT=0

# 重新生成内核配置
$ grub2-mkconfig -o /boot/grub2/grub.cfg

# 重启机器
$ reboot


```





### 2. `Nameserver limits were exceeded`

```bash
# kubelet 的nameserver数量超出限制,不能超过3个
$ cat /etc/resolv.conf
```



### 3. docker私有仓库，本地TLS认证

```bash
#将Harbor 里面的CA证书,放在指定两个位置,并重启docker
$ sudo mkdir /etc/docker/certs.d/harbor.devops.na
$ sudo cp ca.crt /etc/docker/certs.d/harbor.devops.na/
$ sudo cat ca.crt >> /etc/ssl/certs/ca-certificates.crt
$ sudo systemctl restart docker
```



### 4. kubelet报错

```bash
 kubelet_network_linux.go:111] Not using --random-fully in the MASQUERADE rule for iptables because the local version of iptables does not support it
```

解决

```bash
#安装所需的依赖
$ yum install gcc make libnftnl-devel libmnl-devel autoconf automake libtool bison flex  libnetfilter_conntrack-devel libnetfilter_queue-devel libpcap-devel
$ export LC_ALL=C
$ wget wget https://www.netfilter.org/projects/iptables/files/iptables-1.6.2.tar.bz2
$ tar -xvf iptables-1.6.2.tar.bz2
$ cd iptables-1.6.2 \
 ./autogen.sh \
 ./configure \
 make -j4 \
 make install
 #当然可以把cd /usr/local/sbin下面的iptables相关的东西打包然后分发到其它服务器
 $ cd /usr/local/sbin
 $ \cp iptables /sbin
 $ \cp iptables-restore /sbin/
 $ \cp iptables-save /sbin/
 
 # 重启kube-proxy
```



### 5. oracle是否适合跑在kubernetes上，那么跑在docker上呢？
answer
```
首先，oracle已经提供了image,用于oracle容器化安装，但是也提出仅用于`non-production`的使用。所以如果你想快速启动一个oracle用于测试、学习、demo，那么你可以使用docker或者k8s启动一个oracle 12c。
如果使用容器启动oracle，会带来以下问题：
- oracle如果出问题定位困难，因为涉及到容器技术和数据库技术，哪种都不是那么容易。
- 维护困难，以往维护只需SSH就行了，现在DBA维护甚至需要到oracle运行哪台物理机使用docker命令才能进去
- 升级打补丁困难，oracle相对复杂，升级或者打补丁如何持久化，难道靠官网image的更新吗，更新后的image是否能支持之前数据库，这也是个风险
- oracle数据库本省是应用中比较重要的部分，对宿主机容器的升级维护也会影响到数据库

```
可参考
> https://oracle.github.io/weblogic-kubernetes-operator/userguide/overview/database/
> https://oracle-base.com/articles/linux/docker-oracle-dba-guide-to-docker

k8s安装oracle
- 创建拉去镜像的key
```bash
$ kubectl create secret docker-registry regsecret \
        --docker-server=container-registry.oracle.com \
        --docker-username=your.email@some.com \
        --docker-password=your-password \
        --docker-email=your.email@some.com \
        -n database-namespace
```

- 创建deployment,运行oracle
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: database-namespace
  labels:
    app: database
    version: 12.1.0.2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      version: 12.1.0.2
  template:
    metadata:
      name: database
      labels:
        app: database
        version: 12.1.0.2
    spec:
      volumes:
      - name: dshm
        emptyDir:
          medium: Memory
      # add your volume mount for your persistent storage here
      containers:
      - name: database
        command:
        - /home/oracle/setup/dockerInit.sh
        image: container-registry.oracle.com/database/enterprise:12.1.0.2
        imagePullPolicy: IfNotPresent
        resources:
          requests:
            memory: 10Gi
        ports:
        - containerPort: 1521
          hostPort: 1521
        volumeMounts:
          - mountPath: /dev/shm
            name: dshm
          # add your persistent storage for DB files here
        env:
          - name: DB_SID
            value: OraDoc
          - name: DB_PDB
            value: OraPdb
          - name: DB_PASSWD
            value: *password*
          - name: DB_DOMAIN
            value: my.domain.com
          - name: DB_BUNDLE
            value: basic
          - name: DB_MEMORY
            value: 8g
      imagePullSecrets:
      - name: regsecret
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: database-namespace
spec:
  selector:
    app: database
    version: 12.1.0.2
  ports:
  - protocol: TCP
    port: 1521
    targetPort: 1521
```

### 6.kubectl edit configmap 格式乱了

```bash
kubectl get -n kube-system -o yaml cm aws-auth | sed -E 's/[[:space:]]+\\n/\\n/g' | kubectl apply -f -
```