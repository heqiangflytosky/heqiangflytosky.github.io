---
title: Java 基础 -- ArrayList 的使用和原理
categories: Java
comments: true
tags: [Java]
description: 介绍 ArrayList 的使用和原理
date: 2013-6-20 10:00:00
---

## 概述

ArrayList 实现了 List 接口，是我们经常使用的一个数据集合，我们通常把它当成一个可变长度的树组使用。长度可变是说我们可以在创建 ArrayList 时不指定它的长度，添加数据时只需要添加就行了，ArrayList 会动态的扩充容量。
ArrayList 内部是通过一个对象数组来存储数据元素的：

```
transient Object[] elementData;
```

这种底层存储数据结构决定了ArrayList的添加删除操作效率是不高的，而随机访问效率很高。

## 数组构造


```
    public ArrayList(int initialCapacity) {
        super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        // 根据指定的容量大小创建树组
        this.elementData = new Object[initialCapacity];
    }

    public ArrayList() {
        super();
        // 将空的树组赋值给 elementData
        this.elementData = EMPTY_ELEMENTDATA;
    }


    // 通过给定集合创建树组
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        size = elementData.length;
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    }
```

## 元素添加和扩容


```
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
```

在真正添加数据前，首先要检查一下当前数组是否可以容纳数据的添加，如果容量不够，就要进行扩容。

```
    private void ensureCapacityInternal(int minCapacity) {
        // 如果是调用的没有指定初始容量的构造方法
        // 那么就把容量设置为 DEFAULT_CAPACITY = 10
        if (elementData == EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        // modCount 记录了集合的修改次数，是父类 AbstractList 的成员变量
        modCount++;

        // 如果需要的最小空间大于当前树组的长度，那么就需要扩容了
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    private void grow(int minCapacity) {
        // 原数组的长度
        int oldCapacity = elementData.length;
        // 扩展到原来的 1.5 倍大小
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果扩充到1.5倍还是不够，那么就直接扩充到要求的 minCapacity 大小
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 如果比 MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8 还要大
        // 则设置为 Integer.MAX_VALUE
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // 以 newCapacity 大小创建一个新的树组，并将数组复制到新的数组中
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
```

这里的 ensureCapacityInternal 方法只有在添加数组时才会被调用的，因此，在构造 ArrayList 时你没有制定容量大小，它不会立即把树组大小设置为10，除非有添加操作时才会进行设置。

## 是否包含元素

```
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

可见，ArrayList 判断是否包含元素采用的时遍历树组的方法，使用 equals 来判断。
所以我们要重写元素的 equals 方法。
而且，这里的遍历方式为顺序遍历，如果数据量很大，那么就会有很大的性能问题。

## 删除元素

```
    public E remove(int index) {
        // 边界检查
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
        // 记录修改次数
        modCount++;
        // 获取当前位置的元素
        E oldValue = (E) elementData[index];

        int numMoved = size - index - 1;
        // 执行数据复制，覆盖需要删除位置的元素
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // 置空便于GC回收

        return oldValue;
    }
```

```
    // 元素删除操作也是顺序遍历数据进行
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    // fastRemove 仅仅是少了边界检查
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```

```
    // 删除相同元素
    public boolean removeAll(Collection<?> c) {
        return batchRemove(c, false);
    }
    // 删除不同元素，求交集
    public boolean retainAll(Collection<?> c) {
        return batchRemove(c, true);
    }

    private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                // 这里通过contains来判断，如果列表很大，效率不高
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                // clear to let GC do its work
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```

## 迭代器

```
    public Iterator<E> iterator() {
        return new Itr();
    }
    
    public ListIterator<E> listIterator() {
        return new ListItr(0);
    }
    
    
```

```
    private class Itr implements Iterator<E> {
        protected int limit = ArrayList.this.size;

        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor < limit;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            int i = cursor;
            if (i >= limit)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                // 对 expectedModCount 重新赋值
                expectedModCount = modCount;
                limit--;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
        ......
    }
```

这里主要说明一下迭代器的 expectedModCount 变量，它被赋予初始值 modCount，也就是 ArrayList 记录修改次数的变量。在 next() 和 remove() 都回去检查 modCount 和 expectedModCount 变量是否相等，否则就抛出 ConcurrentModificationException 异常。
对于这个异常我们可能并不陌生。比如下面的用法就会抛出这个异常：

```
            Iterator iterator = list.iterator();
            while(iterator.hasNext()){
                String str = (String) iterator.next();
                list.remove(str);
            }
```

在上面的用法中，在使用迭代器遍历集合的时候同时调用集合的方法修改集合元素。这回导致 `modCount != expectedModCount` 不相等，从而抛出异常。
那么正确的做法应该要怎么样呢？

```
            Iterator iterator = list.iterator();
            while(iterator.hasNext()){
                String str = (String) iterator.next();
                iterator.remove(str);
            }
```

因为在迭代器的 remove 方法中，在调用完 ArrayList 的 remove 方法后，会重新对 expectedModCount 赋值，从而保持 expectedModCount == modCount。