# content
- [content](#content)
  - [1. 启动参数](#1-启动参数)
    - [1.1 常用参数](#11-常用参数)
  - [2. 运行命令参数](#2-运行命令参数)
    - [2.1 镜像](#21-镜像)
    - [2.2 容器](#22-容器)

## 1. 启动参数
docker的启动参数比较多,主要放在`systemctl`管理的`/usr/lib/systemd/system/docker.service`文件和`/etc/docker/daemon.json`中，一般修改`/etc/docker/daemon.json`即可达到修改配置的需求.

### 1.1 常用参数

常用参数已经注释解释，未解释参数使用默认值即可.

```json
{
  "authorization-plugins": [],
  # Docker运行时使用的根路径, 默认/var/lib/docker
  "data-root": "",
  # 设定容器的DNS地址
  "dns": [],
  "dns-opts": [],
  "dns-search": [],
  "exec-opts": [],
  "exec-root": "",
  "experimental": false,
  "features": {},
  "storage-driver": "",
  "storage-opts": [],
  "labels": [],
  # 重启docker daemon进程，docker 容器是否关闭，true时，重启docker，容器不会被杀掉
  "live-restore": true,
  # 日志格式,以及日志大小，文件数等属性
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file":"5",
    "labels": "somelabel",
    "env": "os,customer"
  },
  "mtu": 0,
  "pidfile": "",
  "cluster-store": "",
  "cluster-store-opts": {},
  "cluster-advertise": "",
  "max-concurrent-downloads": 3,
  "max-concurrent-uploads": 5,
  "default-shm-size": "64M",
  "shutdown-timeout": 15,
  "debug": true,
  "hosts": [],
  "log-level": "",
  "tls": true,
  "tlsverify": true,
  "tlscacert": "",
  "tlscert": "",
  "tlskey": "",
  "swarm-default-advertise-addr": "",
  "api-cors-header": "",
  "selinux-enabled": false,
  "userns-remap": "",
  "group": "",
  "cgroup-parent": "",
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  },
  "init": false,
  "init-path": "/usr/libexec/docker-init",
  "ipv6": false,
  "iptables": false,
  "ip-forward": false,
  "ip-masq": false,
  "userland-proxy": false,
  "userland-proxy-path": "/usr/libexec/docker-proxy",
  "ip": "0.0.0.0",
  "bridge": "",
  "bip": "",
  "fixed-cidr": "",
  "fixed-cidr-v6": "",
  "default-gateway": "",
  "default-gateway-v6": "",
  "icc": false,
  "raw-logs": false,
  "allow-nondistributable-artifacts": [],
  # 镜像加速，可使用国内本土镜像加速
  "registry-mirrors": [],
  "seccomp-profile": "",
  # 默认拉去镜像使用https,可以在此处添加http仓库。
  "insecure-registries": [],
  "no-new-privileges": false,
  "default-runtime": "runc",
  "oom-score-adjust": -500,
  "node-generic-resources": ["NVIDIA-GPU=UUID1", "NVIDIA-GPU=UUID2"],
  "runtimes": {
    "cc-runtime": {
      "path": "/usr/bin/cc-runtime"
    },
    "custom": {
      "path": "/usr/local/bin/my-runc-replacement",
      "runtimeArgs": [
        "--debug"
      ]
    }
  },
  "default-address-pools":[
    {"base":"172.80.0.0/16","size":24},
    {"base":"172.90.0.0/16","size":24}
  ]
}
```
> 详细可参考官网:
https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file

## 2. 运行命令参数

### 2.1 镜像
1. 查找镜像
```bash
$ docker search nginx
#查找指定版本
$ docker search nginx:1.8
```

2. 拉去镜像
```bash
$ docker pull nginx
# 拉去指定版本镜像
$ docker pull nginx:1.8
```

3. 查看本地镜像
```bash
$ docker images
```

4. 创建镜像
```bash
# build 必须有Dockerfile文件，后续会详解，. 为上线文，可以理解文当前目录
$ docker build -t nginx:1.8 .

```

5. 查看镜像
```bash
# 查看镜像信息
$ docker inspect nginx:1.8

```

6. 删除镜像
```bash
# 查看镜像信息
$ docker rmi nginx:1.8

```

### 2.2 容器
1. 启动容器
```bash
[root@localhost system]# docker run nginx:1.8 pwd
/
# nginxL:1.8 为镜像名称，pwd为命令
```

2. 启动命令使其一直运行后台
```
[root@localhost system]# docker run -d nginx:1.8
a9feab9a1d11a7a963d81ee2507b63bff0b7502de53ac40a975c8072c1af13ec
```
可以看到，未在后面加command，因为一直运行的话需要进程不能结束，bash很明显不行，可以借助镜像默认启动命令

3. 进入一个容器执行命令
```bash
[root@localhost system]# docker exec -ti a9feab9a1d11 bash
root@a9feab9a1d11:/# pwd

```
4. 退出容器
```bash
root@a9feab9a1d11:/# exit
exit
[root@localhost system]# 
```

5. 删除容器
```bash
[root@localhost system]# docker rm 
```