
## install
``` bash
yum install kubernetes

```

## kubectl

- [命令清单](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)

``` bash
kubectl [command] [TYPE] [NAME] [flags]

# TYPE: 资源对象的类型，区分大小写，能以单数、复数或者 简写形式表示
# NAME: 资源对象的名称，区分大小写。 如果不指定名称，系统则将返回属于TYPE的全部对象的列表
# flags：kubectl子命令的可选参数
```

- 示例

```bash

# create, apply, replace

# get
kubectl get all                           # 查看所有资源
kubectl get pod -n namespace              # 查看pod资源
kubectl get pod --show-labels             # 查看pod资源及其标签

# describe
kubectl describe pod hello-minikube

# log
# 返回仅包含一个容器的pod nginx的日志快照
kubectl logs nginx

# 返回pod ruby中已经停止的容器web-1的日志快照
kubectl logs -p -c ruby web-1 -n namespace

# 持续输出pod ruby中的容器web-1的日志
kubectl logs -f -c ruby web-1

# 仅输出pod nginx中最近的20条日志
kubectl logs --tail=20 nginx

# 输出pod nginx中最近一小时内产生的所有日志
kubectl logs --since=1h nginx


```

### namespace

```bash
kubectl get namespaces
kubectl create namespace <name>
```


### deployment
- https://www.kubernetes.org.cn/deployment
- https://www.mirantis.com/blog/introduction-to-yaml-creating-a-kubernetes-deployment/
- https://www.cnblogs.com/wzs5800/p/13534942.html

```bash
Deployment定义的 template 字段, 也叫PodTemplate(Pod模板)
Deployment控制器定义, 包括控制器定义(包括期望状态), 被控制对象模板组成
Deployment 控制 ReplicaSet(版本), ReplicaSet 控制 Pod(副本数)
```


### k8s 服务灰度

- 发布不同的deployment, 实现金丝雀部署, 能否控制请求量?
- 

### 镜像

- Harbor
- Nexus

## 应用链路监控(skywalking/zipkin)

