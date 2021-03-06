---
title: 'MapReduce: 在大型集群上简化数据处理'
date: 2020-10-26 10:05:39
tags:
---

# 概要
> MapReduce 是一种编程模型，用于处理和生成大型数据集的实现。

用户通过指定一个用来处理键值对 (key/value) 的 map 函数来生成一个中间键值对集合。然后，再指定一个 reduce 函数，它用来合并所有的具有相同中间 key 的中间 value.

# 简介
虽然数据处理的逻辑简单，但是由于数据量巨大又需要在有限的时间内完成。计算任务也不得不分配给成百上千台机器去执行。如何并行化计算，分配数据以及处理故障的问题，所有的问题都纠缠在一起，这就需要大量的代码来对它们进行处理。

为了应对这种复杂性，我们设计了一种抽象。大多数计算计算都涉及到对输入中的每个逻辑记录进行 map 的操作，以便于计算出一个中间键值对的集合。然后，为了恰当的整合衍生数据，我们对共用相同键的所有值进行 reduce 操作。通过使用具备用户所指定的 map 和 reduce 操作的函数式模型，这使得我们能够轻松并行化大型计算。

# 编程模型
该计算任务将**一个键值对集合**作为输入，并生成一个键值对集合作为输出。**MapReduce** 这个库的用户将这种计算任务以两个函数进行表达，即 **Map** 和 **Reduce** 。

由用户所编写的 **Map** 函数接收输入，并生成一个中间键值对集合。**MapReduce** 这个库会将所有共用一个键的值组合在一起，并将它们传递给 **Reduce** 函数。

**Reduce** 函数也是由用户编写。它接受一个中间键以及该键的值的集合作为输入。它会将这些值合并在一起，以此来生成一组更小的值的集合。

> key / value 集合 ---**Map**---> 中间 key / value 集合 ---**MapReduce**---> key / value1 / value2 /... ---**Reduce**---> key / value.zip

通常每次调用 **Reduce** 函数所产生的值的结果只有0个或者1个。中间值通过一个迭代器来传递给用户所编写的 **Reduce** 函数。这使得我们可以处理这些因为数据量太大而无法存放在内存中的存储值的list列表了。

## 案例

### 例1

背景：我们要从大量文档中计算出每个单词的出现次数。
```Java
/**
 * key: document name
 * value: document contents
 * 返回一个单词加上它出现的次数的键值对
**/
public void map(string key, string value) {
    for (word w: value) {
        EmitIntermediate(w, "1");
    }
}

/**
 * key: a word
 * values: a list of counts
 * 将该单词的出现次数统计出来
**/
public void reduce(string key, Iterator values) {
    int result = 0;
    for (v: values) {
        result += ParseInt(v);
        Emit(AsString(result));
    }
}
```
### 分布式过滤器
**Map** 函数会发出（emit）匹配某个规则的一行。**Reduce** 函数是一个恒等函数，即把中间数据复制到输出。

### 计算URL的访问频率
**Map** 函数用来处理网页请求的日志，并输出(URL,1)。**Reduce** 函数则用于将相同URL的值全部加起来，并输出(URL, 访问总次数)这样的键值对结果。

### 倒转网络链接图
**Map**函数会在源页面中找到所有的目标URL，并输出<target, source>这样的键值对。**Reduce**函数会将给定的目标URL的所有链接组合成一个列表，输出<target, list(source)>这样的键值对。

### 每台主机上的检索词频率
term（这里是指搜索系统里的某一项东西，这里指检索词）vector（这里指数组）将一个文档或者是一组文档中出现的最重要的单词概括为<单词，频率> 这样的键值对列表，对于每个输入文档，**Map** 函数会输出这样一对键值对<hostname, term vector>（其中hostname是从文档中的URL里提取出来的）。**Reduce** 函数接收给定主机的所有每一个文档的term vector。它会将这些term vector加在一起，并去除频率较低的term，然后输出一个最终键值对<hostname, term vector>。

### 倒排索引
**Map** 函数会对每个文档进行解析，并输出<word, 文档ID>这样的键值对序列。**Reduce** 函数所接受的输入是一个给定词的所有键值对，接着它会对所有文档ID进行排序，然后输出<word, list(文档ID)>。所有输出键值对的集合可以形成一个简单的倒排索引。我们能简单的计算出每个单词在文档中的位置。

### 分布式排序
**Map** 函数会从每条记录中提取出一个key，然后输出<key, record>这样的键值对。**Reduce** 函数对这些键值对不做任何修改，直接输出。这种计算任务依赖分区机制以及排序属性。

# 实现
## 执行概述
将传入 **Map** 函数的输入数据自动切分为 **M** 个数据片段的集合，这样就能将 **Map** 操作分布到多台机器上运行。使用分区函数将 **Map** 函数所生成的中间 **key** 值分成 **R** 个不同的分区，这样就可以将 **Reduce** 操作也分布到多台机器上并行处理。

{% asset_img MapReduce框架.png MapReduce框架 %}

* 用户程序中的 ```MapReduce``` 库会先将输入文件切分为 ```M``` 个片段，通常每个片段的大小在16MB到64MB之间（具体大小可以由用户通过可选参数来进行指定）。接着，它会在集群中启动许多个程序副本。
* 有一个程序副本是比较特殊的，那就是 ```master``` 。剩下的副本都是 ```worker```，```master``` 会对这些 ```worker``` 进行任务分配。这里有 ```M``` 个 **Map** 任务以及 ```R``` 个 **Reduce** 任务要进行分配。```master``` 会给每个空闲的 ```worker``` 分配一个 **map** 任务或者一个 **reduce** 任务。
* 被分配了 **map** 任务的 ```worker``` 会读取相关的输入数据片段。它会从输入数据中解析出键值对，并将它们传入用户定义的 ```Map``` 函数中。Map函数所生成的中间键值对会被缓存在内存中。
* 每隔一段时间，被缓存的键值对会被写入到本地硬盘，并通过分区函数分到 ```R``` 个区域内。这些被缓存的键值对在本地磁盘的位置会被传回 ```master```。```master``` 负责将这些位置转发给执行 ```reduce``` 操作的 ```worker```。
* 当 ```master``` 将这些位置告诉了某个执行 ```reduce``` 的 ```worker``` ，该 ```worker``` 就会使用 ```RPC``` 的方式去从保存了这些缓存数据的 ```map worker``` 的本地磁盘中读取数据。当一个 ```reduce worker``` 读取完了所有的中间数据后，它就会根据中间键进行排序，这样使得具有相同键值的数据可以聚合在一起。之所以需要排序是因为通常许多不同的key会映射到同一个 ```reduce``` 任务中。如果中间数据的数量太过庞大而无法放在内存中，那就需要使用外部排序。
* ```reduce worker``` 会对排序后的中间数据进行遍历。然后，对于遇到的每个唯一的中间键，```reduce worker``` 会将该key和对应的中间value的集合传入用户所提供的 ```Reduce``` 函数中。```Reduce``` 函数生成的输出会被追加到这个 ```reduce``` 分区的输出文件中。
* 当所有的 ```map``` 任务和 ```reduce``` 任务完成后，```master``` 会唤醒用户程序。此时，用户程序会结束对 ```MapReduce``` 的调用。

在成功完成任务后，```MapReduce``` 的输出结果会存放在 ```R``` 个输出文件中（每个 ```reduce``` 任务都会生成对应的文件，文件名由用户指定）。一般情况下，用户无需将这些文件合并为一个文件。他们通常会将这些文件作为输入传入另一个 ```MapReduce``` 调用中。或者在另一个可以处理这些多个分割文件的分布式应用中使用。

## Master 的数据结构
* 保存了每个 ```Map``` 任务和每个 ```Reduce``` 任务的状态 (闲置，正在运行，以及完成)
* 非空闲任务 ```worker``` 机器的 ```ID```
* 保存由 ```map``` 任务所生成的中间文件区域的位置传播给 ```reduce``` 任务。对于每个完成的 ```map``` 任务，```master``` 会保存由 ```map``` 任务所生成的 ```R``` 个中间文件区域的位置和大小。当 ```map``` 任务完成后，会对该位置和数据大小信息进行更新。这些信息会被逐渐递增地推送给那些正在运行的```Reduce```工作。

## 容错
因为 ```MapReduce``` 库的设计旨在使用成百上千台机器来处理海量的数据，所以该库必须能很好地处理机器故障。

### Worker 故障
```master``` 会周期 ```ping``` 下每个 ```worker```. 如果在一定时间内无法收到来自某个 ```worker``` 的响应，那么 ```master``` 就会将该 ```worker``` 标记为 ```failed```. 所有由该 ```worker``` 完成的 ```Map``` 任务都会被重设为初始的空闲 ```idle``` 状态。

### Master 故障

## 地区性

## 任务粒度

## 备用任务


# 参考文献
https://mp.weixin.qq.com/s/sChCf07SxhTudxFIKd8pgA

https://mp.weixin.qq.com/s/h43tPiycGrKf9089pML2tw
