---
title: Java 基础 -- LinkedHashMap 的使用和原理
categories: Java
comments: true
tags: [Java]
description: 介绍 LinkedHashMap 的使用和原理
date: 2013-6-30 10:00:00
---

## 概述

前面我们有介绍 HashMap 的使用，HashMap 也是我们经常会使用的集合，本文我们再介绍一下 LinkedHashMap 的使用和原理。

## 使用方法

LinkedHashMap 一个特点就是有序的存储键值对，下面就通过一个例子来验证一下。

```
        Map<String, String> linkedHashMap = new LinkedHashMap<>();
        linkedHashMap.put("key1","value1");
        linkedHashMap.put("key2","value2");
        linkedHashMap.put("key3","value3");

        Set<Map.Entry<String, String>> set = linkedHashMap.entrySet();
        Iterator<Map.Entry<String, String>> iterator = set.iterator();
        while (iterator.hasNext()){
            Map.Entry entry = iterator.next();
            Log.e("Test","key = "+entry.getKey()+", value = "+entry.getValue());
        }
        
        // 下面的代码主要是和访问排序做对比
        linkedHashMap.get("key2");
        
        Iterator<Map.Entry<String, String>> iterator1 = set.iterator();
        while (iterator1.hasNext()){
            Map.Entry entry = iterator1.next();
            Log.e("Test","key = "+entry.getKey()+", value = "+entry.getValue());
        }
```

```
Test: key = key1, value = value1
Test: key = key2, value = value2
Test: key = key3, value = value3

Test: key = key1, value = value1
Test: key = key2, value = value2
Test: key = key3, value = value3
```

可以看到，输出是有序的键值对，而且，是默认是按照插入顺序排序的。
其实，还有一种排序规则是按照访问顺序排序的，也就是，每次访问LinkedHashMap都会改变它的数据顺序，把当前访问的数据排到尾部。这个规则是 LruCache 的实现基础。

把上面的代码的 LinkedHashMap 构造方式改为如下：

```
Map<String, String> linkedHashMap = new LinkedHashMap<>(0, 0.75f, true);
......

```

```
Test: key = key1, value = value1
Test: key = key2, value = value2
Test: key = key3, value = value3

Test: key = key1, value = value1
Test: key = key3, value = value3
Test: key = key2, value = value2
```

通过打印结果可以看到，对 key2 的一次访问操作会把该记录移到尾部。

## 代码分析

### 构造方法

```
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
```

LinkedHashMap 继承了 HashMap 并实现了 Map 接口，所以 LinkedHashMap 可以完全代替 HashMap 的功能。
下面来看一下它的几个构造方法：

```
    public LinkedHashMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
        accessOrder = false;
    }

    public LinkedHashMap(int initialCapacity) {
        super(initialCapacity);
        accessOrder = false;
    }

    public LinkedHashMap() {
        super();
        accessOrder = false;
    }

    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super(m);
        accessOrder = false;
    }

    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```

这里的 accessOrder 就是上面提到的决定了 LinkedHashMap 的存储顺序。accessOrder 为 false 时是按照插入顺序排序的，为 true 时是按照 访问顺序排序的。
另外，LinkedHashMap 里面还有个 header 的变量：

```
private transient LinkedHashMapEntry<K,V> header;
```

它返回的是 LinkedHashMap 里的双向链表的头节点，LinkedHashMap 正是通过这个链表还实现有序的存储。

## LinkedHashMap

```
    private static class LinkedHashMapEntry<K,V> extends HashMapEntry<K,V> {
        LinkedHashMapEntry<K,V> before, after;

        LinkedHashMapEntry(int hash, K key, V value, HashMapEntry<K,V> next) {
            super(hash, key, value, next);
        }

        private void remove() {
            before.after = after;
            after.before = before;
        }

        private void addBefore(LinkedHashMapEntry<K,V> existingEntry) {
            after  = existingEntry;
            before = existingEntry.before;
            before.after = this;
            after.before = this;
        }

        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            if (lm.accessOrder) {
                lm.modCount++;
                remove();
                addBefore(lm.header);
            }
        }

        void recordRemoval(HashMap<K,V> m) {
            remove();
        }
    }
```

LinkedHashMapEntry 继承 HashMapEntry ，添加了 before 和 after 属性来构建一个双向的链表。

## 方法介绍

### init 方法

init 方法在 HashMap 的构造方法里面调用，在 LinkedHashMap 里面重写了这个方法。

```
    @Override
    void init() {
        header = new LinkedHashMapEntry<>(-1, null, null, null);
        header.before = header.after = header;
    }
```

这里初始化了一个只有头结点的双向链表。

### get 方法

LinkedHashMap 重写了 HashMap 的 get 方法：

```
    public V get(Object key) {
        // 调用 HashMap 的getEntry方法，获取一个数据。
        LinkedHashMapEntry<K,V> e = (LinkedHashMapEntry<K,V>)getEntry(key);
        if (e == null)
            return null;
        // 如果需要就改变排列顺序
        e.recordAccess(this);
        return e.value;
    }
    
        void recordAccess(HashMap<K,V> m) {
            LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
            // 如果是使用了访问顺序排序
            if (lm.accessOrder) {
                lm.modCount++;
                // 移除这个节点
                remove();
                // 添加到头结点的前面变为头结点
                addBefore(lm.header);
            }
        }
```

### 遍历数据

介绍遍历数据就要介绍一下 LinkedHashIterator：

```
    private abstract class LinkedHashIterator<T> implements Iterator<T> {
        LinkedHashMapEntry<K,V> nextEntry    = header.after;
        LinkedHashMapEntry<K,V> lastReturned = null;

        int expectedModCount = modCount;

        public boolean hasNext() {
            return nextEntry != header;
        }

        public void remove() {
            if (lastReturned == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            LinkedHashMap.this.remove(lastReturned.key);
            lastReturned = null;
            expectedModCount = modCount;
        }

        Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (nextEntry == header)
                throw new NoSuchElementException();

            LinkedHashMapEntry<K,V> e = lastReturned = nextEntry;
            nextEntry = e.after;
            return e;
        }
    }
```

可以看到这里其实就是遍历双向链表。

## LinkedHashMap 的应用

在 Android 中提供了一个 LruCache 的缓存框架，是用 LinkedHashMap 来实现的，这个后面再介绍。

## 总结

 - LinkedHashMap 是继承于 HashMap，是基于 HashMap 和双向链表来实现的。
 - HashMap 的数据是无序的，LinkedHashMap 是有序的。
 - LinkedHashMap 相对于 LinkedHashMap 存储数据的方式并没有变，只是多了一个双向链表来维持顺序。
 - LinkedHashMap也是线程不安全的。
