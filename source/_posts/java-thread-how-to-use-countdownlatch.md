
---
title: Java 线程 -- CountDownLatch
categories: Android 线程
comments: true
tags: [Android 线程]
description: 本文介绍 CountDownLatch 的使用
date: 2015-9-5 10:00:00
---

## 概述

`CountDownLatch` 是我们在并发编程中很常见的工具，我们可以使用它来进行线程的同步。它维护一个计数器，等待 `CountDownLatch` 的线程必须等到计数器为 0 时才可以继续执行。 

## CountDownLatch 的使用

我们可以在下面场景中使用 `CountDownLatch`：

 - 某个线程等待n个线程执行完某个操作，比如完成某个任务
 - n个线程等待某个线程执行完某个操作，比如满足某个条件

下面我们通过两个demo来实现一下上面两个场景的应用。

### 场景一

```
    public void testCountDownLatch() {
        final CountDownLatch countDownLatch = new CountDownLatch(3);
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Log.e("Test", "run thread1 0");
                    Thread.sleep(1000);
                    countDownLatch.countDown();
                    Log.e("Test", "run thread1 1");
                    Thread.sleep(1000);
                    countDownLatch.countDown();
                    Log.e("Test", "thread1 end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Log.e("Test", "run thread2 0");
                    Thread.sleep(3000);
                    countDownLatch.countDown();
                    Log.e("Test", "thread2 end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        Thread thread3 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Log.e("Test", "run thread3 0");
                    countDownLatch.await();
                    Log.e("Test", "thread3 end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        thread1.start();
        thread2.start();
        thread3.start();
    }
```

```
22:38:25.283 E/Test: run thread3 0
22:38:25.283 E/Test: run thread2 0
22:38:25.284 E/Test: run thread1 0
22:38:26.287 E/Test: run thread1 1
22:38:27.291 E/Test: thread1 end
22:38:28.285 E/Test: thread2 end
22:38:28.285 E/Test: thread3 end
```

### 场景二

```
    public void testCountDownLatch() {
        final CountDownLatch countDownLatch = new CountDownLatch(1);
        Thread thread1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Log.e("Test", "run thread1 0");
                    countDownLatch.await();
                    Log.e("Test", "thread1 end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        Thread thread2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Log.e("Test", "run thread2 0");
                    countDownLatch.await();
                    Log.e("Test", "thread2 end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        Thread thread3 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Log.e("Test", "run thread3 0");
                    Thread.sleep(2000);
                    countDownLatch.countDown();
                    Log.e("Test", "thread3 end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        thread1.start();
        thread2.start();
        thread3.start();
```

```
22:43:56.425 E/Test: run thread1 0
22:43:56.425 E/Test: run thread2 0
22:43:56.427 E/Test: run thread3 0
22:43:58.429 E/Test: thread3 end
22:43:58.429 E/Test: thread1 end
22:43:58.429 E/Test: thread2 end
```

## 使用须知

计数器的值只能在构造方法中初始化一次，之后没有方法再次对其设置值，当 `CountDownLatch` 使用完毕后，它不能再次被使用。

## 源码分析

知道怎么使用之后，我们就按照惯例来看一下源码，看看它的原理是怎么样的？

```
public class CountDownLatch {
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        int getCount() {
            return getState();
        }

        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        protected boolean tryReleaseShared(int releases) {
            // Decrement count; signal when transition to zero
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c - 1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    public void countDown() {
        sync.releaseShared(1);
    }

    public long getCount() {
        return sync.getCount();
    }

    public String toString() {
        return super.toString() + "[Count = " + sync.getCount() + "]";
    }
}
```

上面就是 `CountDownLatch` 的全部源码，比较简单，重点在于变量 `sync`。
它是内部类 `Sync` 的一个对象，`Sync` 继承自 `AbstractQueuedSynchronizer`，`AbstractQueuedSynchronizer` (AQS)提供了一个FIFO队列，可以看做是一个可以用来实现锁以及其他需要同步功能的框架。它的使用依靠继承来完成，子类通过继承并实现所需的方法来管理同步状态。此类我们后面再详细介绍。
`countDown()` 方法调用的是 AQS 的 `releaseShared(1)` 方法：

```
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```

此法方法会首先调用 Sync 实现的 `tryReleaseShared(1)` 方法，该方法的实现思路是把传入的参数减1,判断是否为0,如果为0,表示可以释放锁。
