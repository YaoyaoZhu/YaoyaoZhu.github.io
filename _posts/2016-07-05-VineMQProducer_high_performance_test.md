---
layout: post
title: RabbitMQ曲线抖动问题追踪
categories: [blog]
tags: [RabbitMQ]
description: 
---

### 服务配置

```json
    "RabbitMQ2": {
      "uri": "amqp://napos:napos@192.168.112.94:5672",
      "exchange": "exchangeTestWG2",
      "routingKey": "action.test",
      "poolSize": 10,
      "waitTime": 3000,
      "asyncPoolSize":200,
      "asyncQueueSize":11000
    }
```

### 压测配置

```html
<ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="测试RabbitMQ线程组" enabled="true">
        <stringProp name="ThreadGroup.on_sample_error">continue</stringProp>
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController" guiclass="LoopControlPanel" testclass="LoopController" testname="循环控制器" enabled="true">
          <boolProp name="LoopController.continue_forever">false</boolProp>
          <stringProp name="LoopController.loops">50000</stringProp>
        </elementProp>
        <stringProp name="ThreadGroup.num_threads">150</stringProp>
        <stringProp name="ThreadGroup.ramp_time"></stringProp>
        <longProp name="ThreadGroup.start_time">1458285834000</longProp>
        <longProp name="ThreadGroup.end_time">1458285834000</longProp>
        <boolProp name="ThreadGroup.scheduler">false</boolProp>
        <stringProp name="ThreadGroup.duration"></stringProp>
        <stringProp name="ThreadGroup.delay"></stringProp>
      </ThreadGroup>
```

### 结果

![Metric](/img/post/2016-07-05/metric.jpg)

### jstack查看日志

```shell
jstack 32014 >/tmp/jstack3
vim /tmp/jstack3
```

```shell
"thread_async_mq_producer178" #679 prio=10 os_prio=0 tid=0x00007f14c8008800 nid=0x13d3 waiting on condition [0x00007f145d7d5000]
   java.lang.Thread.State: TIMED_WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x00000006c9dd67f0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
        at java.util.concurrent.LinkedBlockingDeque.pollFirst(LinkedBlockingDeque.java:522)
        at java.util.concurrent.LinkedBlockingDeque.poll(LinkedBlockingDeque.java:684)
        at me.ele.napos.vine.mq.impl.RabbitConnectionPool.getConnection(RabbitConnectionPool.java:67)
        at me.ele.napos.vine.mq.impl.VineMQProducer.send(VineMQProducer.java:223)
        at me.ele.napos.vine.mq.impl.VineMQProducer.lambda$asyncSend$3(VineMQProducer.java:150)
        at me.ele.napos.vine.mq.impl.VineMQProducer$$Lambda$49/1039297198.run(Unknown Source)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
        at java.lang.Thread.run(Thread.java:745)
```

可以看出在RabbitConnectionPool.getConnection()时，从LinkedBlockingDeque里取连接发现取不到。



原因：暂时未发现，增加取放Connection的操作打点，用tmux跑一夜，明天再看。

```java
    public Connection getConnection() throws VineRMQException {
        StatsdMetricUtil.increment("getConnection");
        Connection connection = null;
        try {
            connection = connList.poll(waitTime, TimeUnit.MILLISECONDS);
        } catch (InterruptedException e) {
            throw new VineRMQException(e);
        }
        if (connection == null) {
            logger.error("rabbitMQ connections pool full and conn are all in using");
            throw new VineRMQException("rabbitMQ connections pool full and conn are all in using");
        }
        if (!connection.isOpen()) {
            connClosedList.offer(connection);
            return getConnection();
        }
        return connection;
    }

    public void returnConnection(Connection connection) {
        StatsdMetricUtil.increment("returnConnection");
        if (!connList.offer(connection)) logger.error("RabbitMQConnectionLost : returnConnection cause ERROR");
    }
```

