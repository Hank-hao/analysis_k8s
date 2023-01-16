
- 部分文件为符号链接, windwows文件系统不支持


## 代码生成器

## build


- https://github.com/kubernetes/kubernetes/blob/master/build/README.md
- https://cloud.tencent.com/developer/article/1433219

```bash

KUBE_BUILD_PLATFORMS=linux/amd64 make all
KUBE_BUILD_PLATFORMS=linux/amd64 make WHAT=cmd/kube-apiserver

```



### 参考

- https://www.guoshaohe.com/cloud-computing/kubernetes-source-read/1320