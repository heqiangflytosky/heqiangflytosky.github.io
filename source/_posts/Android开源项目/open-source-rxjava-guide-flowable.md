---
title: RxJava 使用指南 -- Flowable 和 Subscriber
categories: Android开源项目
comments: true
tags: [Android开源项目, RxJava]
description: 本文介绍 RxJava 中 Flowable 和 Subscriber的用法
date: 2017-10-14 10:00:00
---

## 概述

前面文章已经介绍过，Flowable/Subscriber 也是一对观察者模式的组合，和 Observable/Observer 的区别是 Flowable/Subscriber 是支持背压的，背压是个什么呢？

## Backpressure（背压）

背压是指在异步场景中，被观察者发送事件速度远快于观察者的处理速度的情况下，一种告诉上游的被观察者降低发送速度的策略。
在 Observable/Observer 组合的使用中是不支持背压的，下面通过一个例子来看一下这种场景：

```java
        Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                int i = 0;
                while(i < Long.MAX_VALUE){
                    e.onNext(i);
                    i++;
                }
            }
        }).subscribeOn(Schedulers.newThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Thread.sleep(1000);
                        Log.e("Test","i = "+integer);
                    }
                });
```

这里上游数据发射和下游的数据处理在各自的独立线程中执行，如果在同一个线程中不存在背压的情形。下游对数据的处理会堵塞上游数据的发送，上游发送一条数据后会等下游处理完之后再发送下一条。
在例子中，上游发射数据时，并不知道下游数据有没有处理完，就会源源不断的发射数据，而下游数据会间隔两秒钟才处理一次，这样就会产生很多下游没来得及处理的数据，这些数据既不会丢失，也不会被垃圾回收机制回收，而是存放在一个异步缓存池中，如果缓存池中的数据一直得不到处理，越积越多，最后就会造成内存溢出，这便是 Rxjava 中的背压问题。
可以通过 Monitors 发现内存使用快速增长。

![效果图](/images/open-source-rxjava-guide-flowable/observer.png)

## Flowable

`Flowable` 就是为了解决背压问题的产物，因此才会把它们和 Observable/Observer 区分开来使用。
由于基于Flowable发射的数据流，以及对数据加工处理的各操作符都添加了背压支持，附加了额外的逻辑，其运行效率要比 `Observable` 低得多。
因为只有上下游运行在各自的线程中，且上游发射数据速度大于下游接收处理数据的速度时，才会产生背压问题。
所以，如果能够确定上下游在同一个线程中工作，或者上下游工作在不同的线程中，而下游处理数据的速度高于上游发射数据的速度，则不会产生背压问题，就没有必要使用 `Flowable`，以免影响性能。
`Flowable` 

```java
        Flowable.create(new FlowableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull FlowableEmitter<Integer> e) throws Exception {
                int i = 0;
                while(i < Long.MAX_VALUE){
                    e.onNext(i);
                    i++;
                }
            }
        }, BackpressureStrategy.DROP)
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(Long.MAX_VALUE);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        try {
                            Thread.sleep(100);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        Log.e("Test","i = "+integer);
                    }

                    @Override
                    public void onError(Throwable t) {

                    }

                    @Override
                    public void onComplete() {

                    }
                });
```

这里注意三点：

 1. `Flowable.create` 参数中多了个 `BackpressureStrategy`。
 2. `onSubscribe` 回调的参数不是 `Disposable` 而是 `Subscription`。而且需要调用 `Subscription.request` 发起数据请求，否则Subscriber不会接受数据。
 3. 数据发射器是 `FlowableEmitter` 而不是 `ObservableEmitter`。

![效果图](/images/open-source-rxjava-guide-flowable/drop.png)

打印结果：

```
E/Test: i = 110
E/Test: i = 111
E/Test: i = 112
E/Test: i = 113
E/Test: i = 114
E/Test: i = 115
E/Test: i = 116
E/Test: i = 117
E/Test: i = 118
E/Test: i = 119
E/Test: i = 120
E/Test: i = 121
E/Test: i = 122
E/Test: i = 123
E/Test: i = 124
E/Test: i = 125
E/Test: i = 126
E/Test: i = 127
E/Test: i = 19044130
E/Test: i = 19044131
E/Test: i = 19044132
E/Test: i = 19044133
E/Test: i = 19044134
E/Test: i = 19044135
E/Test: i = 19044136
E/Test: i = 19044137
E/Test: i = 19044138
```

可以看到 127 —— 19044130 中间的数据被丢掉了，这是因为前面128条数据是正常发射的，后面的数据由于异步缓存池处于存满的状态而无法接收，当清理缓存池时上游正在发射19044130，此时可以放入缓存池从而可以正常接收。

## BackpressureStrategy（背压策略）

`Flowable` 的异步缓存池不同于 `Observable`，`Observable`的异步缓存池没有大小限制，可以无限制向里添加数据，直至OOM,而 `Flowable` 的异步缓存池有个固定容量，其大小为128。
`BackpressureStrategy` 的作用便是用来设置 `Flowable` 异步缓存池中的存储数据超限时的策略。
`BackpressureStrategy` 提供了一下几种背压策略：

 - MISSING：这种策略模式下相当于没有指定任何的背压策略，不会对数据做缓存或丢弃处理，需要下游通过背压操作符（onBackpressureBuffer()/onBackpressureDrop()/onBackpressureLatest()）指定背压策略。
 - ERROR：这种策略模式下如果缓存池中的数据超限了，则会抛出 `MissingBackpressureException` 异常
 - BUFFER：这种策略模式下没有为异步缓存池限制大小，可以无限制向里添加数据，不会抛出 `MissingBackpressureException` 异常，但会导致OOM。
 - DROP：这种策略模式下如果异步缓存池满了，会丢掉将要放入缓存池中的数据。
 - LATEST：这种策略模式下与 Drop 策略一样，如果缓存池满了，会丢掉将要放入缓存池中的数据，不同的是，不管缓存池的状态如何，LATEST都会将最后一条数据强行放入缓存池中。

## 背压操作符

RxJava 提供了下面的操作符来指定背压策略。

 - onBackpressureBuffer()：对应BUFFER策略
 - onBackpressureDrop()：对应DROP策略
 - onBackpressureLatest()：对应LATEST策略

因此下面代码效果是等同的：

```java
        Flowable.create(new FlowableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull FlowableEmitter<Integer> e) throws Exception {
                int i = 0;
                while(i < 800){
                    e.onNext(i);
                    i++;
                }
            }
        }, BackpressureStrategy.DROP)
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Thread.sleep(100);
                        Log.e("Test","i = "+integer);
                    }
                });
```

```java
        Flowable.range(0, 800)
                .onBackpressureDrop()
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Thread.sleep(100);
                        Log.e("Test","i = "+integer);
                    }
                });
```

## Subscription

前我们已经介绍过，`Flowable` 的 `subscribe` 方法需要的参数是 `Subscription`。

```java
public interface Subscription {
    public void request(long n);
    public void cancel();
}
```

### request

`request(long n)` 用于发起接收数据的请求，如果不调用这个方法，虽然被观察者会正常发送数据，但是观察者是不会去接收数据的。参数 `n` 代表请求的数据量。
但是要注意一点，上游数据的发送是不受这个影响的，无论你设置多少，上游数据都正常发送。

```java
        Flowable.create(new FlowableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull FlowableEmitter<Integer> e) throws Exception {
                int i = 0;
                while(i < 100){
                    e.onNext(i);
                    i++;
                    Thread.sleep(100);
                }
            }
        }, BackpressureStrategy.DROP)
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.newThread())
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(10);
                    }

                    @Override
                    public void onNext(Integer integer) {
                        try {
                            Thread.sleep(100);
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                        Log.e("Test","i = "+integer);
                    }

                    @Override
                    public void onError(Throwable t) {
                    }

                    @Override
                    public void onComplete() {
                    }
                });
```

上面的代码只完成了 10 条数据的接收。`request(long n)`是可以累加的，比如下面代码可以完成 20 条数据的接收。

```java
                    @Override
                    public void onSubscribe(Subscription s) {
                        s.request(10);
                        s.request(10);
                    }
```

### cancel

`cancel()` 方法用于取消订阅关系。

## FlowableEmitter

`FlowableEmitter` 有如下方法：

 - setDisposable：设置Disposable
 - setCancellable：设置Cancellable
 - requested：当前未完成的请求数量
 - isCancelled：订阅关系是否取消
 - serialize：
 - tryOnError：
