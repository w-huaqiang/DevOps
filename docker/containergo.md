# content
- [content](#content)
    - [1. 对人员要求](#1-对人员要求)
    - [2. 一般准则](#2-一般准则)
### 1. 对人员要求

- 开发人员熟悉Docker虚拟化技术，熟练编写Dockerfile。
- 开发人员需要考虑后期容器编排部署的需求来组织结构和编写代码。（云原生架构）
- 部署人员需要熟悉kubernetes资源清单各参数含义，需要总体把控架构中从上到下各个模块，即其对应kubernetes的资源。
- 运维人员需考虑外部流量引入及后期扩容伸缩

### 2. 一般准则

•分离构建和运行环境

•使用dumb-int等避免僵尸进程

•不推荐直接使用pod，而是deployment/statefulSet等

•推荐容器应用日志打印到stdout和stderr，方便日志插件的处理 log

•由于容器采用了COW,大量写入会有性能问题，所以推荐使用Volume

•生产环境不推荐使用latest，但是开发环境推荐使用

•推荐使用Readliness探针检测服务是否真正起来了. Podredliness-liveness

•引入activeDeadlinSeconds避免快速失败的Job无限重启

•引入sidecar处理代理、请求速率控制和连接控制(service mesh)等