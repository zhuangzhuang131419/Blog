---
title: nginx那些事
date: 2020-08-25 10:40:48
tags:
---

# nginx
在介绍什么是 nginx 之前，我们先了解两个概念
## web服务器
> 负责处理和响应用户请求，一般也称为http服务器，如 Apache, IIS, Nginx
## 应用服务器
> 存放和运行系统程序的服务器，负责处理程序中的业务逻辑，如 Tomcat, Weblogic, Jboss

总结一下, nginx 就是
* 一种轻量级的 web 服务器
* 采用事件驱动的异步非阻塞处理方式框架
* 占用内存少，启动速度快，并发能力强

## Nginx的四大应用
### 动静分离

{% asset_img 动静分离.png 动静分离 %}

通过示意图，我们可以得出，**动静分离**其实就是 nginx 服务器将收到的请求分为**动态请求**和**静态请求**
* 静态请求直接从 nginx 服务器所设定的根目录路径去取对应的资源，动态请求转发给真实的后台去处理，即是应用服务器(Tomcat)
* 这样做不仅能给应用服务器减轻压力，将后台 api 接口服务化，还能将前后端代码分开并行开发和部署。

```nginx
server {  
        listen       8080;        
        server_name  localhost;

        location / {
            root   html; # Nginx默认值
            index  index.html index.htm;
        }
        
        # 静态化配置，所有静态请求都转发给 nginx 处理，存放目录为 my-project
        location ~ .*\.(html|htm|gif|jpg|jpeg|bmp|png|ico|js|css)$ {
            root /usr/local/var/www/my-project; # 静态请求所代理到的根目录
        }
        
        # 动态请求匹配到path为'node'的就转发到8002端口处理
        location /node/ {  
            proxy_pass http://localhost:8002; # 充当服务代理
        }
}
```
* 访问静态资源 nginx 服务器会返回 ```my-project``` 里面的文件
* 访问动态请求 nginx 服务器会将它从8002端口请求到的内容，原封不动的返回回去

### 反向代理
#### 什么是反向代理
反向代理其实就类似你去找代购帮你买东西（浏览器或其他终端向nginx请求），你不用管他去哪里买，只要他帮你买到你想要的东西就行（浏览器或其他终端最终拿到了他想要的内容，但是具体从哪儿拿到的这个过程它并不知道）。
#### 反向代理的作用
1. 保障应用服务器的安全 (增加一层代理，可以屏蔽危险攻击，更方便的控制权限)
2. 实现负载均衡
3. 实现跨域

```nginx
server {  
        listen       8080;        
        server_name  localhost;

        location / {
            root   html; # Nginx默认值
            index  index.html index.htm;
        }
        
        proxy_pass http://localhost:8000; # 反向代理配置，请求会被转发到8000端口
}
```

上面这个例子意思是向 nginx 请求 ```localhost:8080``` 跟请求 ```http://localhost:8000``` 是一样的效果

{% asset_img 反向代理.png 反向代理 %}

nginx 就充当图中的 proxy. 当左边三个client在请求向 nginx 获取内容，是感受不到三个server的存在的
> proxy 就充当了三个server的反向代理

* CDN 服务就是经典的反向代理
* 反向代理也是实现负载均衡的基础

### 负载均衡
随着业务的不断增长和用户的增多，一台服务器已经无法满足系统的需求。这个时候就出现了服务器**集群**。

在服务器集群中, nginx 可以将接收到的客户端请求“均匀地”（严格讲并不一定均匀，可以通过设置权重）分配到这个集群中所有的服务器上。这个就叫做负载均衡。

{% asset_img 负载均衡.png 负载均衡 %}

#### 负载均衡的作用
* 分摊服务器集群压力
* 保证客户端访问的稳定性

nginx 自带**健康检查**功能，会定期轮询向集群里的所有服务器发送健康检查请求，来检查集群中是否有服务器处于异常状态。一旦发现某台服务器出现异常，那么在这以后代理进来的客户端请求都不会被发送到该服务器上，从而保证客户端访问的稳定性。


配置一个负载均衡
```nginx
# 负载均衡：设置domain
upstream domain {
    server localhost:8000;
    server localhost:8001;
}
server {  
        listen       8080;        
        server_name  localhost;

        location / {
            # root   html; # Nginx默认值
            # index  index.html index.htm;
            
            proxy_pass http://domain; # 负载均衡配置，请求会被平均分配到8000和8001端口
            proxy_set_header Host $host:$server_port;
        }
}
```

8000和80001是我本地用 Node.js 起的两个服务，负载均衡成功后可以看到访问 ```localhost:8080``` 有时会访问到8000端口的页面，有时会访问到8001端口的页面。

### 正向代理

{% asset_img 正向代理.png 正向代理 %}

正向代理跟反向道理正好相反。

> 此时，proxy就充当了三个client的正向代理

正向代理，意思是一个位于客户端和原始服务器(origin server)之间的服务器，为了从原始服务器取得内容，客户端向代理发送一个请求并指定目标(原始服务器)，然后代理向原始服务器转交请求并将获得的内容返回给客户端。客户端才能使用正向代理。当你需要把你的服务器作为代理服务器的时候，可以用Nginx来实现正向代理。

科学上网vpn（俗称翻墙）其实就是一个正向代理工具。

该 vpn 会将想访问墙外服务器 server 的网页请求，代理到一个可以访问该网站的代理服务器 proxy 上。这个 proxy 把墙外服务器 server 上获取的网页内容，再转发给客户。


# 参考文献
https://juejin.im/entry/6844903464816869383

https://juejin.im/post/6844904129987526663