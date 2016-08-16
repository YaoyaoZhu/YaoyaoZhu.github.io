---
layout: post
title: 并发(Concurrent)
categories: [blog]
tags: [Java]
description: 
---

#并发(Concurrent)
##乐观锁与悲观锁
乐观锁不锁住竞争资源，而是假设无锁，不断的去尝试访问，直到访问成功。
悲观锁是锁住竞争资源，只有获得锁的线程可以访问，其他的都挂起。
乐观锁有CAS算法，譬如Atomic包下的并发，通过调用C最后通过CPU实现CAS，指令级的消耗小；悲观锁有Synchronized，jvm会将没有获得锁的线程挂起，context切换大；轻度中度使用中建议使用非阻塞算法CAS。

