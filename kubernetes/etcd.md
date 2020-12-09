### etcd

- [etcd](#etcd)
  - [etcd概念](#etcd概念)
    - [为什么叫etcd](#为什么叫etcd)
    - [特性](#特性)
  - [安装](#安装)
    - [安装推荐](#安装推荐)
    - [etcd-keeper](#etcd-keeper)
      - [启动](#启动)
      - [UI 访问](#ui-访问)
  - [备份恢复](#备份恢复)
    - [备份](#备份)
    - [恢复](#恢复)

#### etcd概念

`A distributed, reliable key-value store for the most critical data of a distributed system`

etcd是一个分布式、可靠k-v存储分布式系统，它不仅仅用于存储，还提供共享配置及服务发现。

##### 为什么叫etcd

---

"etcd"这个名字源于两个想法，即unix系统的`etc`文件夹合分布式文件系统`distributed`。`etc`文件夹为单个系统存储配置数据的地方，而etcd存储大规模分布式系统的配置信息。



etcd以一致和容错的方式存储元数据。分布式系统使用etcd作为一致性键值存储，用于配置管理，服务发现和协调分布式工作。使用etcd的通用分布式模式包括领导选举，分布式锁和监视机器活动。

---

##### 特性

- 一致性协议：etcd使用Raft协议。

- 数据存储：etcd多版本并发控制（MVCC）数据模型，支持查询先前版本的键值对。

- API：etcd提供HTTP+JSON，gRPC接口，跨平台跨语言。

- 访问安全方面：etcd支持HTTPS访问

  

#### 安装

按照官网给出的数据， 在 2CPU，1.8G 内存，SSD 磁盘这样的配置下，单节点的写性能可以达到 16K QPS, 而先写后读也能达到12K QPS。这个性能还是相当可观。

##### 安装推荐

| k8s节点数 | vCPUs | Memory(GB) | storage(GB) ssd |
| --------- | ----- | ---------- | --------------- |
| <=50      | 2     | 8          | 50              |
| 50-250    | 4     | 16         | 150             |
| 250-1000  | 8     | 32         | 250             |
| 3000      | 15    | 64         | 500             |

##### etcd-keeper

###### 启动

> 注意： 以下镜像适合于etcd v3

```bash
$ docker run -d \
-p 8888:8080 \
-v /etc/ssl/etcd:/etc/ssl/etcd \
--env CACERT=/etc/ssl/etcd/ssl/ca.pem \
--env CERT=/etc/ssl/etcd/ssl/node-node1.pem \
--env KEY=/etc/ssl/etcd/ssl/node-node1-key.pem \
registry.cn-qingdao.aliyuncs.com/imageofout/etcd-keeper:v2
```

###### UI 访问

http://3.1.20.110:8888/etcdkeeper/

> 源代码:
>
> ```bash
> $ git clone https://github.com/evildecay/etcdkeeper.git
> ```
>
> 

#### 备份恢复

##### 备份

```bash
$ ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/ssl/etcd/ssl/ca.pem \
--cert=/etc/ssl/etcd/ssl/node-node1.pem \
--key=/etc/ssl/etcd/ssl/node-node1-key.pem \
snapshot save /tmp/snapshot-pre-boot.db
```

##### 恢复

```bash
$ ETCDCTL_API=3 etcdctl \
--data-dir=/var/lib/etcd \
--name etcd2 \
--initial-cluster etcd1=https://3.1.20.110:2380,etcd2=https://3.1.20.111:2380,etcd3=https://3.1.20.112:2380 \
--initial-cluster-token k8s_etcd \
--initial-advertise-peer-urls https://3.1.20.111:2380 \
snapshot restore //tmp/snapshot-pre-boot.db
```



> 注意： 1. 如果备份的db文件是`copy`而来，则需加`--skip-hash-check`
>
> ​             2. 恢复前，清空`/var/lib/etcd`


http://thesecretlivesofdata.com/raft/