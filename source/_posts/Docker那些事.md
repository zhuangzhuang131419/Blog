---
title: Docker那些事
date: 2020-08-20 21:31:34
tags:
---
# 容器生态系统
## 容器核心技术
容器核心技术指的是能够让 Container 在 host 上运行起来的技术

### 容器规范
> 我们熟知的Docker只是容器的一种

OCI 发布了以下两个规范
* runtime spec
* image format spec
有了这两个规范，不同组织和厂商开发的容器能够在不同的runtime上运行

### 容器runtime
Java 程序 -> 容器

JVM -> runtime

JVM 为 Java 程序提供运行环境，同样的容器只有在runtime中才能运行

目前主流的三种容器runtime
* lxc
    * Linux 上老牌的容器 runtime
    * Docker 最初也使用的 lxc
* runc
    * 是 Docker 开发的容器，也是 Docker 默认的runtime
* rkt
    * 是 CoreOS 开发的容器

### 容器管理工具
有了 runtime 之后，用户得有工具来管理容器，每种不同的runtime有自己不同的管理工具
* lxc
    * lxd
* runc 
    * docker engine
        * cli
        * deamon
    * 我们提到的 Docker 一般指的就是 docker engine
* rkt
    * rkt cli

### 容器定义工具
允许用户自己定义容器的内容和属性，这样容器可以被保存、共享、重建。
* docker image
    * runtime 依据 docker image 创建容器
* dockerfile
    * 包含若干命令的文本文件，可以通过这个创建出docker image
* ACI (App Container Image) 

### Registries
因为容器是通过模板 image 来创建的，需要有一个仓库来统一存放 image, 这个仓库就叫做 Registry

* Docker Registry
    * 企业通过 Docker Registry 来构建私有的 Registry
* Docker Hub
* Quay.io

### 容器OS
容器 OS 是专门运行容器的操作系统。与常规 OS 相比，容器 OS 通常体积更小，启动更快
## 容器平台技术

容器核心技术使得容器能够在单个 host 上运行，而容器平台技术能够让容器作为集群在分布式环境中运行。

部署在容器中的应用一般采用微服务架构，在这种架构下，应用被划分为不同的组件，并以服务的形式运行在**各自**的容器中。为了保证应用的高可用，每个组件都可能会运行多个相同的容器。这些容器会组成集群，集群中的容器会根据业务需要被动的创建、迁移和销毁。

### 容器编排引擎
编排 (orchestration) 包括容器管理、调度、集群定义和服务发现。通过容器编排引擎，容器被有机组合成微服务应用。

以下三个是当前主流的容器编排引擎
* docker swarm
    * Docker 开发的容器编排阵容
* kubernetes
    * Google 开发的容器编排阵容
* mesos + marathon

### 容器管理平台
容器管理平台是架构在容器编排引擎之上的一个更为通用的平台。
* Rancher
* ContainerShip

### 基于容器的PaaS
基于容器的 PaaS 为微服务应用开发人员与公司提供了开发、部署和管理应用的平台

* Deis
* Flynn
* Dokku

## 容器支持技术

{% asset_img 容器支持技术.png 容器支持技术 %}

### 容器网络
容器的出现使网络拓扑变得更加动态和复杂。用户需要专门的解决方案来管理容器与容器，容器与其他实体之间的连通性和隔离性。

* docker network
* flannel
* weave
* calico

### 服务发现
动态变化是微服务应用的一大特点。当负载增加时，集群会自动创建新的容器; 负载减小，多余的容器会被销毁。容器也会根据 host 的资源使用情况在不同的 host 中迁移，容器的 IP 和端口也会随之发生变化。那么就必须有一种机制可以让 client 能够知道如何访问容器提供的服务

* etcd
* consul
* zookeeper

### 监控
* docker ps/top/stats
* docker stats API
* sysdig
* cAdvisor/Heapster
* Weave Scope

### 数据管理
* Rex-Ray

### 日志管理
* docker logs
    * 是 Docker 原生的日志工具
* logsput

### 安全性
* OpenSCAP


# 容器核心知识概述
## 什么是容器(Container)
容器是一种轻量级、可移植、自包含的软件打包**技术**, 使应用程序可以在几乎任何地方以相同的方式运行。
### 容器与虚拟机
* 容器
    * 不会虚拟硬件层
    * 用 namespace 和 cgroups 进行隔离
      * namespace 使每个进程只看到它自己的系统视图 (文件, 进程, 网络接口, 主机名). 进程只能看到同一个命名空间下的资源。
        * 存在多种类型的多个命名空间. 所以一个进程属于每个类型的一个命名空间
          * Mount (mnt)
          * ProcessID (pid)
          * Network (net)
          * Inter-process communication (ipd)
          * UTS
          * UserID (user)
        * 通过分派两个不同的 UTS 命名空间给一对进程, 能使他们看见不同的本地主机名
        * 一个进程属于什么 Network 命名空间决定了运行在进程里的应用程序能看见什么网络接口.
      * cgroups 限制了进程能使用的资源量(CPU, 内存, 网络带宽)
        * cgroups 是一个 Linux 内核功能, 它被用来限制一个进程或者一组进程的资源使用
    * 应用程序本身
    * **依赖**
        * 应用程序需要的库或其他软件容器在 Host 操作系统的用户空间中运行，与**操作系统的其他进程隔离**
        * 这一点显著区别于虚拟机
* 虚拟机
    * 在一个宿主的平台上又搭建出一个完全隔离的环境
        * 把 CPU, 内存, 硬盘, 网卡, 显卡, 声卡都虚拟化了
        * 在一整套虚拟硬件的基础上，再搭建一个虚拟系统
    * VMWare, KVM, Xen
    * 为了运行应用，除了部署应用本身及其依赖，还得安装整个操作系统

{% asset_img 容器与虚拟机.png 容器与虚拟机 %}
1. 每个虚拟机需要运行自己的一组系统进程, 这就产生了除组件进程消耗以外的额外计算资源损耗
2. 一个容器仅仅是运行在宿主机上被隔离的单个进程, 仅消耗应用容器消耗的资源, 不会有其他进程的开销

## 为什么需要容器
> 容器使软件具备了超强的可移植能力

* 一个应用包含多种服务，这些服务有自己所依赖的库和软件包
* 存在多种部署环境，服务在运行时可能需要动态迁移到不同的环境中

Docker 将集装箱思想运用到软件打包上，为代码提供了一个基于容器的标准化运输系统。Docker 可以将任何应用极其依赖打包成一个轻量级、可移植、自包含的容器。容器可以运行在几乎所有的操作系统上。

只需要配置好标准的 runtime 环境，服务器就可以运行任何容器。容器消除了开发、测试、生产环境的不一致性。

## 容器是怎么工作的
### Docker 架构
Docker 的核心组件
* Docker 客户端: Client
* Docker 服务端: Docker daemon
* Docker 镜像: Image
* Registry
* Docker 容器: Container

{% asset_img Docker架构.png Docker架构 %}

* Docker 采用 Client/Server 架构。
* 客户端向服务端发送请求，服务端负责构建、运行和分发容器
* 客户端和服务端可以运行在同一个 Host 上，客户端也可以通过 socket 或 REST API 与远程的服务器通信

### Docker 客户端
### Docker 服务端
### Docker 镜像
可将 Docker 镜像看成只读模板，通过它可以创建 Docker 容器。
* 从无到有创建镜像
* 下载并使用别人创建好的现成镜像
* 在现有镜像上创建新的镜像

### Docker 容器
Docker 容器就是 Docker 镜像运行的实例

### Registry
Registry 是存放 Docker 镜像的仓库，Registry 分私有和公有两种。


# Docker 镜像
## 镜像的内部结构
### base 镜像
* 不依赖其他镜像，从 scratch 构建
* 其他镜像可以以之为基础进行扩展
* 能称作 base 镜像的通常都是各种 Linux 发行版的 Docker 镜像
    * Ubuntu
    * Debian
    * CentOS

```bash
$ docker pull centos
$ docker images centos
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos              latest              0d120b6ccaa8        6 weeks ago         215MB
```

#### 为什么 SIZE 这么小
Linux 操作系统是由内核空间和用户空间构成的

1. rootfs

* rootfs (用户空间 文件系统)
    * /dev
    * /proc
    * /bin
    * /etc
    * /usr
    * /tmp
    * ...
* bootfs
    * kernel (内核空间)

对于 base 镜像来说，底层直接用 host 的 kernel, 自己只需要提供 rootfs 就好了

2. base 镜像提供的是最小安装的 Linux 发行版

3. 支持多种 Linux OS

### 镜像的分层结构

```Dockerfile
FROM debian
RUN apt-get install emacs
RUN apt-get install apache2
CMD ["/bin/bash"]
```
1. 从 debian base 镜像上构建
2. 安装 emacs 编辑器
3. 安装 apache2
4. 容器启动时运行 bash

{% asset_img 镜像的分层结构.png 镜像的分层结构 %}

#### 可写的容器层
* 当容器启动的时候，一个新的可写层被加载到镜像的顶端
* 通常被称为**容器层**，**容器层**下面的叫做**镜像层**

{% asset_img 镜像层.png 镜像层 %}

所有对容器的改动，无论添加、删除还是修改文件都只会发生在容器层中。只有容器层是可写的，容器层下面的镜像层是只读的。只有当需要修改时才复制出一份数据，这种特性被称之为 Copy-on-Write. 可见容器保存的是镜像变化的部分，不会对镜像本身进行修改。

## 构建镜像
### docker commit
* 运行容器
* 修改容器
* 将容器保存为新的镜像

```bash
$ docker run -it ubuntu
root@d4729bbdc9fb:/#
# -it 参数的作用是已交互模式进入容器，并打开终端。d4729bbdc9fb 是容器内部 ID
```

// todo 跳过

## 分发镜像
* 用相同的 Dockerfile 在其他 host 构建镜像
* 将镜像上传到公共 Registry (比如 Docker Hub) Host 直接下载使用
* 搭建私有的 Registry 供本地 Host 使用



## 参考文献

《每天5分钟玩转Docker容器技术》