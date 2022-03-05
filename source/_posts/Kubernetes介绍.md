---
title: Kubernetes介绍
date: 2020-12-08 16:50:53
tags: 分布式系统
categories: kubernetes
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





### 系统的逻辑部分



#### ReplicationController, pod和服务是如何组合在一起的

通过运行`kubectl run`命令，创建了一个ReplicationController，它用于创建pod实例。为了使该pod能够从集群外部访问，需要让Kubernetes将该ReplicationController管理的所有pod由一个服务对外暴露。

{% asset_img 由ReplicationController,pod和服务组成的系统.png 由ReplicationController,pod和服务组成的系统 %}



> pod和它的容器

* 通常一个pod可以包含任意数量的容器
* pod有自己独立的私有IP地址和主机名



> ReplicationController的角色

* 始终确保一个运行中的pod实例。
* ReplicationController用于复制pod并让他们保持运行。



> 为什么需要服务

pod的存在是短暂的，一个pod可能会在任何时候消失，消失的pod会被ReplicationController替换为新的pod。新的pod与替换它的pod具有不同的ip地址。



服务就是需要解决不断变化的pod IP地址的问题，以及在一个固定的IP和端口对上对外暴露多个pod.



当一个服务被创建的时，它会得到一个静态的IP，在服务的生命周期中这个IP不会发生改变。客户端应该通过固定IP地址连接到服务，而不是直接连接pod。服务会确保其中一个pod接收连接，而不关心pod当前运行在哪里。



### 水平伸缩应用

> Kubernetes最基本的原则：不是告诉Kubernetes应该执行什么操作，而是声明性地改变系统期望状态，并让Kubernetes检查当前的状态是否与期望的状态一致。



```bash
$ kubectl scale rc kubia --replicas=3
replicationcontroller "kubia" scaled

$ kubectl get rc
NAME     DESIRED    CURRENT    READY    AGE
kubia    3          3          2        17m

```



{% asset_img 由同一ReplicationController管理的多实例应用.png 由同一ReplicationController管理的多实例应用 %}





### 查看应用运行在哪个节点上

在Kubernetes的世界中，pod运行在哪个节点上并不重要，只要它被调度到一个可以提供pod正常运行所需的CPU和内存的节点就可以了。





# pod：运行于Kubernetes中的容器

## 介绍pod

* pod是一组并置的容器，代表了Kubernetes中的基本构建模块。
  * 只包含一个单独容器的pod也是非常常见的。
  * 当一个pod包含多个容器时，这些容器总是运行于同一个工作节点上。（一个pod不会跨越多个工作节点）

### 为何需要pod

> 多个容器比单个容器中包含多个进程要好

容器被设计为每个容器只运行一个进程。如果在单个容器中运行多个不相关的进程，那么保持所有进程运行、管理他们的日志等将会是我们的责任。



### 了解pod

> 同一pod中容器之间的部分隔离

容器之间彼此是完全隔离的，但我们期望的是隔离容器组，而不是单个容器，并让每个容器组内的容器共享一些资源，而不是全部。K8s通过配置Docker来让一个pod内的所有容器共享相同的Linux命名空间，而不是每个容器都有自己的一组命名空间。



> 容器如何共享相同的IP和端口空间

由于一个pod中的容器运行与相同的Network命名空间中，因此它们共享相同的IP地址和端口空间。这意味着在同一pod中的容器运行的多个进程需要注意不能绑定到相同的端口号，否则会导致端口冲突，但这只涉及同一pod中的容器。由于每个pod都有独立的端口空间，对于不同pod中的容器来说则永远不会遇到端口冲突。



> 介绍平坦pod间网络

Kubernetes集群中的所有pod都在同一个共享网络地址空间中，这意味着每个pod都可以通过其他pod的IP地址来实现相互访问。

它们之间没有NAT网关。当两个pod彼此之间发送网络数据包时，它们都会将对方的实际IP地址看做数据包中的源IP。



### 通过pod合理管理容器

> 将多层应用分散到多个pod中

不推荐在单个pod中同时运行前端服务器和数据库这两个容器，无法充分利用第二个节点上的计算资源。



> 基于扩缩容考虑而分割到多个pod中

另一个不应该将应用程序都放到单一pod中的原因就是扩缩容。pod也是扩缩容的基本单位。所以需要把前后端的容器进行分离。



> 何时在pod中使用多个容器

将多个容器添加到单个pod的主要原因是应用可能由一个主进程和一个或多个辅助进程组成





## 以YAML或JSON描述文件创建pod



几乎在所有Kubernetes资源中都可以找到的三大重要部分：

* metadata 包括名称、命名空间、标签和关于该容器的其他信息
* spec 包含pod内容的实际说明，例如pod的容器
* status 包含运行中的pod的当前信息，例如pod所处的条件、每个容器的描述和状态，以及内部IP和其他基本信息



我们使用```kubectl create```命令从YAML文件创建pod



```bash
$ kubectl create -f kubia-manual.yaml
pod/kubia-manual created
```



查看应用程序日志

```bash
$ kubectl logs kubia-manual
Kubia server starting...

# 指定容器
$ kubectl logs kubia-manual -c kubia
Kubia server starting...
```



### 向pod发送请求

可以使用```kubectl expose```创建一个service，以便外部访问该pod。这里先介绍一下端口转发



> 端口转发

Kubernetes 将允许我们配置端口转发到该pod。



```bash
$ kubectl port-forward kubia-manual 8888:8080
... Forwarding from 127.0.0.1:8888 -> 8080
... Forwarding from [::1]:8888 -> 8080


$ curl localhost:8888
You've hit kubia-manual
```



## 通过标签选择器列出pod子集

标签要与标签选择器结合在一起。



### 使用标签选择器列出pod

```bash
# 列出creation_method=manual的所有pod
$ kubectl get po -l creation_method=manual
NAME              READY   STATUS    RESTARTS   AGE
kubia-manual      1/1     Running   2          5d17h
kubia-manual-v2   1/1     Running   0          22m

# 列出包含env标签的所有pod
$ kubectl get po -l env
NAME              READY   STATUS    RESTARTS   AGE
kubia-manual-v2   1/1     Running   0          23m

# 列出没有env标签的pod
$ kubectl get po -l '!env'

# creation_method!=manual
# env in (prod,devel)
# env notin (prod,devel)
```



### 将pod调度到特定节点

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
	# 节点选择器要求Kubernetes只将pod部署到包含标签gpu=true的节点上
  nodeSelector:
    gpu: "true"
  containers:
  - image: luksa/kubia
    name: kubia
```



## 注解pod

> 与label不同，annotation并不是为了保存标识信息而存在的，它们不能像标签一样用于对对象进行分组。



查找对象的注解

```bash
$ kubectl get po kubia -o yaml
```



## 停止和移除pod

### 按名称删除pod

```bash
$ kubectl delete po kubia-gpu
pod "kubia-gpu" deleted
```



### 使用标签选择器删除pod

```bash
$ kubectl delete po -l creation_method=manual
pod "kubia-manual" deleted
pod "kubia-manual-v2" deleted
```





删除pod后，一直会出现一个新pod。这是因为我们创建了一个ReplicationController，然后再由ReplicationController创建pod。如果想删除该pod，我们还需要删除这个ReplicationController。



# 副本机制和其他控制器：部署托管的pod

可以创建ReplicationController或Deployment这样的资源，接着由它们来创建并管理实际的pod

## 保持pod健康

Kubernetes可以通过存活探针 (liveness probe) 检查容器是否还在运行。



### 介绍存活探针

Kubernetes可以通过存活探针检查容器是否还在运行。可以为pod中的每个容器单独指定存活探针。如果探测失败，Kubernetes将定期执行探针并重新启动容器。

Kubernetes有三种探测容器机制：

* HTTP GET探针对容器的IP地址执行HTTP GET请求
* TCP套接字探针尝试与容器指定端口建立TCP连接。
* Exec探针在容器内执行任意命令，并检查命令的退出状态码。

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    # 一个HTTP GET 存活探针
    livenessProbe:
      httpGet:
        path: /    # HTTP请求的路径
        port: 8080 # 探针连接的网络端口
```

> 该探针在Kubernetes定期在端口8080路径上执行HTTP GET请求，以确定该容器是否健康。



### 配置存活探针的附加属性

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:

  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 15 # Kubernetes会在第一次探测前等待15秒
```



### 创建有效的存活探针



* 存活探针应该检查什么
* 保持探针轻量
* 无须在探针中实现重试循环



Kubernetes会在容器崩溃或其存活探针失败时，通过重启服务器来保持运行。这项任务由承载pod的节点上的Kubelet执行——在主服务器上运行的Kubernetes Control Plane组件不会参与此过程。但如果节点本身崩溃，那么Control Plane必须为所有随节点停止运行的pod创建替代品。它不 会为你直接创建的pod执行此操作 。 这些pod只被Kubelet 管理 ， 但由于 Kubelet 本身运行在节点上， 所以如果节点异常终止，它将无法执行任何操作。







## 了解ReplicationController

ReplicationController是一种Kubernetes资源，可确保它的pod始终保持运行状态。如果pod因任何原因消失，则ReplicationController会注意到缺少了pod并创建替代pod。



### 控制器的协调流程

ReplicationController不是根据pod类型来执行操作的，而是根据pod是否匹配某个标签选择器。

{% asset_img ReplicationController协调流程.jpg ReplicationController协调流程 %}



一个ReplicationController有三部分组成

* label selector：用于确定ReplicationController作用域中有哪些pod
* replica count：指定营运性的pod数量
* pod template：用于创建新的pod副本



### 使用ReplicationController的好处

* 确保一个pod持续运行（现有pod丢失时启动一个新的pod）
* 集群节点发生故障时，它将为故障节点上运行的所有pod创建替代副本
* 能轻松实现pod的水平伸缩



### 创建一个ReplicationController

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
	# ReplicationController的名字
  name: kubia 
spec:
	# pod 实例的目标数目
  replicas: 3
  # pod 选择器决定了RC的操作对象
  selector:
    app: kubia
  # 创建新pod所用的pod模板
  template:
    metadata:
      labels:
      	# 这个标签要和selector中的标签一致
        app: kubia 
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```







### 创建一个ReplicationController

```bash
$ kubectl create -f kubia-rc.yaml
```



```yaml
apiVersion: v1
kind: ReplicationController
metadata:
	# ReplicationController的名字
  name: kubia
spec:
  replicas: 3
  selector:
  	# pod选择器决定了RC的操作对象
    app: kubia
  # 创建新pod所用的pod模板
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080
```





### 将pod移入或移出ReplicationController的作用域



由ReplicationController创建的pod并不是绑定到ReplicationController。在任何时刻，ReplicationController管理与标签选择器匹配的pod。通过更改pod的标签，可以将它从ReplicationController的作用域中添加或删除。



当更改pod的标签时，ReplicationController发现一个pod丢失了，并启动一个新的pod替换它。



如果不是更改某一个pod的标签而是修改ReplicationController的标签选择器，会让所有的pod脱离ReplicationController的管理，导致它创建三个新的pod。



### 修改pod模板

只会影响之后的pod，并不会影响已经产生的pod

```bash
$ kubectl edit rc kubia
```





### 水平缩放pod

1. 可以通过编辑ReplicationController中的`spec.replicas`的值
2. 可以通过```kubectl scale```命令



### 删除一个ReplicationController

当通过```kubectl delete```删除ReplicationController时，pod也会被删除。可以通过给命令增加```--cascade=false```选项来保持pod的运行。





## 使用ReplicaSet而不是ReplicationController

单个 ReplicationController 无法将 pod 与标签 env=production 和 env = devel 同时匹配。 它只能匹配带有 env = devel 标签的 pod 或带有 env = devel 标签的 pod 。 但是 一 个 ReplicaSet 可以匹配两组 pod 并将它们视为一个大组。



### 定义ReplicaSet

```yaml
apiVersion: apps/v1beta2
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  # 该模板与ReplicationController中的相同
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:

   - name: kubia
     image: luksa/kubia
```



## 运行执行单个任务的pod

当遇到只想运行完成工作后就终止任务的情况。ReplicationController，ReplicaSet和DaemonSet会持续运行任务，永远达不到完成态。



### Job资源

> 它允许你运行一种pod，该pod在内部进程成功结束时，不重启容器。一旦任务完成，pod就被认为处于完成状态。



在发生节点故障时，该节点上由Job管理的pod将按照ReplicaSet的pod的方式，重新安排到其他节点。



### 定义Job资源

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    metadata:
      labels:
        app: batch-job
    spec:
    	# Job不能使用Always为默认的重新启动策略
      restartPolicy: OnFailure
      containers:

   - name: main
     image: luksa/batch-job
```







# 服务：让客户端发现pod并与之通信

pod需要一中寻找其他pod的方法来使用其他pod提供的服务。在非Kubernetes的世界，系统管理员要在用户端配置文件中明确指出服务的精确IP地址或者主机名来配置每个客户端应用，但在Kubernetes中并不适用。

* pod是短暂的——它们随时会启动或者关闭
* Kubernetes在pod启动前会给已经调度到节点上的pod分配IP地址——因此客户端不能提前知道提供服务的pod的IP地址
* 水平伸缩意味着多个pod可能会提供相同的服务——每个pod都有自己的IP地址，客户端无需关心后端提供服务pod的数量，以及各自对应的IP地址



## 介绍服务

> Kubernetes服务是一种为一组功能相同的pod提供单一不变的接入点的资源。

客户端通过IP地址和端口号建立连接，这些连接会被路由到提供服务的任意一个pod上。

{% asset_img 介绍服务.jpg 介绍服务 %}

* 暴露一个单一不变的IP地址让外部的客户端连接pod。同理，可以为后台数据库pod创建服务，并为其分配一个固定的IP地址。
* 尽管pod的IP地址会改变，但是服务的IP地址固定不变。
* 通过创建服务，能够让前端的pod通过环境变量或DNS以及服务名来访问后端服务。



### 创建服务

可以使用标签的方式来指定哪些pod属于服务，哪些pod不属于。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - port: 80
    targetPort: 8080
  # 具有app=kubia标签的pod都属于该服务
  selector:
    app: kubia
```



> 配置服务上的会话亲和性

如果希望特定客户端产生的所有请求每次都指向同一个pod，可以设置服务的sessionAffinity为ClientIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  sessionAffinity: ClientIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: kubia
```



> 同一个服务暴露多个端口

创建的服务可以暴露一个端口，也可以暴露多个端口。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia
spec:
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  selector:
    app: kubia
```





如果创建了不包含pod选择器的服务，Kubernetes将不会创建Endpoint资源。这样就需要创建Endpoint资源来指定该服务的endpoint列表。





## 将服务暴露给外部客户端



* 将服务的类型设置成NodePort，每个集群节点在节点本身上打开一个端口
* 将服务的类型设置成LoadBalance，这使得服务可以通过一个专用的负载均衡器来访问
* 创建一个Ingress资源，通过一个IP地址公开多个服务



### 使用NodePort类型的服务

创建一个服务并将其类型设置为NodePort









## pod就绪后发出信号

pod可能需要时间来加载配置或数据，或者可能需要执行预热过程以防止第一个用户请求时间太长影响了用户体验。在这种情况下，不要将请求转发到正在启动的pod中，直到完全准备就绪。



### 介绍就绪探针

就绪探测器会定期调用，并确定特定的pod是否接收客户端请求。当容器准备就绪探测返回成功时，表示容器已准备好接受请求。



#### 就绪探针类型

* Exec探针，执行进程的地方
* HTTP GET探针，向容器发送HTTP GET请求，通过响应的HTTP状态代码判断容器是否准备好。
* TCP socket探针，它打开一个TCP连接到容器的指定端口。如果连接已建立，则认为容器已准备就绪。



与存活探针不同，如果容器未通过准备检查，则不会被终止或重新启动。存活探针通过杀死异常的容器并用新的正常容器替代它们来保持pod正常工作，而就绪探针确保只有准备好处理请求的pod才可以接受它们。



就绪探针确保客户端只与正常的pod交互，并且永远不会知道系统存在问题



## 连接集群外部的服务



### endpoint

```bash
$ kubectl describe svc kubia
Name:              kubia
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=kubia
Type:              ClusterIP
IP:                10.105.246.179
Port:              <unset>  80/TCP
TargetPort:        8080/TCP
Endpoints:         10.1.2.24:8080,10.1.2.25:8080,10.1.2.26:8080
Session Affinity:  None
Events:            <none>
```



服务并不是和 pod 直接相连的。有一个Endpoint资源介于两者之间。Endpoint资源就是暴露一个服务的IP地址和端口的列表。当客户端连接到服务时，服务代理选择这些IP和端口对中的一个，并将传入连接重定向到在该位置监听的服务器。







# 卷：将磁盘挂载到容器

主要介绍容器是如何访问外部磁盘存储的，以及如何在他们之间共享存储空间。



pod类似逻辑主机，在逻辑主机中运行的进程共享诸如CPU、RAM、网络接口但是不能共享磁盘。pod中的每个容器都有自己独立的文件系统、因为文件系统来自容器镜像。



## 介绍卷 (Volume)

> volume是pod的一个组成部分，因此像容器一样在pod的规范中就定义了。它不是独立的Kubernetes对象，也不能单独创建或删除。

在pod启动时创建卷，并在删除pod时销毁卷。因此，在容器重新启动期间，卷的内容将保持不变，在重新启动容器之后，新容器可以识别前一个容器写入卷的所有文件。





### 介绍持久卷和持久卷声明

PersistentVolume（持久卷，PV）

{% asset_img 持久卷和持久卷声明的关系.jpg 持久卷和持久卷声明的关系%}

当集群用户需要在其pod中使用持久化存储时，他们首先创建持久卷声明（PersistentVolumeClaim, PVC）清单，指定所需要的最低容量要求和访问模式，然后用户将持久卷声明清单提交给Kubernetes API服务器，Kubernetes 将找到可匹配的持久卷并将其绑定到持久卷声明。



**PV不属于任何namespace，而pod和PVC有各自的namespace**



* RWO-ReadWriteOnce-仅允许单个节点挂载读写
* ROX-ReadOnlyMany-允许多个节点挂载只读
* RWX-ReadWriteMany-允许多个节点挂载读写这个卷





# ConfigMap和Secret：配置应用程序



## 利用ConfigMap解耦配置



### ConfigMap介绍

Kubernetes允许将配置选项分离到单独的资源对象ConfigMap中，本质上就是一个键/值对映射。

pod通过环境变量与ConfigMap卷使用ConfigMap











# 从应用访问pod元数据以及其他资源



## 通过Downward API传递数据

> Downward API允许我们通过环境变量或者文件的传递pod的元数据。





### 可用的元数据

* pod的名称
* pod的IP
* pod所在的命名空间
* pod运行节点的名称
* pod运行所归属的服务账户的名称
* 每个容器请求的CPU和内存的使用量
* 每个容器可以使用的CPU和内存限制
* pod的标签
* pod的注解

















# Deployment：声明式地升级应用





## 更新运行在pod内的应用程序



### 删除旧版本pod，使用新版本pod替换

{% asset_img 升级并删除原有pod.jpg 升级并删除原有pod %}

缺点就是删除旧的pod到启动新pod之间短暂的服务不可用



### 先创建新pod再删除旧版本pod

1. 从旧版本立即切换到新版本

* 运行新版本的pod之前，Service只将流量切换到初始版本的pod
* 一旦新版本的pod被创建并且正常运行之后，就可以修改服务的标签选择器并将Service的流量切换到新的pod
* 切换之后，一旦确定新版本的功能运行正常，就可以通过删除旧的ReplicationController来删除旧版本的pod

缺点：这会需要更多的硬件资源，因为你将在短时间内同时运行两倍数量的pod

2. 执行滚动升级操作

   通过逐步对旧版本的ReplicationController进行缩容并对新版本的进行扩容

   {% asset_img 滚动升级.jpg 滚动升级 %}



## 使用Deployment声明式地升级应用



Deployment 由 ReplicaSet 组成，并由它接管 Deployment 的 pod



当创建一个Deployment时，ReplicaSet资源也会随之创建





















# 了解Kubernetes机理



### Kubernetes组件的分布式特性

{% asset_img Kubernetes控制平面以及工作节点的组件.jpg Kubernetes控制平面以及工作节点的组件 %}



> 组件间如何通信

Kubernetes系统组件间只能通过API服务器通信，它们之间不会直接通信。

API服务器是和etcd通信的唯一组件，其他组件不会直接和etcd通信，而是通过API服务器来修改集群状态。



> 单组件运行多实例

尽管工作节点上的组件都需要运行在同一个节点上，控制平面的组件可以被简单地分割在多台服务器上。为了保证高可用性，控制平面的每个组件可以有多个实例。



> 组件是如何运行的

Kubelet是唯一一直作为常规系统组件来运行的组件，它把其他组件作为pod来运行。



### Kubernetes如何使用etcd

创建的所有对象——pod、ReplicationController、服务和私密凭据需要持久化存储到etcd中。



唯一能直接和etcd通信的是Kubernetes的API服务器。Kubernetes存储所有数据到etcd的/registry下。





> 确保etcd集群一致性

etcd使用RAFT一致性算法来保证状态一致性，确保在任何时间点，每个节点的状态要么是大部分节点的当前状态，要么是之前确认过的状态。



### API服务器做了什么

Kubernetes API服务器作为中心组件，其他组件或者客户端（如kubectl）都会去调用它。



API服务器除了提供一种一致的方式将对象存储到etcd，也对这些对象做校验，这样客户端就无法存入非法的对象了。除了校验，还会处理乐观锁，这样对于并发更新的情况，对对象做更改就不会被其他客户端覆盖。



### API服务器如何通知客户端资源变更

API服务器不需要告诉控制器去做什么，它只需要启动这些控制器，以及其他一些组件来监控已部署资源的变更。



{% asset_img API服务器给所有监听者发送更新过的对象.jpg API服务器给所有监听者发送更新过的对象 %}

客户端通过创建到API服务器的HTTP连接来监听变更。通过此连接，客户端会接收到监听对象的一系列变更通知。





### 了解调度器

宏观来看，调度器就是利用API服务器的监听机制等待新创建的pod，然后给每个新的、没有节点集的pod分配节点。



调度器不会命令选中的节点去运行pod。调度器做的就是通过API服务器更新pod的定义。然后API服务器再去通知Kubelet该pod已经被调度过。当目标节点上的Kubelet发现该pod被调度到本节点，它就会创建并且运行pod的容器。



> 使用多个调度器

可以在集群中运行多个调度器而非单个。对于每一个pod，可以通过在pod特性中设置schedulerName属性指定调度器来调度特定的pod。



### 介绍控制器管理器中运行的控制器

API服务器只做了存储资源到etcd和通知客户端有变更的工作；调度器则只是给pod分配节点。所以需要有活跃的组件确保系统真实状态朝API服务器定义的期望的状态收敛。这个工作由控制器管理器里的控制器来实现。



> [控制器源码](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller)，每个控制器一般有一个构造器，内部会创建一个Informer，其实是个监听器，每次API对象有更新就会被调用。通常，Informer会监听特定类型的资源变更事件。然后去看一下`worker()`方法。其中定义了每次控制器需要工作的时候都会调用`worker()`方法。



从比较高纬度的视角看，Controller Manager主要提供了一个分发事件的能力，而不同的Controller只需要注册对应的Handler来等待接收和处理事件



#### Replication 管理器



ReplicationController的操作可以理解为一个无限循环，每次循环，控制器都会查找符合其pod选择器定义的pod的数量，并且将该数值和期望的复制集数量作比较。



控制器不会每次循环去轮询pod，而是通过监听机制订阅可能影响期望的复制集数量或者符合条件pod数量的变更事件。

{% asset_img Replication管理器监听API对象变更.jpg Replication管理器监听API对象变更 %}

Replication管理器通过API服务器操纵pod API对象来完成其工作。所有控制器就是这样运作的。



#### ReplicaSet, DaemonSet以及Job控制器

DaemonSet以及Job控制器比较相似，从它们各自资源集中定义的pod模板创建pod资源。

这些控制器不会运行pod，而是将pod定义到发布API服务器，让Kubelet创建容器并运行。



#### Deployment控制器

Deployment控制器负责使deployment的实际状态与对应Deployment API对象的期望状态同步。



#### StatefulSet控制器



#### Node控制器



#### Namespace控制器

大部分资源归属于某个特定命名空间。当删除一个Namespace资源时，该命名空间里的所有资源都会被删除。当收到删除Namespace对象的通知时，控制器通过API服务器删除所有归属该命名空间的资源。



### Kubelet做了什么

Kubelet就是负责所有运行在工作节点上内容的组件。

* 在API服务器中创建一个Node资源来注册该节点
* 持续监控API服务器是否把该节点分配给pod，然后启动pod容器。
* 告知配置好的容器runtime来从特定容器镜像运行容器。
* Kubelet随后持续监控运行的容器，向API服务器报告它们的状态、事件和资源消耗。



尽管Kubelet一般会和API服务器通信并从中获取pod清单，它也可以基于本地指定目录下的pod清单来运行pod。



### Kubernetes Service Proxy的作用

除了Kubelet，每个工作节点还会运行kube-proxy，用于确保客户端可以通过Kubernetes API连接到你定义的服务。

> kube-proxy确保对服务IP和端口的连接最终能到达支持服务的某个pod处。如果有多个pod支撑一个服务，那么代理会发挥对pod的负载均衡作用







## 服务是如何实现的



### 引入kube-proxy

每个Service有其自己稳定的IP地址和端口。客户端（通常为pod）通过连接该IP和端口使用服务。IP地址是虚拟的，没有被分配给任何网络接口，当数据包离开节点时也不会列为数据包的源或目的IP地址。Service的一个关键细节是，它们包含一个IP、端口对，所以服务IP本身并不代表任何东西。



### kube-proxy如何使用iptables

当在API服务器中创建一个服务时，虚拟IP地址立刻就会分配给它。之后很短时间内，API服务器会通知所有运行在工作节点上的kube-proxy客户端有一个新服务已经被创建了。然后，每个kube-proxy都会让该服务在自己的运行节点上可寻址。



通过建立一些iptables规则，确保每个目的地为服务的IP/端口对的数据包被解析，目的地址被修改，这样数据包就会被重定向到支持服务的一个pod。







# 计算资源管理



## 为pod中的容器申请资源

创建一个pod的时候，可以指定**容器**对CPU和内存的资源请求量，以及资源限制量














# 参考文献
* 《 Kubernetes in Action 中文版 》