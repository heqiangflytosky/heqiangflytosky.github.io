---
title: Java 线程池 -- ThreadPoolExecutor 使用以及优化攻略
categories:  Java 并发
comments: true
tags: [线程池]
description: ThreadPoolExecutor 使用以及优化攻略
date: 2015-9-7 10:00:00
---

在 [Java 线程池 -- 线程池基础](http://www.heqiangfly.com/2015/09/01/java-thread-thread-pool-basic/) 中提到，要谨慎使用 `Executors` 的几个方法来创建线程池，尽量使用 `ThreadPoolExecutor` 的方式，现在我们来介绍在使用 `ThreadPoolExecutor` 时需要注意的问题。
使用的关键是围绕 `ThreadPoolExecutor` 的几个参数：

 - corePoolSize：核心线程数
 - maximumPoolSize：最大线程数量
 - workQueue：等待队列
 - threadFactory：线程工厂
 - handler：线程池的拒绝策略

这些参数的具体介绍，请参考 [Java 线程池 -- 线程池基础](http://www.heqiangfly.com/2015/09/01/java-thread-thread-pool-basic/)。

## 如何选择核心线程数

关于核心线程数量的选择，需要考虑到线程池所进行的工作的性质，比如：是IO密集型还是计算密集型。
《Java 虚拟机并发编程》一书中给出的计算方法是：

线程数 = CPU可用核心数/(1-阻塞系数)，其中阻塞系数取值在0到1之间

计算密集型任务的阻塞系数为0，而IO密集型任务的阻塞系数则接近1。

```
CPU可用核心数 = Runtime.getRuntime().availableProcessors();
```

简单的分析来看，如果是CPU密集型的任务，我们应该设置数目较小的线程数，比如CPU数目加1。如果是IO密集型的任务，则应该设置可能多的线程数，由于IO操作不占用CPU，所以，不能让CPU闲下来。当然，如果线程数目太多，那么线程切换所带来的开销又会对系统的响应时间带来影响。

## 设置 workQueue 来优化线程池的执行顺序

我们都知道线程池执行流程是先corePool再workQueue，最后才是maxPool的一个执行流程。
如果一个线程池执行的任务属于IO密集型，CPU大多属于闲置状态，系统资源未充分利用。如果一瞬间来了大量请求，如果线程池数量大于coreSize时，多余的请求都会放入到等待队列中。等待着corePool中的线程执行完成后再来执行等待队列中的任务。
试想一下，这种场景我们该如何优化？
我们可以修改线程池的执行顺序为corePool->maxPool->workQueue。这样就能够充分利用CPU资源，提交的任务会被优先执行。当线程池中线程数量大于maxSize时才会将任务放入等待队列中。
通过 [Java 线程池 -- 线程池基础](http://www.heqiangfly.com/2015/09/01/java-thread-thread-pool-basic/)以及[Java 线程池 -- ThreadPoolExecutor 源码解析](http://www.heqiangfly.com/2015/09/06/java-thread-sourcecode-of-threadpoolexecutor/)，我们知道，当线程数量大于 corePoolSize 时，在 workQueue 未满的情况下，不管线程数量有没有大于 maximumPoolSize，并不会去创建新的线程。如果我们配置的 workQueue 是默认的 `new LinkedBlockingQueue<Runnable>()`，那么一般情况下（LinkedBlockingQueue允许队列长度为 Integer.MAX_VALUE），maximumPoolSize 参数是不起作用的，它将永远不会启动大于 corePoolSize 的第二个线程。
通过前面的源码分析，我们发现，`ThreadPoolExecutor.execute()` 方法判断请求队列是否有空间是用的 `offer` 方法，那么我们是不是可以覆盖 ThreadPoolExecutor 的 offer 方法，使其永远返回 false，这样当当前线程数大于核心线程数而小于最大线程数时，都会创建线程去执行。当当前线程数大于最大线程数时，回去执行拒绝策略，设置 RejectedExecutionHandler 调用实际的 offer 方法来决定是否执行拒绝策略，然后这里我们在拒绝策略里面才把任务加入到等待队列里面去。
这样就保证了在线程大于核心线程时，会继续创建线程达到 maximumPoolSize，然后才进入到请求队列等待。
这样也就相当于改变了线程池中线程的执行顺序为：corePool->maxPool->workQueue。这也是面试时关于线程池经常会问到的问题。

```
public class AppExecutors {
    private static final WorkQueue<Runnable> mIoWorkQueue = new WorkQueue<>();

    private static final RejectedExecutionHandler mIoRejectedPolicy = new RejectedExecutionHandler() {
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
            mIoWorkQueue.superOffer(r);
        }
    };

    private static final Executor diskIO = new ThreadPoolExecutor(CORE_POOL_SIZE,MAX_POOL_SIZE_IO,
            KEEP_ALIVE_TIME, TimeUnit.MILLISECONDS, mIoWorkQueue,
            new DefaultThreadFactory(THREAD_NAME_IO), mIoRejectedPolicy);

    public static Executor diskIO() {
        return diskIO;
    }

    private static class WorkQueue<E> extends LinkedBlockingQueue<E> {
        @Override
        public boolean offer(E e) {
            return false;
        }

        public void superOffer(E e) {
            super.offer(e);
        }
    }
}

```

上面是借助于等待队列把大于最大线程数的任务添加到等待队列中去，其实这个逻辑我们还可以在 offer 方法中去处理，如果当前线程数小于最大线程数时，则直接返回false，其他情况下返回 `super.offer(e)` 也可以实现，具体参考 dubbo中的 `EagerThreadPoolExecutor`。
https://github.com/apache/dubbo/blob/master/dubbo-common/src/main/java/org/apache/dubbo/common/threadpool/support/eager/TaskQueue.java

```
    @Override
    public boolean offer(Runnable runnable) {
        if (executor == null) {
            throw new RejectedExecutionException("The task queue does not have executor!");
        }

        int currentPoolThreadSize = executor.getPoolSize();
        // have free worker. put task into queue to let the worker deal with task.
        if (executor.getSubmittedTaskCount() < currentPoolThreadSize) {
            return super.offer(runnable);
        }

        // return false to let executor create new worker.
        if (currentPoolThreadSize < executor.getMaximumPoolSize()) {
            return false;
        }

        // currentPoolThreadSize >= max
        return super.offer(runnable);
    }
```

## 如何设置 threadFactory

创建线程或线程池时请指定有意义的线程名称，方便出错时回溯。

```
    private static class DefaultThreadFactory implements ThreadFactory {
        private final String namePrefix;

        DefaultThreadFactory(@NonNull String prefix) {
            namePrefix = prefix;
        }

        @Override
        public Thread newThread(@NonNull Runnable r) {
            Thread t = new Thread(r);
            t.setName(namePrefix + t.getId());
            if (t.isDaemon()) {
                t.setDaemon(false);
            }
            return t;
        }
    }
```

## 推荐文章

《java并发编程实践》
《Java 虚拟机并发编程》
[面试官：线程池如何按照 core、max、queue 的执行循序去执行？](https://mp.weixin.qq.com/s?__biz=MjM5NzMyMjAwMA==&mid=2651488151&idx=2&sn=40872a4979087378d893589ec1d52bfa&chksm=bd2513e88a529afee2e7296260da2882fc034efec1790544aba3f2ed46dc50df589b2475d8fd&scene=21#wechat_redirect)
http://www.imooc.com/wenda/detail/602949
https://www.jianshu.com/p/78c7df3c762d
https://blog.csdn.net/tbdp6411/article/details/78443732