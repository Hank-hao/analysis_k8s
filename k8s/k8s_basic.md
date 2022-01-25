[TOC]


## [概述](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

- 以统一的方式抽象底层基础设施能力(计算, 存储, 网络), 定义任务编排的各种关系(亲密关系, 访问关系, 代理关系), 将这些抽象以声明式API的方式对外暴露, 从而允许平台构建者基于这些抽象进一步构建自己的PaaS乃至任何上层平台
- 平台的平台

## 组件

### Master

- Master, 集群控制与管理节点
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler

### Node

- 除了Master, 其他机器被成为Node, 与Master一样,可以是物理机,也可以是虚机

### Pod

- 核心资源对象之一
- **一个容器组**, **通常是不同的应用不同的容器, 而不是一个应用多个容器**
- 每个pod都有个根容器,Pause容器
    - 与业务无关, 不易死亡, 以他来代表整个容器组的状态
    - 为每个Pod提供1号进程, 并收集Pod内的僵尸进程, Pod的init进程
- Pod里的容器共享pause容器的IP(Pod IP)和Volume
- 任意两个Pod可以直接通信(虚拟二层技术实现)
- 分类
    - 普通pod
        - 存储到 etcd 中, master 调度到某个 node 上运行
        - node 上的 kubelet 进行进行实例化成一组相关的docker容器并启动
    - 静态pod
        - 并没有存放再etcd中, 而是放在某个Node的具体文件中
- 限制CPU/内存(绝对值, 如: 0.1 core)
    - 包含 Requests 与 Limits 两个参数
- POD内容器不会跨节点

### Label
- k-v 结构
- Label可以被附加到各种资源对象上，例如Node、Pod、Service、RC等

### RC
- Replication Controller
- 它其实定义了 一个期望的场景，即声明某种Pod的副本数量在任意时刻都符合某个预期值
  - pod 期待的副本数量
  - 用于筛选目标的pod Label Selector
  - 当Pod的副本数量小于预期数量时, 用于创建新Pod的Pod模板s
- Replica Set与RC当前的唯一区别
- Replica Sets支 持基于集合的Label selector（Set-based selector）
- 而RC只支持基于等 式的Label Selector（equality-based selector)
- 通过定义一个RC实现Pod的创建及副本数量的自动控制
- 在RC里包括完整的Pod定义模板
- RC通过Label Selector机制实现对Pod副本的自动控制
- 通过改变RC里的Pod副本数量，可以实现Pod的扩容或缩容
- 通过改变RC里Pod模板中的镜像版本，可以实现Pod的滚动

### Deployment
- **RC的一次升级**， 内部使用 Replica Set 实现, 更好的解决编排问题
    - 滚动升级和回滚
    - 扩容和缩容
- 我们可以知道当前Pod部署的进度


### (HPA) Horizontal Pod Autoscaler
- 与RC, Deployment一样，也是一种资源
- 通过追踪分析指定RC控制的所有目标Pod的负载变化情况，来确定是否要调整Pod副本数量
- 两种度量指标
  - CPUUtilizationPercentage
  - 应用程序自定义的指标

### StatefulSet
- 有状态服务的管理
- 有状态的中间件: mysql集群, mongo集群, zooKeeper集群等。节点有ID等
- 可以看作RC/Deployment的变种
- 实现
    - 每个POD都有编号
    - POD副本启动顺序是受控的
    - 使用持久化的存储卷(PV/PVC)

### Service

#### 概述

- 核心资源对象之一
- 每个Service就是微服务架构中的**微服务**
- 每个Pod提供一个独立的EndPoind(Pod IP + ContainerPort)供访问, 多个Pod副本提供一个Service
- 一个Service的多个Pod副本，由运行在每个Node上的 kube-proxy 进行智能负载均衡
- 每个Service都被分配了一个全局唯一的虚拟IP地址，这个虚拟IP被称为**Cluster IP**
- Pod的Endpoint地址会随着Pod的销毁和重新创建而发生改变
- Service一旦被创建，Kubernetes就会自动为它分配一个可用的Cluster IP，在整个生命周期内，它的Cluster IP不会发生改变

#### 服务发现机制

- 通过DNS进行服务发现

#### 外部系统访问Service

- 三种IP
   - Node IP: 每个节点的物理网卡IP
   - POD IP: POD间通讯
   - Cluster IP: 提供服务IP, 无法Ping通, 集群外不能访问此IP+端口
- Node IP网, POD IP网, Cluster IP网间的通讯。 采用的是Kubernetes自己设计的一种编程方式的特殊路由规则。
- NodePort方式
    - 集群外通过 NodeIP:NodePort 方式
    - 通常需要集群外的负载均衡来均衡Node见的访问

### Ingress

- 建立在service之上的L7访问入口
    - 支持自定义的service的访问策略
    - 支持tls通信
    - ...

### Job

- 资源对象定义并启动一个批处理任务Job
- Job也控制一组Pod容器。从这个角度来看，Job也是一种特殊的Pod副本自动控制器
- Job所控制的Pod副本是短暂运行的，可以将其视为一组 Docker容器，其中的每个Docker容器都仅仅运行一次
- Job所控制的Pod副本的工作模式能够多实例并行计算

### Volume(共享卷)

- 是Pod中能够被多个容器访问的共享目录。 
- Kubernetes中的Volume被定义在Pod上。 一个Pod里的多个容器挂载到具体的文件目录下
- Kubernetes中的 Volume与Pod的生命周期相同，但与容器的生命周期不相关

- 支持多种类型的Volume
  - GlusterFS
  - Ceph
  - ... ...

### Persistent Volume

- PV可以被理解成Kubernetes集群中的某个网络存储对应的一块存储，它与Volume类似，但有以下区别
  - PV只能是网络存储，不属于任何Node，但可以在每个Node上访问
  - PV并不是被定义在Pod上的，而是独立于Pod之外定义的
  - PV支持的类型
    - AWSElasticBlockStore, AzureFile, Flocker, NFS, iSCSI ...

### Namespace

- 对集群内**资源对象**进行分组
- Namespace在很多情况下用于实现多租户的资源隔离

### Annotation

- 注释, 也使用key/value键值对的形式进 行定义。 
- 不同的是Label具有严格的命名规则
- Annotation则是用户任意定义的附加信息，以便于外部工具查

### ConfigMap

- 解决 docker 中应用配置文件的问题
- 将存储在etcd中的 ConfigMap通过Volume映射的方式变成目标Pod内的配置文件
- 不管目标Pod被调度到哪台服务器上，都会完成自动映射

## Ref

- [怎么读?](https://github.com/kubernetes/kubernetes/issues/44308)