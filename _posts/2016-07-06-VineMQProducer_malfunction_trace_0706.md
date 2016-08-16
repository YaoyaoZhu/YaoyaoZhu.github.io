---
layout: post
title: 07-06抖动问题追踪
categories: [blog]
tags: [RabbitMQ]
description: 
---

07-06抖动问题追踪



asyncSend里的send_in打点和send里发送成功的打点

![sendInAndMQout](/img/post/2016-07-06/sendInAndMQout.jpg)

getConnection and returnConnection 

![getConnAndReturnConn](/img/post/2016-07-06/getConnAndReturnConn.jpg)

结合堆栈信息

```verilog
"pool-5-thread-831" Id=1174 TIMED_WAITING on java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject@7ae661ee
	at sun.misc.Unsafe.park(Native Method)
	-  waiting on java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject@7ae661ee
	at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:215)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2078)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1093)
	at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:809)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	...
```

判断是连接阻塞了。