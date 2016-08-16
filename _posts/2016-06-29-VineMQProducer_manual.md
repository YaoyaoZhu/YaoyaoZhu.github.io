---
layout: post
title: VineMQProducer manual
categories: [blog]
tags: [RabbitMQ,Vine]
description: 
---

# 2016-06-29-VineMQProducer_manual

## gradle配置

```
compile "me.ele.napos:vine-mq-connector:1.2.81-SNAPSHOT"
```

## config配置

```json
 "app": {
      "RabbitMQ3": {
      "uri": "amqp://napos:napos@192.168.112.94:5672",
      "exchange": "exchangeTestWG2",
      "routingKey": "action.test",
      "poolSize": 2,
      "waitTime": 20,
      "sendPoolMaxSize":200,
      "sendQueueSize":11000
    }
 }
```

1. 基本配置简化为uri，uri默认vhost为"/"。

2. routingKey是默认配置，具体发送时可以指定routingKey，如smipleMQProducer.send(msg, routingKey)。

3. sendPoolMaxSize小于处理器数量会logger.warning()

4. sendQueueSize建议配置大小最少10_000，否则会导致大量失败。

   ​

   ## SimpleMQProducer	

```java
public class SimpleMQProducer extends VineMQProducer {

    private static SimpleMQProducer producer = new SimpleMQProducer();

    public static SimpleMQProducer getProducer() {
        return producer;
    }

    @Override
    protected String getConfigKey() {
        return "app.RabbitMQ2";
    }
}	
```



## Execute

```java
public void execute() {
        SimpleMQProducer producer = SimpleMQProducer.getProducer();
        try {
            producer.send("MsgExec test");
            producer.send("MsgExec routingKey_test","routingKey_test");//routingKey需要绑定。
            producer.asyncSend("testThread");//线程组发送
        } catch (VineRMQException e) {
            e.printStackTrace();
        }
    }
```

