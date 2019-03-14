---
title: Java 线程池 -- ThreadPoolExecutor 源码解析
categories:  Java 并发
comments: true
tags: [线程池]
description: ThreadPoolExecutor 源码解析
date: 2015-9-6 10:00:00
---

## 构造方法以及参数解析

先来看一下 `ThreadPoolExecutor` 的几个构造方法：

 - ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue)
 - ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory)
 - ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,RejectedExecutionHandler handler)
 - ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory, RejectedExecutionHandler handler)

下面来解释一下这结果参数的意义：

 - corePoolSize：核心线程的数量。
  - 核心线程默认是没有超时的，也就是说就算线程闲置，也不会被处理。但是如果设置了 `allowCoreTimeOut` 为true，那么当核心线程闲置时，会被回收。
  - 当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的核心线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池核心线程的数量时就不再创建。如果调用了线程池的 `prestartAllCoreThreads` 方法，线程池会提前创建并启动所有核心线程。
  - 核心线程在创建 Worker 的时候会赋值 firstTask。
 - maximumPoolSize：线程池的允许创建的最大线程数量，被 `CAPACITY` 限制，最大职能是 2^29-1。
  - 当 corePoolSize =< 线程数 < maximumPoolSize，且任务队列已满时。线程池会创建新线程来处理任务。
  - 当 maximumPoolSize < 线程数时，且任务队列已满时，线程池会拒绝处理任务而抛出异常。
 - keepAliveTime：线程活动保持时间。线程池的工作线程空闲后，保持存活的时间。
 - unit：线程活动保持时间的单位。
 - workQueue：用于保存等待执行的任务的阻塞队列。
 - threadFactory：设置创建线程的工厂。可用于统一设置线程的一些属性。
 - handler：线程池的拒绝策略。当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。下面介绍源码中给出的四种实现。
  - AbortPolicy：直接抛出 `RejectedExecutionException` 异常。这是 `ThreadPoolExecutor` 的默认策略。
  - CallerRunsPolicy：直接在调用者所处的线程来执行任务。
  - DiscardOldestPolicy：丢弃队列中最早创建的任务，并尝试重新触发执行一次线程池任务。
  - DiscardPolicy：直接丢弃该任务，不做任何处理。
  - 也可以根据应用场景需要来实现 `RejectedExecutionHandler` 接口自定义策略。


## 概念解释


### 任务队列

任务队列就是用于保存等待执行的任务队列。
通过前面的介绍我们可以了解到线程池创建线程的流程：

 - 当线程数 < corePoolSize 时，线程池会创建一个线程来执行任务，即使其他空闲的核心线程能够执行新任务也会创建线程。
 - 当 corePoolSize =< 线程数 < maximumPoolSize，且任务队列已满时。线程池会创建新线程来处理任务。
 - 当 maximumPoolSize < 线程数时，且任务队列已满时，线程池会拒绝处理任务而抛出异常。

因此，任务队列的类型也会影响到线程创建逻辑。
在源码中任务队列的类型是：

```
    private final BlockingQueue<Runnable> workQueue;
```

通过实现 `BlockingQueue` 接口，我们可以自定义任务队列。

 - SynchronousQueue：`Executors.newCachedThreadPool()` 创建的是这种类型的队列。它也是无界队列。它的特点是在某次添加任务后必须等待其他线程取走后才能继续添加。
 - LinkedBlockingQueue：`Executors.newSingleThreadExecutor` 和 `Executors.newFixedThreadPool` 创建的是这种类型的队列。它也是无界队列。使用这种队列有可能造成会堆积大量的请求，从而导致 OOM。
 - DelayedWorkQueue：`Executors.newScheduledThreadPool` 创建的是这种类型的队列。
 - ArrayBlockingQueue：它是有界队列，队列的最大值可以指定，有助于防止资源耗尽。


### Worker

Worker 就是线程池中任务的执行者，它的数量为 corePoolSize 的大小。
Worker 相当于对任务和线程的一层包装，它可以控制一些在线程执行过程中的中断操作。
Worker 继承自 `AbstractQueuedSynchronizer` 并且实现了 `Runnable`。

```
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        private static final long serialVersionUID = 6138294804551838833L;
        // Worker 运行的线程
        final Thread thread;
        // Worker 创建成功后执行的第一个任务。核心线程不为空，非核心线程为空
        // 那么，如果firstTask执行完后，还会继续获取等待队列中的其他任务来执行。
        Runnable firstTask;
        volatile long completedTasks;

        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            // 调用线程工厂创建线程
            this.thread = getThreadFactory().newThread(this);
        }

        public void run() {
            // Worker 开始工作
            runWorker(this);
        }

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }

```

其实核心方法就是 `runWorker(Worker w)` 方法，这个方法会在后面的源码分析中介绍。

## 源码解析

当 `ThreadPoolExecutor` 创建好后，就会调用 `execute(Runnable command)` 方法来执行线程。

```
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        // 如果当前线程池中线程的数量小于 corePoolSize
        // 那么就直接创建线程添加到worker中，成功后直接返回
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 如果任务可以成功的添加到队列中
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 这里还要做二次确认
            // 如果线程池不在运行了，而且成功的将该任务移除
            // 执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果在运行阶段，但是Worker数量为0
            // 则添加到worker中
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 如果无法添加到队列中，而且无法添加以非核心线程的形式添加 Worker
        // 则执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

下面再来看一下 `addWorker` 方法，该方法的主要作用就是

### 创建Worker

```
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

### 执行任务

执行任务是通过 Worker 的 run 方法发起的，核心方法就是 `runWorker(Worker w)`：

```
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        // 获取创建 Worker 时获得的 firstTask
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 如果 firstTask 不为空，或者是可以从任务队列中获取任务
            // getTask() 是个阻塞方法，后面详细介绍
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 确认线程池和线程的工作状态
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 执行 beforeExecute 方法，这个方法默认是个空实现
                    // 可以在子类中定制
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 执行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 执行 beforeExecute 方法，这个方法默认是个空实现
                        // 可以在子类中定制
                        afterExecute(task, thrown);
                    }
                } finally {
                    // 执行执行完毕
                    // task赋值为null
                    task = null;
                    // 增加Worker执行的任务数
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

`runWorker(Worker w)` 方法的任务就是来执行线程池中的任务，对于核心线程来说，当 firstTask 执行完毕后，它还会通过 `getTask()` 方法获取等待队列中的任务来执行，该方法会根据 `allowCoreThreadTimeOut` 的配置和当前核心线程的数量来决定是否阻塞，根据 `keepAliveTime` 的时间来决定阻塞多久。
下面就来看一下 `getTask()` 方法。

```
    private Runnable getTask() {
        boolean timedOut = false; // 初始值设置 timedOut 为false

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // 如果设置了 allowCoreThreadTimeOut 或者是 Worker的数量大于 corePoolSize
            // 才会允许超时返回null
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 如果Worker的数量大于 maximumPoolSize 或者是允许超市且已经超时返回过一次
            // 而且此时Worker数量大于1或者是等待队列已经没有任务了
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                // 此时将 Workder的数量减1，并返回 null，在这种情况下Worker线程会退出。
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                // 根据 timed，如果允许超时，则调用 poll 方法，并根据 keepAliveTime
                // 设置超时返回
                // 如果不允许超时，那么就调用 take 方法，会一直阻塞，直到由新任务出现
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                // 执行到这里说明是超时返回了，进行下一个循环
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```