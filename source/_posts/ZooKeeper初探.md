---
title: ZooKeeper初探
date: 2020-12-30 11:24:34
tags: 分布式系统
---

# ZooKeeper 是什么
> ZooKeeper 是一个分布式协调服务 (service for coordinating processes of distributed applications)

* 协调: 在一个并发的环境里，我们为了避免多个运行单元对共享数据同时进行修改，造成数据损坏的情况出现，我们就必须依赖像锁这样的协调机制.
* 我们在进程内还有各种各样的协调机制(一般我们称之为同步机制)。现在我们大概了解了什么是协调了，但是上面介绍的协调都是在进程内进行协调。在进程内进行协调我们可以使用语言，平台，操作系统等为我们提供的机制。
* 有两台机器A、B，A 对一个数据进行了一个操作，B是如何同步得到这个结果的，在分布式环境中，就需要一个分布式协调服务。
* ZooKeeper = 文件系统 + 通知机制

# ZooKeeper 可以干什么

```ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.```

这句话描述了 ZooKeeper 可以进行: ```配置管理```, ```命名服务```, ```分布式同步```, ```集群管理```.

## 配置管理
如果我们的配置很多，分布式系统中的服务器都需要这个配置。这时候，往往需要寻找一种集中管理配置的办法——我们在集群中的地方修改了配置，所有对这个配置感兴趣的都可以获得变更。

把公用的配置文件提取出来放在一个目录节点进行监听，一旦配置信息发生变化，每个应用程序就会受到 ZooKeeper 的通知，然后从 ZooKeeper 获取新的配置应用信息。由于需要很高的可靠性，一般我们用一个集群来提供配置服务，但是用集群提升可靠性，如何保证及群众的一致性呢？

需要使用一种已经实现了一致性协议的服务。ZooKeeper 就是这种服务，它使用了 Zab 这种一致性协议来提供一致性。
## 命名服务
由于 IP 地址对人非常不友好，我们需要使用域名来访问。但是因为计算机不能识别域名，所以每台机器都有一份域名到 IP 地址的映射。如果域名对应的 IP 地址发生了变化怎么处理呢？

于是我们使用 DNS (Domain Name System) 。它作为将域名和IP地址相互映射的一个分布式数据库，能够使人更方便地访问互联网。

Zookeeper的命名功能就是这么一个服务器。在集群中，相同的一个服务有很多个提供者，这些提供者启动时，提供者的相关信息（服务接口，地址，端口等）注册到 ZooKeeper 中，当消费者要消费某服务的时候，从 ZooKeeper 中获取该服务的所有提供者的信息目录，再根据 Dubbo 的负载均衡机制选择一个提供者。
## 分布式锁
可以利用 ZooKeeper 来协调多个分布式进程之间的活动。使用分布式锁，在某个时刻只让一个服务去干活，当这台服务出问题的时候锁释放，立即 fail over 到另外的服务。

这种机制也被称为 Leader Election
## 集群管理
集群中的机器要感知到其他节点的变化(有新的节点加入进来，或者有老的节点退出集群)

# ZooKeeper 的数据模型
很像数据结构当中的树，也很像文件系统的目录

{% asset_img ZooKeeper数据结构.png ZooKeeper数据结构 %}

这种节点叫做**Znode**，**Znode**的引用方式是**路径引用**：
* /动物/仓鼠
* /植物/荷花

这样的层级结构，让每一个**Znode**节点拥有唯一的路径


## Znode
Znode 包含了数据、子节点引用、访问权限
* data
  * Znode 存储的数据信息
* ACL
  * 记录 Znode 的访问权限，即哪些人哪些 IP 可以访问本节点
* stat
  * 包含 Znode 的各种元数据，比如事务ID，版本号，时间戳，大小等等
* child
  * 当前节点的子节点引用


一共有三种类型的Znode
* Regular
  * 这种 Znode 一旦创建，就永久存在，除非你删除了它
* Ephemeral
  * 如果 ZooKeeper 认为创建它的客户端挂了，他会删除这种类型的 Znode。
  * 这种类型的 Znode 与客户端会话绑在一起，所以客户端会定时发送心跳给 ZooKeeper。
* Sequential
  * 当你想要以特定的名字创建一个文件，ZooKeeper 实际上创建的文件名是你指定的文件名再加上一个数字。
  * 当有多个客户端同时创建 Sequential 文件时，ZooKeeper 会确保这里的数字是递增的。

ZooKeeper 是为读多写少的场景所设计，用于存储少量的状态和配置信息，每个节点的数据最大不能超过 1 MB

# ZooKeeper 基本操作和事件通知

* ```create(path, data, flag)``` 创建节点
  * flag 是表明 Znode 的类型的
  * 如果得到了 yes 的返回，那么说明这个文件之前是不存在的
  * 如果得到了 no 或者一个错误返回，那么说明这个文件之前就已经存在了
* ```delete(path, version)``` 删除节点
  * 当且仅当 Znode 的当前版本号与传入的 version 相同，才执行操作。
* ```exist(path, watch)``` 判断一个节点是否存在
  * watch 可以监听对应文件的变化
  * 判断文件是否存在和 watch 文件的变化在 ZooKeeper 中属于原子操作。
* ```getData(path, watch)``` 获取一个节点的数据
* ```setData(path, data, watch)``` 设置一个节点的数据
  * 当且仅当文件的版本号与传入的 version 一致时，才会更新文件
* ```getChildren(watch)``` 获取节点下的所有子节点

**watch** 是指注册在特定 Znode 上的触发器。当这个 Znode 发生改变，也就是调用了 ```getData```, ```setData```, ```getChildren``` 的时候，将会触发 Znode 上注册的对应事件，请求 **watch** 的客户端会接受到异步通知。

1. 客户端调用 ```getData``` 方法，watch 参数是 true。服务端接到请求，返回节点数据，并且在对应的哈希表里插入被 Watch 的 Znode 路径，以及 Watcher 列表。

{% asset_img watch的具体交互-1.png watch的具体交互-1 %}

2. 当被 Watch 的 Znode 已删除，服务端会查找哈希表，找到该 Znode 对应的所有 Watcher，异步通知客户端，并且删除哈希表中对应的 Key-Value。

{% asset_img watch的具体交互-2.png watch的具体交互-2 %}

# ZooKeeper 的一致性
ZooKeeper 为了防止单机挂掉的情况，维护了一个集群。

{% asset_img ZooKeeper集群.png ZooKeeper集群 %}

* ZooKeeper Service 集群是一主多从结构
* 在更新数据时，首先更新到主节点(服务器, 不是 Znode)，再同步到从节点。
* 在读取数据时，直接读取任意从节点。

## ZAB 协议
ZAB (ZooKeeper Atomic Broadcast) 可以解决 ZooKeeper 集群崩溃恢复以及主从同步数据的问题。

* Looking: 选举状态
* Following: 从节点所处的状态
* Leading: 主节点所处的状态


## 最大ZXID
最大 ZXID 也就是节点本地的最新事务编号，包含 epoch 和计数两部分。

* epoch 相当于 Raft 算法中的 term

### 恢复模式
加入 ZooKeeper 当前的主节点挂掉了，集群会进行崩溃恢复
1. Leader Election
* 选举阶段，此时集群中的节点处于Looking状态。它们会各自向其他节点发起投票，投票当中包含自己的服务器ID和最新事务ID（ZXID）。
* 接下来，节点会用自身的ZXID和从其他节点接收到的ZXID做比较，如果发现别人家的ZXID比自己大，也就是数据比自己新，那么就重新发起投票，投票给目前已知最大的ZXID所属节点。
* 每次投票后，服务器都会统计投票数量，判断是否有某个节点得到半数以上的投票。如果存在这样的节点，该节点将会成为准Leader，状态变为Leading。其他节点的状态变为Following。
2. Discovery
* 发现阶段，用于在从节点中发现最新的ZXID和事务日志。(为什么还要寻找ZXID最大的呢)
* 这是为了防止某些意外情况，比如因网络原因在上一阶段产生多个Leader的情况。
* 所以这一阶段，Leader集思广益，接收所有Follower发来各自的最新epoch值。Leader从中选出最大的epoch，基于此值加1，生成新的epoch分发给各个Follower。
* 各个Follower收到全新的epoch后，返回ACK给Leader，带上各自最大的ZXID和历史事务日志。Leader选出最大的ZXID，并更新自身历史日志。
3. Synchronization
* 同步阶段，把Leader刚才收集得到的最新历史事务日志，同步给集群中所有的Follower。只有当半数Follower同步成功，这个准Leader才能成为正式的Leader。


自此，故障恢复正式完成。

## 广播模式
写入数据，涉及到 ZAB 协议的 Broadcast 阶段
1. 客户端发出写入数据请求给任意Follower。
2. Follower把写入数据请求转发给Leader。
3. Leader采用二阶段提交方式，先发送Propose广播给Follower。
4. Follower接到Propose消息，写入日志成功后，返回ACK消息给Leader。
5. Leader接到半数以上ACK消息，返回成功给客户端，并且广播Commit请求给Follower。

> Zab协议既不是强一致性，也不是弱一致性，而是处于两者之间的单调一致性。它依靠事务ID和版本号，保证了数据的更新和读取是有序的。

# ZooKeeper 的应用
## 分布式锁
这是雅虎研究员设计Zookeeper的初衷。利用Zookeeper的临时顺序节点，可以轻松实现分布式锁。
## 服务注册和发现
利用Znode和Watcher，可以实现分布式服务的注册和发现。最著名的应用就是阿里的分布式RPC框架Dubbo。
## 共享配置和状态信息
Redis的分布式解决方案Codis，就利用了Zookeeper来存放数据路由表和 codis-proxy 节点的元信息。同时 codis-config 发起的命令都会通过 ZooKeeper 同步到各个存活的 codis-proxy。

# 参考文献
* [ZooKeeper简介（浅入）](https://juejin.cn/post/6844903684610981895)
* [漫画：什么是ZooKeeper](http://www.360doc.com/content/18/0521/09/36490684_755631753.shtml)