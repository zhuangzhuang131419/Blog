---
title: Docker课堂笔记
date: 2021-03-17 13:45:54
tags: Docker
---

# Docker简介

# Docker安装

## Docker的基本组成

### 镜像

>  Docker 镜像（Image）就是一个**只读**的模板。镜像可以用来创建 Docker 容器，**一个镜像可以创建很多容器**。



### 容器

>  Docker 利用容器（Container）独立运行的一个或一组应用。**容器是用镜像创建的运行实例**。

* 它可以被启动、开始、停止、删除。每个容器都是相互隔离的、保证安全的平台。

* **可以把容器看做是一个简易版的 Linux 环境**（包括root用户权限、进程空间、用户空间和网络空间等）和运行在其中的应用程序。

* 容器的定义和镜像几乎一模一样，也是一堆层的统一视角，唯一区别在于容器的最上面那一层是可读可写的。



### 仓库

> 仓库（Repository）是集中存放镜像文件的场所。

* 仓库(Repository)和仓库注册服务器（Registry）是有区别的。仓库注册服务器上往往存放着多个仓库，每个仓库中又包含了多个镜像，每个镜像有不同的标签（tag）。

* 仓库分为公开仓库（Public）和私有仓库（Private）两种形式。

* 最大的公开仓库是 Docker Hub(https://hub.docker.com/)，存放了数量庞大的镜像供用户下载。
* 国内的公开仓库包括阿里云 、网易云等



### Docker的架构图

{% asset_img Docker架构图.png Docker架构图 %}



### 小总结

Docker 本身是一个容器运行载体或称之为管理引擎。我们把应用程序和配置依赖打包好形成一个可交付的运行环境，这个打包好的运行环境就似乎 image镜像文件。只有通过这个镜像文件才能生成 Docker 容器。image 文件可以看作是容器的模板。Docker 根据 image 文件生成容器的实例。同一个 image 文件，可以生成多个同时运行的容器实例。

*  image 文件生成的容器实例，本身也是一个文件，称为镜像文件。

*  一个容器运行一种服务，当我们需要的时候，就可以通过docker客户端创建一个对应的运行实例，也就是我们的容器

* 至于仓储，就是放了一堆镜像的地方，我们可以把镜像发布到仓储中，需要的时候从仓储中拉下来就可以了。



## 永远的HelloWorld

#### 启动Docker后台容器（测试运行hello-world）

```dockerfile
-> docker run hello-word # hello-word是一个镜像, 要以这个镜像生成一个容器实例

Unable to find image 'hello-world:latest' locally # 本地没有这个镜像, latest是镜像的tag
latest: Pulling from library/hello-world
b8dfde127a29: Pull complete
Digest: sha256:308866a43596e83578c7dfa15e27a73011bdd402185a84c5cd7f32a88b501a24
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```



## 底层原理

### Docker是怎么工作的

Docker是一个Client-Server结构的系统，Docker守护进程运行在主机上， 然后通过Socket连接从客户端访问，守护进程从客户端接受命令并管理运行在主机上的容器。 **容器，是一个运行时环境，就是我们前面说到的集装箱**。



### 为什么Docker比VM快

* Docker有着比虚拟机更少的抽象层。由于Docker不需要Hypervisor实现硬件资源虚拟化，运行在Docker容器上的程序直接使用的都是实际物理机的硬件资源。因此在CPU、内存利用率上Docker将会在效率上有明显优势。

* Docker利用的是宿主机的内核，而不需要Guest OS。因此，当新建一个容器时，Docker不需要和虚拟机一样重新加载一个操作系统内核。仍而避免引寻、加载操作系统内核返个比较费时费资源的过程，当新建一个虚拟机时，虚拟机软件需要加载Guest OS，返个新建过程是分钟级别的。而Docker由于直接利用宿主机的操作系统,则省略了返个过程，因此新建一个Docker容器只需要几秒钟。



|            | Docker容器              | 虚拟机（VM）                |
| ---------- | ----------------------- | --------------------------- |
| 操作系统   | 与宿主机共享OS          | 宿主机OS上运行虚拟机OS      |
| 存储大小   | 镜像小，便于存储与传输  | 镜像庞大（vmdk, vdi...）    |
| 运行性能   | 几乎无额外性能损失      | 操作系统额外的CPU、内存消耗 |
| 移植性     | 轻便、灵活、适应于Linux | 笨重，与虚拟化技术耦合度高  |
| 硬件亲和性 | 面向软件开发者          | 面向硬件开发者              |
| 部署速度   | 快速，秒级              | 较慢，10s以上               |



# Docker常用命令



## 帮助命令

```dockerfile
-> docker version

-> docker info

-> docker --help
```



## 镜像命令

```dockerfile
-> docker images # 列出本地主机上所有的镜像
REPOSITORY               TAG       IMAGE ID       CREATED         SIZE
hello-world              latest    d1165f221234   11 days ago     13.3kB

# REPOSITORY: 表示镜像的仓库源
# TAG: 镜像的标签
# IMAGE ID: 镜像ID
# CREATED: 镜像创建时间
# SIZE: 镜像大小

-a: 列出本地所有的镜像（含中间映像层）
-q: 只显示镜像ID
--digests: 显示镜像的摘要信息
--no-trunc: 显示完整的镜像信息
```



```dockerfile
-> docker search [OPTIONS]镜像的名字 # 在hub中查找镜像

--no-trunc: 显示完整的镜像描述
-s: 列出收藏数不小于指定值的镜像
--automated: 只列出 automated build类型的镜像
```



```dockerfile
-> docker pull 镜像名称[:TAG] # 下载镜像

# TAG默认latest
-> docker pull tomcat 
-> docker pull tomcat:latest
```




```dockerfile
-> docker rmi -f 镜像ID # 删除单个
-> docker rmi -f 镜像名1:TAG 镜像名2:TAG # 删除多个
-> docker rmi -f $(docker images -qa)  # 删除全部（子语句）
```



## 容器命令

### 新建并启动容器

```dockerfile
-> docker run [OPTIONS] IMAGE [COMMAND][ARG...]

--name="容器新名字": 为容器指定一个名称；
-d: 后台运行容器，并返回容器ID，也即启动守护式容器；

# 用的最多
-i: 以交互模式运行容器，通常与 -t 同时使用；
-t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；

-P: 随机端口映射；
-p: 指定端口映射，有以下四种格式
      ip:hostPort:containerPort
      ip::containerPort
      hostPort:containerPort
      containerPort
      
-> docker run -it centos
# 进入新的实例
```



### 列出当前所有正在运行的容器

```dockerfile
-> docker ps [OPTIONS]

-a: 列出当前所有正在运行的容器+历史上运行过的
-l: 显示最近创建的容器。
-n: 显示最近n个创建的容器。
-q: 静默模式，只显示容器编号。
--no-trunc: 不截断输出。

-> docker ps
CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS         PORTS     NAMES
09aad010c2bd   centos    "/bin/bash"   3 minutes ago   Up 7 seconds             funny_tharp
```



### 退出容器

* exit 
  * 容器停止退出
* Ctrl + P + Q
  * 容器不停止退出

### 启动容器

```dockerfile
-> docker start 容器ID或者容器名
```



### 重启容器

```dockerfile
-> docker restart 容器ID或者容器名
```



### 停止容器

```dockerfile
-> docker stop 容器ID或者容器名
```



### 强制停止容器

```dockerfile
-> docker kill 容器ID或者容器名
```



### 删除已停止的容器

```dockerfile
-> docker rm 容器ID

# 一次性删除多个容器
-> docker rm -f $(docker ps -a -q)
-> docker ps -a -q | xargs docker rm 
```



### 启动守护式线程

与之相对的是启动交互式容器，我们不希望交互，希望后台以守护式进程的方式启动

```dockerfile
-> docker run -d 容器名
-> docker ps
# 发现容器已经退出了
CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS         PORTS     NAMES
```



问题：然后docker ps -a 进行查看, 会发现容器已经退出
很重要的要说明的一点: **Docker容器后台运行,就必须有一个前台进程**。容器运行的命令如果不是那些一直挂起的命令（比如运行top，tail），就是会自动退出的。

这个是docker的机制问题，比如你的web容器，我们以nginx为例，正常情况下，我们配置启动服务只需要启动响应的service即可。例如 service nginx start。但是,这样做，nginx为后台进程模式运行，就导致docker前台没有运行的应用（因为不是以交互的方式启动的），这样的容器后台启动后，**会立即自杀**因为他觉得他没事可做了。所以，最佳的解决方案是，将你要运行的程序以前台进程的形式运行。



### 查看容器日志

```dockerfile
-> docker logs -f -t --tail 容器ID

-t: 加入时间戳
-f: 跟随最新的日志打印
--tail: 数字 显示最后多少条
```



### 查看容器内运行进程

```bash
-> docker top 容器ID
```



### 查看容器内部细节

```bash
-> docker inspect 容器ID
```



### 进入正在运行的容器并以命令行交互

```
-> docker exec -it 容器ID [bashShell]
-> docker attach 容器ID
```

* attach: 直接进入容器启动命令的终端，不会启动新的进程
* exec: 是在容器中打开新的终端，并且可以启动新的进程 （隔山打牛）



### 从容器内拷贝文件到主机上

```bash
-> docker cp 容器ID:容器内路径 目的主机路径
```



# Docker镜像

镜像就是一个“千层饼”

## 镜像的含义


镜像是一种轻量级、可执行的独立软件包，**用来打包软件运行环境和基于运行环境开发的软件**，它包含运行某个软件所需的所有内容，包括代码、运行时、库、环境变量和配置文件。



### UnionFS（联合文件系统）

>  Union文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，它支持**对文件系统的修改作为一次提交来一层层的叠加**，同时可以将不同目录挂载到同一个虚拟文件系统下(unite several directories into a single virtual filesystem)。

Union 文件系统是 Docker 镜像的基础。镜像可以通过分层来进行继承，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。

特性：一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录



### Docker镜像加载原理

{% asset_img Docker层级结构.png Docker层级结构 %}

Docker的镜像实际上由一层一层的文件系统组成，这种层级的文件系统UnionFS。

bootfs(boot file system)主要包含bootloader和kernel。bootloader主要是引导加载kernel, Linux刚启动时会加载bootfs文件系统，**在Docker镜像的最底层是bootfs**。这一层与我们典型的Linux/Unix系统是一样的，包含boot加载器和内核。当boot加载完成之后整个内核就都在内存中了，此时内存的使用权已由bootfs转交给内核，此时系统也会卸载bootfs。

rootfs (root file system) ，在bootfs之上。包含的就是典型 Linux 系统中的 /dev, /proc, /bin, /etc 等标准目录和文件。rootfs就是各种不同的操作系统发行版，比如Ubuntu，Centos等等。 



Q: 平时我们安装进虚拟机的CentOS都是好几个G，为什么docker这里才200M？

对于一个精简的OS，rootfs可以很小，只需要包括最基本的命令、工具和程序库就可以了，因为底层直接用Host的kernel，自己只需要提供 rootfs 就行了。由此可见对于不同的linux发行版, bootfs基本是一致的, rootfs会有差别, 因此不同的发行版可以公用bootfs。



### 分层的原因

最大的好处就是可以共享资源



## 镜像的特点

Docker镜像都是只读的，当容器启动时，一个新的可写层被加载到镜像的顶部。这一层通常被称作“容器层”，“容器层”之下的都叫“镜像层”。



## 镜像commit操作补充

```bash
# 提交容器副本使之成为一个新的镜像
-> docker commit 

-> docker commit -m="提交的描述信息" -a="作者" 容器ID 要创建的目标镜像名:[标签名]
```



# Docker容器数据卷

## 定义

类似Redis里面的rdb和aof文件。

先来看看Docker的理念：
*  将运用与运行的环境打包形成容器运行 ，运行可以伴随着容器，但是我们对数据的要求希望是持久化的
*  容器之间希望有可能共享数据


Docker容器产生的数据，如果不通过docker commit生成新的镜像，使得数据做为镜像的一部分保存下来，
那么当容器删除后，数据自然也就没有了。


为了能保存数据在docker中我们使用卷。



## 作用

* 容器的持久化
* 容器间继承+共享数据



## 数据卷

在容器内添加数据有两种方法：直接命令添加和DockerFile添加



### 直接命令添加

```bash
->  docker run -it -v /宿主机绝对路径目录:/容器内目录    镜像名
->  docker run -it -v /宿主机绝对路径目录:/容器内目录:ro 镜像名 # read-only 只读（只允许主机单向写操作传给它）
```



数据卷挂载成功，容器和宿主机之间数据共享。容器停止退出后，主机修改后数据仍然同步。



### DockerFile添加

> dockerfile与image的关系有点类似使用java文件生成的class文件。可以理解为镜像这个模板的描述文件。



示例：

```dockerfile
# 基于centos构建
FROM centos 
# 新建两个容器卷
VOLUME ["/dataVolumeContainer1", "/dataVolumeContainer2"]
# 运行命令行
CMD echo "finished,-----success1"
CMD /bin/bash
```



整个dockerfile等价于：

```bash
-> docker run -it -v /host1:/dataVolumeContainer1 -v /host2:/dataVolumeContainer2 centos /bin/bash
```

 如果没有指定host1和host2将会使用默认的



## 数据卷容器

### 含义

> 命名的容器挂载数据卷，其它容器通过挂载这个(父容器)实现数据共享，挂载数据卷的容器，称之为数据卷容器

活动硬盘上面挂活动硬盘，实现数据的传递



```bash
-> docker run -it --name dc01 zzyy/centos
-> docker run -it --name dc02 --volumes-from dc01 zzyy/centos
-> docker run -it --name dc03 --volumes-from dc01 zzyy/centos
```



dc02和dc03分别在dataVolumeContainer2各自新增内容，回到dc01可以看到02/03各自添加的都能共享了。

删除dc01，dc02的修改可以被dc03看到。（即使dc03 是 volume from dc01的）

更像是共享而不是继承。



### 结论

容器之间配置信息的的传递，数据卷的生命周期一直持续到没有容器使用它为止。



# DockerFile解析

## 含义

> Dockerfile是用来构建Docker镜像的构建文件，是由一系列命令和参数构成的脚本。



### 构建三步骤

* 编写dockerfile文件
* `docker build`

* `docker run`



### 示例

以centos的dockerfile为例

```dockerfile
# scratch 是一个基础的镜像
FROM scratch
# 作者信息
MAINTAINER The CentOS Project <cloud-ops@centos.org>

ADD c68-docker.tar.xz /
LABEL name="CentOS Base Image" \
	vendor="CentOS" \ 
	license="GPLv2" \
	build-date="2016-06-02"
	
# Default command
CMD ["/bin/bash"]
```



## DockerFile构建过程解析

### Dockerfile内容基础知识

1. 每条保留字指令都必须为大写字母且后面要跟随至少一个参数
2. 指令按照从上到下，顺序执行
3. #表示注释
4. 每条指令都会创建一个新的镜像层，并对镜像进行提交



### Docker执行Dockerfile的大致流程

1. docker从基础镜像运行一个容器
2. 执行一条指令并对容器作出修改
3. 执行类似docker commit的操作提交一个新的镜像层
4. docker再基于刚提交的镜像运行一个新容器
5. 执行dockerfile中的下一条指令直到所有指令都执行完成



### 总结

从应用软件的角度来看，Dockerfile、Docker镜像与Docker容器分别代表软件的三个不同阶段，
*  Dockerfile是软件的原材料
*  Docker镜像是软件的交付品
*  Docker容器则可以认为是软件的运行态。
Dockerfile面向开发，Docker镜像成为交付标准，Docker容器则涉及部署与运维，三者缺一不可，合力充当Docker体系的基石。



1. Dockerfile，需要定义一个Dockerfile，Dockerfile定义了进程需要的一切东西。Dockerfile涉及的内容包括执行代码或者是文件、环境变量、依赖包、运行时环境、动态链接库、操作系统的发行版、服务进程和内核进程(当应用进程需要和系统服务和内核进程打交道，这时需要考虑如何设计namespace的权限控制)等等;

2. Docker镜像，在用Dockerfile定义一个文件之后，docker build时会产生一个Docker镜像，当运行 Docker镜像时，会真正开始提供服务;

3. Docker容器，容器是直接提供服务的。



## DockerFile体系结构（保留字指令）

| 保留字       | 含义                                                         |
| ------------ | ------------------------------------------------------------ |
| `FROM`       | 基础镜像，当前镜像是基于哪个镜像                             |
| `MAINTAINER` | 镜像维护者的姓名和邮箱地址                                   |
| `RUN`        | 容器构建**时**需要运行的命令                                 |
| `EXPOSE`     | 当前容器对外暴露出的端口                                     |
| `WORKDIR`    | 指定在创建容器后，终端默认登陆的进来工作目录，一个落脚点     |
| `ENV`        | 用来在构建镜像过程中设置环境变量（`ENV MY_PATH /usr/mytest`，这样就可以直接使用`MY_PATH`） |
| `ADD`        | 将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包（拷贝+解压） |
| `COPY`       | 类似ADD，拷贝文件和目录到镜像中。将从构建上下文目录中 <源路径> 的文件/目录复制到新的一层的镜像内的 <目标路径> 位置 |
| `VOLUME`     | 容器数据卷，用于数据保存和持久化工作                         |
| `CMD`        | 指定一个容器启动时要运行的命令；Dockerfile 中可以有多个 CMD 指令，**但只有最后一个生效**，CMD 会被 docker run 之后的参数替换 |
| `ENTRYPOINT` | 指定一个容器启动时要运行的命令；ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数 |
| `ONBUILD`    | 当构建一个被继承的Dockerfile时运行命令，父镜像在被子继承后父镜像的onbuild被触发 |



## 示例

```dockerfile
FROM centos
ENV MY_PATH /tmp
WORKDIR $MY_PATH

RUN yum -y install vim
RUN yum -y install net-tools

EXPOSE 80
CMD /bin/bash
```