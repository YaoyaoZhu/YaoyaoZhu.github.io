---
layout: post
title: RabbitMQ_learn
categories: [blog]
tags: [RabbitMQ]
description: 
---

RabbitMQ_learn

1. R.Connection create TCP connection, channels shares a connection. TCP connection is costly.

2. flow

   ```flow
   st=>start: Producer
   e=>end: Consumer
   opExchange=>operation: Exchange
   opBlinding=>operation: Blinding
   opQueue=>operation: Queue
   st->opExchange->opBlinding->opQueue->e
   ```


basic.consumer: 订阅，消费后最近的消息后**自动接收**下一条消息。

basic.get: 获得单条。

Consumer获得消息需要返回basic.ack，一般采用auto_ack。

basic.reject允许Consumer拒绝消息。basic.reject=true?重发:删除

