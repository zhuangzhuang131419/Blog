---
title: Google三驾马车（3）——MapReduce
date: 2021-03-05 15:45:50
tags: [Google, 论文]
categories: 分布式系统
---

这是我认为在三驾马车中相对比较简单的[MapReduce](https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf)，之前跟着MIT6.824中的lab1中模拟了一遍。顺便推荐一下这个神课，网上的资料也非常齐全，算是我的分布式系统入门课程。



# MapReduce

我们一般使用MapReduce作为一个算法的框架来处理海量的数据。假设我们有1TB的数据，我们可能会统计单词数，或者建立倒排索引。这时候就会用到这个MapReduce的框架。



## 什么是Map和Reduce



### 什么是Map

{% asset_img 什么是Map.png 什么是Map %}



### 什么是Reduce

{% asset_img 什么是Reduce.png 什么是Reduce %}



## 如何统计单词出现数

在Map和Reduce的过程，实际上是高度并行的。

{% asset_img 统计单词出现数.png 统计单词出现数 %}



## 如何构建倒排索引

{% asset_img 构建倒排索引.png 构建倒排索引 %}



## MapReduce的框架是什么

{% asset_img MapReduce的架构.png MapReduce的架构 %}

