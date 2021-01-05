---
title: Git的原理
date: 2020-08-22 19:33:22
tags:
---

这篇文章主要是讲一讲git的底层实现原理，之前有了解过一点，现在作一个完整的梳理。
# Git 的信息是怎样被储存的
首先我们初始化git, 可以发现新建了一个.git/目录，Git会将整个数据库储存在.git/目录下
```bash
$ git init
$ tree .git/objects
.git/objects
├── info
└── pack
```
然后我们先创建两个文件
```bash
$ tree .git/objects
.git/objects
├── 58
│   └── c9bdf9d017fcd178dc8c073cbfcbb7ff240d6c
├── c2
│   └── 00906efd24ec5e783bee7f23b5d7c941b0c12c
├── info
└── pack
$ echo '111' > a.txt
$ echo '222' > b.txt
$ git add *.txt
```
可以看一个这个objects里面具体是什么
```bash
$ cat .git/objects/58/c9bdf9d017fcd178dc8c073cbfcbb7ff240d6c
xKOR0a044K%
```

这个乱码的出现其实是git把信息压缩成了二进制文件。但是不用担心，因为Git也提供了一个能够帮助你探索它的api ```git cat-file [-t] [-p]```， -t可以查看object的类型，-p可以查看object储存的具体内容。
```bash
$ git cat-file -t 58c9
blob

$ git cat-file -p 58c9
111
```

可以发现这个object是一个blob类型的节点，他的内容是111，也就是说这个object储存着a.txt文件的内容。这就引出了我们所说的第一个在git中数据存储的类型**blob**

## Blob
它只储存的是一个文件的内容，不包括文件名等其他信息。然后将这些信息经过SHA1哈希算法得到对应的哈希值，这个哈希值就是这个文件在这个Git仓库中的唯一标识。即目前我们的Git 仓库是这样滴

{% asset_img Git原理1.png 示意图 %}

我们继续探索, 我们新建了一个commit
```bash
$ git commit -m "[+] init"
$ tree .git/objects
.git/objects
├── 4c
│   └── aaa1a9ae0b274fba9e3675f9ef071616e5b209
├── 58
│   └── c9bdf9d017fcd178dc8c073cbfcbb7ff240d6c
├── ae
│   └── 202f2a6e3588115a05581d4dc12e082e3e97e4
├── c2
│   └── 00906efd24ec5e783bee7f23b5d7c941b0c12c
├── info
└── pack
```

我们可以看到当我们commit了之后，又多了两个objects。使用cat-file看看具体的内容
```bash
$ git cat-file -t 4caaa1
tree

$ git cat-file -p 4caaa1
100644 blob 58c9bdf9d017fcd178dc8c073cbfcbb7ff240d6c	a.txt
100644 blob c200906efd24ec5e783bee7f23b5d7c941b0c12c	b.txt
```

现在我们碰到了第二个数据类型**tree**

## Tree
它将当前的目录结构打了一个快照。从它储存的内容来看可以发现它储存了一个目录结构（类似于文件夹），以及每一个文件（或者子文件夹）的权限、类型、对应的身份证（SHA1值）、以及文件名。现在我们的git仓库是这样的

{% asset_img Git原理2.png 示意图 %}


```bash
$ git cat-file -t ae20
commit

$ git cat-file -p ae20
tree 4caaa1a9ae0b274fba9e3675f9ef071616e5b209
author zhengchicheng.bob <zhengchicheng.bob@bytedance.com> 1598099521 +0800
committer zhengchicheng.bob <zhengchicheng.bob@bytedance.com> 1598099521 +0800

[+] init
```

现在我们发现了第三种数据类型**commit**

## Commit
它储存的是一个提交的信息，包括对应目录结构的快照tree的哈希值，上一个提交的哈希值（这里由于是第一个提交，所以没有父节点。在一个merge提交中还会出现多个父节点），提交的作者以及提交的具体时间，最后是该提交的信息。现在我们的数据库是这样的

{% asset_img Git原理3.png 示意图 %}


那么目前为止，我们就已经知道了Git是如何去储存一个提交信息的了, 我们已经了解了Blob, Tree, Commit三个基本的数据类型。那么我们平时接触到的分支信息又存储在哪里呢

```bash
$ cat .git/HEAD
ref: refs/heads/master

$ cat .git/refs/heads/master
0c96bfc59d0f02317d002ebbf8318f46c7e47ab2
```
在Git仓库中, HEAD、分支、普通的tag可以简单的理解成一个指针，指向对应commit的SHA1值。

{% asset_img Git原理4.png 示意图 %}

> 本质上Git是一个key-value的数据库加上默克尔树形成的无环图(DAG)

tips: 默克尔树又叫哈希树，主要特点是:
1. 最下面的叶结点包含存储数据或其哈希值
2. 非叶子结点都是他的两个孩子节点内容的哈希值
作用：
1. 可以快速比较大量数据
2. 快速定位修改

# Git 的三个分区
继续上面的例子，当前我们的仓库状态是这样的

{% asset_img Git原理4.png 示意图 %}

## 工作目录
操作系统上的文件，所有代码开发编辑都在这上面完成。

## Index 索引区
可以理解成一个暂存区域，这里面的代码会在下一次commit被提交到Git仓库

## Git 仓库
由Git object记录着每一次提交的快照，以及链式结构记录的提交变更历史

图解Git
https://marklodato.github.io/visual-git-guide/index-zh-cn.html


# 参考文献
https://www.jiqizhixin.com/articles/2019-12-20