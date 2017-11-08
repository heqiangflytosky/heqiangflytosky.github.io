---
title: Retrofit 使用指南
categories: Android开源项目
comments: true
tags: [Android开源项目, Retrofit]
description: 
date: 2017-10-22 10:00:00
---

## 概述

Retrofit 是 Squareup 公司开源的网络请求框架，它其实是对 OkHttp 的一层封装，使用面向接口的方式进行网络请求，利用动态生成的代理类封装了网络接口请求的底层，并且提供了对 RxJava 的支持。
写这篇文章的时候，Retrofit 已经发布 2.3.0 了，本文就以此版本来介绍。
[GitHub 地址](https://github.com/square/retrofit)

## 基本用法

添加依赖：

```
    compile 'com.squareup.retrofit2:retrofit:2.3.0'
    compile 'com.squareup.retrofit2:converter-gson:2.3.0'
    compile 'com.squareup.retrofit2:adapter-rxjava2:2.3.0'
```

注意 `compile 'com.squareup.retrofit2:adapter-rxjava2:2.3.0'` 是添加了对RxJava2的支持，一定要注意RxJava的版本对应问题，否则会报错：

```
                                                  Caused by: java.lang.IllegalArgumentException: Unable to create call adapter for io.reactivex.Observable<com.example.heqiang.testsomething.testretrofit.TestRetrofit$TestBean>
                                                     for method MyService.getDataRx
                                                     at retrofit2.ServiceMethod$Builder.methodError(ServiceMethod.java:752)
                                                     at retrofit2.ServiceMethod$Builder.createCallAdapter(ServiceMethod.java:237)
                                                     at retrofit2.ServiceMethod$Builder.build(ServiceMethod.java:162)
                                                     at retrofit2.Retrofit.loadServiceMethod(Retrofit.java:170)
                                                     at retrofit2.Retrofit$1.invoke(Retrofit.java:147)
                                                     at java.lang.reflect.Proxy.invoke(Proxy.java:393)
                                                     at $Proxy0.getDataRx(Unknown Source)
                                                     at com.example.heqiang.testsomething.testretrofit.TestRetrofit.getDataRx(TestRetrofit.java:68)
                                                     at com.example.heqiang.testsomething.rxjava.RxJavaActivity.testRetrofitRx(RxJavaActivity.java:105)
                                                     	... 11 more
                                                  Caused by: java.lang.IllegalArgumentException: Could not locate call adapter for io.reactivex.Observable<com.example.heqiang.testsomething.testretrofit.TestRetrofit$TestBean>.
                                                   Tried:
                                                    * retrofit2.adapter.rxjava.RxJavaCallAdapterFactory
                                                    * retrofit2.ExecutorCallAdapterFactory
                                                     at retrofit2.Retrofit.nextCallAdapter(Retrofit.java:241)
                                                     at retrofit2.Retrofit.callAdapter(Retrofit.java:205)
                                                     at retrofit2.ServiceMethod$Builder.createCallAdapter(ServiceMethod.java:235)
```

先上代码：

MyService.java

```java
public interface MyService {
    @GET("XX/urls")
    Call<TestRetrofit.TestBean> getData();
    // 支持RxJava
    @GET("XX/urls")
    Observable<TestRetrofit.TestBean> getDataRx();
}

```

TestRetrofit.java

```java
public class TestRetrofit {

    private MyService mMyService;

    public TestRetrofit(Context context){

        Cache cache = new Cache(new File(context.getExternalCacheDir(),"test"),100*1024*1024);
        Log.e("Test", "cache dir = " + context.getExternalCacheDir());
        OkHttpClient client = new OkHttpClient.Builder()
                .retryOnConnectionFailure(true)
                .connectTimeout(20, TimeUnit.SECONDS)
                .cache(cache)
                .build();

        Retrofit retrofit = new Retrofit.Builder()
                //这里如果不指定client，那么会生成一个默认的OkHttpClient
                .client(client)
                .baseUrl("http://*.*.*.*/")
                .addConverterFactory(GsonConverterFactory.create())
                //加入对RxJava2的支持
                //.addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .build();
        mMyService = retrofit.create(MyService.class);
    }

    public void getData(){
        Call<TestBean> call = mMyService.getData();
        call.enqueue(new Callback<TestBean>() {

            @Override
            public void onResponse(Call<TestBean> call, Response<TestBean> response) {
                Log.e("Test","onResponse = "+response.body().toString());
            }

            @Override
            public void onFailure(Call<TestBean> call, Throwable t) {

            }
        });
    }

    public static class TestBean{
        public int  code;
        public String message;
        public String redirect;
        public ArrayList<String> value;

        @Override
        public String toString() {
            return "code = "+code+", message = "+message+",redirect = "+ redirect+", value="+value.toString();
        }
    }
}
```

## 结合RxJava

在上面的代码中去掉 `addCallAdapterFactory(RxJavaCallAdapterFactory.create())` 的注释，再加上定义的 `Observable<TestRetrofit.TestBean> getDataRx()` 方法，就添加了对 RxJava2 的支持。

```java
    public void getDataRx(){
        mMyService.getDataRx().subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<TestBean>() {
                    @Override
                    public void onError(Throwable e) {
                        Log.e("Test","onError");
                    }

                    @Override
                    public void onComplete() {

                    }

                    @Override
                    public void onSubscribe(@NonNull Disposable d) {

                    }

                    @Override
                    public void onNext(TestBean testBean) {
                        Log.e("Test","onNext = "+testBean.toString());
                    }
                });
```

## CallAdapter

`CallAdapter` 其实就是对 `Call` 的转换，上面的对 RxJava 支持时用到的 `RxJavaCallAdapterFactory` 其实就是 `CallAdapter.Factory` 的子类，当然我们也可以自定义实现一个 `CallAdapter`。

## Converter
在默认情况下 Retrofit 只支持将 HTTP 的响应体转换换为 `ResponseBody`，那么接口的返回值都是 `Call<ResponseBody>`，如果我们需要把 `ResponseBody` 转换成我们需要的类型就需要 `Converter`。
`Converter` 就是对数据的转换，代码中 `addConverterFactory(GsonConverterFactory.create())` 就是指定了 Gson 将 `ResponseBody`转换我们想要的类型。
显然我们也可以自定义一个 `Converter`。

<!--  
http://www.jianshu.com/p/308f3c54abdd
-->
