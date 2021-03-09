---
title: Google三驾马车（1）——GFS
date: 2021-03-04 20:12:49
tags: [Google, 论文]
categories: 分布式系统
---

最近阅读了号称Google分布式系统的三驾马车中的"[The Google File System](https://dl.acm.org/doi/pdf/10.1145/945445.945450?casa_token=oR7grA26w34AAAAA:foohVEqDKPuAtLOsysk87pEoO7zTgcYRPFKtOxG7QLzOMI8HnT1vybCb6kibsHO5RWxPm8C4RSRKiTw)"。由于水平尚浅，即便读的是[中译](https://duanmeng.github.io/2017/12/07/gfs-notes/)仍觉得倍感吃力，偶然间发现这个YouTube上的讲解视频——[深入浅出Google File System](https://www.youtube.com/watch?v=WLad7CCexo8)，可谓豁然开朗，打算回头再重读原文，收获颇丰。



# 架构的层次

{% asset_img 架构的层次.png 架构的层次 %}



最底层是一个文件系统（GFS），往上需要你把数据模型抽象出来以便使用（BigTable），再往上就是算法（MapReduce...）。算法除了能和数据模型直接挂钩以外，它也能直接访问文件系统。再往上就是各类应用。



# Google File System 

## 如何保存一个文件

{% asset_img 保存文件.png 保存文件 %}



我们有一个硬盘（disk），这个硬盘最初会有一些原始信息（metadata）。为了寻找到数据在硬盘当中的位置，就用到了索引（index）。



## 如何保存大文件

{% asset_img 保存大文件.png 保存大文件 %}



对于大文件来说，我们就不能保存小块了，因为块数太多了。大块（chunk）由65536个小块（block）构成。但是如果我们用这个方式存储小文件，就会造成空间上的浪费。



## 如何保存超大文件

{% asset_img 保存超大文件.png 保存超大文件 %}



超大文件我们就无法在一个机器上存了，因为一个机器存不下，那么我们就需要一个主从结构。master就是一个存储所有原始信息的地方，索引可以照样建立，只不过现在指向的是chunkserver中的数据。这样的方法也有问题，chunkserver数据任何变动都要通知master



### 如何减少Master存储的数据和流量

{% asset_img 减少Master的数据和流量.png 减少Master的数据和流量 %}



我们可以参考解耦的思想，master不直接对chunk负责，而是转而对chunkserver负责，再由chunkserver去负责chunk。这样就可以减少master的元数据信息，减少master和chunkserver之间的通信。



## 如何发现数据损坏

{% asset_img 发现数据损坏.png 发现数据损坏 %}



为了能够发现数据损坏，我们需要一个机制来建立数据损坏。每个小块（block）保存一个ckecksum，在读取的时候比较一下。这样也不会给整体带来很大的冗余。



## 如何减少ChunkServer挂掉带来的损失

{% asset_img 减少chunkserver挂掉带来的损失.png 减少chunkserver挂掉带来的损失 %}



我们通过把一个chunk存在多个chunkserver上的方式来增加容错。

跨机架胯中心2+1指的是把chunkserver分在三个地方，其中两个放在California的数据中心，另一个放在Washington的数据中心。在同一个数据中心中呢，又可以把两个chunkserver放在不同的机架上。



## 如何恢复损坏的Chunk

{% asset_img 恢复损坏的chunk.png 恢复损坏的chunk %}



利用存储在别的chunkserver上的副本对已损坏的chunk进行修复。



## 如何发现ChunkServer挂掉了

{% asset_img 发现ChunkServer挂掉.png 发现ChunkServer挂掉 %}



chunkserver通过发送心跳包在Master上留下时间戳来告诉Master自己还活着



### ChunkServer挂掉后恢复数据

{% asset_img ChunkServer挂掉后恢复数据.png ChunkServer挂掉后恢复数据 %}



我在索引里就会发现chunkserver4下线了，我就会启动一个修复进程。修复进程中的working list是基于存活副本数的恢复策略。副本越少说明越危险，应当尽早的修复。



## 如何应对热点

{% asset_img 应对热点.png 应对热点 %}



我们引入一个热点平衡进程，记录chunk和chunkserver访问的频次。基于ChunkServer的硬盘和带宽利用率，当副本过度繁忙，我们就把它复制到更多的chunkserver。



## 如何读文件

{% asset_img 读文件过程.png 读文件过程 %}



chunkserver的底层还是利用Linux 的文件系统来实现。



## 如何写文件

{% asset_img 写文件过程.png 写文件过程 %}



primary就是主副本，这是临时被选定的，本身的各个副本之间是没有优先级的。但我的client首先连接的可以不是primary，而是去找一个离它最近的server。因为在服务器和服务器之间有很高的带宽，所以服务器之间可以互相传。

为了更有效的使用网络我们将数据流和控制流分离。控制流从client到达主副本，然后到达其他的所有次副本，而数据则是线性地通过一个仔细选择的chunkserver链像流水线那样推送过去的。我们的目标是充分利用每个机器的网络带宽，避免网络瓶颈和高延时链路，最小化数据推送的延时。

 为了充分利用每个机器的网络带宽，数据通过chunkserver链线性的推送过去而不是以其他的拓扑进行分布比如树型。因此每个机器的带宽可以全部用来发送数据而不是为多个接受者进行切分。



不用设计处理错误的机制，错误机制会引发更多的错误。直接让client重试就好了。