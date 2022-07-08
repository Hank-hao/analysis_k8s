[TOC]


## Kubernetes 网络模型
- IP-per-Pod模型
  - 一个Pod内部的所有容器共享一个网络堆栈
  - 相当于一个网络命名空间，它们的IP地址、网络设备、配置等都是共享的
  - 着同一个Pod内的容器可以通过localhost来连接对方的端口
  - 同一个VM内的进程之间的关系是一样的
  - Pod都能够被看作一台独立的虚拟机或物理机

- k8s对集群网络的要求
    - 所有容器间可以在无NAT方式下通信
    - 所有节点可以在无NAT方式下，同所有容器通信
    - 容器地址与别人看到的是同一个地址


## Docker网络基础

- Network Namespace
- Veth设备对
- 网桥
- iptables
- route

#### 命名空间

- 为了支持网络协议栈的多个实例, **独立的协议栈**, 不同网络栈完全隔离
- 网络命名空间中可以有自己
    - 独立的路由表
    - 独立的iptables设置来提供包转发
    - NAT及IP包过滤等功能

- 需要纳入网络空间的元素
  - 进程
  - 套接字
  - 网络设备

- Linux实现命名空间
  - 全局变量都必须被修改为协议栈私有
  - 让这些全局变量成为一个Net Namespace变量的成员
  - 协议栈的函数调用加入一个Namespace参数
  - 内核代码隐式地使用了命名空间中的变量
  - 网络空间之间通讯，通过Veth设备对
  - 一个设备只能属于一个网络空间
  - 有些设备可以转移, 有些设备不能转移

- 网络空间操作
``` bash
ip netns list 
ethtool -k eth0 | grep netns-local # on为不能在网络空间中转移
```

#### Veth Pair

- 为了**在不同的网络空间之间通信**
- 成对出现, **很像一对以太网卡，中间有一根直连的网线**
- docker会把veth设备名字改为 eth0
- 在docker内部, veth设备也是联通容器与宿主机的主要网络设备
- 一旦将Veth设备对的对端放入另一个命名空间，在本命名空间中就 看不到它了

``` bash
ip link add veth0 type veth peer name veth1  # 创建veth设备对

brctl show   # 查看网桥下的veth设备
```

#### 网桥

![bridge&veth关系图](http://assets.processon.com/chart_image/61bc630e7d9c0868e029bc2e.png)

- 二层网络虚拟设备(虚机二层交换机)
- 遇到未学习到的地址,  发广播包
- 多netns连接, 如果相互创建veth pair, 会非常多, 可以使用网桥将其连接起来
- docker默认会创建docker0的网桥, 凡是与docker0相连的容器, 都可以使用它来通讯
- Linux网桥的实现
    - 内核是通过一个虚拟的网桥设备（Net Device）来实现桥接的
    - 这个虚拟设备可以绑定若干个以太网接口设备，从而将它们桥接起来

```bash
yum install -y bridge-utils
brctl show   # 网桥管理程序
```

#### tun/tap设备

- 给用户态一个机会
- Linux文件系统的角度看，它是用户可以用文件句柄操作的字符设备
- 从网络虚拟化角度看，它是虚拟网卡，一端连着网络协议栈，另一端连着用户态程序
- tun表示虚拟的是点对点设备, tap表示虚拟的是以太网设备, 两种设备包封装不同
- tun/tap设备可以将协议栈处理好的包发给另一个使用tun/tap驱动的进程, 由进程重新处理后发给物理网络, tun/tap进程像埋在用户态的钩子
    - openvpn, vtun, fannel都是基于此实现隧道包封装
- 对于tun/tap设备，它的数据源是用户态而非物理网卡, 最大价值是提前剧透

.tun设备的/dev/tunX文件收发的是IP包
·tap设备的/dev/tapX文件收发的是链路层数据包


#### iptables & netfilter

- 应用广泛
    - 容器与宿主机端口映射
    - k8s service 的默认模式
    - cni的portmap
    - k8s的网络策略

- 网络协议栈中有一组回调函数挂接点，
- 通过这些挂接点挂 接的钩子函数可以在Linux网络栈处理数据包的过程中对数据包进行一些操作
- Netfilter负责在内核中执行各种挂接的规则，**运行在内核模式中**
- iptables是**在用户模式下运行的进程**，负责协助和维护内核中Netfilter的各种规则表
- Netfilter可以挂接的规则点有5个
- 挂接点能挂接的规则也分不同的类型: RAW、MANGLE、NAT和FILTER(优先级由高到低)
- 处理规则
    - 表类型（准备干什么事情）
    - 什么挂接点（什么时候起作用）
    - 匹配的参数是什么（针对什么样的数据包）
    - 匹配后有什么动作（匹配后具体的操作是什么）

#### linux隧道

- ip tunnel help, linux的5种隧道
    - ipip: ipv4 in ipv4, 在ipv4报文上封装ipv4报文
    - gre: 通用路由封装Generic Routing Encapsulation,任意一种网络层协议上封装其他任意一种网络层协议的机制，适用于IPv4和IPv6
    - sit: IPv4报文封装IPv6报文，即IPv6 over IPv4
    - isatap: 站内自动隧道寻址协议, 用于IPv6的隧道封装
    - vti: 虚拟隧道接口, 是思科提出的 一种IPSec隧道技术

##### ipip

```bash
modprobe ipip   # 加载内核模块
```

#### vxlan

- fdb即二层mac表

```bash

# 创建名为vxlan0的接口, 也是vtep
ip link add vxlan0 type vxlan id 42 group 239.1.1.1 dev ens33 dstport 4789
ip link delete vxlan0       # 删除接口
ip -d link show vxlan0      # 查看接口
bridge fdb show dev vxlan0  # 查看接口转发表
```


#### macvlan

- docker与外部网络通讯时，使用macvlan, 比nat, linux bridge, openswitch 拥有更好的性能
- 物理网网口的虚拟子接口，可以有自己的IP和MAC地址
- 5种模式
    - bridge, 同父接口的子接口可直接通讯
    - VEPA(Virtual Ethernet Port Aggregator，虚拟以太网端口聚合), 无论目的地址是啥都先发送父接口
    - private,
    - passthru, 直通模式, 每个父接口 只能和一个Macvlan网卡捆绑
    - source, 只接收指定源mac地址的数据包
- 缺点
    - 无法支持大量的MAC地址
    - 无法工作在无线网络环境中

#### ipvlan

- 虚拟接口使用相同的MAC地址
- DHCP场景需要注意, 不能使用MAC地址为标识
- 2种模式
    - L2
    - L3


## Docker网络实现

- 四种模式
  - host模式
  - container模式
  - none模式
  - **bridge模式**, k8s通常使用此模式

- bridge模式下docker网络
  - docker daemon 第一次启动创建一个虚拟的网桥, 默认名称为**docker0网桥**
  - 根据 rpc1918, 在私有网络中分配一个个网桥
  - 每个容器都会**创建veth设备对**, 一端关联到网桥, 另一端使用命名空间技术映射到容器内的eth0设备
  - 然后网桥地址段给eth0接口分配一个IP地址, MAC地址根据这个ip地址生成
  - docker daemon 创建了 docker0网桥并添加iptables规则
  - docker0网桥与iptalbes都处于**root命名空间**

- 局限性
  - Docker一开始 没有考虑到多主机互联的网络解决方案
  - Docker使用的Libnetwork组件只是将 Docker平台中的网络子系统模块化为一个独立库的简单尝试，
  - 离成熟和 完善还有一段距离

## Kubernetes的网络实现

- 致力于解决如下问题
  - 容器到容器之间的直接通信
  - 抽象的Pod到Pod之间的通信
  - Pod到Service之间的通信
  - 集群外部与内部组件之间的通信

### 容器到容器之间的通信

- Pod内的容器是不会跨宿主机的
- 同一个Pod内的容器共享同一个网络命名空间
- 他们之间**可以使用localhost进行通讯**(方便从已存在的程序中迁移)
    - 容器1使用 localhost:3306 可以访问容器2的 mysql

### POD之间的访问

- 一个Node内的Pod之间的通信
    - Pod1和Pod2都是通过Veth连接到同一个docker0网桥上的
    - 它们的IP地址IP1、IP2都是从docker0的网段上动态获取的
    - 它们和网桥本身的IP3是同一个网段的
    - IP1,IP2,IP3都是同一个网段,直接通信
- **不同Node上Pod之间的通信**
    - 在整个Kubernetes集群中对Pod的IP分配进行规划，不能有冲突
        - 网络增强开源软件Flannel就能够管理 资源池的分配
    - 找到一种办法，将Pod的IP和所在Node的IP关联起来，通过这个关联让Pod可以互相访问
        - google使用GCE来打通底层网络
        - **其它增强网络应用**
        - 静态路由的方式

## Pod与Service网络实战
- pause容器
- kube-proxy

## CNI网络模型

- 容器固定IP地址
- 一个容器多个IP地址
- 多个子网隔离
- ACL控制策略
- 与SDN集成

### CNM模型

- docker公司提出的Container Network Model（CNM）
- 由三个组件组成
  - Network Sandbox
    - 容器内部的网络栈, 包括网络接口、路由表、DNS等配置的管理
    - 可由Linux网络命名空间、FreeBSD Jail等机制实现
    - 一个Sandbox可以包含多个Endpoint
  - Endpoint
    - 用于将容器内的Sandbox与外部网络相连的网络接口
    - 可以使用veth对, 、Open vSwitch的内部port等技术进行实现
    - 一个 Endpoint仅能够加入一个Network
  - Network
    - 可以直接互连的Endpoint的集合
    - 可以通过Linux网 桥、VLAN等技术进行实现
    - 一个Network包含多个Endpoint

### CNI模型

- CoreOS公司提出的 Container Network Interface（CNI）
- **CNI提供了一种应用容器的插件化网络解决方案, 定义对容器网络进行操作和配置的规范，通过插件的形式对CNI接口进行实现**
- CNI的思想就是在kubelet启动infra容器后，就可以直接调用CNI插件为这个infra容器的Network Namespace配置符合预期的网络栈
- 在CNI模型中只涉及两个概念：容器和网络
  - 容器, 拥有独立的Linux网络命名空间
  - 网络, 表示可以互连的一组实体,这些实体拥有各自独立、唯一的IP地址,可以是容器、物理机或者其他网络设备(比如路由器)等
- 对容器网络的设置与操作都通过插件实现
  - CNI Plugin: 为容器配置网络资源
  - IPAM(IP Address Management) Plugin: 对容器的IP地址进行分配和管理
- K8s中使用的插件
  - CNI Plugin
  - kubenet Plugin: ：使用bridge和host-local CNI插件实现一个基本的 cbr0

### K8s网络策略

- 网络策略:基于Pod的源IP的访问控制列表, 限制的是Pod之间的访问
- 引入Network Policy机制
- 对Pod间的网络通信进行限制和准入控制
- 为了使用Network Policy，Kubernetes引入了一个新的资源对象 NetworkPolicy
- 仅定义一个网络 策略是无法完成实际的网络隔离的，还需要一个策略控制器（Policy Controller）进行策略的实现
- 策略控制器由第三方网络组件提供
- 4个维度限制
    - label selector
    - namespace selector
    - port
    - cidr

## 网络插件

- GCE里面是现成的网络模型, 在私有云里搭建Kubernetes集群，就不能假定这种网络已经存在了.
- 需要自己实现这个网络假设，将不同节点上的Docker容器之间的互相访问先打通，然后运行Kubernetes

- 介绍几个常见的 网络组件及其安装配置方法包括
    - Flannel
        - 它能协助Kubernetes，给每一个Node上的Docker容器都分配互 相不冲突的IP地址。
        - 它能在这些IP地址之间建立一个覆盖网络（Overlay Network）,通过这个覆盖网络，将数据包原封不动地传递到目标容器内
    - Open vSwitch
    - 直接路由: 
        - 可以通过部署MultiLayer Switch(MLS)来实现这一点
    - Calico： 一个基于BGP的纯三层的网络方案
    - Canal: 是calico和flannel的结合

### flannel 插件鼻祖

#### udp 模式

![](http://assets.processon.com/chart_image/61c7ed937d9c0830226e6cbb.png)

- docker1(10.1.1.10)向docker2(10.1.2.10)发包过程
    - docker1发现目标地址10.1.2.10不在同一网段, 通过docker1的默认路由,发给网桥
    - 网桥查看宿主机路由表, 需要发送给fannel0接口(fannel创建的路由)
    - fannel0会把数据包交给创建该设备的应用程序fanneld
    - (fanneld从etcd中知道10.1.2.0/24子网分配给了node2)
    - fanneld收到包后判断目标后，封装udp包，udp目标为:192.158.1.11:8285
    - 数据到达192.158.1.11:8285, fanneld侦听的8285, 收到包后进行解包, 并发往fannel0接口
    - 根据宿主机路由, 发往网桥
    - 网桥最终找到了docker2
- tun + udp
- 性能低

#### vxlan 模式

- 当前k8s只支持单网络, 所以在三层网络上只有1个vxlan网络
- 使用vxlan进行二层互通
- 相当于udp的封解包过程放到了内核态

#### Host-Gateway

- 性能最好, 不能跨二层网络
- 大二层

``` bash
ip -d link show flannel0  # 查看接口信息

```


### Calico

##### vRouter模式

- 基于BGP的三层通信方式, 每个容器通过IP直接通信, 中间经过路由转发找到对方
- 在没有IP封装与NAT的情况下通信, 实现裸机性能，简化故障排除, 提供更好的互操作性, 称为vRouter模式
- 组件
    - Felix
        - 节点上的守护程序
        - 管理网络接口: 将有关网络接口的信息编写到内核中, 保证主机正确响应来自每个工作负载的ARP请求, 并将其管理的网卡启用IP Forward
        - 编写路由: 负责将主机上Endpoint的路由编写到Linux内核FIB(转发信息库)中
        - 编写ACL: 负责将ACL编写道Linux内核中, 即iptables规则
        - 状态报告: 负责提供网络相关的健康状态数据
    - BGP Client
        - 节点上的守护程序
        - 读取Felix写到内核的路由信息
        - 进行路由分发
    - BGP Route Reflector(BIRD)
        - BGP 路由反射器
- 瓶颈 
    - 路由信息的数量
    - 不支持vpc
    - 无流量控制

##### overlay模式

- ipip

### Weave

- 创建虚拟网络, 使用gossip协议(是基于流行病传播方式的节点或者进程之间信息交换的协议)\
- 加密

### Cilium

- 基于IP/端口的防火墙方式在微服务架构中的网络和安全面临一系列限制
- BPF
    - 将BPF字节码动态插入Linux内核，为工作负载提供透明的网络链接保护、负载均衡、安全和可观测性支持

### Canal

- 业界还是会习惯性地将Flannel和Calico的组成称为“Canal”



#### CNI-Genie

- 华为开源的多网络插件，集成引导插件，本身无具体功能
- 支持flannel, Calico, Weave Net, Canal, Romana


#### server mesh

- 应用程序间通信的中间层
- 轻量级的网络代理
- 应用程序无感知
- sidecar模式
    - 给应用程序加上一个"边车"的方式拓展应用程序现有的功能，如日志记录, 监控, 流量控制，服务注册, 服务发现, 服务限流, 服务熔断, 鉴权, 访问控制等
    - sdk方式
    - agent方式: 所有通讯都通过这个agent代理
    - 解决服务之间调用越来越复杂的问题
    - 控制与逻辑分离的问题

- Linkerd
    - 工作原理
        - 将服务请求路由到目的地址(可根据参数判断是生产环境还是测试环境)
        - 确定目的地址后，将流量发送到相应的服务发现端点
        - 可根据最近请求的延时时间, 选择最快的实例
        - 将请求发送给该实例, 记录响应类型和延迟数据
        - 故障摘除
        - 将追踪信息发送到metric系统


#### Istio

- 提供微服务连接的、安全的、流量控制的和可观察的开放平台
- 数据平面: Envoy, sidecar代理
- 控制平面: mixers

- Envoy
    - sidecar代理，类似7层代理，更贴近Service Mesh的使用场景
    - 在用户创建Pod中自动注入一个sidecar, 即Envoy
- Pilot
    - 用于配置Envoy
- Mixer
    - 负责整个Service Mesh中执行访问控制和使用策略
- Citadel
    - 提供服务和用户的身份验证

- istio支持混合服务网格
    - 多集群istio, 一个控制平面
    - 多集群istio, 多个控制平面
    - 将虚机纳入istio
