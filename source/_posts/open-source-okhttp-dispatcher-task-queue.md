---
title: OkHttp 源码分析 -- 任务队列
categories: Android开源项目
comments: true
tags: [OkHttp]
description: 介绍 OkHttp 的任务队列
date: 2019-9-25 10:00:00
---


## 概述

OkHttp 的任务队列在内部维护了一个线程池用于执行具体的网络请求，达到方便高效地进行异步请求的目的。在 OkHttp 内部，这个工作是由 Dispatcher 来完成的。
关于线程池的知识，请参考我的博客
[Java 线程池 -- 线程池基础](http://www.heqiangfly.com/2015/09/01/java-thread-thread-pool-basic/) 
[Java 线程池 -- ThreadPoolExecutor 源码解析](http://www.heqiangfly.com/2015/09/06/java-thread-sourcecode-of-threadpoolexecutor/)
[Java 线程池 -- ThreadPoolExecutor 使用攻略](http://www.heqiangfly.com/2015/09/07/java-thread-how-to-use-threadpoolexecutor/)

## Dispatcher

Dispatcher 主要负责创建线程池以及为任务找到合适的执行线程。

```
public final class Dispatcher {
  //  异步队列中正在执行的最大任务个数
  private int maxRequests = 64;
  //  异步队列中相同 host
  正在执行的最大任务数
  private int maxRequestsPerHost = 5;
  // 当线程池中没有可执行任务时执行的空闲任务
  private @Nullable Runnable idleCallback;

  // 线程池
  private @Nullable ExecutorService executorService;

  // 等待执行的异步任务
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  // 正在执行的异步任务
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  // 正在执行的同步任务
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    // 构造线程池
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
  
  ......
  
}
```

Dispatcher 内部有三个任务队列，去执行同步任务和异步任务。
创建的线程池的 corePoolSize 为0，maximumPoolSize 为无限大，意味着线程数量可以无限大。
采用SynchronousQueue装等待的任务，这个阻塞队列没有存储空间，这意味着只要有请求到来，就必须要找到一条工作线程处理他，如果当前没有空闲的线程，那么就会再创建一条新的线程。
keepAliveTime为60S，意味着线程空闲时间超过60S就会被杀死。
也就是说，在实际运行中，当收到10个并发请求时，线程池会创建十个线程，当工作完成后，线程池会在60s后相继关闭所有线程。

## 同步请求

来看一下 `RealCall.execute()` 方法：

```
  @Override public Response execute() throws IOException {
    ......
    try {
      // 将任务加入分发器的队列
      client.dispatcher().executed(this);
      // 执行任务
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      // 通知dispatcher任务已完成，对应任务移除队列
      client.dispatcher().finished(this);
    }
  }
```

```
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

`executed` 方法做的事情就是把任务加到正在执行的同步任务队列中去。

```
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      // 从对应队列中移除
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      // 看是否要执行等待队列中的任务，同步任务不会走这里
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }
    // 如果没有需要执行的任务就执行空闲任务
    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }
```

## 异步请求

来看一下 `RealCall.enqueue()` 方法：

```
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    // 创建一个 AsyncCall 对象
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

`Dispatcher.enqueue()`

```
  synchronized void enqueue(AsyncCall call) {
    // 如果正在执行的异步队列没有超出限制，而且正在执行的同一个host的任务也没有超出限制
    // 就加入到正在执行的异步队列中，并且执行 AsyncCall
    // 否则就加入到异步等待队列中
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```

`AsyncCall.execute()` 方法：

```
    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        // 执行网络请求任务
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          // 执行失败回调
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        // 移除对应队列
        client.dispatcher().finished(this);
      }
    }
```

```
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }
  
  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    ......
      // promoteCalls参数为true，需要执行 promoteCalls 来执行待执行的任务
      if (promoteCalls) promoteCalls();
    ......
  }
  
  private void promoteCalls() {
    // 正在执行的异步队列超容，直接返回
    if (runningAsyncCalls.size() >= maxRequests) return; 
    // 没有需要执行的等待队列，返回
    if (readyAsyncCalls.isEmpty()) return; 
    // 开始遍历等待队列
    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();
      // 如果符合同一host请求数限制，就加入到正在执行的队列，并去执行这个任务
      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }
      // 如果正在执行的队列加满了，就返回
      if (runningAsyncCalls.size() >= maxRequests) return;
    }
  }
```

## 总结
