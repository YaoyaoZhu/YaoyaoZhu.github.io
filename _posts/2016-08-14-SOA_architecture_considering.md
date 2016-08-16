---
layout: post
title: SOA架构的考虑
categories: [blog]
tags: [SOA]
description: 
---

SOA架构的考虑

一个SOA中间件应该有以下部分:

​	Container、RPC框架、Daemon、Heartbeat、LoadBalance、Monitor。

[TOC]

# RPC

​	对于RPC（远程调用），最少我们得有Web API调用方式，一般为RESTful风格，通过http＋约定来实现远程调用，如http+json/xml，对于B/S来说，采用这种方式对B端更加友好。然而，这种方式毕竟效率不够高，对于MicroServer之间的RPC，显然直接使用二进制+TCP，通过C/S来约定更为高效，如ProtocolBuffers、thrift。

## Web API

### RPC通信协议

​	可以基于http+json作为协议，使用Netty来处理外部http请求。

​	如何设计？

```
struts{
	basic:Head,Exception;
	Request extends Head{Exception};
	Response extends Head{Exception};
}
```

### RPC协议传输

http协议。

ps：[RESTful](http://www.cnblogs.com/artech/p/3506553.html)

### RPC请求接收

​	请求接收通过Netty来处理，封一个HttpJsonServer，启动的时候加载一些ChannelHandler：JsonRequestDeserializer，异步处理的handler。我们可以在SOA Container里启动该server。

具体可以看：[异步处理模型]({% post_url 2016-06-13-Async_queue_program_model %})

### RPC请求解析

​	请求解析的实现是反序列化后通过解析json，得到类和接口，在通过动态代理实现调用。可以在这里做些trace。

其实就是把request和response都封到Task里，(再使用异步队列存任务，)Executor执行。



​	即Container—>HttpJsonServer—>HttpJsonTask{request—>pre—>filter—>run—>response}



## ClientToService

### RPC通信协议

基于TCP协议，使用二进制的数据包，可以使用Socket通信。

### RPC协议传输

### RPC请求解析





