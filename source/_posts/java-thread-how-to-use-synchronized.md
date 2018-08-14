---
title: Java 线程 -- synchronized、wait、notify和notifyAll的使用
categories: Android 线程
comments: true
tags: [Android 线程]
description: 本文介绍 synchronized 的使用
date: 2015-9-4 10:00:00
---

## 概述

在并发编程中，多线程同时并发访问的资源叫做临界资源，当多个线程同时访问对象并要求操作相同资源时，分割了原子操作就有可能出现数据的不一致或数据不完整的情况，为避免这种情况的发生，我们会采取同步机制，以确保在某一时刻，方法内只允许有一个线程。
采用 synchronized 修饰符实现的同步机制叫做互斥锁机制，它所获得的锁叫做互斥锁。每个对象都有一个 monitor (锁标记)，当线程拥有这个锁标记时才能访问这个资源，没有锁标记便进入锁池。任何一个对象系统都会为其创建一个互斥锁，这个锁是为了分配给线程的，防止打断原子操作。每个对象的锁只能分配给一个线程，因此叫做互斥锁。


## 使用场景

|  分类  |  被锁对象  | 代码 |
|:-----:|:-----:|:-----:|
| 静态方法 | 类对象 | public static synchronized void method() {} |
| 实例方法 | 类的实例对象 | public synchronized void method() {} |
| 实例对象 | 类的实例对象 | synchronized (this) {} |
| class对象 | 类对象 | synchronized (Demo.class) {} |
| 任意实例对象Object | 实例对象Object | synchronized (mObject) {} |

前面两种是针对方法，后面三种是针对代码块。

## 实现原理

synchronized 关键字经过编译后，会在同步块的前后分别形成 monitorenter 和 monitorexit 这两个字节码指令。根据虚拟机规范的要求，在执行 monitorenter 指令时，首先要尝试获取对象的锁，如果获得了锁，把锁的计数器加 1，相应地，在执行 monitorexit 指令时会将锁计数器减 1，当计数器为 0 时，锁便被释放了。由于 synchronized 同步块对同一个线程是可重入的，因此一个线程可以多次获得同一个对象的互斥锁，同样，要释放相应次数的该互斥锁，才能最终释放掉该锁。

## 使用方法

```
    private Object mObject = new Object();
    public void testSynchronized() {
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (mObject) {
                    try {
                        Log.e("Test", Thread.currentThread().getName() + " start");
                        Thread.sleep(1000);
                        Log.e("Test", Thread.currentThread().getName() + " end");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        thread1.setName("Thread1");
        thread1.setPriority(1);
        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (mObject) {
                    try {
                        Log.e("Test", Thread.currentThread().getName() + " start");
                        Thread.sleep(1000);
                        Log.e("Test", Thread.currentThread().getName() + " end");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        thread2.setName("Thread2");
        thread2.setPriority(2);
        thread1.start();
        thread2.start();
    }
```

```
00:28:48.055 E/Test: Thread1 start
00:28:50.057 E/Test: Thread1 end
00:28:50.059 E/Test: Thread2 start
00:28:51.060 E/Test: Thread2 end

```

可以看到 Thread1 加锁代码块首先执行，首先获得类对象锁，后面 Thread2 run 方法执行到加锁代码块时进入等待状态，Thread1 代码块执行完释放锁之后 Thread2 对应代码块才开始执行。



## wait、notify和notifyAll

先来简单介绍一下这几个方法：

 - wait、notify 和 notifyAll 方法是 Object 的方法，而且是 final 方法，无法被重写。
 - wait 会使当前的线程进入阻塞状态，而且它会释放当前的锁，然后让出CPU，进入等待状态。但前提是必须先获得锁。
 - wait、notify 和 notifyAll 一般是配合 synchronized 使用，用在 synchronized 方法或者代码块内。
 - 当 notify/notifyAll() 被执行时候，才会唤醒一个或多个正处于等待状态的线程，然后继续往下执行，直到执行完 synchronized 代码块的代码或是中途遇到wait() ，再次释放锁。
 - notify/notifyAll() 的执行只是唤醒沉睡的线程，而不会立即释放锁，调用线程依旧持有锁。锁的释放要看代码块的具体执行情况。因此，线程虽被唤醒，但是仍无法获得锁。直到调用线程退出 synchronized 方法或者代码块，释放锁后，等待线程中的一个才有机会获得锁继续执行。所以在编程中，尽量在使用了notify/notifyAll() 后立即退出 synchronized 临界区，以便其他线程唤醒并获得锁以继续执行。
 - notify方法只唤醒一个等待（对象的）线程并使该线程开始执行。如果有多个线程等待一个对象，这个方法只会唤醒其中一个线程，选择哪个线程取决于操作系统对多线程管理的实现。notifyAll 会唤醒所有等待(对象的)线程。如果当前情况下有多个线程需要被唤醒，推荐使用notifyAll 方法。


把上面的 Thread1 代码做一些改动，代码执行中间加入 `wait()` 方法：

```
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (mObject) {
                    try {
                        Log.e("Test", Thread.currentThread().getName() + " start");
                        Thread.sleep(1000);
                        Log.e("Test", Thread.currentThread().getName() + " wait");
                        mObject.wait();
                        Log.e("Test", Thread.currentThread().getName() + " after wait");
                        Thread.sleep(1000);
                        Log.e("Test", Thread.currentThread().getName() + " end");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
```


```
00:30:09.948 E/Test: Thread1 start
00:30:10.949 E/Test: Thread1 wait
00:30:10.950 E/Test: Thread2 start
00:30:11.951 E/Test: Thread2 end
```

可见 `wait()` 方法唤醒 Thread2 获得锁开始执行，Thread2 加锁代码块执行完后 Thread1 仍然再等待状态。

改动一下 Thread2 代码，代码块中加入 `mObject.notify()`。
**注意：**`notify()` 方法必须放到 synchronized 代码块中。

```
        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (mObject) {
                    try {
                        Log.e("Test", Thread.currentThread().getName() + " start");
                        Thread.sleep(1000);
                        Log.e("Test", Thread.currentThread().getName() + " notify");
                        mObject.notify();
                        Thread.sleep(1000);
                        Log.e("Test", Thread.currentThread().getName() + " end");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
```

```
15:13:16.072 E/Test: Thread1 start
15:13:17.073 E/Test: Thread1 wait
15:13:17.073 E/Test: Thread2 start
15:13:18.079 E/Test: Thread2 notify
15:13:19.082 E/Test: Thread2 end
15:13:19.083 E/Test: Thread1 after wait
15:13:20.084 E/Test: Thread1 end

```

可以看到，Thread2 发出 `notify()` 并在 synchronized 执行完释放锁后，Thread1 会得到锁，继续执行 `wait()` 后面的代码。

下面来看一下 `notifyAll` 的使用情况：

```
    private Object mObject = new Object();
    public void testSynchronized() {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                synchronized (mObject) {
                    try {
                        Log.e("Test", Thread.currentThread().getName() + " start");
                        Thread.sleep(1000);
                        Log.e("Test", Thread.currentThread().getName() + " wait");
                        mObject.wait();
                        Log.e("Test", Thread.currentThread().getName() + " after wait");
                        Thread.sleep(1000);
                        Log.e("Test", Thread.currentThread().getName() + " end");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        Thread thread1 = new Thread(runnable);
        thread1.setName("Thread1");
        thread1.setPriority(1);
        Thread thread2 = new Thread(runnable);
        thread2.setName("Thread2");
        thread2.setPriority(2);
        Thread thread3 = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (mObject) {
                    try {
                        Log.e("Test", Thread.currentThread().getName() + " start");
                        Thread.sleep(5000);
                        Log.e("Test", Thread.currentThread().getName() + " notifyAll");
                        mObject.notifyAll();
                        Thread.sleep(1000);
                        Log.e("Test", Thread.currentThread().getName() + " end");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        thread3.setName("Thread3");
        thread3.setPriority(3);
        thread1.start();
        thread2.start();
        thread3.start();
    }
```

```
15:32:48.906 E/Test: Thread1 start
15:32:49.908 E/Test: Thread1 wait
15:32:49.908 E/Test: Thread2 start
15:32:50.909 E/Test: Thread2 wait
15:32:50.910 E/Test: Thread3 start
15:32:55.916 E/Test: Thread3 notifyAll
15:32:56.917 E/Test: Thread3 end
15:32:56.918 E/Test: Thread1 after wait
15:32:57.922 E/Test: Thread1 end
15:32:57.922 E/Test: Thread2 after wait
15:32:58.925 E/Test: Thread2 end
```

这里主要看一点，notifyAll 之后，其他正在wait的县城并没有立即开始执行，而是等到持有锁的线程 synchronized 代码块执行完毕，其他两个线程才会依次根据优先级执行。
