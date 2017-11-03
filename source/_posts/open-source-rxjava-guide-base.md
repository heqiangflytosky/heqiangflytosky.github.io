
---
title: RxJava 使用指南（一）-- 基本概念、数据流创建和线程调度
categories: Android开源项目
comments: true
tags: [Android开源项目, RxJava]
description: 本文介绍 RxJava 的基本概念以及简单使用，介绍 Observable 创建数据流的多种操作符的用法以及线程调度的方法和它们之间的区别。
date: 2017-10-10 10:00:00
---

## 概述

RxJava 很早就开始接触和使用了，但只是仅仅会一些简单的使用而已，于是打算通过一系列的博客来加深对RxJava的理解。
[RxJava Github地址](https://github.com/ReactiveX/RxJava/)
写这篇文章的时候，RxJava最新版本已经是 `2.1.5` 了，那么我们就以最新版本为基础来介绍 RxJava 的使用。
使用之前要加入一下依赖：

```
dependencies {
    compile 'io.reactivex.rxjava2:rxjava:2.1.5'
    compile 'io.reactivex.rxjava2:rxandroid:2.0.1'
}
```

RxAndroid 是一个 RxJava 扩展库，更好的兼容了 Android 特性，比如主线程，UI事件等。

## 基本概念

RxJava 官方的解释是一个在 Java VM 上使用可观测的序列来组成异步的、基于事件的程序的库。简要概括一下，它就是一个实现异步操作的库。它的本质体现在异步两个字上面。
RxJava 的异步的实现，是通过一种扩展的观察者模式来实现的，观察者模式相信我们都不陌生。
想要使用RxJava，我们先来了解一下几个基本概念。

 - Observable (可观察者，即被观察者)：发射数据流
 - Observer (观察者)：接收数据流
 - Consumer：它也是一个 Observer，只有一个 accept() 回调
 - subscribe (订阅)：建立 Observable 和 Observer 的联系
 - subscribeOn：为 Observable 对数据的处理指定一个调度器
 - observeOn：为下游对数据的操作指定一个调度器
 - Operators操作符：可以理解为对数据流的操作，包括创建、过滤、变换、组合、聚合等。



## 基本用法

正如我们实现一个基本的观察者模式一样，你要创建被观察者和观察者，然后通过订阅事件使他们联系起来。
下面介绍一个RxJava的最基本的实现：

### 创建Observable

```java
        Observable<String> observable = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<String> observableEmitter) throws Exception {
                observableEmitter.onNext("Hello ");
                observableEmitter.onNext("World");
                observableEmitter.onComplete();
            }
        });
```

`ObservableEmitter` 相当于一个事件发射器，每执行一次 `onNext()`，观察者就会收到一次数据，数据发送完毕后调用 `onComplete()` 方法。

### 创建Observer

```java
        Observer<String> observer = new Observer<String>() {
            @Override
            public void onSubscribe(@NonNull Disposable d) {

            }

            @Override
            public void onNext(@NonNull String s) {
                Log.i(TAG,"onNext : " + s);
            }

            @Override
            public void onError(@NonNull Throwable e) {

            }

            @Override
            public void onComplete() {
                Log.i(TAG,"onComplete");
            }
        };
```

观察者的 `onNext()` 回调会收到被观察者发送的数据。

### subscribe（订阅）

```java
observable.subscribe(observer);
```

执行后输出：

```
I/RxJava: onNext : Hello 
I/RxJava: onNext : World
I/RxJava: onComplete
```

通过上面三步实现了 RxJava 最简单的用法，其中并没有涉及到线程切换等操作，这些后面再介绍。

## 创建Observable

前面的例子介绍了使用 `Observable.create()` 操作符来创建 `Observable`，下面再介绍一下 RxJava 提供的其他方法。

### just()

`just(T item1, ...)`创建 `Observable` 并自动调用 `onNext()`发射数据，可以接受一个或者多个参数， `just()` 中传递的参数将在 `Observer` 的 `onNext()` 方法中接收到。

```java
        Observable.just("Hello", "World")
                .subscribe(new Observer<String>() {
                    @Override
                    public void onSubscribe(@NonNull Disposable d) {

                    }

                    @Override
                    public void onNext(@NonNull String s) {
                        Log.i(TAG,"onNext : " + s);
                    }

                    @Override
                    public void onError(@NonNull Throwable e) {

                    }

                    @Override
                    public void onComplete() {
                        Log.i(TAG,"onComplete");
                    }
        });
```

```
I/RxJava: onNext : Hello
I/RxJava: onNext : World
I/RxJava: onComplete
```

### defer()

`defer(Callable<? extends ObservableSource<? extends T>> supplier)` 当观察者订阅时，才创建 Observable。

```java
        Observable.defer(new Callable<ObservableSource<String>>() {
            @Override
            public ObservableSource<String> call() throws Exception {
                Log.i(TAG,"defer call");
                return Observable.just("Hello", "World");
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.i(TAG,"accept : " + s);
            }
        });
```

这里新建了一个 `Consumer` 对象来作为观察者。

### fromArray()

`fromArray(T... items)` 接受一个数组参数，创建 `Observable` 并自动调用 `onNext()` 将数组中的数据逐一发送。

```java
        Observable.fromArray(new String[]{"Hello","World"})
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        Log.i(TAG,"accept : " + s);
                    }
                });
```

### fromIterable()

`fromIterable(Iterable<? extends T> source)` 接受一个集合参数，创建 `Observable` 并将集合中的数据逐一发送。

```java
        ArrayList<String> list = new ArrayList();
        list.add("Hello");
        list.add("World");
        Observable.fromIterable(list)
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        Log.i(TAG,"accept : " + s);
                    }
                });
```

### interval()

`interval(long period, TimeUnit unit)` 按照一个固定的时间间隔 `period` 来发射数据，可以作为一个定时器来使用。

```java
        Observable.interval(2, TimeUnit.SECONDS)
                .subscribe(new Observer<Long>() {
                    @Override
                    public void onSubscribe(@NonNull Disposable d) {
                        mDisposable = d;
                    }

                    @Override
                    public void onNext(@NonNull Long aLong) {
                        Log.i(TAG,"onNext : " + aLong);
                        if(aLong == 5)
                            mDisposable.dispose();
                    }

                    @Override
                    public void onError(@NonNull Throwable e) {
                    }

                    @Override
                    public void onComplete() {
                    }
                });
```
上面例子中当数据等于5解除订阅关系，停止发射数据。

### range()

`range(final int start, final int count)` 创建一个被观察者并发射从 `start` 到 `count` 的整数序列给观察者。

```java
        Observable.range(1,5)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.i(TAG,"accept : " + integer);
                    }
                });
```

### timer()

`timer(long delay, TimeUnit unit)`创建一个 Observable 并在它在一个给定的延迟 `delay` 后发射一个特殊的值（0）给观察者。

```java
        Observable.timer(5, TimeUnit.SECONDS)
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(Long aLong) throws Exception {
                        Log.i(TAG,"accept : " + aLong);
                    }
                });
```

## 线程调度

### Scheduler（调度器）

在上面的例子中，并没有涉及到线程切换的操作，但是我们在使用过程中会经常遇到这种情况，比如，我们会将网络请求等耗时操作放到后台线程中，将UI操作放到主线程中执行。
RxJava 提供了线程调度的功能，我们可以借助于 `Scheduler` 来完成。另外 RxAndroid 提供了 `AndroidSchedulers` 调度器来供开发者使用。
`Scheduler` 和 `AndroidSchedulers` 提供了6种线程调度器：

| 调度器 | 使用场景 |
|:--------:|:-------------:|
| Schedulers.io() | 主要用于一些耗时IO操作，比如读写文件，数据库存取，网络交互等。这个调度器具有线程缓存机制，它会根据需要，增加或者减少线程池中的线程数量。需要注意的是Schedulers.io()中的线程池数量是无限制大的，大量的I/0操作将创建许多线程，我们需要在性能和线程数量中做出取舍。 |
| Schedulers.computation() | 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。 |
| Schedulers.newThread() | 开启一个新的线程，不具有线程缓存机制，因为创建一个新的线程比复用一个线程更耗时耗力，因此，Schedulers.newThread( )的效率没有Schedulers.io( )高。 |
| Schedulers.from(Executor executor) | 使用指定的 Executor 来作为线程调度器 |
| Schedulers.single() | 拥有一个线程单例，所有的任务都在这一个线程中执行。 |
| Schedulers.trampoline() | 在当前线程执行一个任务，但不是立即执行，用trampoline()将它加入队列。这个调度器将会处理它的队列并且按程序运行队列中每一个任务。 |
| AndroidSchedulers.mainThread() | Android中的主线程执行任务，为Android开发定制。 |

### 实现线程调度

实现线程的调度可以通过 `subscribeOn()` 和 `observerOn()` 实现。

 - subscribeOn()：指定对数据的处理在哪个调度器上执行，直到遇到observeOn改变线程调度器。
 - observerOn()：指定下游对数据的操作运行在哪个调度器上。

使用时需要注意：

 - `subscribeOn()` 可以多次调用，但只有第一次的调用会起作用。
 - `observerOn()` 可以多次调用，每调用一次切换一次线程。

#### 示例1

在这个例子中，我们通过 `subscribeOn(Schedulers.io())` 指定被观察者在IO线程中进行图片下载，然后通过 `observeOn(AndroidSchedulers.mainThread())` 在主线程中更新UI。

```java
        Observable.create(new ObservableOnSubscribe<Bitmap>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Bitmap> observableEmitter) throws Exception {
                Log.i(TAG,"current thread : "+Thread.currentThread().getName());
                Bitmap bitmap = mHttpModel.getBitmapSync(mUrl);
                observableEmitter.onNext(bitmap);
                observableEmitter.onComplete();
            }
        }).subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<Bitmap>() {
                    @Override
                    public void onSubscribe(@NonNull Disposable d) {
                    }

                    @Override
                    public void onNext(@NonNull Bitmap bitmap) {
                        Log.i(TAG,"current thread : "+Thread.currentThread().getName());
                        mImageView.setImageBitmap(bitmap);
                    }

                    @Override
                    public void onError(@NonNull Throwable e) {
                    }

                    @Override
                    public void onComplete() {
                    }
                });
```

结果：

```
RxJava: current thread : RxCachedThreadScheduler-1
RxJava: current thread : main
```

#### 示例2

这个例子主要来介绍一下线程调度的时机问题，被观察者所在的线程肯定是由 `subscribeOn()` 来指定，然后就直到遇到 `observeOn()` 再切换线程，否则就在当前线程执行下去。
看下面一段伪代码：

```java
Observable.create                           //被观察者在io线程执行，因为后面通过subscribeOn指定io线程
.map                                        //没有遇到线程操作，依然在io线程
.subscribeOn(Schedulers.io())
.map                                        //没有遇到线程操作，依然在io线程
.observeOn(AndroidSchedulers.mainThread())  //切换线程
.map                                        //遇到线程切换，在主线程
.observeOn(Schedulers.io())                 //切换线程
.subscribe                                  //遇到线程切换，在io线程
```

> 如果我们不指定线程调度器，被观察者和观察者会在什么线程执行呢？我们通过在前面的例子中添加一些打印信息会发现，它们会默认在当前线程中执行。
