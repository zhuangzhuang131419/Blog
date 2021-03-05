---
title: Google三驾马车（2）——BigTable
date: 2021-03-05 13:57:06
tags: [Google, 论文]
categories: 分布式系统
---

这一篇是Google的另一个经典论文“[Big Table](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/bigtable-osdi06.pdf)”。这跟上一篇文章同属一个系列的YouTube视频——[深入浅出Big Table](https://www.youtube.com/watch?v=r1bh90_8dsg)



# BigTable

## 如何在文件内快速查询

我们在GFS中已经讲到了文件的读写，那么我们如何在文件内快速的查询呢。



{% asset_img 在文件内快速查找.png 在文件内快速查找 %}

把key排好序后，就可以用二分查找再加上范围查找。



## 如何保存一个很大的表

我们的基本思路是将大表拆成小表（tablet），在元数据（metadata）中保存每一个小表的位置。

{% asset_img 保存很大的表.png 保存很大的表 %}



## 如果保存超大表

如果还是不够，就可以再把小表（tablet）拆成小小表（SSTable）

{% asset_img 保存超大表.png 保存超大表 %}



## 如何向表中写数据

{% asset_img 如何写数据.png 如何写数据 %}



我们在整个架构中，有内存（Memory）和硬盘（GFS/Disk）两部分。我们可以在内存中存放小小表（SSTable）的位置，所以内存就会有小小表（SSTable）的索引。为了加快写速度，我们并不是马上去往硬盘上去写的，我们在硬盘中修改排序是很难的，但是我们在内存中修改排序是简单的，因为在内存中是随机存储。



### 内存表过大时怎么办

{% asset_img 内存表过大.png 内存表过大 %}



当内存表达到64M的时候，就会把自己存到硬盘上，称为一个新的SSTable，并持久性保存。



### 如何避免丢失内存数据

{% asset_img 避免内存数据丢失.png 避免内存数据丢失 %}



我们通过添加log来避免丢失内存数据。



## 如何读数据

{% asset_img 如何读数据.png 如何读数据 %}



我们存储的小小表（SSTable）内部是有序的，各个SSTable之间时无序的。所以如果想要找一个key，需要到所有的SSTable以及内存表中去查找。由于需要到硬盘去查询，查询的效率就会很低。



### 如何加速读数据

{% asset_img 加速读取数据.png 加速读取数据 %}



通过添加索引的方式来进行加速，预先把索引表加到内存中来。



### 如何继续加速读数据

{% asset_img 继续加速读数据.png 继续加速读数据 %}



我们引入布隆过滤器，可以加速判断一个key是否能够命中（**只能起辅助作用**）



## 如何将表存入GFS

{% asset_img 将表存入GFS.png 将表存入GFS %}



BigTable用的是内存的结构，GFS是硬盘的结构，在GFS中给SSTable和log各备份三份。



## 表的逻辑视图是什么

{% asset_img 表的逻辑视图.png 表的逻辑视图 %}



Photo列下面可以记录Steve不同时间戳下的照片，相当于搜索引擎抓到不同页面的不同版本。Weight和Height可以作为一个Column Family。



### 将逻辑视图转换为物理存储

{% asset_img 将逻辑视图转换为物理存储.png 将逻辑视图转换为物理存储 %}



可以将相同的key聚合在一起，加快查询速度。NoSQL天然不支持join操作的。



## BigTable的架构是什么

{% asset_img BigTable的架构.png BigTable的架构 %}



首先需要一个client进行连接，为了连接需要一个library提供库函数以及权限的控制。连接之前需要获得锁服务（Chubby），以及各种表的元数据的获得。获得请求能力之后，就会到特定的Tablet Server去获得Tablet的数据。Tablet的底层有一个master在处理数据的。上面有一个master负责处理各个原始信息以及负载均衡，可以指定Tablet之间的存放。还有一些Cluster Schedule来监控整个服务或者系统的过程。

