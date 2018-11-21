---
title: Java 基础 -- 算法类 Arrays 和 Collections
categories:  Java
comments: true
tags: [String]
description: 简单介绍算法类 Arrays 和 Collections 的使用
date: 2013-6-8 10:00:00
---
## 概述

`Arrays` 和 `Collections` 是 Java 提供的工具类，提供了一些算法类的方法，比如：排序、搜索等。它们都在 java.util.* 包下面。`Arrays` 用来操作数组，`Collections` 用来操作集合。
本文就对这两个类做一些介绍。

## Arrays

### 排序

`Arrays` 提供了两种排序的方法：

 - `sort(T[])：所有的对象都必须实现 `Comparable` 接口，它用来确定对象之间的大小关系。
 - `sort(T[], Comparator) `：对象不必实现 `Comparable` 接口，由 `Comparator` 来确定对象之间的大小关系。使用这种方式我们可以自定义排序的比较方式。

另外在 JDK 1.8 以后还提供了并行排序：

 - `public static <T extends Comparable<? super T>> void parallelSort(T[] a)`
 - `public static <T> void parallelSort(T[] a, Comparator<? super T> cmp)`

并行排序 使用 Fork/Join 框架使排序任务可以在线程池中的多个线程中进行。当小于某个临界值时或者分块为1，其实和普通的 sort 方法一样。否则就用的是 Fork/Join。

先来看一个上面两个方法的应用：

```
        int[] nums = {2,6,9,4,5,0,1};
        Arrays.sort(nums);
        for(int i = 0; i < nums.length; i++)
        System.out.println(nums[i] + ",");
```

由于 `Integer` 实现了 `Comparable` 接口，而且 `compareTo` 的实现是按照升序排列：

```
    public int compareTo(Integer anotherInteger) {
        return compare(this.value, anotherInteger.value);
    }
    public static int compare(int x, int y) {
        return (x < y) ? -1 : ((x == y) ? 0 : 1);
    }
```

如果我们想降序排列要怎么做呢？这就需要使用 `Comparator` 来自定义了。

```
    {
        Integer[] nums = {2,6,9,4,5,0,1};
        Arrays.sort(nums, new MyComparator());
        for(int i = 0; i < nums.length; i++)
        System.out.println(nums[i] + ",");
    }

    public static class MyComparator implements Comparator<Integer> {

        @Override
        public int compare(Integer t1, Integer t2) {
            return (t1 < t2) ? 1 : ((t1 == t2) ? 0 : -1);
        }
    }
```

再来介绍一下排序所用的算法。
其实 `Collections` 的排序算法最后调用的也是 `Arrays.sort` 方法。

```
    public static <T> void sort(T[] a, Comparator<? super T> c) {
        if (c == null) {
            sort(a);
        } else {
            if (LegacyMergeSort.userRequested)
                legacyMergeSort(a, c);
            else
                TimSort.sort(a, 0, a.length, c, null, 0, 0);
        }
    }
```

在不指定 `Comparator` 情况下，用的是 `DualPivotQuicksort`（快速排序） 和 `ComparableTimSort`（归并排序升级版，结合了合并排序和插入排序算法）。
在指定 `Comparator` 情况下一般用的是 `TimSort`，`legacyMergeSort` 基本已经废弃不用。`TimSort` 和 `ComparableTimSort` 的区别是 `ComparableTimSort` 需要对象是 `Comparable` 可比较的，不需要特定 `Comparator` ，而 `TimSort` 利用提供的 `Comparator` 进行排序。

### 其他

 - `binarySearch`：搜索，采用二分法查找数组中的数据对应的索引。
 - `copyOf`：复制。
 - `equals`：判断两个数组是否相等。
 - `fill`：填充
 - `hashCode`：返回数组的内容的哈希码。
 - `toString`：数组转为字符串
 - `stream`：数组转为流

## Collections

### 排序

 - `public static <T extends Comparable<? super T>> void sort(List<T> list)`
 - `public static <T> void sort(List<T> list, Comparator<? super T> c)`

### 返回线程安全的集合

 - `synchronizedCollection(Collection<T> c)`
 - `synchronizedCollection(Collection<T> c, Object mutex)`
 - `synchronizedList(List<T> list)`
 - `synchronizedList(List<T> list, Object mutex)`
 - `synchronizedMap(Map<K,V> m)`：返回一个线程安全的 SynchronizedMap 类，但是同样是线程安全的 ConcurrentHashMap 类性能优于  SynchronizedMap。
 - `synchronizedSet(Set<T> s)`
 - `synchronizedSet(Set<T> s, Object mutex)`
 - `synchronizedSortedMap(SortedMap<K,V> m)`
 - `synchronizedSortedSet(SortedSet<T> s)`

部分方法对应一个可以设置同步对象锁的重载方法。

### 返回只读的集合

 - `unmodifiableCollection(Collection<? extends T> c)`
 - `unmodifiableList(List<? extends T> list)`
 - `unmodifiableMap(Map<? extends K, ? extends V> m)`
 - `unmodifiableSet(Set<? extends T> s)`
 - `unmodifiableSortedMap(SortedMap<K, ? extends V> m)`
 - `unmodifiableSortedSet(SortedSet<T> s)`
 

