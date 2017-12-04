---
title: RxJava 使用指南（二）-- 操作符介绍
categories: Android开源项目
comments: true
tags: [Android开源项目, RxJava]
description: 本文介绍 RxJava 的操作符的用法
date: 2017-10-12 10:00:00
---

## map 和 flatmap （变换）

这一组操作符提供数据的变换工作，就是把数据对象变换成其他类型的数据对象，它们都接受一个 `Function` 类型的参数。但是它们的用法上还是有区别的。

### map

先来说一下 `map()` 操作的使用场景，如果在数据流的传递过程中，我们需要根据当前数据流对象转换或提取生成一种其他类型的数据流对象，可以使用 `map()` 。
它需要一个 `Function` 参数，`Function` 对象的两个参数是转换的源数据和目标数据类型。
比如我们有一个请求网络图片的场景，被订阅者发出的数据是原始的 `byte` 类型数据，在设置给 `ImageView` 前我们要转换成 `Bitmap` 类型的数据，那么就可以用这个操作符。

```java
        Observable.create(new ObservableOnSubscribe<byte []>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<byte []> observableEmitter) throws Exception {
                byte [] data = getBitmapDataSync(mUrl);
                observableEmitter.onNext(data);
                observableEmitter.onComplete();
            }
        }).subscribeOn(Schedulers.io())
                .map(new Function<byte[], Bitmap>() {
                    @Override
                    public Bitmap apply(@NonNull byte[] bytes) throws Exception {
                        return generateBitmap(bytes);
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<Bitmap>() {
                    @Override
                    public void accept(Bitmap bitmap) throws Exception {
                        mImageView.setImageBitmap(bitmap);
                    }
                });
```

### flatmap

`flatMap(Function<? super T, ? extends ObservableSource<? extends R>> mapper)` 中同样需要一个 `Function` 对象作为参数，但是 `Function` 的目标数据类型变成了 `Observable`。
`flatMap` 一般用于输出一个 `Observable`，而其随后的 `subscribe` 中的参数也跟 `Observable` 中的参数一样。
下面再提供一个使用场景，这个场景属于嵌套的网络请求，比如我们想先进行一次网络请求得到图片的url，然后根据url再进行网络请求得到图片，最后设置给 `ImageView` ，这种情况下由url到 `Bitmap` 的转换用 `map` 是无法实现的，可以使用 `flatmap`。

```java
        Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<String> observableEmitter) throws Exception {
                String url = getImageUrl(mUrl);
                observableEmitter.onNext(url);
                observableEmitter.onComplete();
            }
        }).subscribeOn(Schedulers.io())
                .observeOn(Schedulers.io())
                .flatMap(new Function<String, ObservableSource<Bitmap>>() {
                    @Override
                    public ObservableSource<Bitmap> apply(@NonNull final String url) throws Exception {
                        return Observable.create(new ObservableOnSubscribe<Bitmap>() {
                            @Override
                            public void subscribe(@NonNull ObservableEmitter<Bitmap> observableEmitter) throws Exception {
                                Bitmap bitmap = getBitmap(url);
                                observableEmitter.onNext(bitmap);
                                observableEmitter.onComplete();
                            }
                        });
                    }
                })
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<Bitmap>() {
                    @Override
                    public void accept(Bitmap bitmap) throws Exception {
                        mImageView.setImageBitmap(bitmap);
                    }
                });
```

## filter 和 distinct （过滤）

这一组操作符提供数据的过滤工作，`filter` 对不符合条件的数据进行过滤，`distinct` 提供去重的功能。最常用的用法之一是过滤 null 对象,它帮我们免去了在 `onNext()` 函数调用中再去检测 null 值。

### filter

`filter(Predicate<? super T> predicate)` 接受 `Predicate` 对象参数，它的 `test()` 方法给出一个过滤条件，如果满足条件，则继续向下传递，如果不满足，则过滤掉。

```java
        Observable.range(0,10)
                .filter(new Predicate<Integer>() {
                    @Override
                    public boolean test(@NonNull Integer integer) throws Exception {
                        return integer % 2 == 0;
                    }
                }).subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                Log.i(TAG,"accept : " + integer);
            }
        });
```

上面的示例会过滤掉奇数，把偶数打印出来。

### distinct

`distinct()` 过滤掉重复的数据项，过滤规则为：只允许还没有发射过的数据项通过。它还有重载的两个方法 `distinct(Function<? super T, K> keySelector)` 和 `distinct(Function<? super T, K> keySelector, Callable<? extends Collection<? super K>> collectionSupplier)`

```java
        Observable.just(1, 2, 2, 2, 3, 4, 5, 5, 6, 6)
                .distinct()
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.i(TAG,"accept : " + integer);
                    }
                });
```

## mergeWith 和 concatWith （组合）

这一组操作符提供数据的组合工作。

### mergeWith

`mergeWith(ObservableSource<? extends T> other)` 合并两个 `Observable`，它们数据可能会交错发射。

```java
        Observable.just(1, 2, 3, 4, 5)
                .mergeWith(Observable.just(6, 7, 8, 9, 10))
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.i(TAG,"accept : " + integer);
                    }
                });
```


### concatWith

`concatWith(ObservableSource<? extends T> other)`合并两个 `Observable`，它们数据会按顺序发射，一个 `Observable` 的数据发送完了另外一个才会发射。

```java
        Observable.just(1, 2, 3, 4, 5)
                .concatWith(Observable.just(6, 7, 8, 9, 10))
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.i(TAG,"accept : " + integer);
                    }
                });
```

## zipWith（聚合）

`zipWith(ObservableSource<? extends U> other, BiFunction<? super T, ? super U, ? extends R> zipper)` 将两个 `Obversable` 发射的数据，通过一个函数 `BiFunction` 的 `apply()` 方法对对应位置的数据处理后放到一个新的 `Observable` 中发射，所发射的数据个数与最少的 `Observabel` 中的一样。

```java
        Observable.just(1, 2, 3, 4, 5, 6)
                .zipWith(Observable.just("one", "two", "three", "four", "five"), new BiFunction<Integer, String, String>() {
                    @Override
                    public String apply(@NonNull Integer integer, @NonNull String s) throws Exception {
                        return integer + s;
                    }
                }).subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.i(TAG,"accept : " + s);
            }
        });
```

打印结果：

```
I/RxJava: accept : 1one
I/RxJava: accept : 2two
I/RxJava: accept : 3three
I/RxJava: accept : 4four
I/RxJava: accept : 5five
```

## take、 takeLast、takeUntil 和 takeWhile

### take

`take(long count)` 观察者只接受被观察者发出的前 `count` 个数据。

```java
        Observable.just(1, 2, 3, 4, 5, 6)
                .take(3)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.i(TAG,"accept : " + integer);
                    }
                });
```

### takeLast

`takeLast(int count)` 观察者只接受被观察者发出的后面 `count` 个数据。

```java
        Observable.just(1, 2, 3, 4, 5, 6)
                .take(3)
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.i(TAG,"accept : " + integer);
                    }
                });
```

### takeUntil

`takeUntil(Predicate<? super T> stopPredicate)` 当条件满足是停止发射数据，`takeUntil(ObservableSource<U> other)` 当 other 发射第一个数据后即停止第一个 `Observable` 数据的发射。

```java
        Observable.just(1, 2, 3, 4, 5, 6)
                .takeUntil(new Predicate<Integer>() {
                    @Override
                    public boolean test(@NonNull Integer integer) throws Exception {
                        return integer == 3;
                    }
                })
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.i(TAG,"accept : " + integer);
                    }
                });
```

### takeWhile

`takeWhile(Predicate<? super T> predicate)` 当满足条件是才会发射数据，遇到不满足条件的情况，就中断退出发射。

```java
        Observable.just(1, 2, 3, 4, 5, 6)
                .takeWhile(new Predicate<Integer>() {
                    @Override
                    public boolean test(@NonNull Integer integer) throws Exception {
                        return integer < 3;
                    }
                })
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.i(TAG,"accept : " + integer);
                    }
                });
```

## sample

`sample(long period, TimeUnit unit)` 相当于采样操作，它会定时地扫描被观察者发送的数据，并接收被观察者最近发射的数据。

```java
        Observable.interval(2, TimeUnit.SECONDS)
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.io())
                .sample(3, TimeUnit.SECONDS)
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(Long aLong) throws Exception {
                        Log.i(TAG,"accept : " + aLong);
                    }
                });
```

```
I/RxJava: accept : 0
I/RxJava: accept : 1
I/RxJava: accept : 3
I/RxJava: accept : 4
I/RxJava: accept : 6
I/RxJava: accept : 7
I/RxJava: accept : 9
I/RxJava: accept : 10
I/RxJava: accept : 12
```

## skip 和 skipLast

### skip

`skip(long count)` 用于过滤被观察者发送的前 n 项数据。

```java
        Observable.interval(1, TimeUnit.SECONDS)
                .skip(6)
                .subscribe(new Consumer<Long>() {
                    @Override
                    public void accept(Long aLong) throws Exception {
                        Log.i(TAG,"accept : " + aLong);
                    }
                });
```

```
I/RxJava: accept : 6
I/RxJava: accept : 7
I/RxJava: accept : 8
I/RxJava: accept : 9
I/RxJava: accept : 10
```

### skipLast

`skipLast(int count)` 用于过滤最后 n 项数据。

## repeat、repeatUntil 和 repeatWhen

这组操作符提供在调用 `onCompleted()` 事件后提供重复调用 `Observable` 事件的操作。

### repeat

`repeat` 提供重复调用 `Observable` 事件的操作。

 - repeat()：无限次重复（Long.MAX_VALUE）
 - repeat(long times)：重复 times 次

### repeatUntil

`repeatUntil(BooleanSupplier stop)` 重复调用 `Observable` 事件的操作直到 stop 条件满足。

### repeatWhen

`repeatWhen(final Function<? super Observable<Object>, ? extends ObservableSource<?>> handler)` 当满足一定条件重复调用 `Observable` 事件的操作。

## retry、retryUntil 和 retryWhen

这组操作符提供在调用 `onError()` 事件后提供重新调用 `Observable` 事件的操作。

## 时间节点处理操作

特定的时间节点处理方法：

 - doOnEach：发射数据的时候执行
 - doAfterNext：数据发射成功后
 - doOnNext：调用onNext方法时
 - doOnComplete：调用onComplete方法时
 - doOnError：调用onError时
 - doFinally：onComplete，onError或者取消订阅关系后
