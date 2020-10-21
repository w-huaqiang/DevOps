# content
- [content](#content)
  - [1. 部署的历史](#1-部署的历史)
    - [1.1 容器带来的春风](#11-容器带来的春风)
  - [2. docker是什么](#2-docker是什么)
    - [2.1 运维角度看docker](#21-运维角度看docker)
      - [2.1.1 image](#211-image)
      - [2.1.2 container](#212-container)
    - [2.2 开发人员角度看docker](#22-开发人员角度看docker)
  - [3. docker历史](#3-docker历史)
  - [4. 几段爱恨情仇](#4-几段爱恨情仇)
    - [4.1 Docker和CoreOS](#41-docker和coreos)
    - [4.2 Docker和Mesosphere](#42-docker和mesosphere)
    - [4.3 Docker和 OCI](#43-docker和-oci)


## 1. 部署的历史

![dd](../image/runHistory.svg)

作为一个资深运维,对于应用部署我们经历了三个阶段:

- **物理机时代**：在初期我们会采购物理机，在物理机上安装操作系统，然后安装软件。为了做好隔离，使应用之间没有影响，我们会在每台物理机上安装一种应用。但是这样就会造成个问题，就是资源利用率特别低,大部分应用在物理机上的利用率不超过10%。并且随着科技的发展，内存、CPU等硬件设备的更新，单台物理机资源不断增大，使得单台物理只跑一个应用的资源利用率进一步变低。这时可能会想，在一台物理机上部署多个应用，这样资源利用率就提高了，但是这会造成一个新的问题，那就是应用隔离。不同应用部署在同一台物理机上，资源就会争用，其中一个应用就很容易对另外一个应用造成影响，其中还包括安全问题。
- **虚拟机时代**：针对物理机时代的问题，虚拟化技术应运而生。在物理机上安装虚拟机软件，可以将物理机虚拟出多个虚拟机，然后在不同虚拟机上安装不同的应用。这里我们会发现，首先我们需要购置物理机，然后在物理上安装操作系统和Hypervisor虚拟软件，紧接着通过Hypervisor虚拟出虚拟机(vm)，然后在虚拟机上安装操作系统，最后部署应用。这是我们不仅仅提高了资源利用率，也解决了应用之间的隔离问题。
- **容器化时代**：随着技术的进一步发展，大型分布式系统不断出现，应用发布频率不断提高，对IT资源提供的要求进一步提高，像虚拟机那种从操作系统级别部署的模式已经暴露出问题，比如：guest OS占用了大量资源；应用更新涉及到的开发测试环境很容易不一致；开发到运维流程的难以控制，虚拟机启动时间过长等。此时新一代基于内核虚拟化的容器技术普遍被大家接受，容器不在需要Hypersivor的硬件模拟，不再需要安装guest OS，而是直接共享宿主机内核，秒级别启动应用，同时通过内核态的namespace可以满足了不同应用之间的隔离要求。我们对于基础设施的需求不在关注于操作系统，而是将目光转移到了应用级别。

### 1.1 容器带来的春风

- 更高效的利用系统资源
- 快速的启动时间
- 容器镜像的不可变性，能更好的满足持续交付和持续部署的要求
- 不同云厂商，数据中间的迁移
- 方便和快捷的维护
- 大型的编排和调度

以下为容器和虚拟化简短对比

| 特性       | 容器         | 虚拟机         |
| ---------- | ------------ | -------------- |
| 启动       | 秒级         | 分钟级         |
| 存储使用   | 一般为MB     | 至少GB         |
| 性能       | 接近原生     | 弱于原生       |
| 系统支持两 | 单机可达千个 | 一般最多几十个 |



- [x] 容器为当前CI/CD、DevOps、serverless等先进技术理念提供了更好的技术土壤


## 2. docker是什么

Docker是以Docker容器为资源分割和调度的基本单位，封装整个软件运行时环境，为开发者和系统管理员设计的，用于构建、发布和运行分布式应用的平台。他是一个跨平台、可移植并且简单易用的容器解决方案。Docker的源码托管在GitHub上，基于Go语言开发并遵从Apache 2.0协议。Docker可在容器内部快速自动化部署应用，并通过操作系统内核技术（namespace、cgroups等)为容器提供资源隔离与安全保障。

那么docker是什么呢？

- docker是一种容器解决方案
- docker是一种基于内核虚拟化的容器技术
- docker可以跨平台运行在linux、Windows、MacOS之上


### 2.1 运维角度看docker

作为一个运维人员，我们实在是太喜欢一个这样的应用，下载一个zip（或者tar包），然后将其解压，里面包含一个运行的命令，直接`./`运行就跑起来，不需要管任何依赖。
那么从运维角度来看，docker满足了我们的需求，而且帮我们做的更多。

#### 2.1.1 image
docker中的image可以理解为这个zip包，这个image我们可以拷贝到任何地方，不仅如此，docker公司维护了一个repository，上面放了很多很多这种zip包，我们可以直接通过`docker pull`下载，而且允许我们自己搭建这种仓库。

#### 2.1.2 container
当我们把需要运行的一个docker image下载到本地后，我们就可以在他的目录中直接运行程序了，那个目录里包含了所有我们需要的库文件。此时我们需要思考一个问题，如果一个image只运行成一个container，那么将是不合理的。所以docker的容器运行有个`写时拷贝`的技术。我们可以理解为在image之上贴了个透明的玻璃，我们之后所有的操作都是在玻璃上完成的，并未对底层image造成任何影响。当需要创建另外一个container时，就加另外一个玻璃板，这样就可以一个image对应多个container了。

> 具体`image`和`container`将会在后续章节详细介绍

### 2.2 开发人员角度看docker

作为开发人员，其实更多的的经历应该放在如何提供一个可用的image，所以开发人员更多的经历是在写`Dockerfile`上。
Dockerfile 由一行行命令语句组成，最后使用`docker build`生成image。

如下
```bash
FROM alpine

LABEL maintainer="wanghq@bjzdgt.com"

# Install Node and NPM
RUN apk add --update nodejs nodejs-npm

# Copy app to /src
COPY . /src

WORKDIR /src

# Install dependencies
RUN  npm install

EXPOSE 8080

ENTRYPOINT ["node", "./app.js"]
```

我们可以将image理解成oop开发中的`类`,container理解成`实例`。所以我们更应该关注写出可以创建实例的类。然后交付给运维人员去运行。(传统模式)

如果公司有DevOps平台，那么恭喜你，基本不需要运维人员的参与，开发即可直接自服务的开发验证、运行。因为在docker的基础上一切看起来真的那么简单，现在运维可能更多的关注是如何提供一套可靠的CICD、PaaS平台


## 3. docker历史

每一段传奇都有这样一个开头`long long ago`，很久很久以前，有一个叫dotCloud的PaaS提供商，其平台利用了Linux容器技术，为了方便创建和管理这些容器，dotCloud内部由Solomon Hykes所带领的团队开发了一套内部工具，而这套工具就是后来的Docker。

2013年dotCloud的PaaS业务并不景气，公司需要寻求新的突破。于是他们聘请了Ben Golub作为新的CEO，将公司重新命名为`Docker`，放弃了dotCloud PaaS平台，怀揣着"将Docker和容器技术推向全世界"的使命，开启了一段新征程。



2013年3月：Docker正式发布开源版本，GitHub中Docker代码提交盛况空前，风头之劲一时无二。

2013年11月：RedHat 6.5正式版本发布，集成了对Docker的支持，拉开了业界各大厂商竞相支持Docker的序幕。

...

2014年：docker公司发布了容器集群管理项目Swarm。

2014年6月: 云市场巨头AWS、Google、Microsoft Azure相继宣布支持Docker，并着手开发基于容器的全新产品。

2014年6月：google正式宣告了Kubernetes项目的诞生（Borg的开源版本），如同Docker横空出世一样，再一次改变了容器市场的格局。

2014年8月：VMware宣布与Docker建立合作关系，标志了虚拟化市场形成了新的格局

2014年10月：微软宣布将整合Docker进入下一代Windows Server中

2014年12月:  CoreOS发布并开始支持rkt（最初作为Rocket发布）作为Docker的替代品。

...

2015年6月：Linux基金会、AWS、思科、Docker、EMC、富士通、高盛、Google、惠普、华为、IBM、Intel等公司在DockerCon上共同宣布成立容器标准化组织OCP（open container project)，旨在实现容器标准化、为Docker生态圈内成员的协作互通打下良好基础。该组织后更名为OCI(open container initiative)。

2015年7月：为了在容器编排地位取得绝对的优势，同Swarm和Mesos竞争，Google、RedHat等开源基础设施公司，共同发起了一个名为CNCF的基金会：希望以Kubernetes为基础，建立一个由开源基础设施领域厂商主导、按照独立基础会方式运营的平台社区，来对抗以Docker公司为核心的容器商业生态。

...

2016年: Docker放弃公司现有的Swarm项目，将容器编排和集群管理功能内置到Docker中，然而这种改变带来的技术复杂度和维护难度，给Docker项目造成了非常比例的局面

kubernetes支持OpenApi，给开发人员定制化提供了更大的灵活性，从API到容器的每一层，都给开发者暴露出了可扩展的插件机制，kubernetes项目的这个变革很快在整个容器社区催生了大量的、基于kubernetes API扩展接口的二次创新产品：istio、Operator、rook等。

Docker公司在kubernetes社区的崛起和壮大后，败下阵来。

...

2017年: Docker将Containerd捐献给CNCF社区，并宣布将Docker项目改名为Moby，交给社区维护。

2017年10月：Docker宣布将在自己主打产品Docker EE 中内置kubernetes项目，持续2年之争的容器编排落下帷幕，kubernetes胜出。

...

2018年：RedHat宣布2.5亿美元收购CoreOS

Docker 公司CTO Solomon Hykes宣布辞职，容器技术圈子处于稳定。

## 4. 几段爱恨情仇

### 4.1 Docker和CoreOS

2013年2月，Docker建立了一个网站发布它的首个演示版本， 3月，美国加州Alex Polvi正在自己的车库开始 他的 第二次创业 

有了第一桶金的Alex这次准备干一票大的，他计划开发一个足以颠覆传统的服务器系统的Linux发行版。为了提供能够从任意操作系统版本稳定无缝地升级到最新版系统的能力，Alex急需解决应用程序与操作系统之间的耦合问题。因此，当时还名不见经传的Docker容器引起了他的注意，凭着敏锐直觉，Alex预见了这个项目的价值，当仁不让地将Docker做为了这个系统支持的第一套应用程序隔离方案。不久以后，他们成立了以自己的系统发行版命名的组织:CoreOS。

Alex Polvi 认为，由于 Docker 貌似已经从原本做"业界标准容器"的初心转变成打造一款以容器为中心的企业服务平台，CoreOS 才决定开始推出自己的标准化容器产品Rocket（rkt）。

> 也许这才是CoreOS整rkt的原因:
>
> Docker 刚问世就红透半边天，不仅拿了融资，还得到了Google 等巨头的支持。CoreOS此前一直忙于为Docker提供技术支持服务



### 4.2 Docker和Mesosphere

Mesos是大数据最受欢迎的资源管理项目，跟Yarn项目竞争的实力派对手。
大数据关注的计算密集型离线业务，不像Web服务那样适合用容器进行托管和扩容，也没有应用打包的强烈需要，所以Hadoop、Spark等项目现在也没在容器技术投入很大的精力，但是Mesos作为大数据套件之一，天生的两层调度机制让它非常容易从大数据领域独立出来去支持更广泛的Pass业务，所以Mesos公司发布了Marathon项目，成为了Docker Swarm的一个强有力的竞争对手。
虽然不能提供像Swarm那样的Docker API，但是Mesos社区拥有一个非常大的竞争力：超大规模集群管理经验
Mesos+Marathon组合进化成了一个调度成熟的Pass项目，同时能支持大数据业务。

然后在和kubernetes的PK后依然败下阵来。



### 4.3 Docker和 OCI

OCI 是一个旨在对容器基础架构中的基础组件（如镜像格式与容器运行时）进行标准化的管理委员会。

CoreOS的Rocket虽然难以撼动Docker的地位，但是随着kubernetes宣布支持其他容器运行时的时候，Docker就意识到，有google作为靠山的kubernetes是不允许Docker一家做大的。

随后，所有相关方都尽力用成熟的方式处理此事，共同成立了 OCI ——一个旨在管理容器标准的轻量级的、敏捷型的委员会。



