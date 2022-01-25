[TOC]

## k8s security

### api server 认证管理

- k8s资源的变更管理, 都是通过 k8s server api 的 RESTFul 接口实现
- 三种认证方式
    - HTTPS 证书认证
    - HTTP Token 认证
    - HTTP Base 认证

### Secret 私密数据
- https://kubernetes.io/docs/concepts/configuration/secret/
- Secret解决了密码、token、秘钥等敏感数据的配置问题，而不需要把这些敏感数据暴露到镜像或者Pod Spec中
- 保管私密数据
- 用户密码需要base64编码
- 三种类型
    - Opaque(default): 任意字符串
    - kubernetes.io/service-account-token: 作用于ServiceAccount，就是上面说的。
    - kubernetes.io/dockerconfigjson: 作用于Docker registry，用户下载docker镜像认证使用。

``` bash

echo -n 'admin' | base64

# for Docker registry

# 似乎不支持相对目录
kubectl create secret generic harborsecret \
    --from-file=.dockerconfigjson=~/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson

# 查看 secret 内容
kubectl get secrets harborsecret --output="jsonpath={.data.\.dockerconfigjson}" | base64 -d

```