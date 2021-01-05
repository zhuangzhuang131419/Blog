---
title: Ray初步探索
date: 2020-12-02 10:36:24
tags: Ray
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


# Using Ray
## Starting Ray
* 什么是 Ray runtime (内存管理)
  * Ray programs are able to parallelize and distribute by leveraging an underlying Ray runtime.
  * The Ray runtime consists of multiple services/processes started in the background for communication, data transfer, scheduling, and more.
  * The Ray runtime can be started on a laptop, a single server, or multiple servers.
  
## Using Actors
一个 ```actor``` 实际上就是一个有状态的 ```worker```. 当一个新的 actor 被实例化，那么一个新的 worker 就诞生了，同时这个 actor 的 method 就被安排在那个特定的 worker 上。 actor 可以访问并修改那个 worker 的状态. 

* Worker 和 Actor 的区别
  * "Ray worker" 就是一个 Python ```进程```
  * "Ray worker" 要么被用来运行多个 Ray task 或者开始时对应一个专门的 actor


> Task: 当 Ray 在一台机器上运行的时候，会自动开始几个 ```Ray workers```. 他们被用来执行 ```task``` (类似一个进程池)

> Actor: 一个 Ray Actor 也是一个 "Ray worker" 只不过是在 runtime 实例化的. 所有的 methods 都运行在同一个进程里，使用相同的资源. 与 ```Task``` 不同的是，运行 Ray Actor 的进程不会重复使用并且会在 Actor 被删除后终止.


## AsyncIO / Concurrency for Actors
Ray 提供了两种 concurrency 的办法 ```async execution``` 和 ```threading```. Python 的 ```Global Interpreter Lock (GIL)``` 只允许在某一时刻运行一个thread, 那么我们就无法实现真正意义上的 parallelism. 一些常见的库，比如 Numpy, Cython, Tensorflow, PyTorch 在调用 C/C++ 函数的时候会释放 GIL. 但是```async execution``` 和 ```threading```无法绕开 GIL.


### AsyncIO for Actors
* [Python 中 async 与 await 的用法](https://juejin.cn/post/6844904088677662728)

```Python
import ray
import asyncio
ray.init()

@ray.remote
class AsyncActor:
    # multiple invocation of this method can be running in
    # the event loop at the same time
    async def run_concurrent(self):
        print("started")
        await asyncio.sleep(2) # concurrent workload here
        print("finished")

actor = AsyncActor.remote()

# regular ray.get
ray.get([actor.run_concurrent.remote() for _ in range(4)])

# async ray.get
await actor.run_concurrent.remote()
```

### Threaded Actors
有时候使用 asyncio 并不是一个理想的解决方案。比如你可能有一个method在进行大量的计算任务并阻塞了event loop, 且不能通过 ```await``` 去停止。这就对 Async Actor 的整体性能有影响，因为 Async Actor 在某一时刻只能执行一个任务并且依赖```await``` 去进行上下文切换。

可以使用 ```max_concurrency``` Actor 来代替, 类似线程池。

```Python
@ray.remote
class ThreadedActor:
    def task_1(self): print("I'm running in a thread!")
    def task_2(self): print("I'm running in another thread!")

a = ThreadedActor.options(max_concurrency=2).remote()
ray.get([a.task_1.remote(), a.task_2.remote()])
```

每个 threaded actor 的 ```Invocation``` 都会在线程池中运行。线程池的大小是由 ```max_concurrency``` 控制的。

## GPU Support



## Serialization
因为 Ray 进程不是内存空间共享的，数据在 ```workers``` 和 ```nodes``` 之间传输需要序列化和反序列化。Ray 使用 ```Plasma object store``` 高效的把对象传输给不同的 ```nodes``` 和 ```processes```.

### Plasma Object Store

**Plasma** 是一个基于 Apache Arrow 开发的内对象存储。所有的对象在 **Plasma object store** 中都是不可变的并且保存在共享内存中。

每个 node 都有自己的 object store. 当数据存入 object store, 它不会自动的广播通知其他的 node. 数据一直保留在本地直到在别的 node 被别的 task 或者 actor 请求。

### Overview
Ray 使用 ```Pickle protocol version 5``` 作为序列化协议。

## Memory Management 
Ray 当中的内存管理

{% asset_img Ray内存管理.png Ray内存管理 %}

我们把 Ray 的内存分成两个部分: ```Ray system memory``` 和 ```Application memory```.
* Ray system memory (这部分的内存是 Ray 内部在使用)
  * Redis
    * 存储在集群中的一系列 nodes 和 actors
    * 存储这部分用到的内存很小
  * Raylet
    * C++ raylet 进程在每个 node 上运行所需要的空间
    * 这部分不能够被控制，但是用到的内存空间也很小
* Application memory (这部分内存是我们的应用在使用)
  * Worker heap
    * 应用所使用的的内存 (e.g., in Python code or TensorFlow)
    * 需要用应用的 *resident set size* 减去 *shared memory usage* 来衡量。
  * Object store memory
    * 当应用通过 ```ray.put``` 创建对象并且返回的值来自 remote function
  * Object store shared memory
    * 当应用通过 ```ray.get``` 读取对象
    * 如果一个对象已经存在在一个 node 中, 这将不会导致额外的内存分配。这样可以使得一些比较大的对象在各个 actors 和 tasks 中共享的更有效率。

### Raylet 
我们可以抽象一个相对简单的 Worker 和 GCS 的关系:

{% asset_img Worker和GCS关系.png Worker和GCS关系 %}

Raylet 在中间的作用非常关键，包含了以下重要内容
* Node Manager
  * 基于 boost::asio 的异步通信模块，主要是通信的连接和消息处理管理
* Object Manager
  * Object Store 的管理
* gcs_client 或者 gcs server
  * gcs_client 是连接 GCS 客户端。如果设置 RAY_GCS_SERVICE_ENABLED 为 true 的话，这个 Server 就是作为 GCS 启动


#### Raylet 的启动过程

{% asset_img Raylet的启动过程.png Raylet的启动过程 %}

1. Raylet 的初始化，这里包含有很多参数。包括 Node Manager 和 gcs client 的初始化
2. 注册 GCS 准备接收消息。一旦有消息进来，就进入 Node Manager 的 ProcessClientMessage 过程。(TODO ProcessClientMessage的通信模型)

## Placement Groups (置放群组)
**Placement Groups** 允许用户跨多个```nodes```原子性的保存一组资源.(i.e., gang scheduling). **Placement Groups** 可以被用不同的策略来调度打包 ```Ray tasks``` 和 ```actors```.

> 这里的原子性意味着如果有一个 bundle 不适合当前所在的 node, 那么整个 Placement Group 都不会被创建.

**Placement Groups** 有以下应用
* Gang Scheduling: 
* Maximizing data locality
* Load balancing

### 关键概念
* **bundle** : 资源的集合
  * 一个 bundle 必须适合一个在集群中的 node
  * 然后 Bundles 会根据 ```placement group strategy``` 在集群中跨 nodes 放置
* **placement group** : bundle 的集合
  * 每一个 bundle 都被给予了一个在 placement group 的编号
  * 然后 Bundles 会根据 ```placement group strategy``` 在集群中跨 nodes 放置
  * 等 placement group 创建了后，```tasks``` 和 ```actors``` 可以根据 placement group 或者个人 bundles 来调度。

### 策略类型
* STRICT_PACK
  * All bundles must be placed into a single node on the cluster.
* PACK
  * All provided bundles are packed onto a single node on a best-effort basis. If strict packing is not feasible (i.e., some bundles do not fit on the node), bundles can be placed onto other nodes nodes.
* STRICT_SPREAD
  * Each bundle must be scheduled in a separate node.
* SPREAD
  * Each bundle will be spread onto separate nodes on a best effort basis. If strict spreading is not feasible, bundles can be placed overlapping nodes.


## Advanced Usage
### Synchronization
* Inter-process synchronization using FileLock
```Python
import ray
from filelock import FileLock

@ray.remote
def write_to_file(text):
    # Create a filelock object. Consider using an absolute path for the lock.
    with FileLock("my_data.txt.lock"):
        with open("my_data.txt","a") as f:
            f.write(text)

ray.init()
ray.get([write_to_file.remote("hi there!\n") for i in range(3)])

with open("my_data.txt") as f:
    print(f.read())

## Output is:

# hi there!
# hi there!
# hi there!
```
[理解 Python 关键字 "with" 与上下文管理器](https://zhuanlan.zhihu.com/p/26487659)

* Multi-node synchronization using SignalActor

> 当你有多个 tasks 需要等待同一个条件的时候，你可以使用一个 ```SingnalActor``` 来调度。

```Python
# Also available via `from ray.test_utils import SignalActor`
import ray
import asyncio

@ray.remote(num_cpus=0)
class SignalActor:
    def __init__(self):
        self.ready_event = asyncio.Event()

    def send(self, clear=False):
        self.ready_event.set()
        if clear:
            self.ready_event.clear()

    async def wait(self, should_wait=True):
        if should_wait:
            await self.ready_event.wait()

@ray.remote
def wait_and_go(signal):
    ray.get(signal.wait.remote())

    print("go!")

ray.init()
signal = SignalActor.remote()
tasks = [wait_and_go.remote(signal) for _ in range(4)]
print("ready...")
# Tasks will all be waiting for the singals.
print("set..")
ray.get(signal.send.remote())

# Tasks are unblocked.
ray.get(tasks)

##  Output is:
# ready...
# get set..

# (pid=77366) go!
# (pid=77372) go!
# (pid=77367) go!
# (pid=77358) go!
```

* Message passing using Ray Queue
```Python
import ray
from ray.util.queue import Queue

ray.init()
# You can pass this object around to different tasks/actors
queue = Queue(maxsize=100)

@ray.remote
def consumer(queue):
    next_item = queue.get(block=True)
    print(f"got work {next_item}")

consumers = [consumer.remote(queue) for _ in range(2)]

[queue.put(i) for i in range(10)]
```

* Dynamic Remote Parameters

### Dynamic Custom Resources
> 我们可以动态的去调整资源的需求或者返回在运行时调用```.option``` 返回 ```ray.remote``` 的值

```Python
@ray.remote(num_cpus=4)
class Counter(object):
    def __init__(self):
        self.value = 0

    def increment(self):
        self.value += 1
        return self.value

a1 = Counter.options(num_cpus=1, resources={"Custom1": 1}).remote()
a2 = Counter.options(num_cpus=2, resources={"Custom2": 1}).remote()
a3 = Counter.options(num_cpus=3, resources={"Custom3": 1}).remote()
```

# Ray Cluster
Ray 可以在单个机器上运行, 但是 Ray 真正的强大之处在于它可以在一个机器集群上运行.
## Distributed Ray Overview
### 概念
* **Ray Nodes**: 一个 Ray 集群包含一个 ```head node``` 和 一些 ```worker nodes```. 首先运行的是 ```head node```, 然后 ```worker node``` 会得到 ```head node``` 在集群中的地址. Ray 的集群可以自动扩容, 这意味着它可以根据当前的负载创建或者销毁实例.
* **Ports**: Ray 的进程通过 TCP 端口进行交流.
* **Ray Cluster Launcher**: 这是一个可以自动提供机器并且发布一个多节点的 Ray 集群的工具.


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