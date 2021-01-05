---
title: RPC框架
date: 2020-07-23 17:14:11
tags:
---

# RPC 定义

RPC是一种技术思想而非一种规范或协议，常见的RPC技术和框架有
* 应用级的服务框架：阿里的Dubbo/Dubbox, Google的gRPC, Spring Boot/Spring Cloud
* 远程通信协议：RMI, Socket, SOAP(HTTP XML), REST(HTTP JSON)
* 通信框架：MINA和Netty

RPC采用客户机/服务器模式。请求程序就是一个客户机，而服务提供程序就是一个服务器。首先，客户机调用进程发送一个有进程参数的调用信息到服务进程，然后等待应答信息。在服务器端，进程保持睡眠状态直到调用信息到达为止。当一个调用信息到达，服务器获得进程参数，计算结果，发送答复信息，然后等待下一个调用信息，最后，客户端调用进程接收答复信息，获得进程结果，然后调用执行继续进行。

# RPC 流程
{% asset_img RPC框架图.png RPC框架图 %}
1. 本地调用某个函数方法
2. 本地机器的RPC框架把这个调用信息封装起来（调用的函数、入参等），序列化(json、xml等)后，通过网络传输发送给远程服务器。
3. 远程服务器收到调用请求后，远程机器的RPC框架反序列化获得调用信息，并根据调用信息定位到实际要执行的方法，执行完这个方法后，序列化执行结果，通过网络传输把执行结果发送回本地机器。
4. 本地机器的RPC框架反序列化出执行结果，函数return这个结果

# GO RPC 源码分析
## server端
* server端首先进行方法注册，通过反射处理将方法取出后存到map中。
* 然后进行网络调用，主要是监听端口，读取数据包，解码请求，调用反射处理后的方法，将返回值编码，返回给客户端。
### 方法注册

* Register

```go
// Register publishes the receiver's methods in the DefaultServer.
func Register(rcvr interface{}) error { return DefaultServer.Register(rcvr) }

// RegisterName is like Register but uses the provided name for the type
// instead of the receiver's concrete type.
func RegisterName(name string, rcvr interface{}) error {
	return DefaultServer.RegisterName(name, rcvr)
}
```
方法注册的入口函数有两个,分别为Register以及RegisterName,这里interface{}通常是带方法的对象.如果想要自定义方法的接收对象,则可以使用RegisterName.

* 反射处理过程
```go
type methodType struct {
    sync.Mutex // protects counters
    method     reflect.Method    //反射后的函数
    ArgType    reflect.Type      //请求参数的反射值
    ReplyType  reflect.Type      //返回参数的反射值
    numCalls   uint              //调用次数
}


type service struct {
    name   string                 // 服务名,这里通常为register时的对象名或自定义对象名
    rcvr   reflect.Value          // 服务的接收者的反射值
    typ    reflect.Type           // 接收者的类型
    method map[string]*methodType // 对象的所有方法的反射结果.
}
```
反射处理过程,其实就是将对象以及对象的方法,通过反射生成上面的结构,如注册```Arith.Multiply(xx,xx) error ```这样的对象时,生成的结构为``` map["Arith"]service, service 中ethod为 map["Multiply"]methodType```.



### 网络调用

## 参考文献
https://segmentfault.com/a/1190000013532622