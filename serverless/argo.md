


argo是k8s上原生工作流引擎，可以直接编排k8s上面的各个资源。

## 1. 安装



```bash
kubectl create ns argo
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo/stable/manifests/quick-start-postgres.yaml
```

在安装好的k8s上，可以快速启动argo,主要组件包含`argo-server` 、`workflow-controller`、`postgresql`(可以使用其他数据库),postgresql数据库是用来存放执行过的workflow归档信息，如果不需要也可以没有。

安装argo客户端，可以方便操作argo

下载连接地址为:
https://github.com/argoproj/argo/releases

直接在k8s的客户端下使用，是调用k8s api。有些功能不全，需要配置相关环境变量使其连接argo server才能操作全部功能。
```bash
export ARGO_SERVER=10.233.49.21:2746

export ARGO_NAMESPACE=default
export ARGO_TOKEN='Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImhlNy1FOVB6b1I0UFBacHlCakl5eWNCN1FpcGhieEpCcWRNNU1mZVVWcUUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImFyZ28tc2VydmVyLXRva2VuLXYybDU1Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFyZ28tc2VydmVyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiOWY0ZmI5MjAtNDM4Yi00OWUwLTg0MjktYmIwMjViN2NmOTk3Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6YXJnby1zZXJ2ZXIifQ.cDKG659P-QDpaBYV3I2QC6K4tDOXnR0bBbnXnEpgyPy7NyfvibECqTIFFtQtaHhRDCbvLv9uVXMaQNweNEPcKpzJ1mDVoODwWntQADxj5uep-wJq2-i1fdH2BpY5liiIZQl114wfVwlSzbSQz9rkFugGIfKqGZmPMO0enpshks15ZALZALnDhLD5OrBVltVs82Yx912cKL038HOi_1K3ZPxQ2ogCQh-BsRd3hTGL-owmZS0H_x_np9awctpuFrE5icEkqPvkyQ4yE3fDNIMbuPxd2VI8cHE3_K_2kUR8L9QCW0YsPLGV_Teq2Oi8LfsK2CxhhNWzQOSlwYgflZZsAw'

# token 是k8s serviceaccount 对应的secret,注意前面需要加 Bearer
```

## 2. 运行workflow

workflow主要是通过一个yaml文件定义
```yaml
metadata:
  namespace: default
  name: failover-grafana
spec:
  ## 定义template 入口
  entrypoint: failover
  templates:
  - name: failover
    # 定义step 的运行逻辑
    steps:
    - - name: stop-master-grafana
        template: runscript
        arguments:
          parameters:
          - name: host
            value: "3.1.20.108"
          - name: password
            value: "password"
          - name: cmd
            value: "systemctl stop grafana-server"
          - name: user
            value: "root"
      - name: stop-master-db
        template: runscript
        arguments:
          parameters:
          - name: host
            value: "3.1.20.108"
          - name: password
            value: "password"
          - name: cmd
            value: "systemctl stop mariadb"
          - name: user
            value: "root"
    - - name: start-slave-grafana
        template: runscript
        arguments:
          parameters:
          - name: host
            value: "3.1.20.129"
          - name: password
            value: "123456"
          - name: cmd
            value: "systemctl start grafana-server"
          - name: user
            value: "root"
  ## 定义一个template, 这个template是通过sshpass在远程执行命令
  - name: runscript
    inputs:
      parameters:
      - name: host
      - name: password
      - name: cmd
      - name: user
    container:
      image: 'ictu/sshpass'
      command:
      - sshpass
      args:
      - -p
      - '{{inputs.parameters.password}}'
      - ssh
      - -o
      - 'StrictHostKeyChecking no'
      - '{{inputs.parameters.user}}@{{inputs.parameters.host}}'
      - '{{inputs.parameters.cmd}}'
```

运行
```bash
argo submit run.yaml
```

> 此事例模拟的是一个grafana切换的流程


更多文件内容参考 https://argoproj.github.io/argo/quick-start/