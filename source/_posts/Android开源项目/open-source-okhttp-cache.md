---
title: OkHttp 源码分析 -- 缓存
categories: Android开源项目
comments: true
tags: [OkHttp]
description: 介绍 OkHttp 的缓存
date: 2019-9-28 10:00:00
---

## 概述

合理地利用本地缓存可以有效地减少网络开销，减少响应延迟。OkHttp 也支持缓存功能。
关于 OkHttp 缓存的使用，请参考博客[OkHttp 使用指南 -- 基本使用 ](http://www.heqiangfly.com/2019/09/01/open-source-okhttp-usage/)。
本文将会介绍一些 Http 的缓存知识以及OkHttp实现缓存的原理。

## Http 缓存

Http 协议在头信息中定义了一些域来标识缓存功能。我们首先了解一下相关的知识，有助于我们更好理解 OkHttp 缓存。

### Expires

```
Expires: Thu, 12 Mar 2019 12:08:54 GMT
```

超时时间，一般用在服务器的response报头中用于告知客户端对应资源的过期时间。当客户端需要再次请求相同资源时先比较其过期时间，如果尚未超过过期时间则直接返回缓存结果，如果已经超过则重新请求。
该字段存在于HTTP/1.0中，这种机制有一个非常大的问题，因为该字段是存在于响应头中，也就是说它的时间是服务器上的时间，但是客户端的时间是很有可能与服务器上的时间存在误差的，比如不在一个时区，用户修改了自己电脑时间等因素，这样这个字段就没有意义了；在HTTP 1.1开始，使用Cache-Control: max-age=秒 替代；而且现在浏览器均默认使用HTTP 1.1，所以它的作用基本忽略。

### Cache-Control

缓存中非常重要的一个字段，作用与Expires差不多，存在于响应头，都是标注当前资源的有效期。
但是它有很多的值，可以指定较为复杂的缓存规则，如果与Expires同时存在，Cache-Control比Expires优先级更高。

```
Cache-Control:max-age=31536000,public
```

可以组合的值有：

 - public：表明该资源或者说响应可以被任何用户缓存，比如客户端，代理服务器等都可以缓存资源，写法：Cache-Control:public
 - private：表明该资源只能被单个用户缓存，默认是private，即只能被客户端缓存，不能被代理服务器缓存，写法：Cache-Control:private
 - max-age：表明该资源的有效时间，单位是s，写法： Cache-Control:max-age=3600，即在获取该资源后3600s内不需要再向服务器获取
 - no-cache：表明客户端需要忽略已存在的缓存，强制每次请求直接发送给服务器，拉取资源，写法：Cache-Control:no-cache
 - no-store：表明该资源不能被缓存，如果缓存了需要删除，写法：Cache-Control:no-store
 - s-maxage：和max-age含义类似，只不过用于public 修饰的缓存，写法：Cache-Control:s-maxage=3600
 - must-revalidate：表明在使用缓存前必须要验证旧资源状态，并且不可使用过期资源， 写法：Cache-Control:must-revalidate
 - max-stale：表明缓存的资源在过期了但未超过max-stale指定的时间，那么就可以继续使用该缓存，超过后就必须去服务器获取；写法：Cache-Control:max-stale（代表着资源永不过期）； Cache-Control:max-stale=3600(表明在缓存过期后的3600秒内还可以继续用)
 - min-fresh：字面意思是最小新鲜度，跟max-age相对应（最大新鲜度），比如max-age=3600，min-fresh=600，那么 他两的差值就是3000，也就是说缓存真正有效时间只有3000s，超过这个时间就要去服务器拉取了。可以代表客户端告知服务器，如果当前时间加上min-fresh的值，超了该缓存的过期时间，则要给我一个新的。
 - only-if-cached：不管缓存是否过期，或者服务端有更新，只要存在缓存就是用它，写法：Cache-Control:only-if-cached
 - no-transform：不得对资源进行转换，即代理服务器不能修改Content-Encoding, Content-Range, Content-Type等HTTP头；因为有时候代理服务器为了节省缓存空间或者提高传输效率，会对图片等进行压缩；写法： Cache-Control:no-transform、
 - immutable：表示资源在有效期内服务器不会对其更改，这样客户端就不需要再发送验证请求头，比如If-None-Match或If-Modified-Since来检测更新，即使用户主动刷新页面，写法：Cache-Control:immutable

### Last-Modified-Date

客户端第一次请求时，服务器返回：

```
Last-Modified: Tue, 12 Jan 2016 09:31:27 GMT
```

当客户端二次请求时，可以头部加上如下header:

```
If-Modified-Since: Tue, 12 Jan 2016 09:31:27 GMT
```

如果当前资源没有被二次修改，服务器返回304告知客户端直接复用本地缓存。

### ETag

ETag是对资源文件的一种摘要，可以通过ETag值来判断文件是否有修改。当客户端第一次请求某资源时，服务器返回：

```
ETag: "5694c7ef-24dc"
```

客户端再次请求时，可在头部加上如下域：

```
If-None-Match: "5694c7ef-24dc"
```

如果文件并未改变，则服务器返回304告知客户端可以复用本地缓存。

## OkHttp 缓存实现原理

### 相关类

OkHttp 中实现缓存相关的类有下面几个：

 - CacheInterceptor：实现缓存的拦截器，缓存工作都是由它来完成的。
 - Cache：缓存管理器，其内部包含一个 DiskLruCache 将cache写入文件系统。
 - CacheStrategy：缓存策略。其内部维护一个request和response，通过指定request和response来描述是通过网络还是缓存来获取response，或者二者同时使用。
 - CacheStrategy$Factory：缓存策略工厂类，根据实际请求返回对应的缓存策略
 - CacheControl：缓存控制，我们使用 OkHttpClient 时可以使用这个类来控制缓存

## CacheInterceptor

先来看一下 `CacheInterceptor` 的核心代码 `CacheInterceptor.intercept()`

```
  @Override public Response intercept(Chain chain) throws IOException {
    // 尝试获取缓存
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    //获取缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    //如果有缓存，更新下相关统计指标：命中率
    if (cache != null) {
      cache.trackResponse(strategy);
    }
    //如果当前缓存不符合要求，将其close
    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // 如果不能使用网络，同时又没有符合条件的缓存，直接抛504错误
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // 如果有缓存同时又不使用网络请求，则直接返回缓存结果
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
    //进行网络请求
    Response networkResponse = null;
    try {
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // 如果既有缓存，同时又发起了请求，说明此时是一个Conditional Get请求
    if (cacheResponse != null) {
      // 如果服务端返回的是NOT_MODIFIED,缓存有效，将本地缓存和网络响应做合并
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // 更新缓存命中
        cache.trackConditionalCacheHit();
        // 更新缓存
        cache.update(cacheResponse, response);
        return response;
      } else {
        // 如果服务器响应资源有更新，关掉原有缓存
        closeQuietly(cacheResponse.body());
      }
    }

    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      // 如果响应有响应体且响应可以缓存，那就将响应写入到缓存
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // 将网络响应写入cache中
        CacheRequest cacheRequest = cache.put(response);
        // 缓存响应体并返回响应
        return cacheWritingResponse(cacheRequest, response);
      }
      // 通过请求方法判断需不需要进行缓存
      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          // 删除缓存
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```

### CacheStrategy

从上面代码可以看出，缓存的使用都是根据缓存策略来执行的，那么缓存策略是如何生成的呢？我们接下来解析一下。
先来看一下缓存策略工厂：

```
    public Factory(long nowMillis, Request request, Response cacheResponse) {
      this.nowMillis = nowMillis;
      this.request = request;
      this.cacheResponse = cacheResponse;

      if (cacheResponse != null) {
        this.sentRequestMillis = cacheResponse.sentRequestAtMillis();
        this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();
        Headers headers = cacheResponse.headers();
        for (int i = 0, size = headers.size(); i < size; i++) {
          String fieldName = headers.name(i);
          String value = headers.value(i);
          if ("Date".equalsIgnoreCase(fieldName)) {
            servedDate = HttpDate.parse(value);
            servedDateString = value;
          } else if ("Expires".equalsIgnoreCase(fieldName)) {
            expires = HttpDate.parse(value);
          } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {
            lastModified = HttpDate.parse(value);
            lastModifiedString = value;
          } else if ("ETag".equalsIgnoreCase(fieldName)) {
            etag = value;
          } else if ("Age".equalsIgnoreCase(fieldName)) {
            ageSeconds = HttpHeaders.parseSeconds(value, -1);
          }
        }
      }
    }
```

缓存策略工厂是根据 `Request` 和获取的本地缓存 `cacheResponse` 来构造的，主要是对一些变量赋值以及解析一下 `cacheResponse` 的头信息，为后面生成缓存策略做准备。

```
    // 获取缓存策略
    public CacheStrategy get() {
      // 获取合适的缓存策略
      CacheStrategy candidate = getCandidate();
      // 这里的逻辑判断参考下面解析
      if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
        return new CacheStrategy(null, null);
      }

      return candidate;
    }
```

`get()` 方法这里有个对得到的缓存策略的判断，`candidate.networkRequest != null` 意思就是通过对缓存的解析，得到的结果是我们需要通过网络请求获取响应。但是我们的请求头设置了 `only-if-cached`，那这里的意思就不是说只是用缓存，而是不要使用网络。那这两个就出现了矛盾，OKHttp 的解决方法是直接创建一个参数都是 null 的 CacheStrategy，这样在缓存拦截器中将会构建一个504错误返回给用户。

```
    /** Returns a strategy to use assuming the request can use the network. */
    private CacheStrategy getCandidate() {
      // 如果没有缓存 那就将 cacheResponse 参数设置为 null，直接进行网络请求
      if (cacheResponse == null) {
        return new CacheStrategy(request, null);
      }

      // 如果当前请求是HTTPS，而缓存没有TLS握手，重新发起网络请求
      if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
      }

      // 通过响应码以及头部缓存控制字段判断响应能不能缓存，如果不能缓存那就进行网络请求
      if (!isCacheable(cacheResponse, request)) {
        return new CacheStrategy(request, null);
      }
      // 取出请求头缓存控制对象
      CacheControl requestCaching = request.cacheControl();
      // no-cache 表明要忽略本地缓存
      // If-Modified-Since 和 If-None-Match 说明缓存过期，需要服务端验证
      // 这种情况下就需要进行网络请求
      if (requestCaching.noCache() || hasConditions(request)) {
        return new CacheStrategy(request, null);
      }
      // 如果缓存响应是 immutable 响应，在前面已经判断没有过期的情况下，
      // 不去请求网络，而是使用缓存响应
      CacheControl responseCaching = cacheResponse.cacheControl();
      if (responseCaching.immutable()) {
        return new CacheStrategy(null, cacheResponse);
      }
      // 该响应已缓存的时长
      long ageMillis = cacheResponseAge();
      // 该响应可以缓存的时长
      long freshMillis = computeFreshnessLifetime();
      // CacheControl 里面设置了 max-age
      if (requestCaching.maxAgeSeconds() != -1) {
        // 取出上面计算的可以缓存的时长 和 max-age 设置的最小值
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
      }

      long minFreshMillis = 0;
      if (requestCaching.minFreshSeconds() != -1) {
        // 取出请求头中的 min-fresh 值
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
      }

      long maxStaleMillis = 0;
      // 第一个判断：是否要求必须去服务器验证资源状态
      // 第二个判断：获取max-stale值，如果不等于-1，说明缓存过期后还能使用指定的时长
      if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
        // 取出请求头中的 max-stale 值
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
      }
      // 1.如果响应头没有要求忽略本地缓存
      // 2.已缓存时长+最小新鲜度时长 < 最大新鲜度时长 + 过期后继续使用时长
      // 通过不等式转换：最大新鲜度时长减去最小新鲜度时长就是缓存的有效期，再加上过期后继续使用时长，那就是缓存极限有效时长
      //如果已缓存的时长小于极限时长，说明还没到极限，对吧，那就继续使用
      if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        Response.Builder builder = cacheResponse.newBuilder();
        // 如果已过期，但未超过 过期后继续使用时长，那还可以继续使用，只用添加相应的头部字段
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        //如果缓存已超过一天并且响应中没有设置过期时间也需要添加警告
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
        //缓存继续使用，不进行网络请求
        return new CacheStrategy(null, builder.build());
      }

      // 走到这里说明缓存真的过期了，发起conditional get网络请求
      String conditionName;
      String conditionValue;
      //判断缓存的响应头是否设置了Etag
      if (etag != null) {
        conditionName = "If-None-Match";
        conditionValue = etag;
      } else if (lastModified != null) {
        //判断缓存的响应头是否设置了lastModified 
        conditionName = "If-Modified-Since";
        conditionValue = lastModifiedString;
      } else if (servedDate != null) {
        //判断缓存的响应头是否设置了Date
        conditionName = "If-Modified-Since";
        conditionValue = servedDateString;
      } else {
        //如果都没有设置就进行一次常规的网络请求
        return new CacheStrategy(request, null); 
      }
      // 构造 conditional get 网络请求
      Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
      // 将上面判断的字段添加到头部
      Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);
      // 使用新的头部
      Request conditionalRequest = request.newBuilder()
          .headers(conditionalRequestHeaders.build())
          .build();
      return new CacheStrategy(conditionalRequest, cacheResponse);
    }
```

## Cache

Cache 是 OkHtpp 的Cache管理器，负责将数据缓存到文件系统，或者从文件系统上取出响应的缓存信息。Cache 内部通过 DiskLruCache 管理 cache 在文件系统层面的创建，读取，清理等等工作。

### 构造方法

```
  Cache(File directory, long maxSize, FileSystem fileSystem) {
    this.cache = DiskLruCache.create(fileSystem, directory, VERSION, ENTRY_COUNT, maxSize);
  }
```

### 添加缓存

```
  @Nullable CacheRequest put(Response response) {
    // 获取网络请求方法
    String requestMethod = response.request().method();
    // 根据请求方法来判断是否需要缓存
    // POST、PATCH、PUT、DELETE、MOVE方法不做缓存
    if (HttpMethod.invalidatesCache(response.request().method())) {
      try {
        // 如果不是合法的，就移除缓存
        remove(response.request());
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
      return null;
    }
    // 如果请求方法不是get请求，那就直接返回null
    // 不做非get请求的缓存，虽然其它方法的响应可以缓存，但是做起来成本太大且效率低下，所以放弃
    if (!requestMethod.equals("GET")) {
      return null;
    }
    // 如果响应头含有 * 字符，那也不缓存，直接返回null
    if (HttpHeaders.hasVaryAll(response)) {
      return null;
    }
    // 创建Entry对象，这个对象封装了响应的一些信息
    Entry entry = new Entry(response);
    DiskLruCache.Editor editor = null;
    try {
      editor = cache.edit(key(response.request().url()));
      if (editor == null) {
        return null;
      }
      entry.writeTo(editor);
      // 构造 CacheRequestImpl 返回给 CacheInterceptor
      // 用来缓存响应体
      return new CacheRequestImpl(editor);
    } catch (IOException e) {
      abortQuietly(editor);
      return null;
    }
  }
```

### 获取缓存

```
  @Nullable Response get(Request request) {
    // 根据请求 url 生成 key
    String key = key(request.url());
    DiskLruCache.Snapshot snapshot;
    Entry entry;
    try {
      // 根据 key 取出缓存快照
      snapshot = cache.get(key);
      if (snapshot == null) {
        return null;
      }
    } catch (IOException e) {
      // Give up because the cache cannot be read.
      return null;
    }

    try {
      // 将快照中的缓存信息封装到Entry对象
      entry = new Entry(snapshot.getSource(ENTRY_METADATA));
    } catch (IOException e) {
      Util.closeQuietly(snapshot);
      return null;
    }
    // 将缓存中的数据构建成一个响应
    Response response = entry.response(snapshot);
    // 判断请求信息和缓存中的请求信息是否匹配
    if (!entry.matches(request, response)) {
      Util.closeQuietly(response.body());
      return null;
    }

    return response;
  }
```

### 更新缓存

```
  void update(Response cached, Response network) {
    Entry entry = new Entry(network);
    DiskLruCache.Snapshot snapshot = ((CacheResponseBody) cached.body()).snapshot;
    DiskLruCache.Editor editor = null;
    try {
      editor = snapshot.edit(); // Returns null if snapshot is not current.
      if (editor != null) {
        entry.writeTo(editor);
        editor.commit();
      }
    } catch (IOException e) {
      abortQuietly(editor);
    }
  }
```

## 推荐阅读

[OkHttp 3.7源码分析（四）——缓存策略](https://developer.aliyun.com/article/78102)
[OKHttp3--缓存拦截器CacheInterceptor源码解析【八】](https://blog.csdn.net/qq_30993595/article/details/87886518)
[OKHttp3-- HTTP缓存机制解析 缓存处理类Cache和缓存策略类CacheStrategy源码分析 【九】](https://blog.csdn.net/qq_30993595/article/details/87941893)
