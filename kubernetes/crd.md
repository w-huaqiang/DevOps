# comment

## 1. Custom Resource

k8s的强大原因之一离不开其灵活的扩展性，以至于CNCF发展的这么好。在同mesos竞争的时候，正是因为其良好的扩展性，催发了一些优秀的项目，如istio,rook才让其脱颖而出。

kubernetes的扩展有两种方式，一种为创建Custom Resource，另一种是通过聚合层apiserver-aggregation。通过Custom Resource这种方式能够让我们体验如同操作k8s内置资源一样的感觉。

当然，一个完整的CRD应该包含 controller + CRD。其中CRD定义可以方便灵活的通过kubectl创建，而controller需要额外的开发。

## 1.1 定义

我们可以轻松通过熟悉的yaml文件定义CRD。当创建CRD时，kubernetes API服务器会为我们所指定的每一个版本生成一个RESTful资源路径。

如下例子:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: crontabs.stable.example.com
spec:
  # 组名称，用于 REST API: /apis/<组>/<版本>
  group: stable.example.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 每个版本都可以通过 served 标志来独立启用或禁止
      served: true
      # 其中一个且只有一个版本必需被标记为存储版本
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # 可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 名称的复数形式，用于 URL：/apis/<组>/<版本>/<名称的复数形式>
    plural: crontabs
    # 名称的单数形式，作为命令行使用时和显示时的别名
    singular: crontab
    # kind 通常是单数形式的驼峰编码（CamelCased）形式。你的资源清单会使用这一形式。
    kind: CronTab
    # shortNames 允许你在命令行使用较短的字符串来匹配资源
    shortNames:
    - ct
```

## 1.2 创建对象
创建玩CRD定义之后，我们就可以按照其规则创建定制的对象，即Custom Objects。在CRD定义中我们制定了`cronSpec`和`image`自定义字段。

创建
```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```
执行创建
```bash
kubectl apply -f my-crontab.yaml
```
此时便可查看创建的对象
```bash
kubectl get ct
```

> 我们可以对其使用`kubectl`所有基本元语。
> 从 apiextensions.k8s.io/v1beta1 转换到 apiextensions.k8s.io/v1 的 CRD 可能没有结构化的模式定义。应该将其转为v1


## 2. controller
controller源代码:
https://github.com/kubernetes/sample-controller/tree/master/pkg

