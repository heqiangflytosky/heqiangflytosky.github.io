---
title: OkHttp 使用指南 -- 基本使用
categories: Android开源项目
comments: true
tags: [OkHttp]
description: 介绍 OkHttp 的基本使用
date: 2019-9-1 10:00:00
---


## 概述

OkHttp 很久之前就开始使用了，源码也看了部分。近期想着开始写系列关于OkHttp的博客，把这部分只是记录下来，以备查看。
本章节只简单介绍一下 OkHttp 的基本用法，关于源码分析和更深入的介绍参考后面的文章。
本系列文章分析 OkHttp 是基于 3.11.0 版本为基础的，后面有版本更新会尽可能同步更新博客的内容。
[Github 源码](https://github.com/square/okhttp)


## 基本使用

### 引入依赖

```
    api 'com.squareup.okhttp3:okhttp:3.11.0'
```
### 创建 OkHttpClient

```
        Cache cache = new Cache(new File(context.getExternalCacheDir(),"okhttp"), 10*1024*1024);
        mOkHttpClient = new OkHttpClient.Builder()
                .retryOnConnectionFailure(true)
                .connectTimeout(10, TimeUnit.SECONDS)
                //.cache(cache)
                .build();
```

> 在创建 OkHttpClient 对象时需要**注意**一下：最好把该对象做成单例，这样就全局可用，这样可以提高性能。因为每个 OkHttpClient 都拥有自己的连接池和线程池。 重用连接和线程可减少延迟并节省内存。 相反，为每个请求创建一个 OkHttpClient 会浪费空闲池上的资源。

下面就可以建立请求任务了，下面介绍一下常用的 get 和 post 请求的建立。

### get

```
        CacheControl cacheControl = new CacheControl.Builder().maxStale(30,TimeUnit.SECONDS)
                .maxAge(30, TimeUnit.SECONDS)
                .build();

        okhttp3.Request okRequest = new okhttp3.Request.Builder()
                .get()
                .url("https://free-api.heweather.com/s6/air/now?location=zhuhai&key=f464c53cb02240a194640685ee425116")
                //.cacheControl(cacheControl)
                .build();
```

okhttp3.Request.Builder() 的 get() 方法不调用也是可以的，创建Builder时的method默认为GET。

```
    public Builder() {
      this.method = "GET";
      this.headers = new Headers.Builder();
    }
```

### post

```
        FormBody.Builder builder = new FormBody.Builder();
        builder.add("test","test");
        FormBody body = builder.build();

        okhttp3.Request okRequest = new okhttp3.Request.Builder()
                .post(body)
                .url("https://free-api.heweather.com/s6/air/now?location=zhuhai&key=f464c53cb02240a194640685ee425116")
                .build();
```

这里需要注意一下，okhttp 对post请求不做缓存处理。
建立好请求之后，就要加入到请求队列中执行请求任务了。请求任务的执行分为同步和异步两种。

### 同步请求

```
        try {
            Response response = mOkHttpClient.newCall(okRequest).execute();
            if(response  != null){
                Log.e("Test","data = " + response.body().string());
                Log.e("Test","network response = " + response.networkResponse());
                Log.e("Test","cache response = " + response.cacheResponse());
                response.body().close();
            }
        } catch (IOException e) {
            Log.e(TAG , "Http get error ", e);

        }
```

### 异步请求

```
        mOkHttpClient.newCall(okRequest).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                //Log.e("Test","data = " + response.body().string());
                Log.e("Test","network response = " + response.networkResponse());
                Log.e("Test","cache response = " + response.cacheResponse());
                response.body().close();
            }
        });
```

### WebSocket

OkHttp 也提供了对 WebSocket 的支持。

## 相关配置

### 设置超时时间

```
        .connectTimeout(10, TimeUnit.SECONDS)
```

### 打印网络请求日志

```
        //打印网络请求日志
        HttpLoggingInterceptor loggingInterceptor = new HttpLoggingInterceptor(new HttpLoggingInterceptor.Logger() {
            @Override
            public void log(String message) {
                Log.e("HttpLog","retrofitBack = "+message);
            }
        });
        loggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY);

        OkHttpClient.Builder builder  = new OkHttpClient.Builder();
        builder.addInterceptor(loggingInterceptor);
```
