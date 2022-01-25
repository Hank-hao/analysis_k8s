[TOC]

## [Minikube](https://minikube.sigs.k8s.io/docs/)

- 单机版的Kubernetes集群
- 支持Kubernetes的大部分功能，从基础的容器编排管理，到高级特性如负载均衡、Ingress，权限控制等
- 非常适合作为Kubernetes入门，或开发测试环境使用

### Install

``` bash

# docker

# minikube 不能以 root 启动

# 使用docker组
sudo usermod -aG docker $USER && newgrp docker

# start
minikube start --driver=docker --image-mirror-country=cn  # china mainland

minikube start -p cluster2  # Start a second local cluster
minikube stop 
minikube delete       # Delete your local cluster
minikube delete --all # Delete all local clusters and profiles

# dashbord
minikube dashboard --url  # 默认绑定127.0.0.1, 展示 dashboard url 
kubectl proxy --port=8001 --address='10.0.0.1' --accept-hosts='^.*' &  # 使用代理访问dashboard url

# 部署应用
kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.4
kubectl create -f docs/user-guide/nginx-deployment.yaml --record  # 通过yaml文件部署

kubectl expose deployment hello-minikube --type=NodePort --port=8080
kubectl get services hello-minikube
kubectl port-forward service/hello-minikube 7080:8080

kubectl delete deployment hello-minikube # 删除资源

```

- 集群管理

```bash
minikube addons list  # 已安装的k8s服务

```
