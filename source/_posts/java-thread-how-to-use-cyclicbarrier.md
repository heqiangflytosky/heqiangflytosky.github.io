---
title: Java 线程 -- CyclicBarrier
categories: Android 线程
comments: true
tags: [Android 线程]
description: 本文介绍 CyclicBarrier 的使用
date: 2015-9-6 10:00:00
---

## 概述

前面的文章我们介绍了 `CountDownLatch`，本文我们会继续介绍另外一个并发编程中经常遇到的工具 CyclicBarrier。使用它可以使一组线程相互之间等待，达到一个共同点，再继续执行。
我们可以把它理解成一个障碍，所有的线程必须到齐后才能一起通过这个障碍。
关于它和 `CountDownLatch`，可以对比我的上一篇关于 `CountDownLatch` 的博客。
`CountDownLatch` 更像是一个计数器，等待线程等待计数器为0后才会继续执行。而 `CyclicBarrier` 是所有参与的线程都要等待达到某个条件后再一起执行。
另外还有重要一点：`CountDownLatch` 是只触发一次的事件，而 `CyclicBarrier` 可以多次重用。可以调用 `reset()` 来实现重用。

## 使用方法

```
    public void testCyclicBarrier() {
        final CyclicBarrier cyclicBarrier = new CyclicBarrier(3, new Runnable() {
            @Override
            public void run() {
                Log.e("Test", "run barrier thread");
            }
        });
        Thread thread1 = new Thread(new MyRunnable(cyclicBarrier, 1000));
        thread1.setName("Thread1");
        Thread thread2 = new Thread(new MyRunnable(cyclicBarrier, 2000));
        thread2.setName("Thread2");
        Thread thread3 = new Thread(new MyRunnable(cyclicBarrier, 3000));
        thread3.setName("Thread3");
        thread1.start();
        thread2.start();
        thread3.start();
    }

    public static class MyRunnable implements Runnable{
        private int mSleepTime;
        private CyclicBarrier mCyclicBarrier;
        public MyRunnable(CyclicBarrier cyclicBarrier,int sleepTime) {
            mSleepTime = sleepTime;
            mCyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            try {
                Log.e("Test", "run "+Thread.currentThread().getName());
                Thread.sleep(mSleepTime);
                Log.e("Test", Thread.currentThread().getName()+" await");
                int index = mCyclicBarrier.await();
                Log.e("Test", Thread.currentThread().getName()+" end "+index);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
```

```
19:40:23.017 E/Test: run Thread1
19:40:23.017 E/Test: run Thread2
19:40:23.017 E/Test: run Thread3
19:40:24.017 E/Test: Thread1 await
19:40:25.017 E/Test: Thread2 await
19:40:26.019 E/Test: Thread3 await
19:40:26.020 E/Test: run barrier thread
19:40:26.020 E/Test: Thread3 end 0
19:40:26.021 E/Test: Thread1 end 2
19:40:26.024 E/Test: Thread2 end 1
```

所有的线程都到达等待点后，会和 Barrier 线程再一起往下执行。

## 源码分析

下面来简单的分析一下源码：

```
public class CyclicBarrier {
    private static class Generation {
        // 此变量来表示当前障碍是否突破
        boolean broken;         // initially false
    }

    //所有方法都通过这个锁来同步。
    private final ReentrantLock lock = new ReentrantLock();
    ////通过lock得到的一个状态变量
    private final Condition trip = lock.newCondition();
    // 构造方法中传入的总的等待线程的数量
    private final int parties;
    //当障碍突破后运行的程序，通过最后一个调用await的线程来执行
    private final Runnable barrierCommand;
    //当前的Generation。每当障碍失效或者突破之后都会自动替换掉。
    //从而实现重置的功能。
    private Generation generation = new Generation();
    
    //当前还未到达屏障的线程数量
    //当障碍突破或者重置后它的值会被重置为parties
    private int count;

    //重新生成新的Generation
    private void nextGeneration() {
        trip.signalAll();
        count = parties;
        generation = new Generation();
    }
    // 突破屏障
    private void breakBarrier() {
        generation.broken = true;
        count = parties;
        trip.signalAll();
    }

    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            int index = --count;
            // 开始突破障碍
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    // 执行参数传入的runnable
                    if (command != null)
                        command.run();
                    ranAction = true;
                    // 重新生成 Generation
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            // 
            for (;;) {
                try {
                    // 加入到等待队列并释放锁，让其它线程进入
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }

    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    public int getParties() {
        return parties;
    }
    // 返回值表示还未到达屏障的线程数量
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }

    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }

    // 重置屏障
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }
    // 已到达屏障的线程数量
    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
}
```
