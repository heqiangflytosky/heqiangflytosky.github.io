---
title: Java 基础 -- 集合的使用
categories:  Java
comments: true
tags: [String]
description: 简单介绍集合的使用
date: 2013-6-6 10:00:00
---

## 概述

Java 开发过程中，集合类是我们经常要接触和使用的类，从本文开始，将用一系列的文章来对这个知识点进行梳理和总结。
本文先对 Java 集合相关知识做一个概括性的描述，后面会有一系列的文章再细分介绍集合框架。
下面先来看一下 Java 集合框架的类图。

![效果图](http://www.plantuml.com/plantuml/svg/XLF1QXH14BttLmov9r_W8M8EAb6G3Ngkp9RQRE_CsbqRM3Gd8NZqe8W416-Uz21upD_igdzXQgh7ggQpsKEoyrxLgtbLl3bC196My-1EF0LizXx26JH_Bo_MxzzjlbvillnQNrxjETV17DC2Mcn-V_-q-VcnUUtQ02arTsE4LShWNXy2jkJxBhbpAzhy-F3dwktwyxUiuzRv0YAS1Ym9wcCfVGIBQv7l2EFpZ0sQz95mMdAFuSo_uCbCmscY22qrsMw6DvIE_E0SuYexkW_f9VS9UeOmAvbZtyrmMdACo6prO8eTULgDNPa-YOW3vEtkl3auA8pfCnVtzxEJkb2OabyfbDxqBMfFjsr4eINk5awlAqzQiJUkROthsIun89LQKXtAPi3aoaJ1qeOyzebiGD2Y9qjSei57-ChYl72XqLMPtX8dTRAjWsrjgzqoM0nlqx7xZZo4ORg984_Kf2dIM2Zt55lgOmOJWdPcFssUN8ZbLKut7WENg2Bv-ZKfErdQVW3AkvgIz9L7vNzDdc5v2rHMtsttQtYBwp_x3Y67JQHNahHprA0GdOxJmWIpLPF4_qzvL1b8OqU_uniBOnAiN3oC5PQgWp8hzJN7uDGLavpwvDfcyYIZX9O8CYtkJDjmv8frv-uGk-bo7lu1)

整个集合框架就围绕一组标准接口而设计。你可以直接使用这些接口的标准实现，诸如： `LinkedList`、`HashSet`和 `TreeSet` 等,除此之外你也可以通过这些接口实现自己的集合，比如通过实现 `List` 接口来实现线程安全的 `CopyOnWriteArrayList` 类等。
上面图中的类都在 java.util.* 包下面，本文也只介绍这个包下面的类，其他比如 java.util.concurrent.* 包的类将放在另外的文章中介绍。

## 集合类型

Java 集合框架的容器类可以主要分为两种外加一种算法类：

 - 集合（Collection）：存储一个元素集合。
 - 图（Map）：存储键/值对映射。
 - 算法类：Arrays、Collections。它们实现集合接口的对象里的方法执行的一些有用的计算，例如：搜索和排序。

从另外一个维度分，又可以分为：

 - 接口：Map、SortedMap、Collection、List、Queue、Set
 - 抽象类：AbstractMap、AbstrCollection、AbstrList等
 - 具体类：HashMap、TreeMap、Vector、ArrayList等
 - 算法类：Arrays、Collections


## 特点介绍

ArrayList适合于进行大量的随机访问的情况下使用，LinkedList适合在表中进行插入、删除时使用，二者都是非线程安全。

Vector 是线程安全的，它是 ArrayList 的多线程版本。
Vector 内部是通过使用 `synchronized` 方法来保证线程安全的，相对于 `CopyOnWriteArrayList` 来说性能较低。
HashTable 是线程安全的，它是 HashMap 的多线程版本。`HashTable` 内部是通过使用 `synchronized` 方法来保证线程安全的，但在线程竞争激烈的情况下它的效率就比较低。这时可以考虑使用 `ConcurrentHashMap`，这个后面会介绍。
HashTable 即不支持 null key也不支持null value。HashMap 中null可以作为key，这样的key只有一个，可以有一个或者多个value所对应的key为null。
HashMap 非线程安全，底层主要是基于数组和链表来实现的，它之所以有相当快的查询速度主要是因为它是通过计算散列码来决定存储的位置。适用于在Map中插入、删除和定位元素。

Colleciton 用于存放多个单对象，Map 用于存放 Key-Value 形式的键值对。

Set 和 List 都继承自 Collection 接口。它们最大的最大区别就是 Set 中的元素不可以重复，List 中允许重复元素。
Map中不允许出现重复的键（Key）。

WeakHashMap，直接使用 HashMap 有时候会带来内存溢出的风险，使用 WaekHashMap 实例化 Map。当使用者不再有对象引用的时候，WeakHashMap 将自动被移除对应 Key 值的对象。

TreeMap 非线程安全，底层通过红黑树（Red-Black tree）实现，也就意味着 `containsKey()`，`get()`, `put()`, `remove()` 都有着 log(n) 的时间复杂度。TreeMap 是自动实现排序的，按照key的字典顺序来排序（升序）或者自定义的 Comparator 接口来排序。在需要使用排序的 Map 时推荐使用 TreeMap。

Deque 是双端队列，你可以从任意一端插入或者抽取元素。

## 迭代器

 - iterator() 是 Collection 接口的方法；listIterator() 是 List 接口的方法；
 - listIterator 有add()方法，可以向List中添加对象；而 iterator 不能；
 - listIterator 和 iterator 都有hasNext()和next()方法，可以实现顺序向后遍历；但是 listIterator 有hasPrevious()和previous()方法，可以实现逆向（顺序向前）遍历；iterator 就不可以；
 - listIterator 可以定位当前的索引位置，nextIndex()和previousIndex()可以实现；iterator 没有此功能；
 - 都可实现删除对象，但是 listIterator 可以实现对象的修改，set()方法可以实现；iterator 仅能遍历，不能修改；

## 参考文章

http://www.runoob.com/java/java-collections.html
http://www.runoob.com/java/java-vector-class.html


<!-- plantuml 代码 -->

<!--  

@startuml
Title "Java 集合框架图"

namespace 算法 {
class Arrays
class Collections
}

namespace 比较器 {
class Comparable
class Comparator
}

namespace Map {
interface Map
interface SortedMap
interface NavigableMap
abstract class AbstractMap
abstract class Dictionary
class HashMap
class WeakHashMap
class LinkedHashMap
class Hashtable
class IdentityHashMap

class TreeMap

Map <|.. AbstractMap
AbstractMap <|-- HashMap
AbstractMap <|-- WeakHashMap
HashMap <|-- LinkedHashMap
Map <|.. Hashtable
Dictionary  <|-- Hashtable
Map  <|-- SortedMap
SortedMap  <|-- NavigableMap
AbstractMap <|-- TreeMap
NavigableMap <|.. TreeMap
AbstractMap <|-- IdentityHashMap
}

namespace Collection {
interface Collection
interface List
interface Set
interface Queue
interface Deque
interface SortedSet
interface NavigableSet
abstract class AbstractCollection
abstract class AbstractList
abstract class AbstractSet
abstract class AbstractQueue
abstract class AbstractSequentialList

class HashSet
class TreeSet
class LinkedHashSet
class Vector
class Stack
class ArrayList
class LinkedList


Collection <|-- List
Collection <|--  Set
Collection <|--  Queue
Collection <|.. AbstractCollection
AbstractCollection  <|-- AbstractList
List  <|.. AbstractList
AbstractList <|-- AbstractSequentialList
AbstractCollection  <|-- AbstractSet
Set <|.. AbstractSet
Queue <|-- Deque
AbstractCollection  <|-- AbstractQueue
Queue <|.. AbstractQueue
Set <|--  SortedSet
SortedSet  <|--  NavigableSet
AbstractSet <|-- HashSet
AbstractSet <|-- TreeSet
NavigableSet <|.. TreeSet
HashSet <|--  LinkedHashSet
AbstractList <|--  Vector
AbstractList <|--  ArrayList
AbstractSequentialList <|-- LinkedList
Vector <|-- Stack
List <|.. LinkedList
Deque <|.. LinkedList

}
@enduml

-->
