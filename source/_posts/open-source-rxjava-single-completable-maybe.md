---
title: RxJava 使用指南 -- Single、Completable 和 Maybe 的用法
categories: Android开源项目
comments: true
tags: [Android开源项目, RxJava]
description: 本文介绍 RxJava 中 Single、Completable 和 Maybe 的用法
date: 2017-10-18 10:00:00
---

## 概述

前面在 [RxJava 使用指南（一）-- 基本概念、数据流创建和线程调度 ](http://www.heqiangfly.com/2017/10/10/open-source-rxjava-guide-base/) 一文中简单介绍了几种观察者模式组合的组合。

 - Observable/Observer
 - Flowable/Subscriber
 - Single/SingleObserver
 - Completable/CompletableObserver
 - Maybe/MaybeObserver

其中 `Observable/Observer` 和 `Flowable/Subscriber` 比较常用，前面也用了大量篇幅来介绍，本文就来介绍一下 `Single/SingleObserver`、`Completable/CompletableObserver` 和 `Maybe/MaybeObserver` 这三个组合的用法。

## Single/SingleObserver 的用法

先来看一下代码中如何实现：

```
        Single.create(new SingleOnSubscribe<String>() {
            @Override
            public void subscribe(@NonNull SingleEmitter<String> e) throws Exception {
                e.onSuccess("Success");
                //e.onError(new Throwable("Error"));
            }
        }).subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new SingleObserver<String>() {
                    @Override
                    public void onSubscribe(@NonNull Disposable d) {

                    }

                    @Override
                    public void onSuccess(@NonNull String s) {
                        Log.e("Test","onSuccess "+s);
                    }

                    @Override
                    public void onError(@NonNull Throwable e) {

                    }
        });
```

我们先从代码上来分析 `Single/SingleObserver` 和 `Observable/Observer` 的区别。
`SingleEmitter` 和 `ObservableEmitter` 的区别：
`SingleEmitter` 发射数据的方法只有 `onSuccess()` 和 `onError()`，而 `ObservableEmitter` 多了个 `onNext()` 方法。
再来看一下 `SingleObserver` 和 `Observer` 的区别：
`SingleObserver` 处理结果的方法有 `onSubscribe()`、`onSuccess()` 和 `onError()`。而 `Observer` 多了个 `onNext()` 和 `onComplete()` 方法，而没有 `onSuccess()` 方法。
结合 `Single` 本身的名字，我们可以联想到，这个组合是只能发射单个数据或者一条异常通知，不能发射完成通知，其中数据与通知只能发射一个。
那么 `Single/SingleObserver` 有什么使用场景呢？
我们在实际应用中，有时候需要发射的数据并不是数据流的形式，而只是一条单一的数据，比如发起一次网络请求。在这种情况下，如果我们使用 `Observable`，`onComplete` 会紧跟着 `onNext` 被调用，为什么不能将这连个方法合二为一呢。如果再这种情况下我们再使用 `Observable` 就显得有点大材小用，因为我们不需要处理 `onNext()` 的数据。于是，为了满足这种单一数据的使用场景，便出现了 `Single`。
### 转化为其他观察者模式
`Single` 基本上实现了 `Observable` 所有的操作符，如果你发现需要用到一个 `Observable` 的操作符而 `Single` 并不支持，你可以用 `toObservable` 操作符把 `Single<T>` 转换为 `Observable<T>`。
另外 `Single` 还提供了其他转换方法：

 - toCompletable()
 - toMaybe()
 - toFlowable()
 - toFuture()

## Completable/CompletableObserver 的用法

先通过代码来看一下用法：

```
        Completable.create(new CompletableOnSubscribe() {
            @Override
            public void subscribe(@NonNull CompletableEmitter e) throws Exception {
                e.onComplete();
                //e.onError(new Throwable("Error"));
            }
        }).subscribeOn(AndroidSchedulers.mainThread())
                .observeOn(Schedulers.io())
                .subscribe(new CompletableObserver() {
                    @Override
                    public void onSubscribe(@NonNull Disposable d) {

                    }

                    @Override
                    public void onComplete() {
                        Log.e("Test", "onComplete");
                    }

                    @Override
                    public void onError(@NonNull Throwable e) {
                        Log.e("Test", "onError"+e.toString());
                    }
                });
```

照例先来看一下 `CompletableEmitter`：
它提供的数据和通知的方法如下：

 - onComplete()
 - onError()

`CompletableObserver` 的相关的方法：

 - onSubscribe()
 - onComplete()
 - onError()

可以看到，这里面没有数据处理的方法，只有通知相关的方法。它只发射一条完成通知，或者一条异常通知，不能发射数据，其中完成通知与异常通知只能发射一个。
那么 `Completable/CompletableObserver` 有什么使用场景呢？
和前面 `Single/SingleObserver` 的用法比较类似，只是这里不对数据进行处理，只有个通知的结果。比如：我们向服务器发起一个更新数据的请求，服务器更新数据以后是返回的是更新的结果。这个时候我们或许只是关心的是服务器更新数据是否成功，而不需要对数据进行处理，那么这个时候用 `Completable/CompletableObserver`  就可以了。
`Completable` 也提供了对 `Observable`、`Flowable`、`Single` 和 `Maybe` 的转换。

## Maybe/MaybeObserver 的用法

再来看一下 `Maybe` 的用法：

```
        Maybe.create(new MaybeOnSubscribe<String>(){
            @Override
            public void subscribe(@NonNull MaybeEmitter<String> e) throws Exception {
                e.onSuccess("onSuccess");
                //e.onError(new Throwable("Error"));
                //e.onComplete();
            }
        }).subscribeOn(AndroidSchedulers.mainThread())
                .observeOn(Schedulers.io())
                .subscribe(new MaybeObserver<String>() {
                    @Override
                    public void onSubscribe(@NonNull Disposable d) {

                    }

                    @Override
                    public void onSuccess(@NonNull String s) {
                        Log.e("Test","onSuccess");
                    }

                    @Override
                    public void onError(@NonNull Throwable e) {
                        Log.e("Test","onError");
                    }

                    @Override
                    public void onComplete() {
                        Log.e("Test","onComplete");
                    }
                });
```

`Maybe`可发射一条单一的数据，以及发射一条完成通知，或者一条异常通知，其中完成通知和异常通知只能发射一个，发射数据只能在发射完成通知或者异常通知之前，否则发射数据无效。
