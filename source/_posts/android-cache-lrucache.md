---
title: Android -- LruCache 使用和原理
categories: Android
comments: true
tags: [Android]
description: 介绍 LruCache 的使用和原理
date: 2016-9-15 10:00:00
---

## 概述

我们在对数据机型操作过程中，为了得到更好的存取速度，通常会把部分数据缓存在内存中，比如图片。但是在 Android 设备中，内存是非常宝贵的资源，如果持续地缓存一些数据，势必会造成内存吃紧。
为了解决这个问题，Android 提供了缓存数据的 LruCache 类来完成这个任务。
LruCache （Least Recently Used）算法的思想就是最近最少使用算法。一般用来做内存缓存使用。如果想做磁盘缓存，后面我们会介绍 DiskLruCache 来处理这个问题。

## 基本原理

LruCache 内部维护了一个 LinkedHashMap 对象来保存缓存，采用的是访问顺序存储。每当一个缓存数据被访问的时候，这个数据就会被提到列表尾部，这样列表的头部数据就是最近最不常使用的了，当缓存空间不足时，就会删除列表头部的缓存数据。

## 基本用法

```
        Bitmap bitmap  = BitmapFactory.decodeResource(context.getResources(), R.mipmap.ic_launcher);;

        int maxSize = 1024;
        LruCache<String, Bitmap> lruCache = new LruCache<String, Bitmap>(maxSize){
            @Override
            protected int sizeOf(String key, Bitmap value) {
                // 重写这个方法返回图片大小，默认是返回图片数量1。
                return value.getByteCount();
            }
            @Override
            protected void entryRemoved(boolean evicted, String key, Bitmap oldValue, Bitmap newValue) {
                // 这个方法是当某条数据被清除时调用，包括新数据覆盖旧数据
                // 和 cache 超过最大值时清除的数据
                super.entryRemoved(evicted, key, oldValue, newValue);
            }
            
            @Override
            protected Bitmap create(String key) {
                // 当没有缓存时返回的值，如果不重写默认是返回 null
                // 也可自定义这个逻辑
                return super.create(key);
            }
        };

        // 存入缓存
        lruCache.put("bitmap1",bitmap);

        // 获取缓存
        Bitmap bitmap1 = lruCache.get("bitmap1");
        // 可能为空
        if(bitmap1 != null) {
            
        }
```

## 原理分析

### 构造方法

```
    public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        // 设置缓存最大值
        this.maxSize = maxSize;
        // 创建保存数据的 LinkedHashMap，采用访问顺序存储
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    }
```

### put 方法

```
    public final V put(K key, V value) {
        // key 和 value 不能为空
        if (key == null || value == null) {
            throw new NullPointerException("key == null || value == null");
        }

        V previous;
        synchronized (this) {
            putCount++;
            // 增加size
            size += safeSizeOf(key, value);
            previous = map.put(key, value);
            // 不为空表示map中已经存在该key
            if (previous != null) {
                // 纠正前面的计算，把重复的大小减掉
                size -= safeSizeOf(key, previous);
            }
        }

        if (previous != null) {
            // 调用 entryRemoved 处理一些清除数据的操作
            // 这个方法可以由调用者重写
            entryRemoved(false, key, previous, value);
        }
        // 看是否要清除数据
        trimToSize(maxSize);
        return previous;
    }
```

```
    public void trimToSize(int maxSize) {
        while (true) {
            K key;
            V value;
            synchronized (this) {
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }

                if (size <= maxSize) {
                    break;
                }

                Map.Entry<K, V> toEvict = map.eldest();
                if (toEvict == null) {
                    break;
                }
                // 删除队列头部的元素
                key = toEvict.getKey();
                value = toEvict.getValue();
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }

            entryRemoved(true, key, value, null);
        }
    }
```

### get 方法

```
    public final V get(K key) {
        if (key == null) {
            throw new NullPointerException("key == null");
        }

        V mapValue;
        synchronized (this) {
            mapValue = map.get(key);
            if (mapValue != null) {
                hitCount++;
                return mapValue;
            }
            missCount++;
        }

        V createdValue = create(key);
        if (createdValue == null) {
            return null;
        }

        synchronized (this) {
            createCount++;
            mapValue = map.put(key, createdValue);
            // 因为create方法不是线程安全的，
            // 有可能在create过程中put了一个相同的元素
            // 这时要处理冲突
            if (mapValue != null) {
                map.put(key, mapValue);
            } else {
                size += safeSizeOf(key, createdValue);
            }
        }

        if (mapValue != null) {
            entryRemoved(false, key, createdValue, mapValue);
            return mapValue;
        } else {
            trimToSize(maxSize);
            return createdValue;
        }
    }
```