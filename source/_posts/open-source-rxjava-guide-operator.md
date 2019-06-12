---
title: RxJava 使用指南 -- 操作符介绍
categories: Android开源项目
comments: true
tags: [Android开源项目, RxJava]
description: 本文介绍 RxJava 的操作符的用法
date: 2017-10-12 10:00:00
---

本文将会介绍一些常用 RxJava 的操作符的用法。

## 创建Observable 操作符

前面的例子介绍了使用 `Observable.create()` 操作符来创建 `Observable`，下面再介绍一下 RxJava 提供的其他方法。

### create()

前面已有介绍

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

### concat()

`concat()` 方法会将参数中的多个数据源合并，并按顺序发射。

```
        Observable source1 = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<String> e) throws Exception {
                e.onNext("Hello");
                e.onComplete();
                //e.onError(new Exception("Test Error"));
            }
        });

        Observable source2 = Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<String> e) throws Exception {
                e.onNext("World");
                e.onComplete();
            }
        });

        Observable.concat(source1, source2).subscribeOn(Schedulers.io())
        .observeOn(Schedulers.io())
        .subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.e("Test",s);
            }
        }, new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                Log.e("Test",throwable.getMessage());
            }
        });
```

这里需要注意的是，如果前一个数据源发出 `onError`，那么将会中断后面数据的发射。

### first()

再来介绍一下 `first()` 操作符，只发送符合条件的第一个事件，可以与前面的 `contact` 操作符结合使用。

```
        Observable.concat(source1, source2).subscribeOn(Schedulers.io())
                .first("Default")
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        Log.e("Test",s);
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(Throwable throwable) throws Exception {
                        Log.e("Test",throwable.getMessage());
                    }
                });
```

如果 source1 和 source2 都有条件发射数据，那么就只发射 source1 的数据。如果只有 source2 有条件发射数据，那么就发送 source2 的数据。如果都不发射，就发送 `first()` 默认数据。
就是说顺序发射数据时，只要有一个 `Observable` 发射了数据，那么就不会发射后面的数据了。如果都不发射数据，那么就发送 `first(default)` 参数里面的默认数据。

这个操作符做网络缓存的时候很有用。举个例子：依次检查 Disk 与 Network，如果 Disk 存在缓存，则不做网络请求，否则进行网络请求。

## map、flatmap 和 concatMap （变换）

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

### concatMap

concatMap 和 flatmap 的功能是一样的，都提供了转换功能，只不过在转换后进行合并时 flatmap 用的是 merge，而 concatMap 用的是 concat，因此，concatMap 是有序的，可以保证最终的输出顺序和原序列保持一致。而 flatmap 则不一定，有可能出现交错。
下面通过例子来看一下：
先来看一下 flatmap 的输出顺序，为了模拟真实场景，我们在发送第二个数据是延迟了 100ms：

```
        Observable.fromArray(1,2,3,4,5)
                .flatMap(new Function<Integer, ObservableSource<String>>() {
                    @Override
                    public ObservableSource<String> apply(Integer integer) throws Exception {
                        int delay = 0;
                        if (integer == 2) {
                            delay = 100;
                        }
                        return Observable.just(integer.toString()+" string")
                                .delay(delay, TimeUnit.MILLISECONDS);
                    }
                })
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        Log.e(TAG,s);
                    }
                });

```

输出如下，可以看到第二个数据由于有延迟，在最后发送：

```
09:55:51.114 E/RxJava: 1 string
09:55:51.117 E/RxJava: 3 string
09:55:51.117 E/RxJava: 4 string
09:55:51.117 E/RxJava: 5 string
09:55:51.213 E/RxJava: 2 string
```

下面来换做下测试：

```
        Observable.fromArray(1,2,3,4,5)
                .concatMap(new Function<Integer, ObservableSource<String>>() {
                    @Override
                    public ObservableSource<String> apply(Integer integer) throws Exception {
                        int delay = 0;
                        if (integer == 2) {
                            delay = 100;
                        }
                        return Observable.just(integer.toString()+" string")
                                .delay(delay, TimeUnit.MILLISECONDS);
                    }
                })
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        Log.e(TAG,s);
                    }
                });
```

输出，保持了输入序列输出数据：

```
04-17 09:59:07.844 E/RxJava: 1 string
04-17 09:59:07.944 E/RxJava: 2 string
04-17 09:59:07.946 E/RxJava: 3 string
04-17 09:59:07.947 E/RxJava: 4 string
04-17 09:59:07.947 E/RxJava: 5 string
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
如果需要报告错误的状态，可以在 test 方法里抛出 Exception 异常。

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

## reduce、scan、reduceWith 和 scanWith

reduce 和 scan 操作符都可以把观察序列的本次操作结果作为参数传递给下一个Observable使用，可以用来实现累加操作。
但他们还是有区别的。

 - reduce 是把所有的操作都操作完成之后最后只调用一次观察者，把数据一次性输出。
 - scan 是每次操作之后都先把数据输出给观察者，然后再调用scan的回调方法进行第二次操作。

至于 reduceWith 和 scanWith 只是多了一个 Callbale 参数来指定为第一个数据源来使用。

下面通过例子来了解一下它们的用法：

### reduce

```
        Observable.just(1, 3, 5)
                .reduce(new BiFunction<Integer, Integer, Integer>() {
                    @Override
                    public Integer apply(Integer integer, Integer integer2) throws Exception {
                        Log.e("Test","para1 = "+integer +", para2 = "+integer2);
                        return integer + integer2;
                    }
                })
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {

                        Log.e("Test","result = "+integer);
                    }
                });
```

输出：

```
E/Test: para1 = 1, para2 = 3
E/Test: para1 = 4, para2 = 5
E/Test: result = 9
```

可见，调用了两次reduce的回调方法后，得到最后的总结果，最后调用观察者方法。reduce用来实现多个操作式的与（&&）和或（||）操作非常方便。

### scan

```
        Observable.just(1, 3, 5)
                .scan(new BiFunction<Integer, Integer, Integer>() {
                    @Override
                    public Integer apply(Integer integer, Integer integer2) throws Exception {
                        Log.e("Test","para1 = "+integer +", para2 = "+integer2);
                        return integer + integer2;
                    }
                })
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {

                        Log.e("Test","result = "+integer);
                    }
                });
```


```
E/Test: result = 1
E/Test: para1 = 1, para2 = 3
E/Test: result = 4
E/Test: para1 = 4, para2 = 5
E/Test: result = 9
```

使用scan和reduce的最后输出结果是一样的，但是却调用了三次观察者方法。

### reduceWith 和 scanWith

```
        Observable.just(1, 3, 5)
                .reduceWith(new Callable<String>() {
                    @Override
                    public String call() throws Exception {
                        Log.e("Test","call");
                        return "Hello";
                    }
                }, new BiFunction<String, Integer, String>() {
                    @Override
                    public String apply(String string, Integer integer) throws Exception {
                        Log.e("Test","apply  string = "+string + ", integer = "+integer);
                        return string + " "+integer;
                    }
                })
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        Log.e("Test","result = "+s);
                    }
                });
```

输出结果：

```
E/Test: call
E/Test: apply  string = Hello, integer = 1
E/Test: apply  string = Hello 1, integer = 3
E/Test: apply  string = Hello 1 3, integer = 5
E/Test: result = Hello 1 3 5
```

```
        Observable.just(1, 3, 5)
                .scanWith(new Callable<String>() {
                    @Override
                    public String call() throws Exception {
                        Log.e("Test","call");
                        return "Hello";
                    }
                }, new BiFunction<String, Integer, String>() {
                    @Override
                    public String apply(String string, Integer integer) throws Exception {
                        Log.e("Test","apply  string = "+string + ", integer = "+integer);
                        return string + " "+integer;
                    }
                })
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(String s) throws Exception {
                        Log.e("Test","result = "+s);
                    }
                });
```

输出结果：

```
E/Test: call
E/Test: result = Hello
E/Test: apply  string = Hello, integer = 1
E/Test: result = Hello 1
E/Test: apply  string = Hello 1, integer = 3
E/Test: result = Hello 1 3
E/Test: apply  string = Hello 1 3, integer = 5
E/Test: result = Hello 1 3 5
```

## amb、ambArray 和 ambWith

传递两个或多个 Observable 给 amb 或者 ambArray 时，它只发射其中首先发射数据（onNext）或通知（onError或onCompleted）的那个 Observable 的所有数据，而其他所有的 Observable 的将不会被执行直接丢弃。
多个 Observable 是按照顺序执行的。
ambWith的用法：Observable.ambArray(o1,o2)和o1.ambWith(o2)是等价的。

```
        Observable.ambArray(Observable.create(new ObservableOnSubscribe<String>() {

            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                Log.e("Test","111");
                //emitter.onNext("11");
                //emitter.onComplete();
                //emitter.onError(new Exception("1"));

            }
        }),Observable.create(new ObservableOnSubscribe<String>() {

            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                Log.e("Test","222");
                emitter.onNext("22");

            }
        }),Observable.create(new ObservableOnSubscribe<String>() {

            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                Log.e("Test","333");
                emitter.onNext("33");

            }
        })).subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                Log.e("Test","onSubscribe");
            }

            @Override
            public void onNext(String s) {
                Log.e("Test","onNext "+s);
            }

            @Override
            public void onError(Throwable e) {
                Log.e("Test","onError");
            }

            @Override
            public void onComplete() {
                Log.e("Test","onComplete");
            }
        });

```

输出：

```
E/Test: onSubscribe
E/Test: 111
E/Test: 222
E/Test: onNext 22
```

另外，Observable.ambArray 的参数也可以是 Observable.ambArray：

```
        Observable.ambArray(Observable.create(new ObservableOnSubscribe<String>() {

            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                Log.e("Test","111");
                //emitter.onNext("11");
                //emitter.onComplete();
                //emitter.onError(new Exception("1"));

            }
        }),Observable.create(new ObservableOnSubscribe<String>() {

            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                Log.e("Test","222");
                //emitter.onNext("22");

            }
        }),Observable.ambArray(Observable.create(new ObservableOnSubscribe<String>() {

            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                Log.e("Test","333");
                emitter.onNext("33");
                //emitter.onComplete();
                //emitter.onError(new Exception("1"));

            }
        }),Observable.create(new ObservableOnSubscribe<String>() {

            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                Log.e("Test","444");
                emitter.onNext("44");

            }
        }))).subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                Log.e("Test","onSubscribe");
            }

            @Override
            public void onNext(String s) {
                Log.e("Test","onNext "+s);
            }

            @Override
            public void onError(Throwable e) {
                Log.e("Test","onError");
            }

            @Override
            public void onComplete() {
                Log.e("Test","onComplete");
            }
        });
```

应用场景，可以用在数据结果的输出是多个不确定场景时，比如：我们需要获取一个图片，它的获取场景有三个：1.参数直接给，2.本地缓存，3.网络获取。这种情况就可以按照这个顺序依次获取，直到某个场景获取到图片。
是不是和 first() 操作符有点类似呢？

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
