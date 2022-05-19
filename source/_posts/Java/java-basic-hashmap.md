---
title: Java 基础 -- HashMap 的使用和原理
categories: Java
comments: true
tags: [Java]
description: 介绍 HashMap 的使用和原理
date: 2013-6-28 10:00:00
---

本文基于 JDK 1.7 的源码进行分析。

## 概述

HashMap 是我们经常使用的容器，它是基于哈希表的 Map 实现。HashMap 允许存在 Null 键和 Null 值，它是非线程安全的。此类并不保证映射的顺序，特别是它不保证顺序恒久不变。
在 JDK 1.7 中，HashMap 的内部是由数组+链表来实现的。数组是 HashMap 的主体，链表则是为了解决哈希冲突而存在的。也就是说 HashMap 中解决哈希冲突采用了链地址法。

## HashMap 原理

### HashMap 的构造方法

HashMap 有四个构造方法：

 - `HashMap(int initialCapacity, float loadFactor)`
 - `HashMap(int initialCapacity)`
 - `HashMap()`
 - `HashMap(Map<? extends K, ? extends V> m)`

这里介绍一下两个重要的参数：initialCapacity 初始容量和 loadFactor 加载因子。
容量就是哈希表中桶的数量，初始容量就是哈希表在创建时的容量，记载因子就是哈希表在其容量自动增加之前可以达到多满的一种尺度。
当哈希表中的条目数超出了加载因子与当前容量的乘积时，则要对该哈希表进行 rehash 操作（即重建内部数据结构），从而哈希表将具有大约两倍的桶数。
在 HashMap 中，默认的 DEFAULT_INITIAL_CAPACITY = 16，DEFAULT_LOAD_FACTOR = 0.75f。
HashMap 的阈值也就是可以容纳的数据量为：

```
threshold = (int)Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
```

### HashMap 数据存储

HashMap 中的数据是通过 `Entry<K,V>[] table` 树组来存储的，它的基本数据单元是 Entry：

```
    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;

        /**
         * Creates new entry.
         */
        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;
            key = k;
            hash = h;
        }

        ......
}
```

上面也提到过，HashMap 是由数组+链表来实现数据存储，数组中保存的是链表的头部元素。
可以用下面的图来直观一点表示：

![效果图](/images/java-basic-hashmap/hashmap-hash-array.png)

### HashMap put 方法

put 方法的返回值为 HashMap 中已有的该 key 值对应的值，如果没有相同的key，返回 null。

```
    public V put(K key, V value) {
        if (key == null)
            return putForNullKey(value);
        // 根据 key 获得哈希值
        int hash = hash(key);
        // 根据哈希值得到在数组中的存储位置
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            //如果该对应数据已存在，执行覆盖操作。用新value替换旧value，并返回旧value
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                // 返回旧 value
                return oldValue;
            }
        }

        modCount++;
        // 添加新的 Entry
        addEntry(hash, key, value, i);
        // 返回null
        return null;
    }

    static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }
```

这里涉及下面几个操作：

 - 根据key求哈希值；
 - 根据哈希值得到index，也就是在数组中的存储位置
 - 添加新的 Entry

```
    void addEntry(int hash, K key, V value, int bucketIndex) {
        // 如果数据量大于HashMap阈值，则需要对 HashMap 进行扩容
        if ((size >= threshold) && (null != table[bucketIndex])) {
            // 将容量变为原来数组的两倍
            resize(2 * table.length);
            // 求哈希值
            hash = (null != key) ? hash(key) : 0;
            // 得到数组中存储位置
            bucketIndex = indexFor(hash, table.length);
        }
        // 生成一个 Entry
        createEntry(hash, key, value, bucketIndex);
}

    void createEntry(int hash, K key, V value, int bucketIndex) {
        // 获取数组中链表头部的元素
        Entry<K,V> e = table[bucketIndex];
        // 生成一个新的Entry，并放到链表头部
        // 链表头部的元素保存着 table 数组中
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        // 数据量加1
        size++;
}
```

### 容量扩容

```
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        // 容量已经为最大容量时，赋值Integer.MAX_VALUE
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }
        // 初始化一个新的数组
        Entry[] newTable = new Entry[newCapacity];
        boolean oldAltHashing = useAltHashing;
        useAltHashing |= sun.misc.VM.isBooted() &&
                (newCapacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
        // 是否重新计算哈希值
        boolean rehash = oldAltHashing ^ useAltHashing;
        // 将原来的数据迁移到新的数组
        transfer(newTable, rehash);
        table = newTable;
        // 重新赋值阈值 threshold
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
}
```

### HashMap get 方法

这个过程也比较简单：

 - 根据 key 计算哈希值
 - 根据哈希值获取在数组中的位置
 - 遍历该位置的链表，直到找到该key对应的 Entry，返回 value

```
    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        Entry<K,V> entry = getEntry(key);

        return null == entry ? null : entry.getValue();
    }
    
    final Entry<K,V> getEntry(Object key) {
        int hash = (key == null) ? 0 : hash(key);
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }
```

### HashMap keySet() entrySet() 方法

keySet() 和 entrySet() 实现原理相同，这里我么只介绍一下 entrySet()。

```
    public Set<Map.Entry<K,V>> entrySet() {
        return entrySet0();
    }

    private Set<Map.Entry<K,V>> entrySet0() {
        Set<Map.Entry<K,V>> es = entrySet;
        return es != null ? es : (entrySet = new EntrySet());
    }
```

这里的关键点在 entrySet 变量，但是我们通过看源码发现一个奇怪的事情，代码中对 entrySet 只有读取的操作，并没有发现写入数据的操作。
我们来看一下 EntrySet 的实现：

```
    private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public Iterator<Map.Entry<K,V>> iterator() {
            return newEntryIterator();
        }
        public boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<K,V> e = (Map.Entry<K,V>) o;
            Entry<K,V> candidate = getEntry(e.getKey());
            return candidate != null && candidate.equals(e);
        }
        public boolean remove(Object o) {
            return removeMapping(o) != null;
        }
        public int size() {
            return size;
        }
        public void clear() {
            HashMap.this.clear();
        }
    }
    
    Iterator<Map.Entry<K,V>> newEntryIterator()   {
        return new EntryIterator();
    }
```

找到了问题的重点，EntrySet 类重写了 iterator() 方法，生成了 EntryIterator 对象。

```
    private final class EntryIterator extends HashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() {
            return nextEntry();
        }
    }
    
    private abstract class HashIterator<E> implements Iterator<E> {
        Entry<K,V> next;        // next entry to return
        int expectedModCount;   // For fast-fail
        int index;              // current slot
        Entry<K,V> current;     // current entry

        HashIterator() {
            expectedModCount = modCount;
            if (size > 0) { // advance to first entry
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        final Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();

            if ((next = e.next) == null) {
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
            current = e;
            return e;
        }

        public void remove() {
            if (current == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Object k = current.key;
            current = null;
            HashMap.this.removeEntryForKey(k);
            expectedModCount = modCount;
        }
    }
```

HashIterator 是 HashMap 的一个内部类，实现了 Iterator 接口，对于迭代器的各种操作，都是基于 HashMap 的变量 table 来进行的，这也就是为什么我们发现没有对 EntrySet 写入数据的原因。

## HashMap 的遍历

第一种：获取 keySet 遍历，根据 key 再通过 get 方法获取 value

```
        HashMap<String, String> hashMap = new HashMap();

        Set<String> entries = hashMap.keySet();

        for(String key : entries) {
            String vallue = hashMap.get(key);
        }
```

第二种：获取 entrySet，根据Map.Entry 获取key和value，较第一种方法效率高，一次获取可以得到key和value。

```
        HashMap<String, String> hashMap = new HashMap();

        Set<Map.Entry<String, String>> entries = hashMap.entrySet();

        for(Map.Entry<String, String> entry : entries) {
            String key = entry.getKey();
            String value = entry.getValue();
        }
```

第三种：获取 entrySet 的迭代器，然后进行遍历，其实 foreach 的实现也是基于迭代器 iterator() 的。

```
        Iterator iterator = hashMap.entrySet().iterator();

        while (iterator.hasNext()) {
            Map.Entry<String, String> entry = (Map.Entry) iterator.next();
            String key = entry.getKey();
            String value = entry.getValue();
        }
```
