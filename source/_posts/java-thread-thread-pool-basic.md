
---
title: Java 线程池 -- 线程池基础
categories: Android 线程
comments: true
tags: [Android 线程]
description: 本文介绍 Java 多线程编程中线程池的使用
date: 2015-9-20 10:00:00
---

## 概述

谈到 Java 多线程，就不得
不提起线程池，线程池的运用为线程生命周期的开销和资源不足问题提供了解决方案。
Executor 框架是 Java 线程池的一种实现方案。
与 Executor 框架相关的类有：Executor，ExecutorService，AbstractExecutorService，ThreadPoolExecutor，ScheduledExecutorService，ScheduledThreadPoolExecutor，CompletionService，ExecutorCompletionService，Future，Callable ，Executors 等。

## Executor 框架

下面来看一下这些类的关系图：

![效果图](http://www.plantuml.com/plantuml/svg/XP5BJiOW58NdNGKRe2jmK3KwSAIf6q2uMbBADJoCWt_nVnFQbF8eGpZdWtFl6QnZnlb5TL8xCD-C0tdv1-uTciBL2EPFSkZObtM6SKUuOjQIn-sOseBwEHbWuXrHxVeclAAPtr3gLOh_7_a4mYiGwNEHvncNNmLEeZx_jIEv7i5F2laizS-71xyAEqCURfHceoPd4bpLZ3LXnVh-m0exwKf4TRML-v0kV_tIVouYXuFfCdkwxv2-NaUqT4eBjiQO3IekkEKjpXj5jqUbJUS0MlX5tG40)

### Executor

`Executor` 接口中之定义了一个方法 `execute（Runnable command）`，该方法接收一个 `Runable` 实例，它用来执行一个任务，即一个实现了 `Runnable` 接口的类。

```
public interface Executor {
    void execute(Runnable command);
}
```

### ExecutorService

`ExecutorService` 接口继承自 `Executor` 接口，它提供了更丰富的实现多线程的方法，比如，`ExecutorService` 不仅可以执行 `Runnable` 还可以执行 `Callable` 任务。
它还提供了关闭自己的方法，可以调用 `ExecutorService` 的 `shutdown()`方法来平滑地关闭 `ExecutorService`，调用该方法后，将导致 `ExecutorService` 停止接受任何新的任务且等待已经提交的任务执行完成(已经提交的任务会分两类：一类是已经在执行的，另一类是还没有开始执行的)，当所有已经提交的任务执行完毕后将会关闭 `ExecutorService`。
因此我们一般用该接口来实现和管理多线程。

```
public interface ExecutorService extends Executor {
  void shutdown();
  List<Runnable> shutdownNow();
  boolean isShutdown();
  boolean isTerminated();
  boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
  <T> Future<T> submit(Callable<T> task);
  <T> Future<T> submit(Runnable task, T result);
  Future<?> submit(Runnable task);
  <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;
  <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException;
  <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;
  <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
}
```

### AbstractExecutorService

`AbstractExecutorService` 实现了 `ExecutorService` 接口，具体实现了 `submit`、`doInvokeAny`、`invokeAny` 和 `invokeAll` 方法。
它是个抽象类，`execute` 和 `shutdown()` 等方法还需要实现类来实现。

### ThreadPoolExecutor

`ExecutorService` 的默认实现。

### ScheduledExecutorService

调度线程池 `ScheduledExecutorService` 继承自 `ExecutorService` 接口，增加了 `schedule`、`scheduleAtFixedRate`、`scheduleWithFixedDelay` 方法。
它可以实现一些任务的调度工作，比如：实现定时程序、实现循环任务等。

### ScheduledThreadPoolExecutor

`ScheduledExecutorService` 的实现类。

### CompletionService

异步获取并行任务执行结果的线程池接口。

```
public interface CompletionService<V> {
    Future<V> submit(Callable<V> task);
    Future<V> submit(Runnable task, V result);
    Future<V> take() throws InterruptedException;
    Future<V> poll();
    Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
}
```

### ExecutorCompletionService

实现了 `CompletionService` 接口，将 `Executor`（线程池）和 `BlockingQueue`（堵塞队列）结合在一起，同一时候使用Callable作为任务的基本单元，整个过程就是生产者不断把 `Callable` 任务放入堵塞队列，` Executor` 作为消费者不断把任务取出来运行，并返回结果。

### Executors

提供了一系列工厂方法用于创先线程池，返回的线程池都实现了 `ExecutorService` 接口。

 - ExecutorService newFixedThreadPool(int nThreads) 
 - ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory)
 - ExecutorService newSingleThreadExecutor()
 - ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory)
 - ExecutorService newCachedThreadPool()
 - ExecutorService newCachedThreadPool(ThreadFactory threadFactory)
 - ScheduledExecutorService newSingleThreadScheduledExecutor()
 - ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) 
 - ScheduledExecutorService newScheduledThreadPool(int corePoolSize)
 - ScheduledExecutorService newScheduledThreadPool(int corePoolSize, ThreadFactory threadFactory) {

上面的每个方法都有两个重载的方法，不同的是多了个 `ThreadFactory` 参数。这是个线程工厂类，为什么要使用线程工厂呢？其实就是为了统一在创建线程时设置一些参数，如是否守护线程。线程一些特性等，如优先级。通过这个 `TreadFactory` 创建出来的线程能保证有相同的特性。它首先是一个接口类，而且方法只有一个。就是创建一个线程。

```
public interface ThreadFactory {
    Thread newThread(Runnable r);
}
```

 - newFixedThreadPool
  - 创建固定数目线程的线程池。
  - 线程池中有可以重复使用的线程时就重复使用，没有的话就创建一个，但线程数量不能大于最大线程数。
  - 任意时间点，最多只能有固定数目的活动线程存在，此时如果有新的任务要运行，只能放在另外的队列中等待，直到线程池中某个任务执行完有空闲线程出现。
  - FixedThreadPool 没有 IDLE 机制。
 - newSingleThreadExecutor
  - 创建一个单线程化的Executor。
 - newCachedThreadPool
  - 创建一个可缓存的线程池，调用execute将重用以前构造的线程（如果线程可用）。如果现有线程没有可用的，则创建一个新线 程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。
  - 池线程数支持 0-Integer.MAX_VALUE(显然完全没考虑主机的资源承受能力）。
  - 能重用的线程，必须是 timeout IDLE 内的池中线程，缺省 timeout 是 60s，超过这个 IDLE 时长，空闲线程将被移出线程池。
 - newScheduledThreadPool
  - 创建一个可调度线程池
  - 这个池子里的线程可以按 schedule 依次 delay 执行，或周期执行
 - newSingleThreadScheduledExecutor
  - 创建一个单线程的可调度线程池

## 使用方法

### newFixedThreadPool

```
    public void testExecutor() {
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                Log.e("Test","MyThread run " +Thread.currentThread());

                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        for (int i = 0; i < 5; i++) {
            executorService.execute(runnable);
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }
```

```
21:58:00.796 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
21:58:00.896 E/Test: MyThread run Thread[pool-1-thread-2,5,main]
21:58:01.797 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
21:58:01.899 E/Test: MyThread run Thread[pool-1-thread-2,5,main]
21:58:02.799 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
```

可以看到，线程池在重复使用两个线程对象，同时只能有2个线程在执行，待执行的线程必须等到线程池中有空闲线程时才能执行。
如果把间隔事件改为大于一秒，执行结果为重复使用一个线程。

### newCachedThreadPool

```
    public void testExecutor2() {
        ExecutorService executorService = Executors.newCachedThreadPool();
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                Log.e("Test","MyThread run " +Thread.currentThread());

                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        for (int i = 0; i < 5; i++) {
            executorService.execute(runnable);
//            try {
//                Thread.sleep(1000  * 5);
//            } catch (InterruptedException e) {
//                e.printStackTrace();
//            }
        }
    }
```

```
20:51:31.532 E/Test: MyThread run Thread[pool-1-thread-2,5,main]
20:51:31.533 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
20:51:31.538 E/Test: MyThread run Thread[pool-1-thread-3,5,main]
20:51:31.539 E/Test: MyThread run Thread[pool-1-thread-5,5,main]
20:51:31.539 E/Test: MyThread run Thread[pool-1-thread-4,5,main]
```

添加上面代码中注释的部分，线程池执行任务的间隔为 5 秒，这种情况下当前任务执行完成后，下一个任务才开始执行：

```
20:52:12.837 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
20:52:17.839 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
20:52:22.841 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
20:52:27.842 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
20:52:32.846 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
```

可以看到，线程池中重用第一个线程。
下面把间隔事件改为 62 秒，为什么改为62秒呢？因为线程池设定的 IDLE timeout 是 60 秒，这种情况下肯定也是当前任务执行完后下一个任务才开始执行，但是这种情况下并没有重用线程。符合上面的分析，缓存中那些已有 60 秒钟未被使用的线程将被移除出线程池。

```
20:45:56.078 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
20:46:58.081 E/Test: MyThread run Thread[pool-1-thread-2,5,main]
20:48:00.086 E/Test: MyThread run Thread[pool-1-thread-3,5,main]
20:49:02.087 E/Test: MyThread run Thread[pool-1-thread-4,5,main]
20:50:04.088 E/Test: MyThread run Thread[pool-1-thread-5,5,main]
```

### ScheduledThreadPoolExecutor

```
    public void testExecutor() {
        ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(2);
        executor.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                Log.e("Test","MyThread run " +Thread.currentThread());
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, 0, 1000, TimeUnit.MILLISECONDS);

    }
```

```
22:58:28.472 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
22:58:30.474 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
22:58:32.477 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
22:58:34.480 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
22:58:36.483 E/Test: MyThread run Thread[pool-1-thread-1,5,main]
......
```

## 编程规范

关于多线程，在阿里巴巴出品的《Java 开发手册》中提到：

1. 创建线程或线程池时请指定有意义的线程名称，方便出错时回溯。

```
public class TimerTaskThread extends Thread { 
    public TimerTaskThread() {
        super.setName("TimerTaskThread");
        ... 
}
```

2.线程资源必须通过线程池提供，不允许在应用中自行显式创建线程。
> 说明:使用线程池的好处是减少在创建和销毁线程上所花的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者 “过度切换”的问题。

3.线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。 
> 说明:Executors 返回的线程池对象的弊端如下:
1)FixedThreadPool 和 SingleThreadPool:
允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM。 
2)CachedThreadPool 和 ScheduledThreadPool:
允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM。

<!--  

@startuml
interface Executor
interface ExecutorService
abstract class AbstractExecutorService
class ThreadPoolExecutor
class ForkJoinPool
interface ScheduledExecutorService

interface CompletionService
class ExecutorCompletionService

interface BlockingQueue

Executor <|-- ExecutorService
ExecutorService  <|-- ScheduledExecutorService
ScheduledExecutorService <|.. ScheduledThreadPoolExecutor

ExecutorService <|.. AbstractExecutorService
AbstractExecutorService <|-- ThreadPoolExecutor
AbstractExecutorService <|-- ForkJoinPool
ThreadPoolExecutor <|-- ScheduledThreadPoolExecutor

CompletionService <|.. ExecutorCompletionService
Executor <-- ExecutorCompletionService
AbstractExecutorService <-- ExecutorCompletionService
BlockingQueue <-- ExecutorCompletionService

ThreadPoolExecutor <.. Executors
ScheduledThreadPoolExecutor <.. Executors
@enduml

-->
