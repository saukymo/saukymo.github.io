---
title: 在Gitlab-CI中设置kubectl实现自动部署
date: 2017-04-08 15:29:39
tags: [Kubernetes, Gitlab, DevOps]
---

## 参考文档
1. Kubernetes权威指南
2. [How to Add Users to Kubernetes (kubectl)?](http://stackoverflow.com/questions/42170380/how-to-add-users-to-kubernetes-kubectl)
3. [Kubernetes Documentation](https://kubernetes.io/docs/home/)

## 实现思路

Kubernetes集群主要由Master节点控制，Master节点又主要由其中的API Sever(kube-apiserver)提供Rest接口服务。所以我们大体思路就是在CI中配置好正确的kubectl客户端，使它能够直接连到部署的集群上，从而实现远程部署。

目前CI主要分为test, build, deploy等几个阶段，在build阶段就把新的image打包好并且上传到私有registry了，那么deploy阶段就只需要告诉kubernetes新的image编号，kubernetes就会自动帮我们下载新的镜像并且重新部署。

由于目前deploy阶段采用的runner是shell runner，使用的用户是gitlab-runner。可以理解为使用gitlab-runner用户登录到git.hupofin.tech这台机器上去，然后执行ci阶段里定义的命令。

所以我们需要给gitlab-runner配置一个kubectl客户端，这样一来，所有使用shell runner来执行ci的阶段都可以访问集群了

## 手动完成首次部署

首次部署应用到kubernetes不是本次的重点，可以参考kubernetes官网的交互式tutorial，很清晰。

## Service Account
既然是在集群外远程访问控制集群，那么就需要一定的身份认证机制。这里采用Service Account的方式进行验证。

Service Account包括三个部分：
namespace，token和CA证书。

所以我们需要做的就是在集群中创建一个新的sa用于部署，然后将上面三样东西配置在kubectl客户端就可以了。

### 创建一个新的SA
```
	kubectl create sa shaobo
```
### 获得secret的名字
```
	kubectl get sa shaobo -o json | jq -r .secrets[].name
	- shaobo-token-rq201
```
一个Service Account可以包含多个secret，例如刚刚我们创建一个新的SA时自动生成的secret就是用于访问API的secret。
同时，我们也可以自己定义其他的secret，例如在从私有registry下载镜像时可能会需要一个下载镜像的secret，使用https时需要相关证书的secret等等。

### 获取CA证书
```
	kubectl get secret $secret -o json | jq -r '.data["ca.crt"]' | base64 -d > ca.crt
```
我们需要将`ca.crt`证书下载到CI服务器上去。

### 获取token
```
	kubectl get secret $secret -o json | jq -r '.data["token"]' | base64 -d
```

## 将API接口暴露到外网

### kubectl proxy 搭建代理
kubernetes的API默认只有本机才能访问到的，所以我们首先需要搭建一个proxy，转发一下外网的请求，这里采用自带的`kubectl proxy`
```
	kubectl proxy --address='0.0.0.0' --port=8001 --accept-hosts='^*$'
```

这条命令实现了这么几个功能，监听8001端口，任何ip来源的请求，所有来源都准入。集群的准入准出通过阿里云的安全组策略控制，所有我们这里直接允许所有的请求即可。

### screen 后台运行
然后为了保证proxy一直运行下去，通过`screen`命令保留到后台：
```
	screen kubectl proxy --address='0.0.0.0' --port=8001 --accept-hosts='^*$'
```
`ctrl + a` `ctrl + d` detach，进程继续后台运行

## 在新的机器上配置kubectl

### 下载kubectl
[Installing and Setting Up kubectl](https://kubernetes.io/docs/tasks/kubectl/install/)
``` shell
	# Linux
	curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
```

添加到用户目录里
```
	chmod +x ./kubectl
	sudo mv ./kubectl /usr/local/bin/kubectl
```


### 设置cluster
添加一个名字叫`hupo`的集群，server这里设置为集群的ip和端口：
```
	kubectl config set-cluster hupo --embed-certs=true --server=http://101.██.██.██:8081 --certificate-authority=./ca.crt
```

### 设置user credentials
```
	kubectl config set-credentials shaobo --token=$token
```

### 绑定cluster和用户
```
	kubectl config set-context hupo --cluster=hupo --user=shaobo --namespace=default
```

### 切换到刚才设置的context
```
	kubectl config use-context hupo
```

### 测试一下是否完成
```
	kubectl version
```


## CI里使用kubectl自动部署
这里以loop-service为例，直接使用kubectl更新deployment的image设置即可。
```
	kubectl set image deployment/loop-service loop-service=registry-vpc.cn-beijing.aliyuncs.com/hupofintech/loop-service:${ENV}-${CI_BUILD_REF:0:7}
```

这样做会有一个缺点，就是如果更新时服务会停止，且失败之后没有办法简单的回滚。更好的办法是采用Replication Controller实现滚动更新。


