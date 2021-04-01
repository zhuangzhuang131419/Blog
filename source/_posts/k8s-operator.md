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



# Operator 实战 Demo



## 开发环境

### 安装go

```bash
$ brew install go
$ go version
go version go1.16.2 darwin/amd64
```



### 安装kubebuilder





## 创建新项目

```bash
# 进入目录
$ cd $GOPATH/src/github.com
$ mkdir -p operator-demo & cd operator-demo
$ pwd
/Users/zhengchicheng/go/src/github.com/operator-demo
# 初始化项目
$ kubebuilder init --domain my.domain
```

如果报错`outside GOPATH, module path must be specified`。这是因为`go mod init`初始化项目时，需要定义一个module，所以我们先运行`go mod init ProjectName`之后再运行`kubebuilder`命令即可。

```bash
$ tree -L 3
.
├── Dockerfile
├── Makefile
├── PROJECT
├── bin
│   └── manager
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   └── role_binding.yaml
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go

9 directories, 30 files

$ go get github.com/onsi/ginkgo@v1.11.0
$ go get github.com/onsi/gomega@v1.8.1
$ go get github.com/go-logr/logr@v0.1.0

# 在项目根目录下执行下面的命令创建 API。
$ kubebuilder create api --group groupa --version v1 --kind ApiExampleA
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing scaffold for you to edit...
api/v1/guestbook_types.go
controllers/guestbook_controller.go
Running make:
$ make
go: creating new go.mod: module tmp
go get: added sigs.k8s.io/controller-tools v0.2.5
/Users/zhengchicheng/go/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go

# API创建完成后，在项目根目录下查看目录结构
$ tree -L 3
.
├── Dockerfile # 用于构建 Operator 镜像
├── Makefile   # 构建时使用
├── PROJECT    # 项目配置
├── api
│   └── v1
│       ├── apiexamplea_types.go
│       ├── groupversion_info.go
│       └── zz_generated.deepcopy.go
├── bin
│   └── manager
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── crd # 新增 CRD 定义
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── apiexamplea_editor_role.yaml
│   │   ├── apiexamplea_viewer_role.yaml
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   └── role_binding.yaml
│   ├── samples
│   │   └── groupa_v1_apiexamplea.yaml
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── controllers # 新增controller
│   ├── apiexamplea_controller.go
│   └── suite_test.go
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go # 新增处理逻辑

15 directories, 40 files
```





### 安装CRD

```bash
$ make install
```



### 部署Controller

有两种方式运行controller

* 本地运行，用于调试
* 部署到kubernetes上运行，作为生产使用



#### 本地运行controller

```bash
$ make run
go run ./main.go
2021-04-01T17:21:14.672+0800	INFO	controller-runtime.metrics	metrics server is starting to listen	{"addr": ":8080"}
2021-04-01T17:21:14.673+0800	INFO	setup	starting manager
2021-04-01T17:21:14.673+0800	INFO	controller-runtime.manager	starting metrics server	{"path": "/metrics"}
2021-04-01T17:21:14.673+0800	INFO	controller-runtime.controller	Starting EventSource	{"controller": "guestbook", "source": "kind source: /, Kind="}
2021-04-01T17:21:14.777+0800	INFO	controller-runtime.controller	Starting Controller	{"controller": "guestbook"}
2021-04-01T17:21:14.777+0800	INFO	controller-runtime.controller	Starting workers	{"controller": "guestbook", "worker count": 1}
```





#### 将controller部署到kubernetes

执行下面的命令部署controller到kubernetes上，这一步将会在本地构建controller镜像，并推送到DockerHub上，然后在kubernetes上部署Deployment资源。

```bash
make docker-build docker-push IMG=jimmysong/kubebuilder-example:latest
make deploy IMG=jimmysong/kubebuilder-example:latest
```



利用以下命令行查看Deployment对象和Pod资源

``` bash
$ kubectl get deployment -n kubebuilder-example-system
NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
kubebuilder-example-controller-manager   1/1     1            1           4m25s

$ kubectl get pod -n kubebuilder-example-system
NAME                                                      READY   STATUS    RESTARTS   AGE
kubebuilder-example-controller-manager-689bf786fb-xcqbk   2/2     Running   0          5m16s
```



### 创建 CR

``` bash
# 部署CR
$ kubectl apply -f config/samples/groupa_v1_apiexamplea.yaml

# 查看新建的CR
$ kubectl get apiexampleas.groupa.k8s.zhuang.com
NAME                 AGE
apiexamplea-sample   26s

$ kubectl get apiexampleas.groupa.k8s.zhuang.com -o yaml
apiVersion: v1
items:
- apiVersion: groupa.k8s.zhuang.com/v1
  kind: ApiExampleA
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"groupa.k8s.zhuang.com/v1","kind":"ApiExampleA","metadata":{"annotations":{},"name":"apiexamplea-sample","namespace":"default"},"spec":{"foo":"bar"}}
    creationTimestamp: "2021-04-01T12:12:23Z"
    generation: 1
    managedFields:
    - apiVersion: groupa.k8s.zhuang.com/v1
      fieldsType: FieldsV1
      fieldsV1:
        f:metadata:
          f:annotations:
            .: {}
            f:kubectl.kubernetes.io/last-applied-configuration: {}
        f:spec:
          .: {}
          f:foo: {}
      manager: kubectl-client-side-apply
      operation: Update
      time: "2021-04-01T12:12:23Z"
    name: apiexamplea-sample
    namespace: default
    resourceVersion: "415357"
    selfLink: /apis/groupa.k8s.zhuang.com/v1/namespaces/default/apiexampleas/apiexamplea-sample
    uid: 55e88f71-f660-430c-93b5-dd78dd517eed
  spec:
    foo: bar
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

至此一个基本的 Operator 框架已经创建完成，但这个 Operator 只是修改了 etcd 中的数据而已，实际上什么事情也没做，因为我们没有在 Operator 中的增加业务逻辑。



## 增加业务逻辑

下面我们将修改 CRD 的数据结构并在 controller 中增加一些日志输出



1. 初始化项目和API
2. 安装CRD
3. 部署Controller
4. 创建CR



# GitHub Demo

[operator-demo](https://github.com/zhuangzhuang131419/operator-demo)



# 参考资料

* [从零开始入门 K8s | Kubernetes API 编程利器：Operator 和 Operator Framework](https://www.kubernetes.org.cn/6746.html)
* [使用 kubebuilder 创建 operator 示例](https://jimmysong.io/kubernetes-handbook/develop/kubebuilder-example.html)

