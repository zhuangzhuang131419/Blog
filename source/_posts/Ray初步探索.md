---
title: Ray初步探索
date: 2020-12-02 10:36:24
tags: Ray
categories: Ray
---
# Overview of Ray

## Why Ray ?
有很多教程解释了怎么去使用 Python 的 ```multiprocessing module```. 但不幸的是, ```multiprocessing module``` 在解决现代应用的需求时有很大的局限性. 我们现在应用的需求有:
* 在多台计算机上运行相同的代码
* 在不同服务器之间可以进程通信
* 可以轻松解决debug
* 有效地处理大型对象和数值数据

> Ray是一个分布式执行引擎。同样的代码可以在一台机器上实现高效的多处理，也可以在集群是用于大型的计算。

## Necessary Concept
传统的编程依赖两个核心的概念: **function** 和 **classes**. 但是当我们把我们的应用迁移到分布式系统的时候，这时候概念就变了. 

> Ray takes the existing concepts of **functions** and **classes** and translates them to the distributed setting as **tasks** and **actors**.

### From Classes to Actors
如何理解 class 和 actor 之间的关系呢

Python 允许你用 ```@ray.remote``` 去修饰一个 class. 当这个 class 被实例化的时候, Ray 就会在集群中创建一个 ```actor``` 进程, 它拥有这个object 的副本. 在这个 ```actor``` 上的方法调用会转变成 ```task``` 在 ```actor``` 进程上运行并且可以访问和修改 ```actor```的状态.

单个 ```actor``` 线性的执行函数 (每一个单独的函数都是原子的)

## Starting Ray

调用```ray.init()``` 启动所有 Ray 相关的进程.

* 一些 ```worker``` 进程启动, 用来并行的处理 Python function (基本是一个 CPU 核一个```worker```)
* 一个 ```scheduler``` 进程启动, 用来分配 ```tasks``` 给 ```workers```. ```task``` 是 Ray 用来分配任务的单位, 对应一个 function invocation 或者 method invocation.
* 创建一个 ```shared-memory object store```, 用来在 ```workers``` 之间高效的共享对象 (而不是通过创造副本)
* 一个在内存中的数据库用来对元数据排序，元数据可以作为 machine failures 事件的返回

### 进程介绍
当我们使用Ray时，涉及到多个进程。

* 多 ```worker``` 进程执行多个任务并将结果存储在对象存储中，每个 ```worker``` 都是一个独立的进程。
  * 为什么不是对应的 thread 是因为多线程在 Python 中由于 global interpreter lock 的影响有很大的限制. (即某一时刻, 只能有一个 thread 在运行)
* 每个 ```node``` 上的对象存储都将**不可变**的对象存储存在共享内存 (shared memory) 中, 允许 ```worker``` 以少量的复制和并行化有效的共享同一 ```node``` 上的存储对象。
* 每个 ```node``` 上的本地调度将任务分配给同一 ```node``` 上的 ```worker```. (一个```node``` 上的本地调度把任务分配给本 ```node``` 上的 ```worker```)
* 一个 ```driver``` 是用户控制 python 进程. 例如，如果用户正在运行脚本或者使用 python shell, 那么 ```driver``` 就是运行脚本或者 shell 的 python 进程。```driver``` 和 ```worker``` 很相似，他们都可以提交任务给本地调度并从对象存储中获取对象，但是不同之处是本地调度不会将任务分配给 ```driver``` 执行。
* Redis 服务器维护系统的大部分状态。

Q: Ray 执行异步任务时，是怎样实现平行的
> A: Ray 集群中每个 ```node``` 共享本 ```node``` 本地存储, ```node``` 中 ```worker``` 并行运行, 集群间 ```worker``` 的并行

Q: Ray 是怎么使用对象 ID 来表示不可变对象的远程对象的
> A: 任务执行前, 对象值已经存入存储对象中, 任务执行是通过对象 ID 调用存储对象的

## 不可变的远程 (remote) 对象
* 在 Ray 中, 我们可以在对象上创建和计算. 我们将这些对象称为远程对象, 并使用**对象 ID** 来引用它们.
* 远程对象是被存储在对象存储中的, 在集群中每个节点都有一个存储对象.
* 在集群设置中, 我们可能实际上不知道每个存储对象的位置.
* 由于远程对象是不可变的，那么他们的值在创建之后不能更改。这允许在多个对象存储中复制远程对象，而不需要同步副本

## Put, Get

Ray 中的 ```ray.get``` 和 ```ray.put``` 是用作 Python 对象和对象 ID 之间的转换

```Python
x = "example"
# 把一个 Python 对象复制到本地对象存储中(本地意味着同一节点). 一旦对象被存入存储对象后, 他的值就不能被改变了.
ray.put(x)  # ObjectID(b49a32d72057bdcfc4dda35584b3d838aad89f5d)
```

Ray的 ```ray.put()``` 返回的是一个对象 ID (其实就是一个引用), 可以被用来创建新的远程对象.

```Python
x_id = ray.put("example")
# 接受一个对象 ID, 并从相应的远程对象创建一个 Python 对象.
ray.get(x_id) # "example"
```

对于数组这样的对象, 我们可以使用内存的共享从而避免复制对象. 对于其他对象, 它将对象从对象存储中复制到 worker 进程的堆. 如果与对象 ID ```x_id``` 对应的远程(remote)对象与调用 ```ray.get(x_id)``` 的 ```worker``` 不在同一 ```node``` 上, 那么远程对象将首先从拥有它的对象存储区转移到需要他的对象存储区


> tips: function先后顺序，影响ray，例如b函数调用a函数，那么a不加修饰器，就必须放置b前面


## 参考文献
* [Ray 入门指南(1) --- Ray 分布式框架的介绍](https://blog.csdn.net/weixin_43255962/article/details/88689665)
* [Modern Parallel and Distributed Python: A Quick Tutorial on Ray](https://towardsdatascience.com/modern-parallel-and-distributed-python-a-quick-tutorial-on-ray-99f8d70369b8)





# RAY CORE 



## Ray Core Walkthrough



## Using Ray

见



## Configuring Ray



## Ray Dashboard









# Ray Cluster
Ray 可以在单个机器上运行, 但是 Ray 真正的强大之处在于它可以在一个机器集群上运行。



## Distributed Ray Overview
### 概念
* **Ray Nodes**: 一个 Ray 集群包含一个 ```head node``` 和 一些 ```worker nodes```. 首先运行的是 ```head node```, 然后 ```worker node``` 会得到 ```head node``` 在集群中的地址. Ray 的集群可以自动扩容, 这意味着它可以根据当前的负载创建或者销毁实例.
* **Ports**: Ray 的进程通过 TCP 端口进行交流.
* **Ray Cluster Launcher**: 这是一个可以自动提供机器并且发布一个多节点的 Ray 集群的工具.



一个Ray集群包含了一个**head node**和一堆**worker node**还有一个中心的全局控制储存实例（Global Control Store, **GCS**）。**head node**需要先启动，所有的**worker node**都会有**head node**的地址。



系统的一些元数据由GCS管理，例如actor的地址。

{% asset_img Ray集群.png Ray集群 %}



我们可以使用**Ray Cluster Launcher**配置计算机并启动多节点Ray群集。可以在AWS、GCP、Azure、Kubernetes、内部部署和Staroid上使用集群启动器，甚至可以在您的自定义节点提供程序上使用。





### 所有权关系

{% asset_img 所有权关系.png 所有权关系 %}

每个worker进程管理并拥有它提交的task以及task的返回（ObjectRef）。这个所有者对task是否执行以及ObjectRef对应的值是否能够被解析负责。worker拥有过它通过`ray.put`创造的object



### 组件

#### worker

一个Ray实例包含了多个**worker nodes**。每个节点包含以下物理进程：

* 一个或者多个work processes，负责task的提交和执行。
* 一个所有权的对应表。记录 objects 和对应 worker 的引用计数(ref counts)
* 一个进程内的存储。用于存储小的 object
* Raylet ，Raylet 在同一个集群共享所有的 jobs 。raylet 有两个主要的线程：
  * 调度线程。负责资源管理和在分布式对象存储里写入任务参数。集群里面的独立的调度都包含 Ray 的分布式调度
  * 共享内存对象（plasma 对象存储）。负责存储和转换大的对象。集群里面每个对象存储都包含 Ray 分布式对象存储



#### head

区别于其他进程，head承担以下任务：

* Global Control Store(GCS)。GCS是一个 key-value 的服务器，包含了系统级别的元数据，例如 objects 和 actors 的位置。有一个进行中的对 GCS 的优化，以便 GCS 可以运行在任意或者多个节点，而不是设定的 head 节点。
* Driver进程。dirver 是一个特殊的 worker 进程（运行`ray.init()`的进程），它执行上层应用（例如，python 里的`__main__`）。它能够提交 tasks，但是自己本身不能执行。Driver 进程能够运行在任何 node 熵，但是默认是在 head node里。



### 具体如何工作的

Ray集群将自动启动一个基于负载的autoscaler。autoscaler资源要求scheduler查看集群中的挂起任务、参与者和放置组资源需求，并尝试添加能够满足这些需求的最小节点列表。当工作节点空闲超过一定时间时，它们将被删除（头节点永远不会被删除，除非集群被拆除）。



Autoscaler使用一个简单的装箱算法将用户需求装箱到可用的集群资源中。剩余的未满足需求被放置在满足需求的最小节点列表上，同时最大化利用率（从最小节点开始）。











## 参考文献

* [Ray 1.0架构解读](https://zhuanlan.zhihu.com/p/344736949)





# Ray Serve

Ray Serve 是基于 Ray 构建的可伸缩模型服务库

## Ray Serve: Scalable and Programmable Serving
* 框架不可知 (Framework Agnostic): 使用相同的工具包即可提供服务, 从使用 PyTorch 或 TensorFlow & Keras 等框架构建的深度学习模型到 Scikit-Learn 模型或任意业务逻辑.
* Python 优先 (Python First): 使用纯 Python 代码配置服务的模型 - 不再需要 YAML 或 JSON 配置.
* 面向性能 (Performance Oriented): 启用批处理, 流水线处理和 GPU 加速, 以提高模型的吞吐量.
* 本机组合 (Composition Native): 允许你将多个模型组合在一起以创建单个预测, 从而创建"模型管道".
* 水平可扩展 (Horizontally Scalable): 服务可以随着你添加更多计算机而线性扩展.

## Key Concepts
### Backends

### Endpoints





# 参考文献
  * [官方文档](https://docs.ray.io/en/latest/index.html)
  * [Ray 分布式计算框架介绍](https://zhuanlan.zhihu.com/p/111340572)
  * [取代Python多进程！高性能分布式执行框架 - Berkeley Ray](https://www.jiqizhixin.com/articles/2020-09-11-11)