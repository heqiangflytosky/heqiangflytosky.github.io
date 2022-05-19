---
title: Retrofit 2 源码分析二 -- RxJava2
categories: Android开源项目
comments: true
tags: [Android开源项目, Retrofit]
description: Retrofit 2 中 RxJava2 相关部分源码
date: 2017-11-10 10:00:00
---

## 概述

RxJava的优势这里不再赘述，可以参考我前面 RxJava 系列的博客。Retrofit 2 良好的扩展性使它非常方便的添加了 RxJava 的支持。

## 使用方法

参考我的博客[Retrofit 2 使用指南 ](http://www.heqiangfly.com/2017/10/22/open-source-retrofit-guide/)。
和上一篇博客中用内置的适配器不同的是：
方法声明：

```
public interface RequestService {
    ...
    // 支持RxJava
    @GET("heqiang/urls")
    Observable<RequestManager.TestBean> getDataRx();
    ...

}
```

初始化请求：

```
        Retrofit retrofit = new Retrofit.Builder()
                .client(client)
                .baseUrl("http://172.17.137.69/")
                .addConverterFactory(GsonConverterFactory.create())
                //加入对RxJava2的支持
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
        mMyService = retrofit.create(RequestService.class);        
```

方法执行：

```
    public void getDataRx(final CallBack<TestBean> callBack){
        mMyService.getDataRx().subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<TestBean>() {
                    @Override
                    public void onError(Throwable e) {
                        Log.e("Test","onError");
                        callBack.onFail(new Exception(e));
                    }

                    @Override
                    public void onComplete() {

                    }

                    @Override
                    public void onSubscribe(Disposable d) {

                    }

                    @Override
                    public void onNext(TestBean testBean) {
                        Log.e("Test","onNext = "+testBean.toString());
                        callBack.onSuccess(testBean);
                    }
                });
    }
```

## 源码解析

Rxjava 相关代码结构：

![效果图](/images/open-source-retrofit-source-code-analysis-rxjava/rxjava-classes.png)

可以看到代码量不是很多。
`RxJava2CallAdapterFactory` 和 `RxJava2CallAdapter` 两个类分别对应上篇博客分析使用内置适配器时的 `ExecutorCallAdapterFactory` 和 `ExecutorCallAdapterFactory.CallAdapter` 类，是适配器工厂和适配器类。

### RxJava2CallAdapterFactory

先来看一下适配器工厂类。
上面的Demo中通过 `Retrofit.Builder()` 的 `addCallAdapterFactory(RxJava2CallAdapterFactory.create())` 方法指定了适配器工厂。
`RxJava2CallAdapterFactory` 提供了三个静态的方法生成适配器工厂，它们分别可以生成同步的 `Observable`、异步的 `Observable` 以及可以制定线程调度器同步 `Observable`。
不理解没关系，下面会详细介绍他们的区别。
先来看一下这三个方法：

```
  // 生成同步的 Observable 的适配器工厂
  public static RxJava2CallAdapterFactory create() {
    return new RxJava2CallAdapterFactory(null, false);
  }

  // 生成异步的 Observable 的适配器工厂
  public static RxJava2CallAdapterFactory createAsync() {
    return new RxJava2CallAdapterFactory(null, true);
  }

  // 生成同步的 Observable 并制定线程调度器的适配器工厂
  @SuppressWarnings("ConstantConditions") // Guarding public API nullability.
  public static RxJava2CallAdapterFactory createWithScheduler(Scheduler scheduler) {
    if (scheduler == null) throw new NullPointerException("scheduler == null");
    return new RxJava2CallAdapterFactory(scheduler, false);
  }
```
因此在动态代理类的 `serviceMethod.callAdapter.adapt(okHttpCall)` 其实执行的是 `RxJava2CallAdapterFactory.get()` 方法生成的 `RxJava2CallAdapter` 对象的 `adapt()` 方法。
我们先来看一下 `RxJava2CallAdapterFactory.get()` 方法：

```
  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    // 这里的 returnType 指的是请求方法定义的类型，比如 Observable<RequestManager.TestBean>
    // 获取返回数据类型，比如Call<Requestbody>的Call，Observable<RequestManager.TestBean> 中的 Observable
    Class<?> rawType = getRawType(returnType);

    if (rawType == Completable.class) {
      // 当定义的返回值是Completable类时，对于Completable我们前面也介绍过，它的特点是用户只关心 onComplete 事件
      return new RxJava2CallAdapter(Void.class, scheduler, isAsync, false, true, false, false,
          false, true);
    }

    boolean isFlowable = rawType == Flowable.class;
    boolean isSingle = rawType == Single.class;
    boolean isMaybe = rawType == Maybe.class;
    if (rawType != Observable.class && !isFlowable && !isSingle && !isMaybe) {
      return null;
    }

    boolean isResult = false;
    boolean isBody = false;
    Type responseType;
    if (!(returnType instanceof ParameterizedType)) {
      ......
    }
    // 获取响应数据类型，即泛型的参数 如 Call<Requestbody> 中 Requestbody
    Type observableType = getParameterUpperBound(0, (ParameterizedType) returnType);
    // 再次获取数据类型，observableType 是 Observable<RequestManager.TestBean> 中的 TestBean，这里还是返回 TestBean
    Class<?> rawObservableType = getRawType(observableType);
    if (rawObservableType == Response.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException("Response must be parameterized"
            + " as Response<Foo> or Response<? extends Foo>");
      }
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
    } else if (rawObservableType == Result.class) {
      if (!(observableType instanceof ParameterizedType)) {
        throw new IllegalStateException("Result must be parameterized"
            + " as Result<Foo> or Result<? extends Foo>");
      }
      responseType = getParameterUpperBound(0, (ParameterizedType) observableType);
      isResult = true;
    } else {
      responseType = observableType;
      isBody = true;
    }

    return new RxJava2CallAdapter(responseType, scheduler, isAsync, isResult, isBody, isFlowable,
        isSingle, isMaybe, false);
  }
```

这个方法就是生成来一个 `RxJava2CallAdapter` 对象。

### RxJava2CallAdapter

先来看一下构造函数：

```
  RxJava2CallAdapter(Type responseType, @Nullable Scheduler scheduler, boolean isAsync,
      boolean isResult, boolean isBody, boolean isFlowable, boolean isSingle, boolean isMaybe,
      boolean isCompletable) {
    this.responseType = responseType;
    this.scheduler = scheduler;
    this.isAsync = isAsync;
    this.isResult = isResult;
    this.isBody = isBody;
    this.isFlowable = isFlowable;
    this.isSingle = isSingle;
    this.isMaybe = isMaybe;
    this.isCompletable = isCompletable;
  }
```

这里是一些条件的配置。
对于这个类重点关注它的 `adapt()` 方法。

```
  @Override public Object adapt(Call<R> call) {
    // 这里根据同步或者异步生成不同的 Observable，后面介绍
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);

    // 生成不同的 Observable
    Observable<?> observable;
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
      observable = new BodyObservable<>(responseObservable);
    } else {
      observable = responseObservable;
    }

    if (scheduler != null) {
      observable = observable.subscribeOn(scheduler);
    }

    if (isFlowable) {
      return observable.toFlowable(BackpressureStrategy.LATEST);
    }
    if (isSingle) {
      return observable.singleOrError();
    }
    if (isMaybe) {
      return observable.singleElement();
    }
    if (isCompletable) {
      return observable.ignoreElements();
    }
    return RxJavaPlugins.onAssembly(observable);
  }
```

### CallEnqueueObservable 和 CallExecuteObservable

从名字就可以看出来，这两个类分别处理异步和同步的Observable。
这里主要看一下这两个类的 `subscribeActual()` 方法。

CallEnqueueObservable.subscribeActual()

```
  @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallCallback<T> callback = new CallCallback<>(call, observer);
    observer.onSubscribe(callback);
    call.enqueue(callback);
  }
```


CallExecuteObservable.subscribeActual()

```
  @Override protected void subscribeActual(Observer<? super Response<T>> observer) {
    // Since Call is a one-shot type, clone it for each new observer.
    Call<T> call = originalCall.clone();
    CallDisposable disposable = new CallDisposable(call);
    observer.onSubscribe(disposable);

    boolean terminated = false;
    try {
      Response<T> response = call.execute();
      if (!disposable.isDisposed()) {
        observer.onNext(response);
      }
      if (!disposable.isDisposed()) {
        terminated = true;
        observer.onComplete();
      }
    } catch (Throwable t) {
      Exceptions.throwIfFatal(t);
      if (terminated) {
        RxJavaPlugins.onError(t);
      } else if (!disposable.isDisposed()) {
        try {
          observer.onError(t);
        } catch (Throwable inner) {
          Exceptions.throwIfFatal(inner);
          RxJavaPlugins.onError(new CompositeException(t, inner));
        }
      }
    }
  }
```

其实这里主要做了下面几件事：
 1. clone了原有的call，因为OkHttp.Call只能使用一次
 2. 设置了 CallDisposable，可用于解除订阅
 3. 调用 Call 的 execute() 或者 enqueue() 方法。
 4. 同步方法可能会调用onNext和onComplete，异步方法设置回调函数，在回调函数中调用这些方法。

