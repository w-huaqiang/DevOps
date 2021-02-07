# content
- [content](#content)
    - [1. `kernel: XFS: possible memory allocation deadlock in kmem_alloc(mode: 0x250)`](#1-kernel-xfs-possible-memory-allocation-deadlock-in-kmem_allocmode-0x250)
    - [2. `Nameserver limits were exceeded`](#2-nameserver-limits-were-exceeded)
    - [3. docker私有仓库，本地TLS认证](#3-docker私有仓库本地tls认证)
    - [4. kubelet报错](#4-kubelet报错)
    - [5. oracle是否适合跑在kubernetes上，那么跑在docker上呢？](#5-oracle是否适合跑在kubernetes上那么跑在docker上呢)
    - [6.kubectl edit configmap 格式乱了](#6kubectl-edit-configmap-格式乱了)
    - [7. `Unable to perform initial IP allocation check: unable to refresh the service IP block`](#7-unable-to-perform-initial-ip-allocation-check-unable-to-refresh-the-service-ip-block)
    - [8. docker无法启动](#8-docker无法启动)
    - [9. docker容器网络无法和docker0联通，也无法通过暴露的port访问内部容器](#9-docker容器网络无法和docker0联通也无法通过暴露的port访问内部容器)




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


### 7. `Unable to perform initial IP allocation check: unable to refresh the service IP block`

apiserver无法启动，并报错
```bash
F0826 11:14:55.383445       1 controller.go:157] Unable to perform initial IP allocation check: unable to refresh the service IP block: Get https://[::1]:6443/api/v1/services: net/http: TLS handshake timeout
```

考虑使用IPv6,使之无法链接，禁用主机ipv6

```bash
cat >> /etc/sysctl.conf <<EOF
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
EOF
sysctl -p
```

### 8. docker无法启动
```bash
Feb 05 19:56:18 DisasterRecovery dockerd[36614]: time="2021-02-05T19:56:18.733293521+08:00" level=info msg="Loading containers: start."
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: INVALID_TYPE: structure size mismatch 16 != 13
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t filter -C FORWARD -j DOCKER-ISOLATION' failed: iptables v1.4.21: Couldn't load target `DOCKER
                                                  
                                                  Try `iptables -h' or 'iptables --help' for more information.
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t nat -D PREROUTING -m addrtype --dst-type LOCAL -j DOCKER' failed: iptables: No chain/target/m
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t nat -D OUTPUT -m addrtype --dst-type LOCAL ! --dst 127.0.0.0/8 -j DOCKER' failed: iptables: N
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t nat -D OUTPUT -m addrtype --dst-type LOCAL -j DOCKER' failed: iptables: No chain/target/match
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t nat -D PREROUTING' failed: iptables: Bad rule (does a matching rule exist in that chain?).
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t nat -D OUTPUT' failed: iptables: Bad rule (does a matching rule exist in that chain?).
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t filter -F DOCKER-ISOLATION' failed: iptables: No chain/target/match by that name.
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t filter -X DOCKER-ISOLATION' failed: iptables: No chain/target/match by that name.
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t nat -n -L DOCKER' failed: iptables: No chain/target/match by that name.
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t filter -n -L DOCKER' failed: iptables: No chain/target/match by that name.
Feb 05 19:56:18 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:18 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t filter -n -L DOCKER-ISOLATION-STAGE-1' failed: iptables: No chain/target/match by that name.
Feb 05 19:56:19 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:19 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t filter -n -L DOCKER-ISOLATION-STAGE-2' failed: iptables: No chain/target/match by that name.
Feb 05 19:56:19 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:19 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t filter -C DOCKER-ISOLATION-STAGE-1 -j RETURN' failed: iptables: Bad rule (does a matching rul
Feb 05 19:56:19 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:19 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t filter -C DOCKER-ISOLATION-STAGE-2 -j RETURN' failed: iptables: Bad rule (does a matching rul
Feb 05 19:56:19 DisasterRecovery dockerd[36614]: time="2021-02-05T19:56:19.055530840+08:00" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can be use
Feb 05 19:56:19 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:19 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -t nat -C DOCKER -i docker0 -j RETURN' failed: iptables: Bad rule (does a matching rule exist in
Feb 05 19:56:19 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:19 ERROR: COMMAND_FAILED: '/sbin/iptables -w2 -D FORWARD -i docker0 -o docker0 -j DROP' failed: iptables: Bad rule (does a matching rule exist
Feb 05 19:56:19 DisasterRecovery firewalld[1179]: 2021-02-05 19:56:19 ERROR: INVALID_ZONE: docker
F
```

解决： 未安装iptables ,可以安装iptables
```bash
$ yum install iptables -y
```


### 9. docker容器网络无法和docker0联通，也无法通过暴露的port访问内部容器

现象是： 1. docker 通过 -p 暴露的端口在外面无法访问
        2. 宿主机中的容器之间可以互相访问，但是无法访问docker0等桥接设备
        3. docker network create 创建的网络设备也不行

解决： 极大可能是内核家在bridge.ko模块有问题，通过升级内核解决。