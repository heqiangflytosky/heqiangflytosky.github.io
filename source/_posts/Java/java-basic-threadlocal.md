---
title: Java 基础 -- ThreadLocal
categories: Java
comments: true
tags: [Java]
description: 介绍 ThreadLocal 的用法
date: 2016-11-16 10:00:00
---

## 什么是 ThreadLocal

ThreadLocal 从字面意思理解很容易让人认为是一个“本地线程”，其实，ThreadLocal 并不是一个 Thread ，而是 Thread 的局部变量，也许把它命名为 ThreadLocalVariable 更容易让人理解一些。
ThreadLocal 保存了一些变量，对于同一个 ThreadLocal，在多线程环境中不同线程通过 get 或者 set 方法操作的变量都是互相隔离的，也就是操作的只是属于自己线程的变量，这个操作不会影响其他线程的变量。即使它们使用的是同一个 ThreadLocal 对象。

## 用法介绍

ThreadLocal 的几个方法：

 - get()：获取 ThreadLocal 中当前线程的“本地变量”的值。
 - set(T value)：设置 ThreadLocal 中当前线程的“本地变量”的值。
 - remove()：删除 ThreadLocal 中当前线程的“本地变量”的值。
 - initialValue()：调用get方法时，如果当前线程没有设置本地变量，就调用这个方法返回默认值。可以重写这个方法返回自己想要的默认值。

下面简单写个 demo 来介绍一下它的用法：

```
    private ThreadLocal<String> mThreadLocal = new ThreadLocal<>();

        new Thread(new Runnable() {
            @Override
            public void run() {
                Log.e("Test","thread1 set");
                mThreadLocal.set(Thread.currentThread().getName());
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                Log.e("Test","thread1 get = "+mThreadLocal.get());
            }
        },"Thread1").start();


        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(1000);
                    Log.e("Test","thread2 set");
                    mThreadLocal.set(Thread.currentThread().getName());

                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                Log.e("Test","thread2 get = "+mThreadLocal.get());
            }
        },"Thread2").start();
```

```
thread1 set
thread2 set
thread1 get = Thread1
thread2 get = Thread2
```

对于同一个 mThreadLocal 变量，在 Thread1 中的保存的变量是字符串 “Thread1”，在 Thread2 中的保存的变量是字符串 “Thread2”。


## ThreadLocal 原理和源码分析

Thread 类中有个变量 

```
ThreadLocal.ThreadLocalMap threadLocals = null
```

也就是说每个线程中都存在一个这样的 ThreadLocalMap 映射表，

ThreadLocalMap 又是怎么和 ThreadLocal 联系起来的呢？我们不妨先来看一下 ThreadLocal 几个方法的源码：

```
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

set 方法会从当前 Thread 中获取  threadLocals 变量，它是 ThreadLocalMap 对象，然后以调用 set 方法的 ThreadLocal 对象为key，把值设置到该映射表中去。

```
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
```

get 方法是把调用 get 方法 ThreadLocal 对象为key的值从当前 Thread 的映射表中取出来。

在为每个 Thread 初始化 ThreadLocalMap 时，会把 ThreadLocal 对象和要存储的本地变量作为参数传入：

```
        ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
            // 设置初始容量为 16
            table = new Entry[INITIAL_CAPACITY];
            // 以 ThreadLocal 的 threadLocalHashCode 来作为在数组中的存储位置。
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```

每个 ThreadLocalMap 对象有个 Entry 数组 table，Entry 对象包含 key（当前 ThreadLocal 对象） 和 value （要保存的本地变量）两个属性 ：

```
        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }
```

Entry 继承 WeakReference 可以保证不再使用的 ThreadLocal 可以及时的垃圾回收。
那么如果保证不同的 ThreadLocal 在 数组中的位置不同呢？我们来看一下 ThreadLocal 的 threadLocalHashCode 变量。

```
    private final int threadLocalHashCode = nextHashCode();
    private static final int HASH_INCREMENT = 0x61c88647;
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```

这里以静态变量从0开始不断加 0x61c88647 来作为不同 ThreadLocal 的哈希值，这里为什么选用 0x61c88647，可以参考网络上介绍的很多文章：为了让 hash code 能更好地均匀分布在尺寸为 2 的 N 次方的数组里。
这里又有个问题，如果仅仅为了保证均匀分布，那为什么不采用直接从0自增的方式来分配hash code，然后对数组大小取模来生成数组位置。这里我们需要注意的是 Entry 是可能会被回收的，这样可能会造成空间的浪费。而 ThreadLocalMap 的这种方法是跳跃分布分配，这样即使出现了冲突，也能通过解决hash冲突的方法来解决。ThreadLocalMap 解决hash冲突的方法是开地址法。

```
        private void set(ThreadLocal key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

到此我们基本明白了 ThreadLocal 原理：在每个 Thread 对象中有个映射表，这个映射表用一个数组键值对的形式来保存在当前线程中通过 ThreadLocal set 方法设置的局部变量，这个键值对的key是 ThreadLocal。

## 为什么要用 ThreadLocal

我们为什么要用 ThreadLocal 呢？当然使用其他方法也同样是可以满足我们的需求的。
比如使用自定义线程的局部变量：其实 ThreadLocal 也是定义了线程的局部变量 ThreadLocalMap。它已经替我们封装好了，隐藏了实现细节，方便我们使用。
比如使用自定义全局变量：全局变量在多线程使用要注意同步问题，会影响效率。

通过上面的分析，我们知道 ThreadLocal 的特点：

 - ThreadLocal 提供了获取本线程局部变量的能力，多线程编程时，每个线程都往同一个 ThreadLocal 对象中存取所需要的变量就可以了，使用 ThreadLocal 存取的变量，就像是每个线程自己的局部变量，不受其他线程运行状态的影响。
 - ThreadLocal 以空间换时间解决了对象在被共享访问带来线程安全问题。每个线程都有一个ThreadLocalMap映射表，正是利用了这个映射表所占用的空间，使得多个线程都可以访问自己的这片空间，不用担心考虑线程同步问题，效率自然会高。打个比方说，现在有100个同学需要填写一张表格但是只有一支笔，同步就相当于A使用完这支笔后给B，B使用后给C用......老师就控制着这支笔的使用顺序，使得同学之间不会产生冲突。而threadLocal就相当于，老师直接准备了100支笔，这样每个同学都使用自己的，同学之间就不会产生冲突。很显然这就是两种不同的思路，同步机制以“时间换空间”，由于每个线程在同一时刻共享对象只能被一个线程访问造成整体上响应时间增加，但是对象只占有一份内存，牺牲了时间效率换来了空间效率即“时间换空间”。而threadLocal，为每个线程都分配了一份对象，自然而然内存使用率增加，每个线程各用各的，整体上时间效率要增加很多，牺牲了空间效率换来时间效率即“空间换时间”。



## 使用场景

### Looper 中的运用

是 Looper 控制消息队列的类，每个拥有消息队列的线程，都会有一个独立的Looper类，用于处理本线程的消息。

```
public final class Looper {

    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
}
```

在 Looper 中自定义了一个 sThreadLocal 变量，在 prepare 时设置了一个 Looper 对象，调用 myLooper() 时，会返回当前线程对应的 Looper 对象。

## ThreadLocal 为避免内存泄漏而做的努力

前面我们已经提到过，`Entry extends WeakReference<ThreadLocal>` Entry 继承 WeakReference，那么 ThreadLocal 是以弱引用身份被 Entry 作为key 来引用的，那么如果 ThreadLocal 没有被外部的强引用所使用，那么这个 ThreadLocal 会在下次 GC 的时候被回收。
这里就有个问题，虽然 ThreadLocal 被回收了，ThreadLocalMap 就会出现了个 null key 的情况，但是 value 并没有被回收，如果不做处理，就会一直存在这里，如果当前线程生命周期很长的话，就有可能造成内存泄漏。
其实 ThreadLocal 已经考虑到了这种情况，并做了一些措施来保证及时清除被回收的 ThreadLocal 对应的value。在 ThreadLocal 的 get set remove 方法调用时，如果发现 key 为null 时，会调用 replaceStaleEntry 或者 expungeStaleEntry 方法类替换或者清除 value。
下面来看一下代码：

```
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
```

```
        private Entry getEntry(ThreadLocal key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```

```
        private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```

在 get 方法中，会调用 ThreadLocalMap getEntry 方法，如果返现key为null的情况，就调用 expungeStaleEntry 清除 value。并继续循环往下检查是否有null的key，有的话一并删除。

## 参考

https://www.jianshu.com/p/dde92ec37bd1
http://duanqz.github.io/2018-03-15-Java-ThreadLocal