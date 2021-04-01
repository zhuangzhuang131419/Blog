---
title: Ray分布式计算框架——Using Ray
date: 2021-03-23 09:54:20
tags:
categories: Ray
---



# Using Ray

## Starting Ray

* 什么是 Ray runtime (内存管理)
  * Ray programs are able to parallelize and distribute by leveraging an underlying Ray runtime.
  * The Ray runtime consists of multiple services/processes started in the background for communication, data transfer, scheduling, and more.
  * The Ray runtime can be started on a laptop, a single server, or multiple servers.
* 启动Ray runtime的几种方式
  * 隐式的通过`ray.init()`
    * 不需要指定`address`
    * 直接在本机启动一个runtime，本机成为一个**head node**。
    * 当运行`ray.init()`的进程终止了，runtime也就终止了。也可以使用`ray.shutdown()`强制终止。
  * 显示的通过CLI
  * 显示的通过cluster launch
    * `ray up`使用Ray cluster launcher在云上启动一个集群，创建一个指定的**head node**和一些**worker node**。
    * 想要和现有的集群连接，调用`ray.init`并且指定集群的ip地址。

## Using Actors

一个 ```actor``` 实际上就是一个有状态的 ```worker```. 当一个新的 actor 被实例化，那么一个新的 worker 就诞生了，同时这个 actor 的 method 就被安排在那个特定的 worker 上。 actor 可以访问并修改那个 worker 的状态. 

* Worker 和 Actor 的区别
  * "Ray worker" 就是一个 Python `进程`
  * "Ray worker" 要么被用来运行多个 Ray task 或者开始时对应一个专门的 actor


> Task: 当 Ray 在一台机器上运行的时候，会自动开始几个 ```Ray workers```. 他们被用来执行 ```task``` (类似一个进程池)

> Actor: 一个 Ray Actor 也是一个 "Ray worker" 只不过是在 runtime 实例化的. 所有的 methods 都运行在同一个进程里，使用相同的资源. 与 ```Task``` 不同的是，运行 Ray Actor 的进程不会重复使用并且会在 Actor 被删除后终止。



下面将介绍两种actor同步的方式




## AsyncIO / Concurrency for Actors

Ray 提供了两种 concurrency 的办法 ```async execution``` 和 ```threading```. 

Python 的 ```Global Interpreter Lock (GIL)``` 只允许在某一时刻运行一个thread, 那么我们就无法实现真正意义上的 parallelism. 一些常见的库，比如 Numpy, Cython, Tensorflow, PyTorch 在调用 C/C++ 函数的时候会释放 GIL. 但是```async execution``` 和 ```threading```无法绕开 GIL.


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

Ray可以在`ray.remote`修饰器中对远程函数和对象中指定GPU的需求



### Starting Ray with GPUs

Ray会自动监测本机上可用的GPU数量。但我们也可以通过`ray.init(num_gpus=N)`或者`ray start --num-gpus=N`来重写。



Tips: Ray没有对设置超过实际数量GPU的操作采取保护机制。设置过多的GPU可能会导致GPU不存在的报错。



### Using Remote Functions with GPUs

```python
import os

@ray.remote(num_gpus=1)
def use_gpu():
    print("ray.get_gpu_ids(): {}".format(ray.get_gpu_ids()))
    print("CUDA_VISIBLE_DEVICES: {}".format(os.environ["CUDA_VISIBLE_DEVICES"]))
```







## Serialization

因为 Ray 进程不是内存空间共享的，数据在 ```workers``` 和 ```nodes``` 之间传输需要序列化和反序列化。Ray 使用 ```Plasma object store``` 高效的把对象传输给不同的 ```nodes``` 和 ```processes```.



### Overview

Ray 使用 ```Pickle protocol version 5``` 作为序列化协议。



### Plasma Object Store

**Plasma** 是一个基于 Apache Arrow 开发的内对象存储。所有的对象在 **Plasma object store** 中都是不可变的并且保存在共享内存中。

每个 node 都有自己的 object store. 当数据存入 object store, 它不会自动的广播通知其他的 node. 数据一直保留在本地直到在别的 node 被别的 task 或者 actor 请求。



### Numpy Arrays

* Numpy array中存储的都是只读的对象，同一节点上的所有Ray Worker都可以读取对象存储中的numpy array而无需复制（零复制读取）。
* 工作进程中的每个numpy array对象都持有一个指向共享内存中相关数组的指针。
* 对只读对象的任何写入都需要用户首先将其复制到本地进程内存中。



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



### Object 生命周期

{% asset_img Object生命周期.png Object生命周期 %}



Object的所有者是初始化`ObjectRef`，**并且**提交了task或者调用了`ray up`的worker。

* Ray 如果确认所有者还存在，Object最后会解析为它的值
* 如果所有者不存在，试图获取Object值得操作不会悬停，而是抛出异常，即使存在Object物理拷贝



每个worker储存它所属的Object的引用次数。引用仅仅在以下情况被统计：

* 传递一个`ObjectRef`或者包含`ObjectRef`这个参数的task
* Task返回一个`ObjectRef`或者包含`ObjectRef`的返回



### 对象管理

{% asset_img 对象管理.png 对象管理 %}

通常，小的对象存储在它自己的进程里，而大的对象储存在分布式对象存储里。

在进程中的对象能够通过直接内存拷贝的方式快速解析，但是这可能会带来更多的内存占用。如果对象被很多进程引用，就会造成额外的拷贝操作。

而解析一个分布式内存的对象至少需要一个worker之间的IPC（进程间通信）。如果worker本地共享不存在这个对象的拷贝，还会产生一个RPC通信。因为共享内存存储是通过“共享内存机制”实现的，一个节点上的多个worker可能引用同一块对象的拷贝。采用“零拷贝反序列化”的方式就可以降低总内存占用。这种用法意味着一个进程可以应用一个超过单台机器内存容量的对象，因为对象的多个拷贝可以储存在不同的节点上。



### 对象解析

object的值能够通过一个叫`ObjectRef`来解析。`ObjectRef`包含了两个字段：

* 一个唯一的20-byte的标识符。这是产生这个object的task id和object id的组合
* object所有者进程的地址。包含worker进程的唯一ID，IP地址和端口，和本地raylet的ID



{% asset_img 对象解析.png 对象解析 %}

1. 在GCS查找Object的地址
2. 选择一个地址并且发出了一个获取Object的请求
3. 获得了这个Object



### 使用`ray memory`命令行

当一个Ray应用在运行的时候使用`ray memory`会返回所有当前被集群中driver, actor和task持有的的`ObjectRef`的索引。



此输出中的每个条目都对应于一个ObjectRef，该ObjectRef当前将对象固定在对象存储中，以及引用的位置（在driver、worker等中）、引用的类型（有关引用类型的详细信息）、对象的大小（以字节为单位），实例化对象的进程ID和IP地址，以及在应用程序中创建引用的位置。



#### Local ObjectRef Reference

```python
@ray.remote
def f(arg):
    return arg

a = ray.put(None)
b = f.remote(None)
```

{% asset_img LocalObjectRefReferences.png LocalObjectRefReferences %}



有两个`ObjectRef`，都是`LOCAL_REFERENCE`。但是其中一个的Reference是`put object`（对应a），另一个是`task call`（对应b）



#### Objects pinned in memory

```python
import numpy as np

a = ray.put(np.zeros(1))
b = ray.get(a)
del a
```

{% asset_img ObjectsPinnedInMemory.png ObjectsPinnedInMemory %}



在这种情况下，对象仍然固定在对象存储中，因为反序列化副本（存储在b中）直接指向对象存储中的内存。



#### Pending task reference

```python
@ray.remote
def f(arg):
    while True:
        pass

a = ray.put(None)
b = f.remote(a)
```

{% asset_img PendingTaskReference.png PendingTaskReference %}

我们发现同一个ObjectID在worker和driver中都出现了分别是`USED_BY_PENDING_TASK`和`PINNED_IN_MEMORY`。因为Python `arg`直接引用了plasma中的内存。



#### Serialized ObjectRef references

我们对a进行一个封装，这个例子中是作为一个list传入的

```python
@ray.remote
def f(arg):
    while True:
        pass

a = ray.put(None)
b = f.remote([a])
```

{% asset_img SerializedObjectRefReferences.png SerializedObjectRefReferences %}



这时候就变成了`LOCAL_REFERENCE`



#### Captured ObjectRef references

```python
a = ray.put(None)
b = ray.put([a])
del a
```

{% asset_img CapturedObjectRefReferences.png CapturedObjectRefReferences %}



在内存中a是`CAPTURED_IN_OBJECT`的类型



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

* Gang Scheduling: 应用程序要求所有任务/参与者都安排在同一时间开始。
* Maximizing data locality: 您希望将task和actor安排在数据附近，以避免对象传输开销。
* Load balancing: 为了提高应用程序可用性并避免资源过载，您希望尽可能将参与者或任务放置到不同的物理机器中。

#### 关键概念

* **bundle** : 资源的集合
  * 一个 bundle 必须适合一个在集群中的 node
  * 然后 Bundles 会根据 ```placement group strategy``` 在集群中跨 nodes 放置
* **placement group** : bundle 的集合
  * 每一个 bundle 都被给予了一个在 placement group 的编号
  * 然后 Bundles 会根据 ```placement group strategy``` 在集群中跨 nodes 放置
  * 等 placement group 创建了后，```tasks``` 和 ```actors``` 可以根据 placement group 或者个人 bundles 来调度。

#### 策略类型

* STRICT_PACK
  * 所有bundles都必须放在集群上的单个节点中。
* PACK
  * 所有提供的bundle都尽量打包到单个节点上。如果严格打包不可行（即某些bundle不适合节点），则可以将bundle放在其他节点上。
* STRICT_SPREAD
  * 每个bundle必须安排在单独的节点中。
* SPREAD
  * 每个bundle将以最大努力的方式分布到单独的节点上。如果严格展开不可行，则可以将bundle放置在重叠的节点上。



#### 容错

如果包含placement group的某些bundle的节点死亡，GCS将在不同的节点上重新调度所有束。这意味着placement group的初始创建是“原子的”，但一旦创建，就可能存在部分placement group。

Placement group可以容忍工作节点故障（死区节点上的包被重新调度）。但是，放置组目前无法容忍头部节点故障（GCS故障），这是Ray的单点故障。




## Advanced Usage

#### Synchronization

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

#### Dynamic Custom Resources

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

# 