- **question 1**

Set configuration context `$ kubectl config use-context k8s` Monitor the logs of Pod foobar and

1. Extract log lines corresponding to error file-not-found
2. Write them to /opt/KULM00201/foobar

Question weight 5%



*Set configuration context $ kubectl config use-context k8s*

*Monitor the logs of Pod ***foobar\*** and Extract log lines corresponding to error ***unable to access website\***. Write them to /opt/KULM00201/foobar .*



answer

```bash
$ kubectl logs foobar | grep "Getting the checksum" > /opt/KULM00201/foobar
```



- **question 2**

Set configuration context $ `kubectl config use-context k8s`

List all PVs sorted by ***\*name\**** saving the full kubectl output to /opt/KUCC0010/my_volumes . Use kubectl’s own functionally for sorting the output, and do not manipulate it any further.

Question weight 3%



*List all PVs sorted by name, saving the full kubectl output to /opt/KUCC0010/my_volumes* 

*Use kubectl’s own functionality for sorting the output, and do not manipulate it any further.*



answer

```bash
$ kubectl get pv --sort-by=.metadata.name > /opt/KUCC0010/my_volumes
```





- **question 3**

Set configuration context `$ kubectl config use-context k8s`

Ensure a single instance of Pod nginx is running on each node of the kubernetes cluster where nginx also represents the image name which has to be used. Do no override any taints currently in place.

Use ***\*Daemonsets\**** to complete this task and use ***\*ds.kusc00201\**** as Daemonset name. Question weight 3%



answer

```bash
# 可以先创建一个deployment，然后修改，节省时间
$ kubectl create deploy ds.kusc00201 --image=nginx --dry-run=client -o yaml > daemonset.yaml

$ cat daemonset.yaml
apiVersion: apps/v1
#kind: Deployment
#kind: DaemonSet 更改成DaemonSet类型
metadata:
  creationTimestamp: null
  labels:
    app: ds.kusc00201
  name: ds.kusc00201
spec:
#  replicas: 1 删除
  selector:
    matchLabels:
      app: ds.kusc00201
#  strategy: {} 删除
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: ds.kusc00201
    spec:
      containers:
      - image: nginx
        name: nginx
#        resources: {} 删除
#status: {} 删除
```



- **question 4**

Set configuration context `$ kubectl config use-context k8s` Perform the following tasks

1. Add an init container to `lumpy-koala` (Which has been defined in spec file `/opt/kucc00100/pod-spec-KUCC00100.yaml`)
2. The init container should create an empty file named `/workdir/calm.txt`
3. If `/workdir/calm.txt` is not detected, the Pod should exit
4. Once the spec file has been updated with the init container definition, the Pod should be created.

Question weight 7%



answer

```bash
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: lumpy-koala
  name: lumpy-koala
spec:
  initContainers:
  - image: busybox
    name: install
    command:
    - touch
    - "/workdir/calm.txt"
    volumeMounts:
    - name: workdir
      mountPath: /workdir
  containers:
  - image: nginx
    name: lumpy-koala
    command:
    - sh
    - -c
    - "[ -f /workdir/calm.txt ] && sleep 36000 || exit"
    volumeMounts:
    - name: workdir
      mountPath: /workdir
  volumes:
  - name: workdir
    emptyDir: {}
```



- question 5

Set configuration context $ kubectl config use-context k8s

Create a pod named `kucc4` with a single container for each of the following images running inside (there may be between 1 and 4 images specified): nginx + redis + memcached + consul

Question weight: 4%



answer

```bash
# 先创建一个pod文件，然后添加其他几个container
$ kubectl run kucc4 --image=nginx --dry-run=client -o yaml > kucc4.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: kucc4
  name: kucc4
spec:
  containers:
  - image: nginx
    name: nginx
  - image: redis
    name: redis
  - image: memcached
    name: memcached
  - image: consul
    name: consul
```



- question 6

Set configuration context `$ kubectl config use-context k8s` Schedule a Pod as follows:

1. Name: nginx-kusc00101
2. Image: nginx
3. Node selector: disk=ssd 

Question weight: 2%ku



answer

```bash
$ kubectl run nginx-kusc00101 --image nginx --dry-run=client -o yaml > nginx-kusc00101.yaml
# 先创建pod文件，然后添加nodeSelector
$cat nginx-kusc00101.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx-kusc00101
  name: nginx-kusc00101
spec:
  nodeSelector:
    disk: ssd
  containers:
  - image: nginx
    name: nginx-kusc00101
```



- question 7

Set configuration context `$ kubectl config use-context k8s` Create a deployment as follows

1. Name: nginx-app
2. Using container nginx with version 1.10.2-alpine
3. The deployment should contain 3 replicas

Next, deploy the app with new version 1.13.0-alpine by performing a rolling update and record that update.

Finally, rollback that update to the previous version 1.10.2-alpine 

Question weight: 4%



answer

```bash
$ kubectl create deploy nginx-app --image=nginx:1.10.2-alpine --dry-run=client -o yaml > nginx-app.yaml
# 将replicas 改成 3
$ kubectl apply -f nginx-app.yaml --record
$ kubectl set image deploy nginx-app nginx=nginx:1.13.0-alpine
$ kubectl rollout undo deploy/nginx-app
```



- question 8

Set configuration context `$ kubectl config use-context k8s`

Create and configure the service front-end-service so it’s accessible through NodePort and routes to the existing pod named front-end

Question weight: 4%



```bash
$ kubectl expose pod lumpy-koala --name front-end-service --port=80 --target-port=80 --dry-run=client -o yaml > front-end-service.yaml
# 添加 type: NodePort
cat front-end-service.yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: lumpy-koala
  name: front-end-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  type: NodePort
  selector:
    run: lumpy-koala
$ kubectl apply -f front-end-service.yaml
```



- question 9

Set configuration context `$ kubectl config use-context k8s` Create a Pod as follows:

1. Name: jenkins
2. Using image: jenkins
3. In a new Kubenetes namespace named website-frontend 
4. Question weight 3%



answer

```bash
$ kubectl create ns website-frontend
$ kubectl run jenkins --image=jenkins --namespace website-frontend
```



- question 10

Set configuration context `$ kubectl config use-context k8s` Create a deployment spec file that will:

1. Launch 7 replicas of the redis image with the label: app_env_stage=dev
2. Deployment name: kual00201

Save a copy of this spec file to /opt/KUAL00201/deploy_spec.yaml (or .json)

When you are done, clean up (delete) any new k8s API objects that you produced during this task

Question weight: 3%



answer

```bash
$ kubectl create deploy kual00201 --image=redis --dry-run=client -o yaml > /opt/KUAL00201/deploy_spec.yaml

# 修改，replicas 1--> 7;添加label 共3处
$ cat /opt/KUAL00201/deploy_spec.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: kual00201
    app_env_stage: dev
  name: kual00201
spec:
  replicas: 7
  selector:
    matchLabels:
    	app_env_stage: dev
      app: kual00201
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app_env_stage: dev
        app: kual00201
    spec:
      containers:
      - image: redis
        name: redis
        resources: {}
status: {}

# 创建测试，然后删除资源
$ kubectl apply -f /opt/KUAL00201/deploy_spec.yaml
$ kubectl delete -f /opt/KUAL00201/deploy_spec.yaml
```



- question 11

Set configuration context `$ kubectl config use-context k8s`

Create a file /opt/KUCC00302/kucc00302.txt that lists all pods that implement Service ***\*foo\**** in Namespace ***\*production\****.

The format of the file should be one pod name per line.

Question weight: 3%



answer

```bash
$ kubectl get svc foo -n production -o custom-columns=NAME:.metadata.name,Selector:.spec.selector

# 查到label之后可以直接列出相关pod 名称,若label多个，则用空格隔开'app=kual00201,app_env_stage=dev'
$ kubectl get pod -l 'run=lumpy-koala' -o custom-columns=NAME:.metadata.name |grep -v NAME
```





- quesiton 12

Set configuration context `$ kubectl config use-context k8s` Create a Kubernetes Secret as follows:

1. Name: super-secret
2. Credential: alice or username:bob 

Create a Pod named ***pod-secrets-via-file\*** using the ***redis\*** image which mounts a secret named ***super-secret\*** at /secrets

Create a second Pod named ***pod-secrets-via-env\*** using the ***redis\*** image, which exports credential/username as ***TOPSECRET / CREDENTIALS\***

Question weight: 9%



answer

```bash
$ kubectl create secret generic super-secret --from-literal=username=bob
$ kubectl run pod-secrets-via-file --image=redis --dry-run=client -o yaml > pod-secrets-via-file.yaml
$ kubectl run pod-secrets-via-file --image=redis --dry-run=client -o yaml > pod-secrets-via-env.yaml

#创建文件使用
$ cat pod-secrets-via-file.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod-secrets-via-file
  name: pod-secrets-via-file
spec:
  containers:
  - image: redis
    name: pod-secrets-via-file
    volumeMounts:
    - name: super-secret
      mountPath: /secrets
      readOnly: true
    resources: {}
  volumes:
  - name: super-secret
    secret:
      secretName: super-secret
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

$ cat pod-secrets-via-env.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod-secrets-via-env
  name: pod-secrets-via-env
spec:
  containers:
  - image: redis
    env:
    - name: CREDENTIALS
      valueFrom:
        secretKeyRef:
          name: super-secret
          key: username
    name: pod-secrets-via-file
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```



- question 13

Set configuration context `$ kubectl config use-context k8s` Create a pod as follows:

1. Name: non-persistent-redis
2. Container image: redis
3. Named-volume with name: cache-control
4. Mount path: /data/redis

It should launch in the `pre-prod` namespace and the volume MUST NOT be persistent.

Question weight: 4%



answer

```bash
$ kubectl run non-persistent-redis --image=redis --dry-run=client -o yaml > non-persistent-redis.yaml

$ cat non-persistent-redis.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: non-persistent-redis
  name: non-persistent-redis
spec:
  containers:
  - image: redis
    name: non-persistent-redis
    volumeMounts:
    - name: cache-control
      mountPath: /data/redis
    resources: {}
  volumes:
  - name: cache-control
    emptyDir: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

$ kubectl apply -f non-persistent-redis.yaml -n pre-prod


```



> 类似题型

创建一个pod名称为test，镜像为nginx，Volume名称cache-volume为挂在在/data目录下，且Volume是non-Persistent的

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod 
spec:
  containers:
  - name: test
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

### 



- question 14

Set configuration context `$ kubectl config use-context k8s` Scale the deployment `webserver` to 6 pods

Question weight: 1%



answer

```bash
$ kubectl scale deployment/webserver --replicas=6
```





- question 15

Set configuration context `$ kubectl config use-context k8s`

Check to see how many nodes are ready (not including nodes tainted NoSchedule) and write the number to ***\*/opt/nodenum\****

Question weight: 2%



answer

```bash
# 查看NoSchedule的label
$ kubectl get no -o custom-columns=name:.metadata.name,taints:.spec.taints,Ready:.status.conditions[4].type

# 数完了写进去,注意不包含带有taints的node
```





- question 16

Set configuration context $ `kubectl config use-context k8s`

From the Pod label ***\*name=cpu-utilizer\****, find pods running high CPU workloads and write the name of the Pod consuming most CPU to the file /opt/cpu.txt (which already exists)

Question weight: 2%



answer

```bash
$ kubectl top pod -l name=cpu-utilizer --sort-by=cpu

# 找到CPU最高的写进 /opt/cpu.txt
```



- question 17

Set configuration context `$ kubectl config use-context k8s` Create a deployment as follows

1. Name: nginx-dns
2. Exposed via a service: nginx-dns
3. Ensure that the service & pod are accessible via their respective DNS records
4. The container(s) within any Pod(s) running as a part of this deployment should use the ***\*nginx\**** image

Next, use the utility nslookup to look up the DNS records of the service & pod and write the output to /opt/service.dns and /opt/pod.dns respectively.

Ensure you use the busybox:1.28 image(or earlier) for any testing, an the latest release has an unpstream bug which impacts thd use of nslookup.

Question weight: 7%



answer

```bash
$ kubectl create deploy nginx-dns --image=nginx
$ kubectl expose deploy nginx-dns --port=80 --target-port=80 --name nginx-dns

# 启动nslookup
$ kubectl run nslookup-dns --image=busybox:1.28 --command -- sleep 3600

# 扫描svc
$ kubectl exec -ti nslookup-dns -- nslookup nginx-dns > /opt/service.dns 

# 查看pod的IP
$ kubectl get pod -o wide

# 扫描pod
$ kubectl exec -ti nslookup-dns -- nslookup 10-244-186-255.nginx-dns
```



- question 18

No configuration context change required for this item

Create a snapshot of the etcd instance running at https://127.0.0.1:2379 saving the snapshot to the file path /data/backup/etcd-snapshot.db

The etcd instance is running etcd version 3.1.10

The following TLS certificates/key are supplied for connecting to the server with etcdctl

1. CA certificate: /opt/KUCM00302/ca.crt
2. Client certificate: /opt/KUCM00302/etcd-client.crt
3. Clientkey:/opt/KUCM00302/etcd-client.key 

Question weight: 7%



```bash
$ ETCDCTL_API=3
$ etcdctl --endpoint=https://127.0.0.1:2379 --cacert=/opt/KUCM00302/ca.crt --cert=/opt/KUCM00302/etcd-client.crt --key=/opt/KUCM00302/etcd-client.key sanpshot save  /data/backup/etcd-snapshot.db

#检查
$ etcdctl --endpoint=https://127.0.0.1:2379 --cacert=/opt/KUCM00302/ca.crt --cert=/opt/KUCM00302/etcd-client.crt --key=/opt/KUCM00302/etcd-client.key snapshot status  /data/backup/etcd-snapshot.db
```





- question 19

Set configuration context `$ kubectl config use-context ek8s`

Set the node labelled with name=ek8s-node-1 as unavailable and reschedule all the pods running on it.

Question weight: 4%



answer

```bash
$ kubectl get node -l name=ek8s-node-1

# --delete-local-data --ignore-daemonsets --force 可以根据提示按需添加
$ kubectl drain node02 --delete-local-data --ignore-daemonsets --force
```



- question 20

Set configuration context `$ kubectl config use-context wk8s`

A Kubernetes worker node, labelled with ***\*name=wk8s-node-0\**** is in state NotReady . Investigate why this is the case, and perform any appropriate steps to bring the node to a Ready state, ensuring that any changes are made permanent.

Hints:

1. You can ssh to the failed node using $ ssh wk8s-node-0
2. You can assume elevated privileges on the node with the following command $ sudo -i Question weight: 4%



answer

```bash
$ kubectl get node
$ ssh wk8s-node-0
$ sudo -i

# 查看kubelet状态，按照修改，一般是这个服务被启动
$ systemctl status kubelet
$ systemctl start kubelet

$ systemctl enable kubelet

#检查
$ kubectl get node
```





- question 21

Set configuration context `$ kubectl config use-context wk8s`

Configure the kubelet systemd managed service, on the node labelled with ***\*name=wk8s-node-1\****, to launch a Pod containing a single container of image nginx named myservice automatically. Any spec files required should be placed in the /etc/kubernetes/manifests directory on the node.

Hints:

1. You can ssh to the failed node using $ ssh wk8s-node-1
2. You can assume elevated privileges on the node with the following command $ sudo -i Question weight: 4%





answer

```bash
$ kubectl get node -l name=wk8s-node-1

#书写静态pod，放在目录 /etc/kubernetes/manifests
$ ssh wk8s-node-1
$ sudo -i 

$ kubectl run myservice --image=nginx --prot=80 --dry-run=client -o yaml > myservice.yml

$ sudo cp myservice.yml /etc/kubernetes/manifests/
```



- question 22

Set configuration context `$ kubectl config use-context ik8s`

In this task, you will configure a new Node, ***\*ik8s-node-0\****, to join a Kubernetes cluster as follows:

1. Configure kubelet for automatic certificate rotation and ensure that both server and client CSRs are automatically approved and signed as appropnate via the use of RBAC.
2. Ensure that the appropriate cluster-info ConfigMap is created and configured appropriately in the correct namespace so that future Nodes can easily join the cluster
3. Your bootstrap kubeconfig should be created on the new Node at /etc/kubernetes/bootstrap-kubelet.conf (do not remove this file once your Node has successfully joined the cluster)
4. The appropriate cluster-wide CA certificate is located on the Node at /etc/kubernetes/pki/ca.crt . You should ensure that any automatically issued certificates are installed to the node at /var/lib/kubelet/pki and that the kubeconfig file for kubelet will be rendered at /etc/kubernetes/kubelet.conf upon successful bootstrapping
5. Use an additional group for bootstrapping Nodes attempting to join the cluster which should be called system:bootstrappers:cka:default-node-token
6. Solution should start automatically on boot, with the systemd service unit file for kubelet available at /etc/systemd/system/kubelet.service

To test your solution, create the appropriate resources from the spec file located at /opt/..../kube-flannel.yaml This will create the necessary supporting resources as well as the kube-flannel -ds DaemonSet . You should ensure that this DaemonSet is correctly deployed to the single node in the cluster.

Hints:

1. kubelet is not configured or running on ***\*ik8s-master-0\**** for this task, and you should not attempt to configure it.
2. You will make use of TLS bootstrapping to complete this task.
3. You can obtain the IP address of the Kubernetes API server via the following command $ ssh ik8s-node-0 getent hosts ik8s-master-0
4. The API server is listening on the usual port, 6443/tcp, and will only server TLS requests
5. The kubelet binary is already installed on ***\*ik8s-node-0\**** at /usr/bin/kubelet . You will not need to deploy kube-proxy to the cluster during this task.
6. You can ssh to the new worker node using $ ssh ik8s-node-0
7. You can ssh to the master node with the following command $ ssh ik8s-master-0
8. No further configuration of control plane services running on ***\*ik8s-master-0\**** is required
9. You can assume elevated privileges on both nodes with the following command $ sudo -i
10. Docker is already installed and running on ***\*ik8s-node-0\****

Question weight: 8%



answer

```bash

```







- question 23

Set configuration context `$ kubectl config use-context bk8s`

Given a partially-functioning Kubenetes cluster, identify symptoms of failure on the cluster. Determine the node, the failing service and take actions to bring up the failed service and restore the health of the cluster. Ensure that any changes are made permanently.

The worker node in this cluster is labelled with ***\*name=bk8s-node-0\**** Hints:

1. You can ssh to the relevant nodes using $ ssh $(NODE) where $(NODE) is one of ***\*bk8s-master-0\**** or ***\*bk8s-node-0\****
2. You can assume elevated privileges on any node in the cluster with the following command$ sudo -i 

Question weight: 4%



answer

```bash
$ kubectl get cs
# 查看systemd管理的service，启动失败的service
$ ls ls /etc/systemd/system/
$ ssh bk8s-master-0
$ docker ps #检查是否有的服务没起来
# 若是apiserver等服务没起来，检查/etc/kubernetes/manifests里面是否有相关文件，若有怎检查kubelet的配置文件是否未涵盖staticPodPath
```



- question 24

Set configuration context `$ kubectl config use-context hk8s`

Creae a persistent volume with name app-config of capacity 1Gi and access mode ReadWriteOnce. The type of volume is hostPath and its location is /srv/app-config

Question weight: 3%



answer

```bash
# 不知道如何写，可以使用kubectl explain查询
$ cat app-config
apiVersion: v1
kind: Persistentvolume
metadata:
  name: app-config
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/srv/app-config"

$ kubectl apply -f 
```

