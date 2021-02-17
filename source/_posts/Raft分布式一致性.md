---
title: Raft分布式一致性
date: 2020-10-25 18:59:00
tags: 分布式系统
---

这篇文章就是简单描述一下Raft的原理



 # 背景介绍

首先假设我们有一个单节点的系统，我们可以认为这个节点是一个存储了一个值的数据库服务器

{% asset_img raft1.png raft1 %}

我们同时还有一个客户端可以发送值给这个节点

{% asset_img raft2.png raft2 %}

在单节点的情况下，非常容易达成一致，也就是共识（consensus）

{% asset_img raft3.gif raft3 %}

但是我们如何在多节点的情况下达成共识？这就是**distributed consensus**问题。**Raft**就是一个用来实现**distributed consensus**的协议。

{% asset_img raft4.png raft4 %}



# Raft 原理

一个节点有三种状态：*Follower*, *Leader*, *Candidate*。

## 整体流程 

### Leader Election

1. 刚开始所有的节点都处于*Follower*状态。

   {% asset_img raft5.png raft5 %}

2. 当这些*followers*没有感知到*leader*，它们就会变成*candidate*。

   {% asset_img raft6.png raft6 %}

3. 然后，*candidate*向其他节点拉票

   {% asset_img raft7.gif raft7 %}

4. 其他节点发起投票回复。

   {% asset_img raft8.gif raft8 %}

5. *candidate*如果收到了半数节点以上的票，就会成为*leader*。

   {% asset_img raft9.gif raft9 %}

   

### Log Replication

1. 接下来所有对系统的更改都要经过*leader*。每一个改动都被作为一条entry添加到节点的日志(log)中，现在这条log entry还是未提交的，所以不会更新节点里的值。

   {% asset_img raft10.gif raft10 %}

2. 为了提交这个entry，*leader*把自己的log entry复制了一份给其他的*follower*。

   {% asset_img raft11.gif raft11 %}

3. 然后*leader*等待直到超过半数的节点已经写入了entry。(每个*follower*写入entry就会通知*leader*)

   {% asset_img raft12.gif raft12 %}

4. 现在*leader*的log entry可以提交了，并把节点的值修改为“5”

   {% asset_img raft13.png raft13 %}

5. *leader*然后通知所有的*followers* log entry已经提交了。

   {% asset_img raft14.gif raft14 %}

6. 现在这个集群对系统的状态达成了共识。

   {% asset_img raft15.png raft15 %}

   

## 具体细节

### Leader Election

Raft中有控制选举的两个超时机制: election timeout

#### Election Timeout

> 指一个*follower*等待直到成为*candidate*的时间。

一般是150mm~300mm之间的一个随机数。



选举超时后，一个*follower*成为*candidate*开始一个新的选举周期并投票给自己

{% asset_img raft16.gif raft16 %}

同时向其他的节点发起拉票请求

{% asset_img raft17.png raft17 %}

如果收到拉票请求的节点在这个周期还没有投过票，那么他就会把票投给当前的*candidate*并重置自己的election timeout

{% asset_img raft18.gif raft18 %}

一旦一个*candidate*获得超过了半数的选票就会成为*leader*。*leader*开始发送*Append Entries*消息给*followers*。这些消息以*heartbreak timeout*为间隔发送，*followers* 然后回复这些消息。这个选举周期一直到一个*follower*停止接收心跳并且成为一个*candidate*。

{% asset_img raft19.gif raft19 %}

#### Leader Stop

当一个*leader*停止后，会重新选举出*leader*。节点B现在是第二周期的*leader*。

{% asset_img raft20.gif raft20 %}

#### Split Vote

如果两个节点同时成为*candidate*，那么会发生split vote。两个节点在同一个周期同时发起选举。

{% asset_img raft21.gif raft21 %}

并且每个*candidate*都先于另一个接触到其中一个*follower*

{% asset_img raft22.gif raft22 %}

现在每个*candidate*有两票并且在这个周期收不到更多的票了

{% asset_img raft23.gif raft23 %}

那么节点就等待直到一个新的选举。节点C在第五周期收获了绝大多数的投票成为了*leader*。

{% asset_img raft24.gif raft24 %}

### Log Replication

一旦我们确定了*leader*，下一步就是给所有的结点复制所有的改动。运用用于心跳的相同的Append Entry消息来完成。

{% asset_img raft25.gif raft25 %}

1. 首先一个客户端向*leader*发送一个改动，这个改动被追加到*leader*的日志中。

   {% asset_img raft26.gif raft26 %}

2. 然后这次改动会在下一次心跳的时候通知所有的*followers*

   {% asset_img raft27.gif raft27 %}

3. 一旦超过半数的*follower*确认了，那么这次改动就被提交上去了并且给客户端发送一个回应。

   {% asset_img raft28.gif raft28 %}

   

例：客户端现在发送一个"ADD 2"请求

{% asset_img raft29.gif raft29 %}



#### Partition

Raft可以同样在网络分区中保持特性。假设现在把A, B和C, D, E区分开来。由于网络分区，我们现在有在不同周期有两个*leader*。

{% asset_img raft30.gif raft30 %}

现在我们额外增加一个客户端并尝试同时去分别更新两个*leader*。节点B不能收到超过半数的回复，所以无法提交log entry。

{% asset_img raft31.gif raft31 %}

另一个客户端尝试去把节点C更新为8。因为可以复制给超过半数的节点，所以成功了。

{% asset_img raft32.gif raft32 %}

现在我们修复网络分区。节点B看到了更高的周期所以进行下一步，

{% asset_img raft33.gif raft33 %}

节点A, B会回滚他们未提交的log entry并且匹配新*leader*的日志。

{% asset_img raft34.gif raft34 %}



# 参考文献

[图解Raft流程](http://thesecretlivesofdata.com/raft/)

