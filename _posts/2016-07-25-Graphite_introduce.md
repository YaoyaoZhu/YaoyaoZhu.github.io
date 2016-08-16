---
layout: post
title: Graphite介绍
categories: [blog]
tags: [Vine]
description: 服务调用时间和频次等信息等监控工具
---

## Graphite

Vine的监控使用Graphite，工具包提供埋点工具，在FrameWork中埋点。结合Grafana实现丰富的图形监控。

**前端：**渲染图表

**后端：**存储时间序列数据

**组件：**

​	Load Balancer|

​	**Carbon**|

​	**Whisper**|

​	Filesystem|

[***Carbon***](https://www.google.com/url?q=https%3A%2F%2Fgithub.com%2Fgraphite-project%2Fcarbon&sa=D&sntz=1&usg=AFQjCNE9GGsiJvLxwhgUEeeyX4R89IzbEA)实际上是一系列守护进程，组成一个Graphite安装的存储后端。这些守护进程用一个名为[Twisted](https://twistedmatrix.com/trac/)的事件驱动网络引擎监听时间序列数据。Twisted框架让*Carbon*守护进程能够以很低的开销处理大量的客户端和流量。

[***Whisper***](https://www.google.com/url?q=https%3A%2F%2Fgithub.com%2Fgraphite-project%2Fwhisper&sa=D&sntz=1&usg=AFQjCNFvOPixjQv6Trh9S7pvkFv2ktbx3Q)是一个用于存储时间序列数据的数据库，之后应用程序可以用*create**，update**和fetch*操作获取并操作这些数据。

***指标项（metric* *）***是一种随着时间不断变化的可度量的数量，例如：

- 每秒请求数
- 请求处理时间
- CPU使用情况

A *datapoint* is a tuple containing:

**数据点（datapoint）**是包含如下信息的三元组：

- 指标项名称
- 度量值
- 时间序列上某个特定的点（通常是一个时间戳）

客户端应用程序通过将数据点发送至Carbon进程发布指标项。应用程序在Carbon进程所监听的端口上建立TCP连接，然后以简单平文本格式发送数据点信息。

## 

书籍：[Jason Dixon-Monitoring with Graphite-O'Reilly Media (2015)](http://ebooks.readbook5.com/search.php?req=Monitoring%20with%20Graphite&nametype=orig)

[安装教程－中文](http://www.infoq.com/cn/articles/graphite-intro)

[Installation Tutorial](https://www.infoq.com/articles/graphite-intro)



