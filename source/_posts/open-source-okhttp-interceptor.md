---
title: OkHttp 源码分析 -- 拦截器
categories: Android开源项目
comments: true
tags: [OkHttp]
description: 介绍 OkHttp 的拦截器
date: 2019-9-20 10:00:00
---

## 概述

在 OkHttp 中，拦截器是非常重要的一个知识点，它将整个请求网络的过程的每一步都封装在不同的 Interceptor 里。这里使用到了设计模式中的责任链模式，即每个拦截器只处理与自己相关的业务逻辑。
OkHttp 不仅提供了 5 个固定的拦截器，而且我们还可以添加自定义的拦截器。就想在[OkHttp 使用指南 -- 基本使用]() 中介绍的添加 `HttpLoggingInterceptor` 一样。
我们先来看一下 OkHttp 提供的5个基础拦截器：

 - RetryAndFollowUpInterceptor：重试及重定向拦截器，负责请求的重试和重定向。
 - BridgeInterceptor：桥接拦截器，给请求添加对应的 header 信息，处理响应结果的 header 信息。
 - CacheInterceptor：缓存拦截器，根据当前获取的状态选择 网络请求 、读取缓存、更新缓存。
 - ConnectInterceptor：连接拦截器，建立 http 连接。
 - CallServerInterceptor：读写拦截器，读写网络数据。
 
有了这些拦截器，我们才能通过 OkHttp 进行网络的请求，这里先做一个概念性的了解，后面会做详细的介绍。

## 工作流程

### 创建拦截器

在我们创建网络请求任务后，不管是同步请求还是异步请求，网络请求的结果都是通过 `getResponseWithInterceptorChain()` 方法来获得的：


```
  @Override public Response execute() throws IOException {
    ......
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      ......
  }
```

```
  final class AsyncCall extends NamedRunnable {
    ......

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        ......
      } catch (IOException e) {
        ......
      }
    }
```

接下来，我们就研究一下 `getResponseWithInterceptorChain()` 方法。

```
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

它的工作流程大致分为两个部分：

 1.创建一系列的拦截器，包括用户自定义拦截器和框架提供的基础拦截器。
 2.创建一个拦截器链 `RealInterceptorChain`，调用 `proceed` 方法开始执行拦截器。

在这里创建 `RealInterceptorChain` 时，StreamAllocation、HttpCodec 和 RealConnection 参数都为 null，不用担心，这些参数在需要的时候会在后面的拦截器里面创建相应的对象。

### 拦截器链

拦截器链的作用就是负责按照拦截器的添加顺序执行拦截器定义的相关操作，来获取服务器的响应返回。
我们先来分析一下拦截器链的主要代码：

```
  public RealInterceptorChain(List<Interceptor> interceptors, StreamAllocation streamAllocation,
      HttpCodec httpCodec, RealConnection connection, int index, Request request, Call call,
      EventListener eventListener, int connectTimeout, int readTimeout, int writeTimeout) {
    this.interceptors = interceptors;
    this.connection = connection;
    this.streamAllocation = streamAllocation;
    this.httpCodec = httpCodec;
    this.index = index;
    this.request = request;
    this.call = call;
    this.eventListener = eventListener;
    this.connectTimeout = connectTimeout;
    this.readTimeout = readTimeout;
    this.writeTimeout = writeTimeout;
  }

  ......
  
  @Override public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
}
```

 1.首先判断一下拦截器是否执行完毕，执行完毕就抛出一个 AssertionError 异常，一般不会满足这个条件。
 2.为下一个拦截器创建一个新的拦截器链。
 3.从拦截器数组中取出当前需要执行的拦截器，开始执行拦截器的实现 `intercept` 方法，并把创建的下一个拦截器链作为参数传递进去。获取拦截器的返回结果。
 4.从 `intercept` 开始就会递归调动拦截器数组里面所有的拦截器。在拦截器的 `intercept` 里面一般会在调用新创建的拦截器链的 `proceed` 方法之前（请求信息）或者之后（响应信息）加入自己需要执行的代码。直到调用最后一个拦截器 `CallServerInterceptor`，由于是最后一个拦截器，它的 `intercept` 不会再调用参数 `chain` 的 `proceed` 方法。
 5.执行完后就再一层一层的返回响应信息。

其实看我后我们就会觉得 OkHttp 的实现其实挺简单，就是这些链接器的执行过程。下面开始简单介绍一下拦截器。

## 拦截器

拦截器的执行一般为三个部分：

 - 在发起请求前对request进行处理。
 - 调用下一个拦截器，获取response。
 - 对response进行处理，返回给上一个拦截器。

这就是 OkHttp 拦截器机制的核心逻辑。所以一个网络请求实际上就是一个个拦截器执行其 intercept 方法的过程。
拦截器关键的代码就是 `chain.proceed()` 方法的执行，我们知道在下一个拦截器链中又会执行下一个拦截器的intercept方法。所以整个执行链就在拦截器与拦截器链中交替执行，最终完成所有拦截器的操作。这也是 OkHttp 拦截器的链式执行逻辑。

### RetryAndFollowUpInterceptor

重试及重定向拦截器，负责请求的重试和重定向。主要是初始化的一些工作，创建 StreamAllocation 对象，用来传递给后面的拦截器。

```
  @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();
    // 创建 StreamAllocation 对象
    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;

    int followUpCount = 0;
    Response priorResponse = null;
    // 从这里开始，整个方法进入循环
    while (true) {
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;
      try {
        //执行下一个拦截器链的proceed方法
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // 如果不需要重试一起没有设置多路由等逻辑，这里直接抛出异常，否则就重新请求
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getFirstConnectException();
        }
        // 无需释放资源
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // IOException 异常时也要判断 recover 逻辑
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        // 抛出异常后，释放资源
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Request followUp;
      try {
        // 处理重定向逻辑，如果返回不为null，表示需要重定向
        followUp = followUpRequest(response, streamAllocation.route());
      } catch (IOException e) {
        streamAllocation.release();
        throw e;
      }
      // 返回null时，释放资源，跳出while循环返回 response
      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      closeQuietly(response.body());
      // 判断是否大于重定向次数
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }
      // 重新执行重定向请求
      request = followUp;
      priorResponse = response;
    }
  }
```

`recover` 方法用来判断是否要重新进行网络请求。

```
  private boolean recover(IOException e, StreamAllocation streamAllocation,
      boolean requestSendStarted, Request userRequest) {
    streamAllocation.streamFailed(e);

    // The application layer has forbidden retries.
    if (!client.retryOnConnectionFailure()) return false;

    // We can't send the request body again.
    if (requestSendStarted && userRequest.body() instanceof UnrepeatableRequestBody) return false;

    // This exception is fatal.
    if (!isRecoverable(e, requestSendStarted)) return false;

    // No more routes to attempt.
    if (!streamAllocation.hasMoreRoutes()) return false;

    // For failure recovery, use the same route selector with a new connection.
    return true;
  }
```

### BridgeInterceptor

桥接拦截器主要进行下面的工作：

 - 将用户的 Request 信息构建一个能进行网络访问的请求，并给请求添加缺少的一些必要的 header 信息。
 - 处理响应结果的 header 信息。

```
  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    // 对header信息进行补充
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }
    // 默认是保持连接
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }
    //默认是 GZIP 压缩的
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }
    //设置默认的 user-egent "okhttp/3.11.0"
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }
    //发送网络请求，执行下一个拦截器，接收返回结果
    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
    //当服务器返回的数据是 GZIP 压缩的，那么客户端去进行解压操作
    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      String contentType = networkResponse.header("Content-Type");
      responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }
    //构建一个Response
    return responseBuilder.build();
  }
```

### CacheInterceptor

缓存拦截器，根据当前获取的状态选择 网络请求 、读取缓存、更新缓存。

### ConnectInterceptor

连接拦截器，建立 http 连接。是 OkHttp 核心拦截器，网络交互的关键。

```
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    // StreamAllocation 对象是在 RetryAndFollowUpInterceptor 中创建的
    // 在这里拦截使用
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    // 通过socket建立和服务器的连接
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    // 得到前面建立的连接
    RealConnection connection = streamAllocation.connection();
    // 执行下一个拦截器
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

ConnectInterceptor 的执行流程：

```
├── ConnectInterceptor.intercept
    ├── StreamAllocation.newStream
        ├── StreamAllocation.findHealthyConnection
            ├── StreamAllocation.findConnection
                ├── RealConnection.connect
                    ├── RealConnection.connectSocket
                        ├── SocketFactory().createSocket()
                        ├── Platform.get().connectSocket
                            ├── AndroidPlatform.connectSocket
                                ├── Socket.connect
```

**注意**：这里直接调用的是 `RealInterceptorChain` 重载的 `proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec, RealConnection connection)` 方法，传递的都是非空的参数，和在 `RealCall` 的 `getResponseWithInterceptorChain()`  方法以及前面的拦截器里面调用的 `proceed` 有所区别。

### CallServerInterceptor

读写拦截器，读写网络数据。主要负责将请求写入到 IO 流当中，并且从 IO 流当中获取服务端返回给客服端的响应数据，它也是 OkHttp 核心拦截器，网络交互的关键。
