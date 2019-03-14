---
title: Java 并发编程 -- Lock
categories: Java 并发
comments: true
tags: [Java 并发]
description: 介绍Lock接口
date: 2015-7-15 10:00:00
---

## 概述

锁是用来控制多个线程访问共享资源的方式，一般来说，一个锁能够防止多个线程同时访问共享资源（有些锁可以允许多个线程并发的访问共享资源，比如读写锁）。在 Lock 接口出现之前，Java 程序是靠 synchronized 关键字实现锁功能的，而 Java SE 5 之后，并发包中新增了 Lock 接口（以及相关实现类）用来实现锁功能，它提供了与 synchronized 关键字类似的同步功能，只是在使用时需要显式地获取和释放锁。虽然它缺少了（通过synchronized块或者方法所提供的）隐式获取释放锁的便捷性，但是却拥有了锁获取与释放的便捷性、可中断的获取锁以及超时获取锁等多种 synchronized 关键字所不具备的同步特性。

## 和 synchronized 区别

 - 尝试非阻塞地获取锁：当前线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，则成功获取并持有锁，如果被其他线程持有，则立即返回false。
 - 能被中断地获取锁：与synchronized 不同，获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常将会被抛出，同时锁会被释放。
 - 超时获取锁：在指定的截止时间之前获取锁，如果截止时间到了仍旧无法获取锁，则返回

## Lock 接口

先来看一下 Lock 接口的类关系图：

![效果图](http://poc98qeya.bkt.clouddn.com/images/main/java-concurrency-interface-lock/interface-lock.png)

Lock 接口的使用也很简单：

```
        Lock lock = new ReentrantLock();
        lock.lock();
        try {
            
        } finally {
            lock.unlock();
        }
```

在finally块中释放锁，目的是保证在获取到锁之后，最终能够被释放。
另外，不要讲获取锁的过程写在try块中，因为如果在获取锁时发生了异常，异常抛出的同时，也会导致锁无故释放。

Lock 是一个接口，它定义了锁获取和释放的基本操作，Lock的API如下：

 - lock()：获取锁，调用该方法后当前线程会获取锁，并返回
 - lockInterruptibly()：可中断地获取锁，该方法可以相应中断，在锁的获取中可以中断当前线程
 - tryLock()：尝试非阻塞地获取锁，调用该方法会立刻返回，成功获取锁返回true，否则返回false
 - tryLock(long time, TimeUnit unit)：超时地获取锁
 - unlock()：释放锁
 - newCondition()：获取等待通知组件，该组件和当前的锁绑定，当前线程只有获得了锁，才能调用该组件的wait方法，而调用后，当前线程释放锁

<!--
@startuml
Title "Lock关系图"
interface Lock {
  ~ package void lock()
  ~ package void lockInterruptibly()
  ~ package boolean tryLock()
  ~ package boolean tryLock(long time, TimeUnit unit)
  ~ package void unlock()
  ~ package Condition newCondition()
}

interface ReadWriteLock {
  ~ package Lock readLock()
  ~ package Lock writeLock()
}

abstract class AbstractOwnableSynchronizer
abstract class AbstractQueuedSynchronizer
abstract class Sync
class NonfairSync
class FairSync

Lock <|.. ReentrantLock
ReadWriteLock <|.. ReentrantReadWriteLock
Lock <|.. ReadLock
Lock <|.. WriteLock
ReadLock <-- ReentrantReadWriteLock
WriteLock <-- ReentrantReadWriteLock

AbstractOwnableSynchronizer <|-- AbstractQueuedSynchronizer
AbstractQueuedSynchronizer <|-- Sync
Sync <|-- NonfairSync
Sync <|-- FairSync
NonfairSync <-- ReentrantLock
FairSync <-- ReentrantLock
NonfairSync <-- ReentrantReadWriteLock
FairSync <-- ReentrantReadWriteLock
@enduml
-->

<!-- 
http://www.plantuml.com/plantuml/png/ZP3FJiCm3CRlUGfh9v3Ode33458b90HY375EMsz4ovmfTPZ67suy1O-n8n9lC_4Q92lTkYejNApuztrsR0yBbfRTN8knOetGkpJPRFE-_bv_RZw-Ua8Hevt83248y2m0tc0XivcS8ZmQbOFs_EWupYz2jNKBLgbUDKofCHeb0TjLQFs7gWrDWTKSJs3iunqf1kT3v6D7aP7E3UMAbI4WNEuIRteLjHr7AFDxgnWZoswHzOPgsgQsh0hBhZ8jsCgC8TEoAE3iDxrUaamrtgueUx26r1FQDkkDGuTvbpDeednU6Pf8PMiagLAn7U_qPJ204IAnbSG1YTsw4SE1LkiGU0FjRK4iUR_VrgfwTPf4nxdyfxwmuqXnQLyQY0YXJAlB7TAaGZKvJDmuOT8knGeZgoR_y0oHSZVNFm00
-->