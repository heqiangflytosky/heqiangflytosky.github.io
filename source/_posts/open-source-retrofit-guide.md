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
                                                  Caused by: java.lang.IllegalArgumentException: Unable to create call adapter for io.reactivex.Observable<com.example.heqiang.testsomething.testretrofit.RequestManager$TestBean>
                                                     for method RequestService.getDataRx
                                                     at retrofit2.ServiceMethod$Builder.methodError(ServiceMethod.java:752)
                                                     at retrofit2.ServiceMethod$Builder.createCallAdapter(ServiceMethod.java:237)
                                                     at retrofit2.ServiceMethod$Builder.build(ServiceMethod.java:162)
                                                     at retrofit2.Retrofit.loadServiceMethod(Retrofit.java:170)
                                                     at retrofit2.Retrofit$1.invoke(Retrofit.java:147)
                                                     at java.lang.reflect.Proxy.invoke(Proxy.java:393)
                                                     at $Proxy0.getDataRx(Unknown Source)
                                                     at com.example.heqiang.testsomething.testretrofit.RequestManager.getDataRx(RequestManager.java:68)
                                                     at com.example.heqiang.testsomething.rxjava.RxJavaActivity.testRetrofitRx(RxJavaActivity.java:105)
                                                     	... 11 more
                                                  Caused by: java.lang.IllegalArgumentException: Could not locate call adapter for io.reactivex.Observable<com.example.heqiang.testsomething.testretrofit.RequestManager$TestBean>.
                                                   Tried:
                                                    * retrofit2.adapter.rxjava.RxJavaCallAdapterFactory
                                                    * retrofit2.ExecutorCallAdapterFactory
                                                     at retrofit2.Retrofit.nextCallAdapter(Retrofit.java:241)
                                                     at retrofit2.Retrofit.callAdapter(Retrofit.java:205)
                                                     at retrofit2.ServiceMethod$Builder.createCallAdapter(ServiceMethod.java:235)
```

还有个库是 `com.squareup.retrofit2:adapter-rxjava`，这个是仅支持RxJava1，如果想使用 RxJava2，就要用上面的依赖。
如果需要使用 Gson 作为转换器，就需要依赖 `com.squareup.retrofit2:converter-gson`，如果不想使用，可以自定义，比如 FastJson 等。

先上代码：

CallBack.java

```
public interface CallBack<T> {
    void onSuccess(T t);
    void onFail(Exception e);
}
```

RequestService.java

```java
public interface RequestService {
    @GET("hq/urls")
    Call<RequestManager.TestBean> getData();
    // 支持RxJava
    @GET("hq/urls")
    Observable<RequestManager.TestBean> getDataRx();
}

```

RequestManager.java

```java
public class RequestManager {

    private RequestService mRequestService;
    private static volatile RequestManager sInstance;

    private RequestManager(Context context){

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
                //.addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build();
        mRequestService = retrofit.create(RequestService.class);
    }

    public static RequestManager getInstance(){
        if(sInstance == null){
            synchronized (RequestManager.class){
                if(sInstance == null){
                    sInstance = new RequestManager();
                }
            }
        }
        return sInstance;
    }

    public void getData(){
        Call<TestBean> call = mRequestService.getData();
        // 异步请求
        call.enqueue(new Callback<TestBean>() {

            @Override
            public void onResponse(Call<TestBean> call, Response<TestBean> response) {
                Log.e("Test","onResponse = "+response.body().toString());
            }

            @Override
            public void onFailure(Call<TestBean> call, Throwable t) {

            }
        });
        // 同步请求
        try {
            Response<TestBean> response = call.execute();
        } catch (IOException e) {
            e.printStackTrace();
        }
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

## 结合RxJava和RxAndroid

在上面的代码中去掉 `addCallAdapterFactory(RxJava2CallAdapterFactory.create())` 的注释，再加上定义的 `Observable<RequestManager.TestBean> getDataRx()` 方法，就添加了对 RxJava2 的支持。
如果需要使用 RxAndroid 还要在build.gradle中添加依赖：

```
compile 'io.reactivex.rxjava2:rxandroid:2.0.1'
```


```java
    public void getDataRx(final CallBack<TestBean> callBack){
        mRequestService.getDataRx().subscribeOn(Schedulers.io())
                // 依赖RxAndroid
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
                    public void onSubscribe(@NonNull Disposable d) {

                    }

                    @Override
                    public void onNext(TestBean testBean) {
                        Log.e("Test","onNext = "+testBean.toString());
                        callBack.onSuccess(testBean);
                    }
                });
```

## CallAdapter

`CallAdapter` 其实就是对 `Call` 的转换，上面的对 RxJava 支持时用到的 `RxJava2CallAdapterFactory` 其实就是 `CallAdapter.Factory` 的子类，当然我们也可以自定义实现一个 `CallAdapter`。

## Converter
在默认情况下 Retrofit 只支持将 HTTP 的响应体转换换为 `ResponseBody`，那么接口的返回值都是 `Call<ResponseBody>`，如果我们需要把 `ResponseBody` 转换成我们需要的类型就需要 `Converter`。
`Converter` 就是对数据的转换，代码中 `addConverterFactory(GsonConverterFactory.create())` 就是指定了 Gson 将 `ResponseBody`转换我们想要的类型。
显然我们也可以自定义一个 `Converter`，比如使用 FastJson 等。

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
 - FieldMap：POST请求，和@Filed作用一致，用于不确定表单参数
 - Body：POST请求，以对象的形式提交，多用于post请求发送非表单数据,比如想要以post方式传递json格式数据

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

```java
    @FormUrlEncoded
    @POST("add2gank")
    void add2Gank(@FieldMap Map<String, String> map)
```

可以看到这里多了一个 `@FormUrlEncoded` 注解，如果去掉 `@FromUrlEncoded` 在 post 请求中使用 `@Field` 和 `@FieldMap`，那么程序会抛出 `java.lang.IllegalArgumentException: @Field parameters can only be used with form encoding. (parameter #1)` 的错误异常。
如果将 `@FromUrlEncoded` 添加在 `@GET` 上面呢，同样的也会抛出 `java.lang.IllegalArgumentException:FormUrlEncoded can only be specified on HTTP methods with request body (e.g., @POST)` 的错误异常。
如果是使用 `@Body`，也是不用加 `@FormUrlEncoded` 注解的。

 - Url：实现动态Url

前面的使用都是基于BaseUrl的，如果我们想请求不同Url时只能重新生成一个 Retrofit 实例，实际上我们还可以通过 `@Url` 来操作。

```java
    @GET
    public Call<ResponseBody> getImage(@Url String url);
```

如果参数指定的 url 是一个以 `http` 开头的 url，那么就完全使用这个url，和前面指定的 BaseUrl 完全没有联系，如果 url 指定的只有类似 `/id/name` 的path，而没有指定 scheme 和 host，那么它就会用 BaseUrl 的 scheme 和 host。
** <font color=#ff0000>这里注意</font> **：只是用 BaseUrl 的 scheme 和 host，如果 BaseUrl 是 http://www.test.com/v/ ，那么还是会把 path 中的 v  去掉，组成 http://www.test.com/id/name 。

 - Header：用于在方法参数里动态添加请求头：

```
Call<ResultBean> postSayHi(@Header("city") String city);
```

 - Part
 - PartMap

这两个用于上传文件，与 `@MultiPart` 注解结合使用

```
    @Multipart
    @POST("test/upload")
    Call<ResultBean> upload(@Part("file\"; filename=\"launcher_icon.png") RequestBody file);
```

### 方法注解

 - FormUrlEncoded：上面已经介绍过了。
 - Headers：用于在方法添加请求头

```
    @POST("test/sayHi")
    @Headers("Accept-Encoding: application/json")
    Call<ResultBean> postSayHi(@Body UserBean userBean, @Header("city") String city);
```

 - Streaming：如果您正在下载一个大文件，Retrofit2将尝试将整个文件移动到内存中。为了避免这种，我们必须向请求声明中添加一个特殊的注解

```
@Streaming
@GET
Call<ResponseBody> downloadFileByDynamicUrl(@Url String fileUrl);
```

<!--  
http://www.jianshu.com/p/308f3c54abdd
-->
