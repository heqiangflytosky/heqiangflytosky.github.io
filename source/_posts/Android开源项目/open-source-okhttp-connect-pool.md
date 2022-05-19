---
title: OkHttp 源码分析 -- 连接池
categories: Android开源项目
comments: true
tags: [OkHttp]
description: 介绍 OkHttp 的连接池
date: 2019-10-15 10:00:00
---

## 简介

连接池也是OkHttp的核心部分。通过维护连接池，最大限度重用现有连接，减少网络连接的创建开销，以此提升网络请求效率。
OkHttp 使用 ConnectInterceptor 拦截器来负责建立与服务器的连接，那么连接池就工作在这一阶段。

## keep-alive机制

### HTTP 1.0

在 Http 1.0 中，每次连接只处理一个请求，服务端在响应客户端的请求后，就主动断开连接，不继续维护该连接。但是如果一个网页包含很多图片，典型场景下这些资源中大部分来自同一个站点。按照 Http 1.0 的做法，每个图片的请求都要创建一个连接，意味着一次 Socket 的创建销毁，创建一个TCP连接需要3次握手，而释放连接则需要2次或4次握手，又是一个相对比较费时的过程。重复的创建和释放连接极大地影响了网络效率，同时也增加了系统开销。

### HTTP 1.1

为了有效地解决这一问题，HTTP/1.1提出了Keep-Alive机制：当一个HTTP请求的数据传输结束后，TCP连接不立即释放，如果此时有新的HTTP请求，且其请求的Host通上次请求相同，则可以直接复用为释放的TCP连接，从而省去了TCP的释放和再次创建的开销，减少了网络延时。在现代浏览器中，一般同时开启6～8个keepalive connections的socket连接，并保持一定的链路生命，当不需要时再关闭；而在服务器中，一般是由软件根据负载情况(比如FD最大值、Socket内存、超时时间、栈内存、栈数量等)决定是否主动关闭。
在该版本中默认是打开持久连接的，即在请求头中添加Connection:keep-alive，如果要关闭，需要在请求头中指定Connection:close，keep-alive不会永久保持连接，它有一个保持时间，可以在不同的服务器软件（如Apache）中设定这个时间。

## HTTP 2.0

HTTP 2.0 的改进如下：

 - 报头压缩：HTTP/2使用HPACK压缩格式压缩请求和响应报头数据，减少不必要流量开销。
 - 新的二进制格式：http/1.x使用的是明文协议，其协议格式由三部分组成：request line，header，body，其协议解析是基于文本，但是这种方式存在天然缺陷，文本的表现形式有多样性，要做到健壮性考虑的场景必然很多，二进制则不同，只认0和1的组合；基于这种考虑，http/2.0的协议解析决定采用二进制格式，实现方便且健壮。
 - 多路复用：HTTP 2通过引入新的二进制分帧层实现了完整的请求和响应复用，客户端和服务器可以将HTTP消息分解为互不依赖的帧，然后交错发送，最后再在另一端将其重新组装。在一个 TCP 连接上，我们可以向对方不断发送帧，每帧的 stream identifier 的标明这一帧属于哪个流，然后在对方接收时，根据 stream identifier 拼接每个流的所有帧组成一整块数据。把 HTTP/1.1 每个请求都当作一个流，那么多个请求变成多个流，请求响应数据分成多个帧，不同流中的帧交错地发送给对方，这就是 HTTP/2 中的多路复用。流的概念实现了单连接上多请求 - 响应并行，解决了线头阻塞的问题，减少了 TCP 连接数量和 TCP 连接慢启动造成的问题.http2 对于同一域名只需要创建一个连接，而不是像 http/1.1 那样创建 6~8 个连接。
 - 数据流优先级：将 HTTP 消息分解为很多独立的帧之后，我们就可以复用多个数据流中的帧，客户端和服务器交错发送和传输这些帧的顺序就成为关键的性能决定因素。为了做到这一点，HTTP/2 标准允许每个数据流都有一个关联的权重和依赖关系

## 源码分析

无论是HTTP/1.1的Keep-Alive机制还是HTTP/2的多路复用机制，在实现上都需要引入连接池来维护网络连接。接下来看下OkHttp中的连接池实现。
OkHttp内部通过 ConnectionPool 来管理连接池，主要涉及下面几个类：

 - StreamAllocation
 - RealConnection
 - ConnectionPool
 - HttpCodec：两个子类：Http1Codec和Http2Codec，分别对应Http1.1协议以及Http2.0协议。具体在介绍 CallServerInterceptor 时详细介绍。

### ConnectInterceptor

```
public final class ConnectInterceptor implements Interceptor {
  public final OkHttpClient client;

  public ConnectInterceptor(OkHttpClient client) {
    this.client = client;
  }

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
}
```

### 流程

```
├── StreamAllocation.newStream
    ├── StreamAllocation.findHealthyConnection
        ├── StreamAllocation.findConnection
            ├── Internal.instance.get 尝试从连接池中获取连接，如果存在，返回该连接
            ├── routeSelector.next() 寻找新的路由
                ├── RouteSelector.nextProxy()
                    ├── RouteSelector.resetNextInetSocketAddress
                        ├── address.dns().lookup dns解析
            ├── RealConnection.connect 进行 TCP + TLS 握手
                ├── RealConnection.connectSocket
                ├── RealConnection.establishProtocol
                    ├── RealConnection.connectTls
                    ├── RealConnection.startHttp2
    ├── RealConnection.newCodec 确定是使用http1还是http2协议
```

### 获取连接

重点介绍一下 `StreamAllocation.findConnection` 方法：

```
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new streams.
      // 一个StreamAllocation刻画的是一个Call的数据流动，
      // 一个Call可能存在多次请求(重定向，Authenticate等)，
      // 所以当发生类似重定向等事件时优先使用原有的连接
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {
        // 如果有正在连接的连接时就直接使用
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }

      if (result == null) {
        // 如果上面没有正在连接的连接，就从连接池里面找
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // 获取路由配置
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // 进行 TCP + TLS 握手
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // 将新建的连接放入到连接池中
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```


### 清理连接

ConnectionPool有一个独立的线程cleanupRunnable来清理连接池。

```
  private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
        //对连接池进行清理，返回下次清理等待时间
        long waitNanos = cleanup(System.nanoTime());
        if (waitNanos == -1) return;
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
              // 进入超时等待
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };
```

当连接池中put新的连接时，开始执行 cleanupRunnable。

```
void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
```
这段死循环实际上是一个阻塞的清理任务，首先进行清理(clean)，并返回下次需要清理的间隔时间，然后调用wait(timeout)进行等待以释放锁与时间片，当被唤醒后，再次进行清理，并返回下次要清理的间隔时间。
其唤醒时机有两个：

 - 超时时间到。
 - 调用 connectionBecameIdle 时，这个方法在 StreamAllocation.deallocate 方法中调用。

```
  boolean connectionBecameIdle(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (connection.noNewStreams || maxIdleConnections == 0) {
      connections.remove(connection);
      return true;
    } else {
      // 唤醒cleanup线程
      notifyAll(); // Awake the cleanup thread: we may have exceeded the idle connection limit.
      return false;
    }
  }
```

再来看一下 cleanup 方法：

```
  long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // 遍历连接池中的所有连接
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // 如果当前连接正在使用，开始遍历下一个连接
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }
        // 空闲连接数+1
        idleConnectionCount++;

        //选择排序法，标记出空闲时间最长的连接
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // 如果空闲连接已经空闲了 5 秒
        // 或者如果空闲连接超过 5 个
        // 将改连接从连接池中移除
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // 如果有空闲连接，就返回到期的时间
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // 如果没有空闲连接，就 5 秒后再执行清理
        return keepAliveDurationNs;
      } else {
        //没有任何连接，跳出循环
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // 立即进行再次清理
    return 0;
  }
```

## 推荐文章

https://developer.aliyun.com/article/78101
https://mp.weixin.qq.com/s/TeQhe4T4wRjdAEPz6Ne45g
https://blog.csdn.net/qq_30993595/article/details/94405694