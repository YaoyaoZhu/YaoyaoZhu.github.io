---
layout: post
title: 定位问题——终端环境下使用jstack分析jvm
categories: [blog]
tags: [Java]
description: 
---

定位问题——终端环境下使用jstack分析jvm

终端环境下使用jstack分析jvm(现在一般在Framwork层集成trace)

一般来说，解决问题先看log，再看stack。

#### jstack定向dump文件

```shell
jps |grep Main|awk '{print $1}'|xargs -t jstack >dump1
```

#### 统计所有线程状态

```shell
grep java.lang.Thread.State dump1|awk '{print $2$3$4$5}'| sort | uniq -c
```

#### 结果如下：

```
    222 RUNNABLE
      5 TIMED_WAITING(onobjectmonitor)
     10 TIMED_WAITING(parking)
      2 TIMED_WAITING(sleeping)
      3 WAITING(onobjectmonitor)
    257 WAITING(parking)
```

#### 打开dump查看257 WAITING(parking)

```verilog
"thread_async_mq_producer27" #565 prio=10 os_prio=0 tid=0x00007f9fb0009000 nid=0x669a waiting on condition [0x00007f9f3326c000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000006c8e45a08> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1067)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1127)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
```

调用量太低，线程组处于闲置