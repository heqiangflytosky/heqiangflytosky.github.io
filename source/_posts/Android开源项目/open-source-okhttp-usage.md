---
title: OkHttp 使用指南 -- 基本使用
categories: Android开源项目
comments: true
tags: [OkHttp]
description: 介绍 OkHttp 的基本使用
date: 2019-9-1 10:00:00
---


## 概述

OkHttp 可以说是使用最广泛的网络请求库了，基于 Socket 对 HTTP 协议和 WebSocket 协议进行了封装，我们可以很方便地用它来实现 HTTP 和 WebSocket 请求。
OkHttp 其实就是一个 Http 客户端，类似一个浏览器。下面介绍一些 OkHttp 的特点，其实大部分也是 Http 协议的特点：

 - 支持 Http2.0，允许同一主机的所有请求共享套接字。
 - 使用连接池减少请求延迟（如果HTTP / 2不可用）。
 - 透明GZIP缩小下载大小。
 - 响应缓存可以避免重复请求的网络。
 - 内部实现任务队列，提高并发访问的效率。
 - 实现拦截器链（InterceptorChain），实现request与response的分层处理

很久之前就开始使用 OkHttp 了，源码也看了部分。近期想着开始写系列关于OkHttp的博客，把这部分只是记录下来，以备查看。
本章节只简单介绍一下 OkHttp 的基本用法，关于源码分析和更深入的介绍参考后面的文章。
本系列文章分析 OkHttp 是基于 3.11.0 版本为基础的，后面有版本更新会尽可能同步更新博客的内容。
[Github 源码](https://github.com/square/okhttp)

## 对比

|  | HTTPURLConnection | Apache HttpClient | OkHttp |
|:-------------:|:-------------:|:-------------:|:-------------:|
| 提供分析记录请求各个阶段的能力 | 不支持 | 不支持 | 支持 |
| 支持响应缓存 | 支持 | 支持 | 支持 |
| 支持重试 | 支持 | 支持 | 支持 |
| 支持连接池复用连接 | 不支持 | 支持 | 支持 |
| 易用性 | JDK标准量，未做任何封装，使用起来非常原始，不方便 | 较易用 | 易用 |
| 支持HTTP/1.0、HTTP/1.1 | 支持 | 支持 | 支持 |
| 支持HTTP/2.0 | 不支持 | 不支持 | 支持 |
| 支持同步和异步请求 | 支持 | 支持 | 支持 |
| 支持取消请求 | 支持 | 支持 | 支持 |
| 支持设置超时 | 支持 | 支持 | 支持 |
| 稳定性 | 稳定 | 历史悠久，bug少，功能全 | Google在Android6.0剔除了HttpClient，采用OKHttp；现已广泛使用 |
| 性能 | 优 | 一般 | 优 |
| 支持WebSocket | 不支持 | 不支持 | 支持 |
| 引入对用户影响 | JDK自带，无影响 | 需要引入org.apache.http.legacy | 需要引入 okhttp 依赖 |


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

### 缓存

OKHttp 提供了缓存机制以将我们的的 HTTP 和 HTTPS 请求的响应缓存到文件系统中，但是它默认是不使用缓存的，所以如果我们需要使用缓存，就得在实例化 OKHttpClient 的时候进行相关的配置。

#### 配置缓存

配置缓存需要进行下面两个步骤：

 - 构造 OkHttpClient 时配置 Cache，设置缓存路径已经缓存大小
 - 构造 Request 时配置 CacheControl，配置缓存相关属性

```
        Cache cache = new Cache(new File(context.getExternalCacheDir(),"okhttp"), 10*1024*1024);
        mOkHttpClient = new OkHttpClient.Builder()
                .retryOnConnectionFailure(true)
                .connectTimeout(10, TimeUnit.SECONDS)
                .cache(cache)
                .build();

        CacheControl cacheControl = new CacheControl.Builder().maxStale(30,TimeUnit.SECONDS)
                .maxAge(30, TimeUnit.SECONDS)
                .build();

        okhttp3.Request okRequest = new okhttp3.Request.Builder()
                .url("http://www.qq.com/")
                .cacheControl(cacheControl)
                .build();

        try {
            Response response = mOkHttpClient.newCall(okRequest).execute();
            if(response  != null){
                Log.e("Test","network response = " + response.networkResponse());
                Log.e("Test","cache response = " + response.cacheResponse());
                response.body().close();
            }
        } catch (IOException e) {
            Log.e(TAG , "Http get error ", e);
        }
```

#### 常见问题

没有缓存的原因：
一. 看应用是否申请了响应权限：

```
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
```

二. 没有调用 `response.body().close()`

如果在缓存目录里面出现了下面的文件：

```
d0dae34b88fc0cc86e305ce4c60e1670.0.tmp d0dae34b88fc0cc86e305ce4c60e1670.1.tmp journal
```

d0dae34b88fc0cc86e305ce4c60e1670.0.tmp 显示了一些返回的头信息，但是 d0dae34b88fc0cc86e305ce4c60e1670.1.tmp 为空，journal 里面只有

```
libcore.io.DiskLruCache
1
201105
2

DIRTY d0dae34b88fc0cc86e305ce4c60e1670
```

而没有类似 `CLEAN d0dae34b88fc0cc86e305ce4c60e1670 651 71465` 的字段，那基本是上面的代码没有调用导致写入缓存不成功。具体原因后面的源码分析再介绍。

三. okhttp 对post请求不做缓存处理

### pingInterval

通过跟源码我们可以看到,这个值只有http2和webSocket中有使用。

Http2Connection.java

```
    if (builder.pingIntervalMillis != 0) {
      writerExecutor.scheduleAtFixedRate(new PingRunnable(false, 0, 0),
          builder.pingIntervalMillis, builder.pingIntervalMillis, MILLISECONDS);
    }
```

RealWebSocket.java

```
      if (pingIntervalMillis != 0) {
        executor.scheduleAtFixedRate(
            new PingRunnable(), pingIntervalMillis, pingIntervalMillis, MILLISECONDS);
      }
```

如果设置了这个值会定时的向服务器发送一个消息来保持长连接。
