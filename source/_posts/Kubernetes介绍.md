---
title: Kubernetes介绍
date: 2020-12-08 16:50:53
tags: 分布式系统
---

# 深入浅出地了解 Kubernetes
Kubernetes 是一个**软件系统**, 它允许你在其上很容易地部署和管理容器化的应用.

{% asset_img Kubernetes的核心功能.png Kubernetes的核心功能 %}

组件被部署在哪一个结点对于开发者和系统管理员来说都不用关心

## Kubernetes 集群架构
在硬件级别, 一个 Kubernetes 集群由很多节点组成, 这些节点被分成以下两种类型:

* 主节点: 承载着 Kubernetes 控制和管理整个集群系统的控制面板
  * 控制面板
    * *Kubernetes* API 服务器, 你和其他控制面板组件都要和它通信
    * *Scheculer* 它调度你的应用 (为应用的每个可部署组件分配一个工作节点)
    * *Controller Manager* 它执行集群级别的功能, 如复制组件、持续跟踪工作节点、处理节点失败
    * *etcd* 一个可靠的分布式数据存储, 它能持久化存储集群配置
  * 控制面板的组件持有并控制集群状态, 但是它们不运行你的应用程序, 是由工作节点完成的.
* 工作结点: 它们运行用户实际部署的应用
  * Docker, rtk 或其他的容器类型
  * *Kubelet* 它与 API 服务器通信, 并管理它所在节点的容器
  * *Kubernetes Service Proxy (kube-proxy)* 它负责组件之间的负载均衡网络流量

{% asset_img Kubernetes集群组件.png Kubernetes集群组件 %}

## 在 Kubernetes 中运行应用
1. 首先需要将应用打包进一个或多个容器镜像
2. 再将镜像推送到镜像仓库
3. 然后将应用的描述发布到 Kubernetes API 服务器

### 描述信息怎样成为一个运行的容器
1. 当 API 服务器处理应用的描述时, 调度器调度指定组的容器到可用的工作节点 (基于每组所需的计算资源以及调度时每个节点未分配的资源)
2. 那些节点上的 Kubelet 指示容器运行时拉取所需的镜像并运行容器.

{% asset_img Kubernetes体系结构.png Kubernetes体系结构 %}

### 保持容器运行
一旦应用程序运行起来, Kubernetes 就会不断地确认应用程序的部署状态始终与你提供的描述相匹配.

如果整个工作节点死亡或无法访问, Kubernetes 将为在故障节点上运行的所有容器选择新节点, 并在新选择的节点上运行它们.
### 扩展副本数量

当应用程序运行的时候, 可以决定要增加或减少副本量, 而 Kubernetes 将分别增加附加的或停止多余的副本

### 命中移动目标
为了让客户能够轻松地找到提供特定服务的容器, 可以告诉 Kubernetes 哪些容器提供相同的服务. Kubernetes 将通过一个静态IP 地址暴露所有容器, 并将该地址暴露给集群中运行的所有应用程序. *kube-proxy* 将确保到服务的连接可跨提供服务的容器实现负载均衡.


# 开始使用 Kubernetes 和 Docker
## Hello World

```bash
docker run busybox echo "Hello World"
```
1. Docker 首先会检查 busybox:latest 镜像是否已经存在于本机
2. 如果没有, Docker 会从镜像中心拉取 busybox 镜像
3. Docker 在被隔离的容器里运行 ```echo "Hello World"```

{% asset_img 基于Dockerfile构建一个新的容器镜像.png 基于Dockerfile构建一个新的容器镜 %}


> 镜像不是一个大的二进制块, 而是由多层组成的, 不同镜像可能会共享分层.

## 运行容器镜像

```bash
docker run --name kubia-container -p 8080:8080 -d kubia
# 基于 kubia 镜像创建一个叫 kubia-container 的新容器. 这个容器与命令行分离 (-d), 意味着后台运行. 本机上的 8080 端口会被映射到容器内的 8080 端口 (-p 8080:8080), 所以可以通过 http://localhost:8080 来访问应用.
```

* 容器使用独立的 PID Linux 命名空间并且有着独立的系列号, 完全独立于进程树.
* 容器的文件系统也是独立的.


## 介绍 pod
Kubernetes 不直接处理单个容器, 它使用多个共存容器的理念. 这组容器就叫做 **pod**.

> 一个 pod 是一组紧密相关的容器, 他们总是运行在同一工作节点上, 以及同一个 Linux 命名空间中.

{% asset_img 容器、pod及物理工作节点之间的关系.png 容器、pod及物理工作节点之间的关系 %}

* 每个 pod 就像一个独立的逻辑机器, 拥有自己的 IP, 主机名, 进程等, 运行一个独立的应用程序.
* 应用程序可以是单个进程, 运行在单个容器中, 也可以是一个主应用进程或者其他支持进程, 每个进程都在自己的容器中运行.
* 一个 pod 所有的容器都运行在同一逻辑机器上, 而其他 pod 中的容器, 即使运行在同一工作节点上, 也会出现在不同的节点上.

```bash
kubectl run kubia --image=luksa/kubia --port=8080 --generator=run/v1

# 现在还处于创建容器的阶段
kubectl get pods
# NAME    READY   STATUS              RESTARTS   AGE
# kubia   0/1     ContainerCreating   0          14s

# 现在 pod 已变成运行状态
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# kubia   1/1     Running   0          90s
```

上述幕后发生的事情
1. 构建镜像并将其推送到 Docker Hub (在本机上构建的镜像只能在本地机器上可用, 但是需要使它可以访问运行在工作节点上的 Docker 守护进程)
2. 运行 ```kubectl``` 命令时, 它通过向 Kubernetes API 服务器发送一个 REST HTTP 请求, 在集群中创建一个新的 ReplicationController 对象.
3. ReplicationController 创建了一个新的 pod, 调度器将其调度到一个工作节点上.
4. Kubelet 看到 pod 被调度到节点上, 就告知 Docker 从镜像中心中拉取指定的镜像, 因为本地没有该镜像.
5. 下载镜像后, Docker 创建并运行容器.

{% asset_img 在Kubernetes中运行容器镜像.png 在Kubernetes中运行容器镜像 %}




# 参考文献
* 《 Kubernetes in Action 中文版 》