


## 1. openLDAP Server
### 1.1 cert-manager生成证书
CA证书生成过程可以参考cert-manager中CA部分,此处直接通过CA的issuer创建服务器所需的certificates，其中`secretName`后面会用到，`commonName`可以写ldap server的域名
```bash
$ cat openldap-server-cert.yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: hq-opnldap1
  namespace: default
spec:
  secretName: hq-openldap1
  issuerRef:
    name: ca-issuer
    kind: Issuer
  commonName: hqopenldap.com
$ kubectl apply -f openldap-server-cert.yaml
```
### 1.2 helm安装
检查是否包含openldap的helm仓库，仓库地址为：http://mirror.azure.cn/kubernetes/charts/
```bash
$ helm repo list
NAME         	URL                                                                      
stable       	http://mirror.azure.cn/kubernetes/charts/                                
local        	http://127.0.0.1:8879/charts                                             
ali-incubator	https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts-incubator/
ali-stable   	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts                   
jetstack     	https://charts.jetstack.io   
```

> 若没有仓库，可以添加
> ```bash
> $ helm repo add stable http://mirror.azure.cn/kubernetes/charts/
> ```

**安装**
如果不想本地保存chart，而是直接安装则可执行以下命令,其中env.LDAP_TLS_VERIFY_CLIENT=try表示client可以没有证书。
```bash
$ helm install --name hqopenldap \
--namespace default \
--set tls.enabled=true \
--set tls.secret=hq-openldap1 \
--set env.LDAP_DOMAIN=stopenldap.com \
--set adminPassword=admin \
--set configPassword=config \
--set persistence.enabled=true \
--set persistence.size=2Gi \
--set env.LDAP_TLS_VERIFY_CLIENT=try \
stable/openldap
```

**下载chart安装**
- 查找openldap chart，并下载到本地

```bash
$ helm search openldap
NAME           	CHART VERSION	APP VERSION	DESCRIPTION                      
stable/openldap	1.2.3        	2.4.48     	Community developed LDAP software
$ helm fetch stable/openldap
# 解压文件
$  tar -xzvf openldap-1.2.3.tgz 
```
- 修改values.yaml文件

```bash
$ cd openldap
$ cat values.yaml |grep -v '^ *#'|grep -v '^$'
replicaCount: 1
strategy: {}
image:
  repository: osixia/openldap
  tag: 1.2.4
  pullPolicy: IfNotPresent
existingSecret: ""
tls:
  enabled: true
  secret: "hq-openldap1"  # The name of a kubernetes.io/tls type secret to use for TLS
  CA:
    enabled: false
    secret: "ca-key-pair"  # The name of a generic secret to use for custom CA certificate (ca.crt)
extraLabels: {}
podAnnotations: {}
service:
  annotations: {}
  clusterIP: ""
  ldapPort: 389
  sslLdapPort: 636  # Only used if tls.enabled is true
  externalIPs: []
  loadBalancerIP: ""
  loadBalancerSourceRanges: []
  type: ClusterIP
env:
  LDAP_ORGANISATION: "Example Inc."
  LDAP_DOMAIN: "stopenldap.com"
  LDAP_BACKEND: "hdb"
  LDAP_TLS: "true"
  LDAP_TLS_ENFORCE: "false"
  LDAP_REMOVE_CONFIG_AFTER_SETUP: "true"
adminPassword: admin
configPassword: config
persistence:
  enabled: true
  accessMode: ReadWriteOnce
  size: 2Gi
resources: {}
initResources: {}
nodeSelector: 
  run: openldap
tolerations: []
affinity: {}
test:
  enabled: true
  image:
    repository: dduportal/bats
    tag: 0.4.0
```
主要修改的地方有：
- 1.`tls.enabled: true`     启动tls认证

- 2.`tls.secret: hq-openldap1`   tls认证证书的secret，为cert-manager生成`secertNAME`

​- 3.`env.LDAP_DOMAIN: "stopenldap.com" ` ldap DN名称

​- 4.`adminPassword: admin`  admin的密码

​- 5.`configPassword: config`  config的密码

​- 6.`persistence.enabled:true`  设置持久化存储，默认时候default storageclass

​- 7. `persistence.size:2G` 持久化存储大小

**安装**

在当前目录下执行helm install安装
```bash
$ helm install --name hqopenldap --namespace default .
```
检查是否启动成功
```bash
$ kubectl get pod -l app=openldap
NAME                          READY   STATUS    RESTARTS   AGE
hqopenldap-85d59d49b4-6jl7c   1/1     Running   0          138m
```

## 2. 客户端安装及测试
### 2.1 phpldapadmin

- 创建phpldapadmin相关资源

```bash
$ cat phpldapadmin.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: phpldapadmin-deploy-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpldapadmin-dev
  template:
    metadata:
      labels:
        app: phpldapadmin-dev
    spec:
      containers:
        -
          name: phpldapadmin-dev
          image: osixia/phpldapadmin:0.7.1
          ports:
            - containerPort: 80
          env:
            -
              name: PHPLDAPADMIN_LDAP_HOSTS
              value: "openldap"
            # -
            #   name: PHPLDAPADMIN_LDAP_PORT
            #   value: "389"
            -
              name: PHPLDAPADMIN_HTTPS
              value: "false"
---
apiVersion: v1
kind: Service
metadata:
  name: phpldapadmin-service-dev
spec:
  type: ClusterIP
  selector:
    app: phpldapadmin-dev
  ports:
    -
      name: http
      protocol: TCP
      port: 80
      targetPort: 80

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    certmanager.k8s.io/issuer: "ca-issuer"
  name: ingress-phpldapadmin
spec:
  rules:
  - host: ldap.devops.nari
    http:
      paths:
      - backend:
          serviceName: phpldapadmin-service-dev
          servicePort: 80
        path: /
  tls:
  - hosts:
    - ldap.devops.nari
    secretName: openldap-tls

$ kubectl apply -f phpldapadmin.yaml 
```
> 注意：
> 更改env参数`PHPLDAPADMIN_LDAP_HOSTS`指定openldap server的ip


### 2.2 LDAP Admin Tool

由于测试是使用本地安装的LDAP Admin Tool，所有需要把k8s中生成的证书文件拿下来
- 导出证书

```bash
$ kubectl get secret/client-test1 -o jsonpath="{['data']['tls\.crt']}"|base64 --decode > ldap.crt
```
- 导出私钥
  
```bash
$ kubectl get secret/client-test1 -o jsonpath="{['data']['tls\.key']}"|base64 --decode > ldap.key
```

> 注意： 由于LDAP Admin Tool使用的私钥为pkcs#8，所以需要将私钥转换下
> ```bash
> $ openssl pkcs8 -topk8 -inform PEM -in ldap.key -outform pem -nocrypt -out ldap.pem
> ```

- 倒入证书和私钥

`Security`->`manage client certificates`->`Add Certificate`   导入证书

`Security`->`manage client certificates`->鼠标选中证书-`Set Private Key` 导入私钥

- 连接LDAP Server
连接信息中：Base DN: dc=stopenldap,dc=com    参照helm `values.yaml`中`env.LDAP_DOMAIN`

​                       User DN: cn=admin,dc=stopenldap,dc=com 参照helm `values.yaml`中`env.LDAP_DOMAIN`

​                       Port: 636

​                       Password: admin       参照helm `values.yaml`中`adminPassword`


### 3. 备注

ldap server启动TLS时，默认需要验证client端的证书，若不想给客户端签发证书,可通过设置server端不验证证书即可，可以通过添加values.yaml中的一个参数设置:`env.LDAP_TLS_VERIFY_CLIENT: "try"`

> never：不验证客户端证书.
> allow：检查客户端证书，没有证书或证书错误，都允许连接。
> try：检查客户端证书，没有证书（允许连接），证书错误（终止连接）。
> demand | hard | true：检查客户端证书，没有证书或证书错误都将立即终止连接。
> helm 安装的openldap默认为demand。

ldapadd导入数据
```bash
cat > base.ldif << EOF

dn: dc=jinni,dc=com
o: jinni com
dc: jinni
objectClass: top
objectClass: dcObject
objectclass: organization
dn: cn=root,dc=jinni,dc=com

cn: root
objectClass: organizationalRole
description: Directory Manager
dn: ou=People,dc=jinni,dc=com
ou: People
objectClass: top
objectClass: organizationalUnit
dn: ou=Group,dc=jinni,dc=com
ou: Group
objectClass: top
objectClass: organizationalUnit
EOF
```
ldapadd -x -w "admin@123" -D "cn=root,dc=jinni,dc=com" -f /root/base.ldif
```