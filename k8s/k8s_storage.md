## 共享存储 

### 共享存储机制

为了能够屏蔽底层存储实现的细节，让用户方便使用，同时让管理员方便管理，Kubernetes从1.0版本就引入PV和PVC两个资源对象来实现对存储的管理子系统  
PV是对底层网络共享存储的抽象，将共享存储定义为一种“资源  
PV由管理员创建和配 置，它与共享存储的具体实现直接相关  
GlusterFS、iSCSI、RBD或GCE或AWS公有云提供的共享存储, 通过插件式的机制完成与共享存储的对接  
PVC则是用户对存储资源的一个“申请”。就像Pod“消费”Node的资源一样 
PVC能够“消费”PV资源。PVC可以申请特定的存储空间和访问模式  
资源对象StorageClass，用于标记存储资源的特性和性能  
到1.6版本时，StorageClass和动态资源供应的机制得到了完善，实现了存储卷的按需创建  
Kubernetes从1.9版本开始引入容器存储接口Container Storage Interface（CSI）机制



### PV详解

#### 关键配置参数  
- 存储能力 Capacity
- 存储卷模式 Volume Mode
- 访问模式 Access Mode
- 存储类型 Class
- 回收策略 Reclaim Policy
- 挂载参数 Mount Options
- 节点亲和性 Node Affinity

#### PV生命周期
- Available: 可用状态，还未与某个PVC绑定
- Bound: 已与某个PVC绑定
- Released: 绑定的PVC已经删除，资源已释放，但没有被集群 回收
- Failed: 自动资源回收失败


### PVC详解

- 用户对存储资源的需求申请

#### PVC的关键配置参数
- 资源请求 Resources
- 访问模式 Access Modes
- 存储卷模式 Volume Modes
- PV选择条件 Selector
- 存储类别 Class


### PV与PV的生命周期
- 资源供应
- 资源绑定
- 资源使用
- 资源释放
- 资源回收

### Storage Class详解
作为对存储资源的抽象定义，对用户设置的PVC申请屏 蔽后端存储的细节。
一方面减少了用户对于存储资源细节的关注，另一 方面减轻了管理员手工管理PV的工作，由系统自动完成PV的创建和绑定，实现了动态的资源供应。


### 动态存储管理实战: GlusterFS



### CSI存储机制详解
用于在Kubernetes和外部存储系统之间建立一套 标准的存储管理接口 