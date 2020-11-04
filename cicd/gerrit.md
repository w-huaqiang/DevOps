## 1. gerrit介绍

Gerrit实际上一个Git服务器，它为在其服务器上托管的Git仓库提供一系列权限控制，以及一个用来做Code Review是Web前台页面。目前，其主要功能就是用来做Code Review。


一般他在 code review 中所处的位置如下图所示
<img src="../image/gerrit.png" alt="gerrit" width="400" />

## 2. gerrit 安装



### 2.1 安装

- 创建命名空间

```bash
$ kubectl create ns jenkins
```

#### 2.1.1 chart安装(不是很好用)

- 下载gerrit chart

由于没有chart仓库中包含的gerrit chart，故下载github上的chart

```bash
$ wget https://github.com/GerritCodeReview/k8s-gerrit/archive/v0.1.tar.gz
```

> **注意**:
>
> 此chart中会依赖mysql chart，而release中不包含，故需要自己处理
>
> ```bash
> $ tar -xzvf v0.1.tar.gz
> $ cd k8s-gerrit-0.1/helm-charts/gerrit-master
> $ mkdir charts
> $ cd charts
> $ helm fetch stable/mysql
> $ tar -xzvf mysql-1.6.2.tgz
> ```
>
>

- 安装gerrit-master(由于安装后配置较多，启动ldap，参考下面template后安装)

```bash
$ cd k8s-gerrit-0.1/helm-charts
$ helm install gerrit-master \
--namespace jenkins \
--set storageClasses.default.name=nfs-client \
--set storageClasses.shared.name=nfs-client \
--set gerritMaster.keystore="Z2Vycml0Cg==" \
--set gerritMaster.ingress.host=gerrit.devops.na \
--set mysql.persistence.storageClass=nfs-client \
./gerrit-master
```

> **注意**:
>
> `gerritMaster.keystore ` chart中默认为`gerrit`,此处转换成base64
>
> ```bash
> $ echo gerrit|base64
> ```
>
> 错误解决
>
> 1. 运行完了后会报错如下：
>
> `Error: failed to start container "gerrit-master": Error response from daemon: OCI runtime create failed: container_linux.go:349: starting container process caused "exec: \"/bin/bash\": stat /bin/bash: no such file or directory": unknown`
>
> 解决：
>
> 将deployment下的command中`/bin/bash` 更改成`sh`

#### 2.1.2 yaml 安装

- 创建ssh key

此ssh key为gerrit连接gitlab使用，生成后将public key添加到gitlab的管理员中

> 注意:
>
> gerrit使用的key为`-----begin RSA private key---`开头格式，不可以为`-----Begin openssh private key---`,否则会报错`Invalid privatekey`
>
> 生成密钥命令为
>
> ```bash
> ssh-keygen -t rsa -m PEM
> ```
>
>

创建k8s资源文件以备使用

```bash
$ cat sshkey-configmap.yaml
apiVersion: v1
data:
  id_rsa: |
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEAsBIVI64umIlOefyqMP+TUGcZ1b5hyZnvjFsKuUxN3OBAnLlc
    G/dXHpXO1aMkErPaijFhDq/2e0G9yOXne2H03neX9I79etyEweRn4wNEt6FYHSO6
    wIoCpHv42cUwhc1di+baUyYljUEd3e70I9kfn9v8q74i0yuxaw9vwwDFxZYTLnGl
    NaBozY4Ot1WiGRuhTtcHfB2ikx/0ofhDKxGITUuffBzu0JaTY+nDvevP1zyj5Y+k
    sBY5gJg1mfWhFGmklxqPSGX1pZbVbqFtPJTyPWIiZlBj3A3aJ/ifaK0nA89LyEkh
    filhi5fIK8zIilevcR2NrFOE8uPhfhm4BFLstwIDAQABAoIBAAkiZwZa2v/+35tk
    iE/ScooRTz3PirMy0UKu1neWhdOXrsJyu1yYEGT90KU0uYJe4k+ZEDf62Qa7+mSZ
    yNCV3GZOPgvp/yyIlDqa5ujYGpCMRk38tIuYnVp7NAXYPRQz+05uw5TcvcX+QtZo
    tzHE6ecZOpqOWrAp7CJpwpqTR69ga8auZLL6OBSozEfS12+lbccrizSKj0aFdZj5
    JkjxKC3GcR7x964W1goOOO7nMOtM7NQijNy+45W2HSrylaAdeXR2EhkXdkN41C6w
    dTBzWFU24rrCeIJ/xYE2DNoxOMIIXMgdSkh0iN5v1sblMgkHIlxSvbbrnpADI/mt
    RPyE28ECgYEA1lhxYLNMOi7ckRLYYGZ7b6SrmJ6pVjVvNHR5hAVBflhRKUp3Tytd
    j3+PePTAuP1lk2jW6pdrYaLztx3KQ/WQsElTwnKvQL/AKoY9eCr7s5vh6ZtVAkJl
    awVSM9RKPnZ4jMWwwVjR9DSTqWO402OZpY/sZKuybxU8wq/vvkqbYR0CgYEA0kl9
    TXDxKnzlOO1wqbeuPffGDxWkMvxa6A0P+Da4dnX8WhSAYZlJkfrr1iVRpzpg/lY3
    v5IPaUnwLHwwI7GWk8gXBYThhbGAjqIrC0mTw7Vv9mkCXQJyONl/acEbAZsbzgh2
    dppJeSnJTKfaqf2xvISl3PsbjXXrfQMTygFCEOMCgYEAi+7Hber/68gmJOvZqO2e
    zh8CC6whPu+5ftQ/dIn93N8JdVxxEZVtCj6ryrp2iw6a3qaGEDb0WpSIaVagcseB
    QyVL8ljuwuWsLdbfkHD3S6WL0fefgs8X6MtbhAymv6qWPsjdAQzcXIJqmB5LTm3q
    o+SW3o4c+loyBaPYbzHDyJ0CgYB969vIn7UKSDfK2wP33THNka1cIw1sKokVI51I
    eRwEc6plFpm519yiOas3VKmMhfzG0EIZyRDJb6kOOl9pQFJ2s/51I5kYQa5Fuv4j
    iC+fpeohl8f+ghR7zmHppeTSFVuVCKRHaSROFaOg73KVEIF57cSUixl4oAAfn9Qd
    xpsVGQKBgEDVOyojLGLGD2+xt710fJdiurVaYhZPWpNpWvjeCIxTZZQ1oaIxTB1b
    T13HRKVloRuiwiVba7GNn3XthP6iXDWq1RP0ykCY5XPgVzqrHz2tSG23fh+a3xhL
    7z+2PjP8rfBJj4huu6/DvbPX/0Z0levYNqWecACxoh5Lz6VjKgPS
    -----END RSA PRIVATE KEY-----
  id_rsa.pub: |
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCwEhUjri6YiU55/Kow/5NQZxnVvmHJme+MWwq5TE3c4ECcuVwb91celc7VoyQSs9qKMWEOr/Z7Qb3I5ed7YfTed5f0jv163ITB5GfjA0S3oVgdI7rAigKke/jZxTCFzV2L5tpTJiWNQR3d7vQj2R+f2/yrviLTK7FrD2/DAMXFlhMucaU1oGjNjg63VaIZG6FO1wd8HaKTH/Sh+EMrEYhNS598HO7QlpNj6cO968/XPKPlj6SwFjmAmDWZ9aEUaaSXGo9IZfWlltVuoW08lPI9YiJmUGPcDdon+J9orScDz0vISSF+KWGLl8grzMiKV69xHY2sU4Ty4+F+GbgEUuy3 root@gerrit-master-gerrit-master-deployment-7d5677b94f-wfxxd
  known_hosts: |
    10.108.249.222 ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDGeiOT8gllCV7n8VV49EBZnOs0DCVPT8mM39+UczmME3+44D5ulhcikypjybG4yfh7Q94QYxiJlW9puxG0NtWbE8iAnt1/NOLZrRKSf2TkpID9hib90tPSxxvyWjf5haApZhIozOe/4x+B9dmY83pi21Y6PjR3sq83qJF+VLozIzto8pYVCbmv82UftacwpIneOOxNv4J8XFB5Pvp3j9NWJ4PKGi9UTD2COmMhgI7hutTElNVCDBU/OH47g+AzBNZvNYLXsxp5C3wQlp2SgEj4hGAMVf+Js7aMEF5u9DvhJZpcz/Nfud6H6ZS+wk4aSz3Ch+RMg9aKB7DK0mHONAAT
    gitlab-pro-gitlab-shell.gitlab ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDGeiOT8gllCV7n8VV49EBZnOs0DCVPT8mM39+UczmME3+44D5ulhcikypjybG4yfh7Q94QYxiJlW9puxG0NtWbE8iAnt1/NOLZrRKSf2TkpID9hib90tPSxxvyWjf5haApZhIozOe/4x+B9dmY83pi21Y6PjR3sq83qJF+VLozIzto8pYVCbmv82UftacwpIneOOxNv4J8XFB5Pvp3j9NWJ4PKGi9UTD2COmMhgI7hutTElNVCDBU/OH47g+AzBNZvNYLXsxp5C3wQlp2SgEj4hGAMVf+Js7aMEF5u9DvhJZpcz/Nfud6H6ZS+wk4aSz3Ch+RMg9aKB7DK0mHONAAT
  config: |
   Host gitlab-pro-gitlab-shell.gitlab
   User Administrator
   IdentityFile ~/.ssh/id_rsa
   StrictHostKeyChecking no
   UserKnownHostsFile /dev/null
kind: ConfigMap
metadata:
  creationTimestamp: null
  name: ssh-key
  namespace: jenkins
# 以上key 可以使用自己生成的
$ kubectl apply -f sshkey-configmap.yaml
```

- 创建gerrit资源

```bash
$ cat gerrit.yaml
---
# Source: gerrit-master/templates/gerrit-master.secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name:  gerrit-master-gerrit-master-secure-config
  namespace: jenkins
  labels:
    app: gerrit-master
    chart: gerrit-master-0.1.0
    heritage: Helm
    release: gerrit-master
data:
  keystore: Z2Vycml0Cg==
  secure.config: IyBEYXRhYmFzZSBjcmVkZW50aWFscyBzaG91bGQgYmUgdGhlIHNhbWUgYXMgY29uZmlndXJlZCBmb3IgdGhlIGRhdGFiYXNlCiMgR2Vycml0LW1hc3RlciBjaGFydCwgaWYgdGhlIGRhdGFiYXNlIHdhcyBpbnN0YWxsZWQgdXNpbmcgdGhlIGNoYXJ0LgpbZGF0YWJhc2VdCiAgdXNlcm5hbWUgPSBnZXJyaXQKICBwYXNzd29yZCA9IHNlY3JldAoKIyBQYXNzd29yZCBmb3IgdGhlIGtleXN0b3JlIGFkZGVkIGFzIHZhbHVlIGZvciAnZ2Vycml0TWFzdGVyLmtleXN0b3JlJwpbaHR0cGRdCiAgc3NsS2V5UGFzc3dvcmQgPSBnZXJyaXQKCiMgQ3JlZGVudGlhbHMgZm9yIHJlcGxpY2F0aW9uIHRhcmdldHMKIyBbcmVtb3RlICJzbGF2ZSJdCiMgdXNlcm5hbWUgPSBnaXQKIyBwYXNzd29yZCA9IHNlY3JldA==
type: Opaque
---
# Source: gerrit-master/templates/gerrit-master.configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gerrit-master-gerrit-master-configmap
  namespace: jenkins
  labels:
    app: gerrit-master
    chart: gerrit-master-0.1.0
    heritage: Helm
    release: gerrit-master
data:
  gerrit.config: |-
    [gerrit]
      basePath = git # FIXED
      # The canonical web URL has to be set to the Ingress host, if an Ingress
      # is used. If a LoadBalancer-service is used, this should be set to the
      # LoadBalancer's external IP. This can only be done manually after installing
      # the chart, when you know the external IP the LoadBalancer got from the
      # cluster.
      serverId = 97aedfdb-41b2-4c71-8d28-f85623356bcc
      canonicalWebUrl = http://gerrit.devops.na:30080/
      #disableReverseDnsLookup = true
    [plugin "events-log"]
      storeUrl = jdbc:h2:/var/gerrit/review_site/db/ChangeEvents
    [index]
      type = LUCENE
    [auth]
      type = LDAP
    [sshd]
      listenAddress = *:30022
    [httpd]
      listenUrl = http://*:8080/
    [user]
      name = Gerrit Code Review
      email = gerrit@example.com
      anonymousCoward = Unnamed User
    [cache]
      directory = cache
    [container]
      user = root
      javaHome = /usr/lib/jvm/java-1.8-openjdk/jre # FIXED
      javaOptions = "-Dflogger.backend_factory=com.google.common.flogger.backend.log4j.Log4jBackendFactory#getInstance"
      javaOptions = "-Dflogger.logging_context=com.google.gerrit.server.logging.LoggingContext#getInstance"
      #javaOptions = -Djavax.net.ssl.trustStore=/var/gerrit/etc/keystore # FIXED
      #javaOptions = -Xms200m
      # Has to be lower than 'gerritMaster.resources.limits.memory'. Also
      # consider memories used by other applications in the container.
      #javaOptions = -Xmx4g
    [ldap]
      server = ldap://ldap.devops.na
      username = cn=admin,dc=devops,dc=na
      password = admin
      accountBase = dc=devops,dc=na
      groupbase = dc=devops,dc=na
  replication.config: |-
    [remote "gitlab"]
      url = git@gitlab-pro-gitlab-shell.gitlab:aiops/${name}.git
      push = +refs/heads/*:refs/heads/*
      push = +refs/tags/*:refs/tags/*
      push = +refs/changes/*:refs/changes/*
      timeout=30
      threads=3
    # [remote "slave"]
    # url = http://gerrit-slave.example.com/git/${name}.git
    # replicationDelay = 0
    # timeout = 30
---
# Source: gerrit-master/templates/git-gc.storage.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gerrit-master-git-gc-logs-pvc
  namespace: jenkins
  labels:
    app: git-gc
    chart: gerrit-master-0.1.0
    heritage: Helm
    release: gerrit-master
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs-client
---
# Source: gerrit-master/templates/git-repositories.storage.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: gerrit-master-git-filesystem-pvc
  namespace: jenkins
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs-client
---
# Source: gerrit-master/templates/gerrit-master.service.yaml
apiVersion: v1
kind: Service
metadata:
  name: gerrit-master-gerrit-master-service
  namespace: jenkins
  labels:
    app: gerrit-master
    chart: gerrit-master-0.1.0
    heritage: Helm
    release: gerrit-master
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: ssh
    port: 29418
    targetPort: 30022
  selector:
    app: gerrit-master
  type: NodePort
---
# Source: gerrit-master/templates/gerrit-master.deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gerrit-master-gerrit-master-deployment
  namespace: jenkins
  labels:
    app: gerrit-master
    chart: gerrit-master-0.1.0
    heritage: Helm
    release: gerrit-master
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gerrit-master
  template:
    metadata:
      labels:
        app: gerrit-master
    spec:
      securityContext:
        fsGroup: 100
      containers:
      - name: gerrit-master
        image: openfrontier/gerrit:latest
        imagePullPolicy: Always
        env:
        - name: GERRIT_USER
          value: root
        command:
        - sh
        - -c
        args:
        - |
          mkdir /var/gerrit/review_site/etc
          cp /var/keystore /var/gerrit/review_site/etc/keystore
          cp /var/config/gerrit.config /var/gerrit/review_site/etc/gerrit.config
          cp /var/config/replication.config /var/gerrit/review_site/etc/replication.config
          cp /var/config/secure.config /var/gerrit/review_site/etc/secure.config
          ssh-keygen -t rsa -N ” -f ~/.ssh/id_rsa -q
          cat /root/ssh/id_rsa > /root/.ssh/id_rsa
          cat /root/ssh/id_rsa.pub > /root/.ssh/id_rsa.pub
          cp /root/ssh/known_hosts /root/.ssh/known_hosts
          cp /root/ssh/config /root/.ssh/config

          java -jar /var/gerrit/gerrit.war init \
              --batch \
              --install-plugin replication \
              --install-plugin commit-message-length-validator \
              --install-plugin download-commands \
              --install-plugin reviewnotes \
              -d /var/gerrit/review_site

          java -jar /var/gerrit/gerrit.war reindex \
              -d /var/gerrit/review_site

          JAVA_OPTIONS=$(git config --file /var/gerrit/review_site/etc/gerrit.config --get-all container.javaOptions)
          java ${JAVA_OPTIONS} -jar /var/gerrit/gerrit.war daemon -d /var/gerrit/review_site --enable-httpd &
          tail -F -n +1 /var/gerrit/logs/{error,httpd,sshd}_log
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: git-filesystem
          mountPath: "/var/gerrit/review_site"
        - name: gerrit-config
          mountPath: "/var/config/gerrit.config"
          subPath: gerrit.config
        - name: gerrit-config
          mountPath: "/var/config/replication.config"
          subPath: replication.config
        - name: gerrit-master-secure-config
          mountPath: "/var/config/secure.config"
          subPath: secure.config
        - name: gerrit-master-secure-config
          mountPath: "/var/keystore"
          subPath: keystore
        - name: ssh-key
          mountPath: /root/ssh
        resources:
          limits:
            cpu: 1
            memory: 6Gi
          requests:
            cpu: 1
            memory: 5Gi
      volumes:
      - name: git-filesystem
        persistentVolumeClaim:
          claimName: gerrit-master-git-filesystem-pvc
      - name: ssh-key
        configMap:
          name: ssh-key
      - name: gerrit-config
        configMap:
          name: gerrit-master-gerrit-master-configmap
      - name: gerrit-master-secure-config
        secret:
          secretName: gerrit-master-gerrit-master-secure-config
---
# Source: gerrit-master/templates/git-gc.cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: gerrit-master-git-gc
  namespace: jenkins
  labels:
    app: git-gc
    chart: gerrit-master-0.1.0
    heritage: Helm
    release: gerrit-master
spec:
  schedule: "0 6,18 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          securityContext:
            runAsUser: 1000
            fsGroup: 100
          imagePullSecrets:
          - name: image-pull-secret
          containers:
          - name: git-gc
            imagePullPolicy: Always
            image: k8sgerrit/git-gc:latest
            command:
            - /var/tools/gc-all.sh
            resources:
              limits:
                cpu: 100m
                memory: 256Mi
              requests:
                cpu: 100m
                memory: 256Mi
            volumeMounts:
            - name: git-filesystem
              mountPath: "/var/gerrit/git"
            - name: git-gc-logs
              mountPath: "/var/log/git"
          volumes:
          - name: git-filesystem
            persistentVolumeClaim:
              claimName: gerrit-master-git-filesystem-pvc
          - name: git-gc-logs
            persistentVolumeClaim:
              claimName: gerrit-master-git-gc-logs-pvc
---
# Source: gerrit-master/templates/gerrit-master.ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    kubernetes.io/ingress.provider: nginx
  name: gerrit-master-gerrit-master-ingress
  namespace: jenkins
  labels:
    app: gerrit-master
    chart: gerrit-master-0.1.0
    heritage: Helm
    release: gerrit-master
spec:
  rules:
  - host: gerrit.devops.na
    http:
      paths:
      - backend:
          serviceName: gerrit-master-gerrit-master-service
          servicePort: 80
# 文件太大可拆分多个小文件保存      
$ kubectl apply -f gerrit.yaml
```



#### 配置中注意问题

- git push代码到gerrit

```bash
git push origin master:refs/for/master
# 若无法push，执行下面命令，删除change id
$ git commit --amend
```

- 安装插件

登陆jenkins管理界面，安装plugin `Gerrit Trigger`

- 添加gerrit-server

> 由于我的环境中gerrit-server和jenkins都是在k8s内，故在jenkins中添加gerrit-server时使用gerrit-server svc的名称，注意端口为: 29418

点击`Manage Jenkins`->`Gerrit Trriger`->`Add New Server`



<img src="../image/gerrit2.png" alt="image-20200422141658470"  width="400" />

- gerrit添加verified 标签功能

登陆管理员，选择`BROWSE`->`Repositories`->`All-Projects`->`Access`,查看`Reference: refs/heads/*`选项是否包含如下标签，若没有则添加

<img src="../image/gerrit3.png" alt="image-20200422141658470"  width="400" />



> 添加方式为,选择顶端`EDIT`,找到`Reference: refs/heads/*`最下端`Add permission`
>
> <img src="../image/gerrit4.png" alt="image-20200422142212757" width="400" />
>
> 若此处没有这个标签，则按照如下方式添加：
>
> ```bash
> $ git init cfg; cd cfg
> $ git config --global user.name 'wanghq'
> $ git config --global user.email '845789131@qq.com'
> $ git remote add origin ssh://wanghq@gerrit.devops.na:30022/All-Projects
> $ git pull origin refs/meta/config
> $ vi project.config
> [label "Verified"]
>         function = MaxWithBlock
>         value = -1 Fails
>         value = 0 No score
>         value = +1 Verified
> $ git commit -a -m "Updated permissions"
> $ git push origin HEAD:refs/meta/config
> ```
>
> 更改后，需要重启gerrit
>
> 配置参考
>
> 参考 :https://www.cnblogs.com/kevingrace/p/5651447.html

- gerrit中将jenkins用户添加至`Non-Interactive Users`组

登陆管理员，选择`BROWSE`->`Repositories`->`Groups`->`Non-Interactive Users`->`Members`添加jenkins到`Non-Interactive Users`组
