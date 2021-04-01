---
title: k8s operator
date: 2021-04-01 11:34:34
tags:
categories: kubernetes
---

# Operator概述

## 基本概念

首先介绍一下本文涉及的基本概念

* **CRD(Custom Resource Definition)**： 允许用户自定义Kubernetes资源，是一个类型。
* **CR(Custom Resource)**： CRD的一个具体实例

* **webhook**： 本质上是一种HTTP回调，会注册到apiserver上。在apiserver特定事件发生时，会查询已注册的webhook，并把相应的消息发送出去。按照处理类型的不同，可以分成两类：
  * mutating webhook: 可能会修改传入对象
  * validating webhook: 会只读传入对象
* **工作队列**：Controller的核心组件。它会监控集群内的资源变化，并把相关的对象，包括它的动作与key
* **Controller**：它会循环地处理上述工作队列，按照各自的逻辑把集群状态向预期状态推动。不同的controller处理的类型不同。
* **Operator**：Operator是描述、部署和管理Kubernetes应用的一套机制。可以理解为CRD配合可选的webhook与controller来实现用户业务逻辑，即 operator = CRD + webhook + controller



## 常见的operator工作模式



{% asset_img operator工作模式.png operator工作模式 %}



1. 用户创建一个自定义资源（CRD）
2. apiserver 根据自己注册的一个 pass 列表，把该 CRD 的请求转发给 webhook
3. webhook 一般会完成该 CRD 的缺省值设定和参数检验。webhook 处理完之后，相应的 CR 会被写入数据库，返回给用户
4. 与此同时，controller 会在后台监测该自定义资源，按照业务逻辑，处理与该自定义资源相关联的特殊操作
5. 上述处理一般会引起集群内的状态变化，controller 会监测这些关联的变化，把这些变化记录到 CRD 的状态中



# 参考资料

* [从零开始入门 K8s | Kubernetes API 编程利器：Operator 和 Operator Framework](https://www.kubernetes.org.cn/6746.html)

