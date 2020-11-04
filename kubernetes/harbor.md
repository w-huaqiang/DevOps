# content
- [content](#content)
      - [1. 生成证书](#1-生成证书)
      - [2. 安装](#2-安装)

#### 1. 生成证书

- 创建命名空间

```bash
$ kubectl create ns harbor 
```



- 创建证书
> 需要提前安装cert-manager，若未安装，则需要使用openssl生成

```bash
$ cat certificate.yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: harbor-tls
  namespace: harbor
spec:
  secretName: harbor-tls
  issuerRef:
    name: ca-issuer
    kind: ClusterIssuer
  commonName: harbor.devops.na
  organization:
  - CA
  dnsNames:
  - devops.na
  - harbor.devops.na
  - notary.devops.na
```

- 创建证书资源

```bash
$ kubectl apply -f certificate.yaml
```



#### 2. 安装

- 添加镜像仓库

```bash
$ helm repo add harbor https://helm.goharbor.io
```

- chart 安装

```bash
$ helm install harbor harbor/harbor \
--namespace harbor \
--version 1.3.2 \
--set expose.tls.secretName=harbor-tls \
--set expose.tls.notarySecretName=harbor-tls \
--set expose.ingress.hosts.core=harbor.devops.na \
--set expose.ingress.hosts.notary=notary.devops.na \
--set externalURL=https://harbor.devops.na \
--set persistence.persistentVolumeClaim.registry.size=100Gi \
--set persistence.persistentVolumeClaim.chartmuseum.size=50Gi \
--set persistence.persistentVolumeClaim.database.size=5Gi \
--set persistence.persistentVolumeClaim.redis.size=1Gi 

```

