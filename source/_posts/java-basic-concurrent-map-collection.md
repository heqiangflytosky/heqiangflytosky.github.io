---
title: Java 基础 -- 并发集合的使用
categories:  Java
comments: true
tags: [并发集合]
description: 并发包中的集合的使用
date: 2015-10-18 10:00:00
---

## 概述

前面大多数集合并不适合用在并发应用程序中，因为它们没有控制并发访问数据。如果一些并发任务共享一个数据结构，而这个数据结构并不适合用在并发任务中，你将会有数据不一致的错误，这将影响到程序的正确运行。比如 ArrayList 类。
为了方便开发者并发编程，在 JDK 5.0 中加入了并发包 java.util.concurrent.*，它提供了可以在并发程序中使用的类，本文仅介绍并发结合的特点和使用方法。

![效果图](http://www.plantuml.com/plantuml/svg/SoWkIImgAStDuGh9BCb9LL3oIYnBL7YwSzlJ_ec-YGKlPxSzdT3nR67RitdRdixUfyJ5bPbNabgKbfYSgW2KHk85vnULfAQWYlabbcMc9oRbfA8AE-Vd9PSM5QNcbU0IHz6Oc5HSKfIONAAGd9DONApW2EM2f20Y27qUYSKPsCI3ipCBV3ABmNf004WhsDJewI4R1XO4YkhgeZaGIGr46jC-50qA4ACnLI4iG0GMd0MHGF4uWeDfCuf2X31-bPX-mH56s7KZ_8Mfmo4rBmMOYW00)

![效果图](http://www.plantuml.com/plantuml/svg/XLJBgjim4DtxAyJTt_i5NIHDLYNqWGDTAyua5L6aL9O3Xgv3TstNWK9_8j1D_yQ1_iN5FfXed37P6C-SCsSU6cbIHXrYjRVPqaHBwkcTFchL_BqqFtxDlBLKHEFT__Ef-Vdz-kTq_Nrfpl-UiiouI66Z2r8tLkBCAQQM3v7MtFTU7yrMhonnzyAKgVcfeeeI-wtrnSwuxT0_fwyYR-XV1ktA3GN4wrKPWowYAgmkIpM56PEEHXWvnJ48ol3Dl2kg4CZ5V6SERMjjO8yTwW_kSp2HfY7ekaL7e5jGP_QzoPaMnkreZRw1pSdgEYM3maSa8VSnV7n0yjeLNu9tlYfJzyTWV31nK6rscwrwy-tbHPWvfpm_TxkKK-Jytjaoalt2Nm_m9RjFfS1ADPD_WF8Pfi8eJEODUi5wjny42Kl0Iq3Ka2OfOxxT5mu-U010SEBTbrC1i429C3f8VWcG4uroW6S0Gzfj5eKK0WsTG4Mu1sB149FPFAlC73Oe4yG6akl8se0Eq95WyUOo2o8IxQ7QM4cFQL3ZRyDeEavZbahpFPJZxuvSEkV8PokzLBAQJ9oGMrTx-me0)

## 并发 Map 框架

### ConcurrentHashMap

ConcurrentHashMap 是线程安全且高效的 HashMap，底层采用分段的数组+链表实现，线程安全ConcurrentHashMap允许多个修改操作并发进行，其关键在于使用了锁分离技术。它使用了多个锁来控制对hash表的不同部分进行的修改。ConcurrentHashMap内部使用段(Segment)来表示这些不同的部分，每个段其实就是一个小的Hashtable，它们有自己的锁。只要多个修改操作发生在不同的段上，它们就可以并发进行。

JDK1.8的实现已经摒弃了Segment的概念，而是直接用Node数组+链表+红黑树的数据结构来实现，此时锁加在key上，并发控制使用Synchronized和CAS来操作，整个看起来就像是优化过且线程安全的HashMap，虽然在JDK1.8中还能看到Segment的数据结构，但是已经简化了属性，只是为了兼容旧版本。

### ConcurrentSkipListMap

`ConcurrentSkipListMap` 的 key 是有序的，底层是通过跳表来实现的。

## 并发 Collection 框架

### BlockingQueue

BlockingQueue 就是阻塞队列，它是一个先进先出的队列，当获取队列元素但是队列为空时，会阻塞等待队列中有元素再返回。当添加元素时，如果队列已满，那么等到队列可以放入新元素时再放入。
BlockingQueue 的实现都是线程安全的。
BlockingQueue 通常用于实现生产者-消费者队列的场景，当然，你也可以将它当做普通的 Collection 来用。

#### ArrayBlockingQueue

数组阻塞队列，它实现了 BlockingQueue 接口。其内部实现是将对象放到一个数组里。

#### DelayQueue

延迟队列，它实现了 BlockingQueue 接口。只有在延迟期满时才能从中提取元素。该队列的头部是延迟期满后保存时间最长的Delayed 元素。

#### LinkedBlockingQueue

链阻塞队列，内部以一个链式结构(链接节点)对其元素进行存储。

#### PriorityBlockingQueue

具有优先级的阻塞队列，所有插入到 PriorityBlockingQueue 的元素必须实现 java.lang.Comparable 接口。因此该队列中元素的排序就取决于你自己的 Comparable 实现。
基于数组，数据结构为二叉堆.

#### SynchronousQueue

它的内部同时只能够容纳单个元素。如果该队列已有一元素的话，试图向队列中插入一个新元素的线程将会阻塞，直到另一个线程将该元素从队列中抽走。同样，如果该队列为空，试图向队列中抽取一个元素的线程将会阻塞，直到另一个线程向队列中插入了一条新的元素。

#### BlockingDeque

阻塞双端队列，在不能够插入元素时，它将阻塞住试图插入元素的线程；在不能够抽取元素时，它将阻塞住试图抽取的线程。

### CopyOnWriteArrayList

`CopyOnWriteArrayList` 可以说是 `ArrayList` 的多线程版本，它是线程安全的。
它使用了一种叫写时复制的方法，当有新元素添加到 `CopyOnWriteArrayList` 时，先从原有的数组中拷贝一份出来，然后在新的数组做写操作，写完之后，再将原来的数组引用指向到新数组。

<!-- 
http://www.importnew.com/26461.html
https://www.cnblogs.com/cccw/p/5837448.html
https://blog.csdn.net/king866/article/details/53945400
https://www.cnblogs.com/vijozsoft/p/5585620.html
http://ifeve.com/concurrent-collections-1/
https://blog.csdn.net/ink4t/article/details/76696728
https://www.cnblogs.com/bushi/p/6681543.html
https://www.baidu.com/s?word=CopyOnWriteArrayList&tn=50000021_hao_pg&ie=utf-8&sc=UWd1pgw-pA7EnHc1FMfqnHR1rHb1rj0sPjDknBuW5y99U1Dznzu9m1YknHc3PjTdn6&ssl_sample=s_10%2Cs_77&srcqid=2489131420700060184
https://www.jianshu.com/p/5f570d2f81a2
http://www.runoob.com/java/java-collections.html
http://www.runoob.com/java/java-hashTable-class.html
-->


<!-- 
@startuml
Title "Java 并发Map集合框架图"

interface Map
interface SortedMap
interface NavigableMap
interface ConcurrentMap
interface ConcurrentNavigableMap
abstract class AbstractMap
class ConcurrentHashMap
class ConcurrentSkipListMap




Map <|.. AbstractMap
Map  <|-- ConcurrentMap
AbstractMap <|-- ConcurrentHashMap
ConcurrentMap  <|.. ConcurrentHashMap
Map  <|-- SortedMap
SortedMap <|-- NavigableMap
NavigableMap <|-- ConcurrentNavigableMap
ConcurrentMap <|-- ConcurrentNavigableMap
AbstractMap <|-- ConcurrentSkipListMap
ConcurrentNavigableMap <|.. ConcurrentSkipListMap
@enduml
-->

<!-- 

@startuml
Title "Java 并发Collection集合框架图"

interface Collection
interface Set
interface SortedSet
interface NavigableSet
interface List
interface Queue
interface BlockingQueue
interface Deque
interface BlockingDeque
interface TransferQueue
abstract class AbstractCollection
abstract class AbstractSet
abstract class AbstractQueue
class ConcurrentSkipListSet
class CopyOnWriteArrayList
class CopyOnWriteArraySet
class ArrayBlockingQueue
class ConcurrentLinkedDeque
class DelayQueue
class LinkedBlockingDeque
class LinkedBlockingQueue
class LinkedTransferQueue
class SynchronousQueue
class PriorityBlockingQueue

Collection <|.. AbstractCollection
Collection <|-- Set
Set <|.. AbstractSet
Set <|-- SortedSet
SortedSet <|-- NavigableSet
AbstractCollection  <|-- AbstractSet
AbstractSet <|-- ConcurrentSkipListSet
NavigableSet <|.. ConcurrentSkipListSet
Collection <|-- List
List <|.. CopyOnWriteArrayList
AbstractSet  <|-- CopyOnWriteArraySet
Collection <|-- Queue
Queue <|-- BlockingQueue
Queue  <|-- Deque
Deque <|-- BlockingDeque
BlockingQueue <|-- BlockingDeque
AbstractCollection  <|--  AbstractQueue
Queue <|.. AbstractQueue
BlockingQueue  <|..  ArrayBlockingQueue
AbstractQueue <|--  ArrayBlockingQueue
Deque <|.. ConcurrentLinkedDeque
AbstractCollection <|-- ConcurrentLinkedDeque
AbstractQueue  <|-- ConcurrentLinkedQueue
Queue <|.. ConcurrentLinkedQueue
AbstractQueue <|--  DelayQueue
BlockingQueue <|.. DelayQueue
AbstractQueue <|-- LinkedBlockingDeque
BlockingDeque <|.. LinkedBlockingDeque
AbstractQueue <|-- LinkedBlockingQueue
BlockingQueue <|.. LinkedBlockingQueue
BlockingQueue <|-- TransferQueue
AbstractQueue <|-- LinkedTransferQueue
TransferQueue  <|.. LinkedTransferQueue
AbstractQueue <|-- SynchronousQueue
BlockingQueue <|.. SynchronousQueue
AbstractQueue  <|-- PriorityBlockingQueue
BlockingQueue <|.. PriorityBlockingQueue
@enduml
-->