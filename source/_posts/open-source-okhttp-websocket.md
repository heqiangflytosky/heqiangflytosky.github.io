---
title: OkHttp 源码分析 -- WebSocket 实现原理
categories: Android开源项目
comments: true
tags: [OkHttp]
description: 介绍 OkHttp 的缓存
date: 2019-10-10 10:00:00
---

## 概述

WebSocket 是在单个 TCP 连接上进行全双工通讯的协议。WebSocket 使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在 WebSocket API 中，客户端和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。
WebSocket 是真正意义上的全双工模式，也就是我们俗称的「长连接」。
WebSocket 是基于TCP/IP协议，独立于 HTTP 协议的通信协议。是应用层的通信协议。
协议标识符变成了 "ws" 或者 "wss"，分别表示明文和加密的 WebSocket 协议。
OkHttp 实现了对 WebSocket 的支持，本文简单介绍一下 OkHttp 实现 WebSocket 的基本原理。

## 基础知识

### WebSocket 和 http 协议的 keep-alive

keep-alive 是 http 协议规定的一种连接模式，在非KeepAlive模式时，每个请求/应答客户和服务器都要新建一个连接，完成之后立即断开连接（HTTP协议为无连接的协议）；当使用Keep-Alive模式（又称持久连接、连接重用）时，Keep-Alive功能使客户端到服务器端的连接持续有效，当出现对服务器的后继请求时，Keep-Alive功能避免了建立或者重新建立连接。
http 1.0中默认是关闭的，需要在http头加入"Connection:Keep-Alive"，才能启用Keep-Alive；http 1.1中默认启用Keep-Alive，如果加入"Connection:close"，才关闭。目前大部分浏览器都是用http1.1协议，也就是说默认都会发起Keep-Alive的连接请求了，所以是否能完成一个完整的Keep- Alive连接就看服务器设置情况。
http keep-alive只是一种为了达到复用tcp连接的“协商”行为，双方并没有建立正真的连接会话，服务端也可以不认可，也可以随时（在任何一次请求完成后）关闭掉。它是指在一次 TCP 连接中完成多个 http请求，但是对每个请求仍然要单独发header，所以除了真正的数据部分外，服务器和客户端还要大量交换httpheader，信息交换效率很低，这样建立的“长连接”都是伪长连接。
websocket 不同，它本身就规定了是正真的、双工的长连接，两边都必须要维持住连接的状态。

### 断线重连机制

造成WebSocket断线原因：
 
 - 网络状态不好（网络断开、网络信号差）。
 - 数据受各种阻塞（路由器、防火墙、代理服务器）。
 - Web服务端故障。

解决 websocket 断线方法：心跳重连。

1. 通过服务端实现 Pings / Pongs 方式实现心跳。
即通过服务端向浏览器（客户端）发送ping 0x9 消息，浏览器会自动返回pong 0xA消息。（基于session技术）
在经过握手之后的任意时刻里，无论客户端还是服务端都可以选择发送一个ping给另一方。 当ping消息收到的时候，接受的一方必须尽快回复一个pong消息。可以使用这种方式来确保客户端的连接状态。
一个ping或者pong都只是一个常规的帧，只是这个帧是一个控制帧。ping消息的opcode字段值为0x9，pong消息的opcode值为0xA。当你得到一个ping消息时，回复一个跟ping消息有相同载荷数据的pong消息。
2. 通过浏览器（客户端）自带的心跳机制。
即监听网络，浏览器（客户端）定时发送消息到服务端。
PS：这里消息指的websockect协议的数据帧，不需要通过代码实现。不同浏览器发送的消息时间间隔有所差异。
3. 通过代码在浏览器（客户端）实现心跳机制。

**注意**：心跳重连只能由服务端实现，不建议由客户端实现。


## 使用方法

借助 [https://www.websocket.org/](https://www.websocket.org/echo.html)，该网站提供了一个测试接口wss://echo.websocket.org。

```
    private void websocket() {

        if (mWebsocket != null) {
            mWebsocket.send("test");
            return;
        }

        okhttp3.Request okRequest = new okhttp3.Request.Builder()
                .url("wss://echo.websocket.org")
                .build();
        mWebsocket = mOkHttpClient.newWebSocket(okRequest, new WebSocketListener() {
            @Override
            public void onOpen(WebSocket webSocket, Response response) {
                super.onOpen(webSocket, response);
            }

            @Override
            public void onMessage(WebSocket webSocket, String text) {
                super.onMessage(webSocket, text);
            }

            @Override
            public void onMessage(WebSocket webSocket, ByteString bytes) {
                super.onMessage(webSocket, bytes);
            }

            @Override
            public void onClosing(WebSocket webSocket, int code, String reason) {
                super.onClosing(webSocket, code, reason);
            }

            @Override
            public void onClosed(WebSocket webSocket, int code, String reason) {
                super.onClosed(webSocket, code, reason);
            }

            @Override
            public void onFailure(WebSocket webSocket, Throwable t, Response response) {
                super.onFailure(webSocket, t, response);
            }
        });
    }
    
    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mWebsocket != null) {
            mWebsocket.close(1000,"test");
        }
    }
```

建立成功连接后会回调 `onOpen`，后面我们用 `mWebsocket.send` 向服务器发送一条消息，www.websocket.org 服务器马上会通过 `onMessage` 返回一条消息给我们。
而且每隔 30 秒，www.websocket.org 服务器会向客户端发送一条 ping 消息来确保连接的状态。
另外，我们在创建 OkHttpClient 对象时，可以调用 `pingInterval(long interval, TimeUnit unit)` 方法来配置客户端向服务端发送心跳包，可以定时向服务器发送 ping 消息。

## 实现原理

我们就从 `OkHttpClient.newWebSocket` 方法开始分析。

```
  @Override public WebSocket newWebSocket(Request request, WebSocketListener listener) {
    RealWebSocket webSocket = new RealWebSocket(request, listener, new Random(), pingInterval);
    // 连接服务器
    webSocket.connect(this);
    return webSocket;
  }
```

`RealWebSocket` 是 `WebSocket` 的实现类：

```
  public RealWebSocket(Request request, WebSocketListener listener, Random random,
      long pingIntervalMillis) {
    if (!"GET".equals(request.method())) {
      throw new IllegalArgumentException("Request must be GET: " + request.method());
    }
    this.originalRequest = request;
    this.listener = listener;
    this.random = random;
    this.pingIntervalMillis = pingIntervalMillis;

    byte[] nonce = new byte[16];
    random.nextBytes(nonce);
    this.key = ByteString.of(nonce).base64();
    // 创建向服务器发送消息的线程
    this.writerRunnable = new Runnable() {
      @Override public void run() {
        try {
          while (writeOneFrame()) {
          }
        } catch (IOException e) {
          failWebSocket(e, null);
        }
      }
    };
  }

```

通过调用 connect 方法，最终建立与服务器的连接：

```
  public void connect(OkHttpClient client) {
    client = client.newBuilder()
        .eventListener(EventListener.NONE)
        .protocols(ONLY_HTTP1)
        .build();
    // 添加 WebSocket 先关的一些头信息
    final Request request = originalRequest.newBuilder()
        .header("Upgrade", "websocket")
        .header("Connection", "Upgrade")
        .header("Sec-WebSocket-Key", key)
        .header("Sec-WebSocket-Version", "13")
        .build();
    // 创建一个 RealCall 对象
    call = Internal.instance.newWebSocketCall(client, request);
    // 加入到请求队列中，长连接会使线程在线程池里不会得到释放，所以使用异步请求
    call.enqueue(new Callback() {
      @Override public void onResponse(Call call, Response response) {
        try {
          // 有响应返回时，检查响应
          checkResponse(response);
        } catch (ProtocolException e) {
          failWebSocket(e, response);
          closeQuietly(response);
          return;
        }

        // Promote the HTTP streams into web socket streams.
        StreamAllocation streamAllocation = Internal.instance.streamAllocation(call);
        streamAllocation.noNewStreams(); // Prevent connection pooling!
        Streams streams = streamAllocation.connection().newWebSocketStreams(streamAllocation);

        // Process all web socket messages.
        try {
          // 回调 onOpen 方法
          listener.onOpen(RealWebSocket.this, response);
          String name = "OkHttp WebSocket " + request.url().redact();
          // 初始化消息读写器
          initReaderAndWriter(name, streams);
          streamAllocation.connection().socket().setSoTimeout(0);
          // 进入while循环，开始轮询读取服务器发来的消息
          loopReader();
        } catch (Exception e) {
          failWebSocket(e, null);
        }
      }

      @Override public void onFailure(Call call, IOException e) {
        failWebSocket(e, null);
      }
    });
  }
```

```
  public void loopReader() throws IOException {
    while (receivedCloseCode == -1) {
      // 处理消息
      reader.processNextFrame();
    }
  }
  
  void processNextFrame() throws IOException {
    // 读取服务器消息的头信息
    readHeader();
    // 判断是控制帧还是消息帧
    if (isControlFrame) {
      readControlFrame();
    } else {
      readMessageFrame();
    }
  }
```

```
  private void readHeader() throws IOException {
    if (closed) throw new IOException("closed");

    // Disable the timeout to read the first byte of a new frame.
    int b0;
    long timeoutBefore = source.timeout().timeoutNanos();
    source.timeout().clearTimeout();
    try {
      // 服务器没有消息时会阻塞在这里
      b0 = source.readByte() & 0xff;
    } finally {
      source.timeout().timeout(timeoutBefore, TimeUnit.NANOSECONDS);
    }
    // 根据读取信息标记位判断消息的类别
    opcode = b0 & B0_MASK_OPCODE;
    isFinalFrame = (b0 & B0_FLAG_FIN) != 0;
    isControlFrame = (b0 & OPCODE_FLAG_CONTROL) != 0;

    // Control frames must be final frames (cannot contain continuations).
    if (isControlFrame && !isFinalFrame) {
      throw new ProtocolException("Control frames must be final.");
    }

    boolean reservedFlag1 = (b0 & B0_FLAG_RSV1) != 0;
    boolean reservedFlag2 = (b0 & B0_FLAG_RSV2) != 0;
    boolean reservedFlag3 = (b0 & B0_FLAG_RSV3) != 0;
    if (reservedFlag1 || reservedFlag2 || reservedFlag3) {
      // Reserved flags are for extensions which we currently do not support.
      throw new ProtocolException("Reserved flags are unsupported.");
    }

    int b1 = source.readByte() & 0xff;

    boolean isMasked = (b1 & B1_FLAG_MASK) != 0;
    if (isMasked == isClient) {
      // Masked payloads must be read on the server. Unmasked payloads must be read on the client.
      throw new ProtocolException(isClient
          ? "Server-sent frames must not be masked."
          : "Client-sent frames must be masked.");
    }

    // Get frame length, optionally reading from follow-up bytes if indicated by special values.
    frameLength = b1 & B1_MASK_LENGTH;
    if (frameLength == PAYLOAD_SHORT) {
      frameLength = source.readShort() & 0xffffL; // Value is unsigned.
    } else if (frameLength == PAYLOAD_LONG) {
      frameLength = source.readLong();
      if (frameLength < 0) {
        throw new ProtocolException(
            "Frame length 0x" + Long.toHexString(frameLength) + " > 0x7FFFFFFFFFFFFFFF");
      }
    }

    if (isControlFrame && frameLength > PAYLOAD_BYTE_MAX) {
      throw new ProtocolException("Control frame must be less than " + PAYLOAD_BYTE_MAX + "B.");
    }

    if (isMasked) {
      // Read the masking key as bytes so that they can be used directly for unmasking.
      source.readFully(maskKey);
    }
  }
```

处理服务器的消息：

```
  private void readControlFrame() throws IOException {
    if (frameLength > 0) {
      source.readFully(controlFrameBuffer, frameLength);

      if (!isClient) {
        controlFrameBuffer.readAndWriteUnsafe(maskCursor);
        maskCursor.seek(0);
        toggleMask(maskCursor, maskKey);
        maskCursor.close();
      }
    }
    // 判断控制帧的类别
    switch (opcode) {
      case OPCODE_CONTROL_PING:
        // 处理ping消息，向服务器发送一条pong消息
        frameCallback.onReadPing(controlFrameBuffer.readByteString());
        break;
      case OPCODE_CONTROL_PONG:
        frameCallback.onReadPong(controlFrameBuffer.readByteString());
        break;
      case OPCODE_CONTROL_CLOSE:
        int code = CLOSE_NO_STATUS_CODE;
        String reason = "";
        long bufferSize = controlFrameBuffer.size();
        if (bufferSize == 1) {
          throw new ProtocolException("Malformed close payload length of 1.");
        } else if (bufferSize != 0) {
          code = controlFrameBuffer.readShort();
          reason = controlFrameBuffer.readUtf8();
          String codeExceptionMessage = WebSocketProtocol.closeCodeExceptionMessage(code);
          if (codeExceptionMessage != null) throw new ProtocolException(codeExceptionMessage);
        }
        frameCallback.onReadClose(code, reason);
        closed = true;
        break;
      default:
        throw new ProtocolException("Unknown control opcode: " + toHexString(opcode));
    }
  }
  
  private void readMessageFrame() throws IOException {
    int opcode = this.opcode;
    if (opcode != OPCODE_TEXT && opcode != OPCODE_BINARY) {
      throw new ProtocolException("Unknown opcode: " + toHexString(opcode));
    }
    // 读取消息体
    readMessage();
    // 调用客户端回调 onMessage
    if (opcode == OPCODE_TEXT) {
      frameCallback.onReadMessage(messageFrameBuffer.readUtf8());
    } else {
      frameCallback.onReadMessage(messageFrameBuffer.readByteString());
    }
  }
```

