---
title: Retrofit 2 使用指南
categories: Android开源项目
comments: true
tags: [Android开源项目, Retrofit]
description: 介绍 Retrofit 2 的一些基本用法
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
    @GET("hq/urls")
    Call<TestRetrofit.TestBean> getData();
    // 支持RxJava
    @GET("hq/urls")
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

## 注解的使用

### 请求方法

 - GET
 - POST
 - PUT
 - DELETE
 - PATCH
 - HEAD
 - OPTIONS
 - HTTP

前面 6 种都是HTTP协议请求方式，使用方法和前面例子中的类似，都接受一个字符串来和 BaseUrl 组成请求Url。
HTTP 方式是对上面 6 中的扩展，用法：

```java
    @HTTP(method = "GET", path = "blog", hasBody = false)
    Call<ResponseBody> getBlog();
```

### 请求参数

 - Path
 - Query
 - QueryMap

上面三种都是一个完整Url的组成部分，Path 用在 Url 的 path 部分，Query 和 QueryMap 用在参数部分，也就是Url中的 ? 后面的部分。QueryMap 相当于多个 Query。

```java
    @GET("day/{year}/{month}/{day}")
    Observable<DailyDataResponse> getDailyData(
            @Path("year") int year,@Path("month") int month,@Path("day") int day);
```

```java
    @GET("day")
    Call<Response> getDailyData(
            @Query("year") int year,@Query("month") int month,@Query("day") int day);
```

```java
    @GET("day")
    Call<Response> getDailyData(
            @QueryMap Map<String, Integer> map);
```

 - Field ：POST请求，提交单个数据
 - Body：POST请求，相当于多个@Field，以对象的形式提交

```java
    @FormUrlEncoded
    @POST("add2gank")
    Observable<AddToGankResponse> add2Gank(@Field("url") String url, @Field("desc") String desc, @Field("who") String who,
                                @Field("type") String type,@Field("debug") boolean debug);
```

```java
    @POST("add2gank")
    Call<Response> add2Gank(@Body ExtrasBean bean);
```

### 动态Url

前面的使用都是基于BaseUrl的，如果我们想请求不同Url时只能重新生成一个 Retrofit 实例，实际上我们还可以通过 `@Url` 来操作。

```java
    @GET
    public Call<ResponseBody> getImage(@Url String url);
```

如果参数指定的 url 是一个以 `http` 开头的 url，那么就完全使用这个url，和前面指定的 BaseUrl 完全没有联系，如果 url 指定的只有类似 `/id/name` 的path，而没有指定 scheme 和 host，那么它就会用 BaseUrl 的 scheme 和 host。
** <font color=#ff0000>这里注意</font> **：只是用 BaseUrl 的 scheme 和 host，如果 BaseUrl 是 http://www.test.com/v/，那么还是会把 path 中的 v  去掉，组成 http://www.test.com/id/name。

<!--  
http://www.jianshu.com/p/308f3c54abdd
-->
