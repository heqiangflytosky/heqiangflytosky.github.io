
---
title: Java 并发编程 -- ReentrantLock 与 Condition
categories: Java 并发
comments: true
tags: [Java 并发]
description: 本文介绍 ReentrantLock 与 Condition 的使用
date: 2015-7-20 10:00:00
---

## 概述

`ReentrantLock`  顾名思义就是可重入锁，即当前持有该锁的线程能够多次获取该锁，无需等待。

## ReentrantLock 和 synchronized 的区别

### 相同点

 - 都是可重入锁

### 不同点

 - `synchronized` 是依赖于JVM实现的，而 `ReenTrantLock` 是JDK实现的
 - `synchronized` 的使用比较方便简洁，并且由编译器去保证锁的加锁和释放，而 `ReenTrantLock` 需要手工声明来加锁和释放锁，为了避免忘记手工释放锁造成死锁，或者在加锁的代码块内由于程序异常退出而没有释放锁造成死锁，所以最好在 `finally` 中声明释放锁。
 - `ReenTrantLock` 可以指定是公平锁还是非公平锁。而 `synchronized` 只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。
 - `ReenTrantLock` 提供了一个 `Condition`（条件）类，用来实现分组唤醒需要唤醒的线程们，而不是像 `synchronized` 要么随机唤醒一个线程要么唤醒全部线程。一个 `Condition` 对象的 `signal`（`signalAll`）方法和该对象的 `await` 方法是一一对应的，也就是一个 `Condition` 对象的 `signal`（`signalAll`）方法不能唤醒其他 `Condition` 对象的 `await` 方法。
 - `ReenTrantLock` 提供了一种能够中断等待锁的线程的机制，通过 `lock.lockInterruptibly()` 来实现这个机制。

## Condition 和 Object的wait和notify/notifyAll 的区别

### 类似点

 - `Condition` 对象的 `awiat` 方法和 `Object` 对象的 `wait` 方法等效。调用时也必须获得锁。
 - `Condition` 对象的 `signal` 方法和 `Object` 对象的 `notify` 方法等效。调用时也必须获得锁。
 - `Condition` 对象的 `signalAll` 方法和 `Object` 对象的 `notifyAll` 方法等效。调用时也必须获得锁。

### 不同点

 - Condition能够支持多个等待队列（new 多个Condition对象），而Object方式只能支持一个

## 使用方法

`ReentrantLock(boolean fair)` 可以根据传入的参数来创建两种不同的锁：

 - new ReentrantLock(true)：公平锁，线程获取锁的顺序是按照加锁顺序来的
 - new ReentrantLock(false)：非公平锁，抢锁机制，先lock的线程不一定先获得锁。

```
    private ReentrantLock mLock = new ReentrantLock();
    public void testReentrantLock() {
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                mLock.lock();
                try {
                    Log.e("Test", Thread.currentThread().getName() + " start");
                    Thread.sleep(1000);
                    Log.e("Test", Thread.currentThread().getName() + " end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    // 最好在finally代码块种释放，能够避免死锁的发生
                    mLock.unlock();
                }
            }
        };

        Thread thread1 = new Thread(runnable);
        thread1.setName("Thread1");
        thread1.setPriority(1);
        Thread thread2 = new Thread(runnable);
        thread2.setName("Thread2");
        thread2.setPriority(2);

        thread1.start();
        thread2.start();
    }
```

```
17:16:01.072 E/Test: Thread1 start
17:16:02.114 E/Test: Thread1 end
17:16:02.115 E/Test: Thread2 start
17:16:03.156 E/Test: Thread2 end
```

配合 Condition 使用：

```
    private ReentrantLock mLock = new ReentrantLock(true);
    private Condition condition1 = mLock.newCondition();
    private Condition condition2 = mLock.newCondition();
    public void testReentrantLock() {
        Thread thread1 = new Thread(new MyRunnable(condition1));
        thread1.setName("Thread1");
        Thread thread2 = new Thread(new MyRunnable(condition1));
        thread2.setName("Thread2");
        Thread thread3 = new Thread(new MyRunnable(condition2));
        thread3.setName("Thread3");
        Thread thread4 = new Thread(new Runnable() {
            @Override
            public void run() {
                mLock.lock();
                try {
                    Log.e("Test", Thread.currentThread().getName() + " start");
                    Thread.sleep(5000);
                    Log.e("Test", Thread.currentThread().getName() + " condition1.signalAll()");
                    condition1.signalAll();
                    //Log.e("Test", Thread.currentThread().getName() + " condition2.signalAll()");
                    //condition2.signalAll();
                    Thread.sleep(5000);
                    Log.e("Test", Thread.currentThread().getName() + " end");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    mLock.unlock();
                }
            }
        });
        thread4.setName("Thread4");

        thread1.start();
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread2.start();
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread3.start();
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        thread4.start();
    }

    private class MyRunnable implements Runnable{

        private Condition mCondition;
        public MyRunnable(Condition condition) {
            mCondition = condition;
        }

        @Override
        public void run() {
            mLock.lock();
            try {
                Log.e("Test", Thread.currentThread().getName() + " start");
                Thread.sleep(1000);
                Log.e("Test", Thread.currentThread().getName() + " start await");
                mCondition.await();
                Log.e("Test", Thread.currentThread().getName() + " end await");
                Thread.sleep(1000);
                Log.e("Test", Thread.currentThread().getName() + " end");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                mLock.unlock();
            }
        }
    }
```

```
17:27:52.250 E/Test: Thread1 start
17:27:53.251 E/Test: Thread1 start await
17:27:53.252 E/Test: Thread2 start
17:27:54.254 E/Test: Thread2 start await
17:27:54.255 E/Test: Thread3 start
17:27:55.259 E/Test: Thread3 start await
17:27:55.260 E/Test: Thread4 start
17:28:00.261 E/Test: Thread4 condition1.signalAll()
17:28:05.264 E/Test: Thread4 end
17:28:05.264 E/Test: Thread1 end await
17:28:06.268 E/Test: Thread1 end
17:28:06.269 E/Test: Thread2 end await
17:28:07.273 E/Test: Thread2 end
```

从这里可以看出，只调用 `condition1.signalAll()` 后，Thread3 后面没有被唤醒。
下面来看一下在 Thread4 中调用 `condition2.signalAll()` 的执行情况。

```
                    Log.e("Test", Thread.currentThread().getName() + " condition1.signalAll()");
                    condition1.signalAll();
                    Log.e("Test", Thread.currentThread().getName() + " condition2.signalAll()");
                    condition2.signalAll();
                    Thread.sleep(5000);
```

```
17:36:23.026 E/Test: Thread1 start
17:36:24.027 E/Test: Thread1 start await
17:36:24.028 E/Test: Thread2 start
17:36:25.031 E/Test: Thread2 start await
17:36:25.032 E/Test: Thread3 start
17:36:26.034 E/Test: Thread3 start await
17:36:26.034 E/Test: Thread4 start
17:36:31.039 E/Test: Thread4 condition1.signalAll()
17:36:31.039 E/Test: Thread4 condition2.signalAll()
17:36:36.044 E/Test: Thread4 end
17:36:36.044 E/Test: Thread1 end await
17:36:37.047 E/Test: Thread1 end
17:36:37.047 E/Test: Thread2 end await
17:36:38.048 E/Test: Thread2 end
17:36:38.049 E/Test: Thread3 end await
17:36:39.050 E/Test: Thread3 end
```

Thread3 被唤醒，得到锁后得以继续执行。
