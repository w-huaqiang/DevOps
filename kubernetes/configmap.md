

## 1. configmap

configmap是一种k8s资源对象，使用k-v的形式来存储非敏感性的数据。pod可以通过环境变量，命令行参数，或者挂载的形式使用configmap。

由于container image这种不可变基础设施，我们没法将一些需要变化的信息存储在image中（当然可以存储，不过变化的时候就要重新build image，这将造成相当大的繁琐流程）。通过configmap 我们可以将容易变的数据独立出来，这样可以让image更轻便并且从环境中解耦。

> 注意: configmap不提供加密选项，若有加密的需求可以使用secret或者第三方解决方案

和k8s其他资源对象不同，configmap没有spec，而是使用`data`和`binaryData`字段。`data`和`binaryData`都是可选的。`data`字段用来保存`UTF-8`字节序列，而`binaryData`则被设计用来存储二进制数据。一般都是使用`data`，`binaryData`使用较少。

## 2. configmap资源定义

### 2.1 定义要求
- **name**

configmap名称必须是合法的`DNS子域名`

`DNS子域名要求`

  - 不能超过253个字符
  - 只能包含字母、数字，’-‘和‘.’
  - 必须以字母数字开头
  - 必须以字母数字结尾

> DNS标签名要求(RFC 1123定义)
>  - 最多63个字符
>  - 只能包含字母数字，‘-’
>  - 必须以字母数字开头
>  - 必须以字母数字结尾

- **key**
`data`或者`binaryData`字段下面的每个键的名称必须由字母、数字或者‘-’、‘_’、‘.’。若`data`和`binaryData`同时存在，则键的名称不能有重叠。

### 2.2 资源创建
1. 通过资源文件创建configmap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # 类属性键；每一个键都映射到一个简单的值
  player_initial_lives: "3"
  sleep_time: "3600"
  ui_properties_file_name: "user-interface.properties"

  # 类文件键
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
```
```bash
kubectl apply -f configmap.yaml
```

如上为一个configmap的事例，当为配置文件方式时一般使用'|'分割。

2. 通过cli快速创建

```bash
kubectl create configmap game-demo --from-file file.name

kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2

## 可以通过以下命令获取更多创建选项
## kubectl create configmap --help
```

## 3. configmap使用

configmap标准使用有三种方式:
- 容器的环境变量 （不会自动更新)
- 容器运行的命令行参数
- 在只读卷里面添加一个文件，让应用来读取 （内容会自动更新,更新的时间取决于kubelet周期性检查)

此外，如果有特殊需求，可以通过API来直接读取ConfigMap里面的内容，如跨namespace来使用configmap、订阅configmap变化时自动处理应用程序。

3中使用方式举例如下资源文件
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      # 2. 命令行参数使用
      command: ["sleep", "$(SLEEP_TIME)"]
      env:
        # 1. 定义环境变量使用方式
        - name: PLAYER_INITIAL_LIVES # 请注意这里是环境变量的key名称
          valueFrom:
            configMapKeyRef:
              name: game-demo           # 这个值来自 ConfigMap name
              key: player_initial_lives # 需要取值的configmap key
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
        - name: SLEEP_TIME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: sleep_time
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # 3. 你可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中
    - name: config
      configMap:
        # 提供你想要挂载的 ConfigMap 的名字
        name: game-demo
        # 来自 ConfigMap 的一组键，将被创建为文件,若不设置items,则默认按照5个键值生成5个文件
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"
```

configmap典型的用法就是给pod使用，当然也不一定暴露给pod，可以按照第四种使用方式来调整各种应用的行为。

## 4. 需要注意的点

1. configmap被pod使用，若configmap被更新，那么pod中的值是否会更新？在上面3种使用方式中，若以volume挂载的方式，当kubelet周期性检查时会更新pod里面的值。但是若以环境变量的方式使用，必须重启pod才能应用更新。
2. 设置configmap不可变。在k8s 1.19版本，新添加了一个特性`immutable`。当这只为true，configmap被创建之后将不可能更改，若要更新，必须删除重建。