[TOC]

## 目录树

```bash
kubernetes{
	pkg{

	}
	# 各个模块的启动代码
	cmd{
		kubectl
		kube-apiserver
	}
	# 貌似vendor目录包发布前的目录
	staging{

	}

	vendor{
	}
}

```

## kubectl

- 使用 cobra 使用命令行
- 请求的 kube-apiserver 的接口
- Builders, Visitors
- https://topsli.github.io/2018/12/23/kubernetes_source_kubectl.html

```go
/*
    entry: cmd/kubectl/kubectl.go
           vendor/k8s.io/kubectl/pkg/cmd/cmd.go

*/ 

// Builders, Visitors, vendor/k8s.io/cli-runtime/pkg/resource/


// create, vendor/k8s.io/kubectl/pkg/cmd/create/create.go

```

## kube-apiserver

```go

```