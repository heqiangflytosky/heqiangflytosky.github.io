---
title: Java 线程 -- Semaphore
categories: Android 线程
comments: true
tags: [Android 线程]
description: 本文介绍 Semaphore 的使用
date: 2015-9-7 10:00:00
---

## 概述

在操作系统中，信号量是个很重要的概念，它在控制进程间的协作方面有着非常重要的作用，通过对信号量的不同操作，可以分别实现进程间的互斥与同步。当然它也可以用于多线程的控制，我们完全可以通过使用信号量来自定义实现类似 Java 中的 `synchronized`、`wait`、`notify` 机制。

Java 并发包中的信号量 `Semaphore` 实际上是一个功能完毕的计数信号量，从概念上讲，它维护了一个许可集合，对控制一定资源的消费与回收有着很重要的意义。`Semaphore` 可以控制某个资源被同时访问的任务数，它通过 `acquire()` 获取一个许可，`release()` 释放一个许可。如果被同时访问的任务数已满，则其他 `acquire` 的任务进入等待状态，直到有一个任务被 `release` 掉，它才能得到许可。
`Semaphore` 仅仅是对资源的并发访问的任务数进行监控，而不会保证线程安全，因此，在访问的时候，要自己控制线程的安全访问。

## 使用方法

下面给出一个采用 `Semaphore` 控制并发访问数量的示例程序：

```
    Semaphore mSemaphore = new Semaphore(2);
    public void testSemaphore() {
        Thread thread1 = new Thread(new MyRunnable());
        thread1.setName("Thread1");
        Thread thread2 = new Thread(new MyRunnable());
        thread2.setName("Thread2");
        Thread thread3 = new Thread(new MyRunnable());
        thread3.setName("Thread3");
        Thread thread4 = new Thread(new MyRunnable());
        thread4.setName("Thread4");
        Thread thread5 = new Thread(new MyRunnable());
        thread5.setName("Thread5");

        thread1.start();
        thread2.start();
        thread3.start();
        thread4.start();
        thread5.start();
    }

    private class MyRunnable implements Runnable{
        @Override
        public void run() {
            try {
                Log.e("Test","Thread "+Thread.currentThread().getName()+" run");
                mSemaphore.acquire();
                Log.e("Test","Thread "+Thread.currentThread().getName()+" acquire");
                Thread.sleep(1000);
                mSemaphore.release();
                Log.e("Test","Thread "+Thread.currentThread().getName()+" release");
                Thread.sleep(1000);
                Log.e("Test","Thread "+Thread.currentThread().getName()+" end");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
```

```
00:08:15.645 E/Test: Thread Thread1 run
00:08:15.645 E/Test: Thread Thread1 acquire
00:08:15.645 E/Test: Thread Thread5 run
00:08:15.645 E/Test: Thread Thread5 acquire
00:08:15.645 E/Test: Thread Thread3 run
00:08:15.645 E/Test: Thread Thread4 run
00:08:15.645 E/Test: Thread Thread2 run
00:08:16.650 E/Test: Thread Thread1 release
00:08:16.650 E/Test: Thread Thread5 release
00:08:16.650 E/Test: Thread Thread3 acquire
00:08:16.650 E/Test: Thread Thread4 acquire
00:08:17.652 E/Test: Thread Thread5 end
00:08:17.652 E/Test: Thread Thread3 release
00:08:17.652 E/Test: Thread Thread1 end
00:08:17.652 E/Test: Thread Thread4 release
00:08:17.652 E/Test: Thread Thread2 acquire
00:08:18.654 E/Test: Thread Thread3 end
00:08:18.654 E/Test: Thread Thread4 end
00:08:18.655 E/Test: Thread Thread2 release
00:08:19.656 E/Test: Thread Thread2 end
```

可以看出，Semaphore 允许并发访问 `mSemaphore.acquire()` 和 `mSemaphore.release()` 之间资源的任务数一直为 2。