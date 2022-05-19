---
title: Android IPC 之异常的跨进程传递
categories: Android
comments: true
tags: [Android, IPC]
description: 介绍 Android 异常的跨进程传递
date: 2015-2-8 10:00:00
---

## 概述

如果我们在 Server 端出现的异常，想让 Client 端知晓，这个要怎么做呢？

## 验证和分析

现在我们通过一个 ContentProvider 来测试一下这个场景，比如我们在 Server 端抛一个 `RuntimeException` 异常，发现在 Client 端并未收到这个异常，在 Server 端有下面的日志：

```
JavaBinder: *** Uncaught remote exception!  (Exceptions are not yet supported across processes.)
JavaBinder: java.lang.RuntimeException: Test
JavaBinder: 	at com.**.*Service$*ServiceStub.hasBook(*Service.java:105)
JavaBinder: 	at com.**.I*Service$Stub.onTransact(I*Service.java:131)
JavaBinder: 	at android.os.Binder.execTransact(Binder.java:739)
```

那是不是进程间通信不能传递异常呢？开发的经验告诉我们这是不可能的，那究竟是什么原因呢？现在我们通过源码来分析一下：

先从 Server 端的代码分析一下：

```
    @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
    {
      java.lang.String descriptor = DESCRIPTOR;
      ...
        case TRANSACTION_hasQuickApp:
        {
          data.enforceInterface(descriptor);
          java.lang.String _arg0;
          _arg0 = data.readString();
          boolean _result = this.hasBook(_arg0);
          reply.writeNoException();
          reply.writeInt(((_result)?(1):(0)));
          return true;
        }
        
```

上面的 Stub 方法是 Binder.execTransact 调用的：

```
    private boolean execTransact(int code, long dataObj, long replyObj,
            int flags) {
        Parcel data = Parcel.obtain(dataObj);
        Parcel reply = Parcel.obtain(replyObj);
        // theoretically, we should call transact, which will call onTransact,
        // but all that does is rewind it, and we just got these from an IPC,
        // so we'll just call it directly.
        boolean res;
        // Log any exceptions as warnings, don't silently suppress them.
        // If the call was FLAG_ONEWAY then these exceptions disappear into the ether.
        try {
            res = onTransact(code, data, reply, flags);
        } catch (RemoteException|RuntimeException e) {
            if (LOG_RUNTIME_EXCEPTION) {
                Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
            }
            if ((flags & FLAG_ONEWAY) != 0) {
                if (e instanceof RemoteException) {
                    Log.w(TAG, "Binder call failed.", e);
                } else {
                    Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
                }
            } else {
                reply.setDataPosition(0);
                reply.writeException(e);
            }
            res = true;
        } catch (OutOfMemoryError e) {
            // Unconditionally log this, since this is generally unrecoverable.
            Log.e(TAG, "Caught an OutOfMemoryError from the binder stub implementation.", e);
            RuntimeException re = new RuntimeException("Out of memory", e);
            reply.setDataPosition(0);
            reply.writeException(re);
            res = true;
        }
        checkParcel(this, code, reply, "Unreasonably large binder reply buffer");
        reply.recycle();
        data.recycle();

        // Just in case -- we are done with the IPC, so there should be no more strict
        // mode violations that have gathered for this thread.  Either they have been
        // parceled and are now in transport off to the caller, or we are returning back
        // to the main transaction loop to wait for another incoming transaction.  Either
        // way, strict mode begone!
        StrictMode.clearGatheredViolations();

        return res;
    }
```

可以看到，`execTransact` 方法已经通过 `reply.writeException` 写入了 Server端抛出的异常，那为什么 Client 端没有收到呢？再来看一下 `Parcel.writeException` 方法：

```
    public final void writeException(@NonNull Exception e) {
        int code = 0;
        if (e instanceof Parcelable
                && (e.getClass().getClassLoader() == Parcelable.class.getClassLoader())) {
            // We only send Parcelable exceptions that are in the
            // BootClassLoader to ensure that the receiver can unpack them
            code = EX_PARCELABLE;
        } else if (e instanceof SecurityException) {
            code = EX_SECURITY;
        } else if (e instanceof BadParcelableException) {
            code = EX_BAD_PARCELABLE;
        } else if (e instanceof IllegalArgumentException) {
            code = EX_ILLEGAL_ARGUMENT;
        } else if (e instanceof NullPointerException) {
            code = EX_NULL_POINTER;
        } else if (e instanceof IllegalStateException) {
            code = EX_ILLEGAL_STATE;
        } else if (e instanceof NetworkOnMainThreadException) {
            code = EX_NETWORK_MAIN_THREAD;
        } else if (e instanceof UnsupportedOperationException) {
            code = EX_UNSUPPORTED_OPERATION;
        } else if (e instanceof ServiceSpecificException) {
            code = EX_SERVICE_SPECIFIC;
        }
        writeInt(code);
        StrictMode.clearGatheredViolations();
        if (code == 0) {
            if (e instanceof RuntimeException) {
                throw (RuntimeException) e;
            }
            throw new RuntimeException(e);
        }
        writeString(e.getMessage());
        final long timeNow = sParcelExceptionStackTrace ? SystemClock.elapsedRealtime() : 0;
        if (sParcelExceptionStackTrace && (timeNow - sLastWriteExceptionStackTrace
                > WRITE_EXCEPTION_STACK_TRACE_THRESHOLD_MS)) {
            sLastWriteExceptionStackTrace = timeNow;
            final int sizePosition = dataPosition();
            writeInt(0); // Header size will be filled in later
            StackTraceElement[] stackTrace = e.getStackTrace();
            final int truncatedSize = Math.min(stackTrace.length, 5);
            StringBuilder sb = new StringBuilder();
            for (int i = 0; i < truncatedSize; i++) {
                sb.append("\tat ").append(stackTrace[i]).append('\n');
            }
            writeString(sb.toString());
            final int payloadPosition = dataPosition();
            setDataPosition(sizePosition);
            // Write stack trace header size. Used in native side to skip the header
            writeInt(payloadPosition - sizePosition);
            setDataPosition(payloadPosition);
        } else {
            writeInt(0);
        }
        switch (code) {
            case EX_SERVICE_SPECIFIC:
                writeInt(((ServiceSpecificException) e).errorCode);
                break;
            case EX_PARCELABLE:
                // Write parceled exception prefixed by length
                final int sizePosition = dataPosition();
                writeInt(0);
                writeParcelable((Parcelable) e, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                final int payloadPosition = dataPosition();
                setDataPosition(sizePosition);
                writeInt(payloadPosition - sizePosition);
                setDataPosition(payloadPosition);
                break;
        }
    }
```

看明白了吗？这里只识别到了下面 8 中异常：

 - SecurityException
 - BadParcelableException
 - IllegalArgumentException
 - NullPointerException
 - IllegalStateException
 - NetworkOnMainThreadException
 - UnsupportedOperationException
 - ServiceSpecificException

如果将上面测试代码抛出的异常换成上面中的一种，那么 Client 端就可以收到了。
再来看一下 Client 代理类的方法：

```
      @Override public boolean hasBook(java.lang.String pkg) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        boolean _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeString(pkg);
          boolean _status = mRemote.transact(Stub.TRANSACTION_hasQuickApp, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().hasBook(pkg);
          }
          _reply.readException();
          _result = (0!=_reply.readInt());
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }
```

可以看到是可以通过 `Parcel.readException` 来读取对端的异常的。

```
    private Exception createException(int code, String msg) {
        switch (code) {
            case EX_PARCELABLE:
                if (readInt() > 0) {
                    return (Exception) readParcelable(Parcelable.class.getClassLoader());
                } else {
                    return new RuntimeException(msg + " [missing Parcelable]");
                }
            case EX_SECURITY:
                return new SecurityException(msg);
            case EX_BAD_PARCELABLE:
                return new BadParcelableException(msg);
            case EX_ILLEGAL_ARGUMENT:
                return new IllegalArgumentException(msg);
            case EX_NULL_POINTER:
                return new NullPointerException(msg);
            case EX_ILLEGAL_STATE:
                return new IllegalStateException(msg);
            case EX_NETWORK_MAIN_THREAD:
                return new NetworkOnMainThreadException();
            case EX_UNSUPPORTED_OPERATION:
                return new UnsupportedOperationException(msg);
            case EX_SERVICE_SPECIFIC:
                return new ServiceSpecificException(readInt(), msg);
        }
        return new RuntimeException("Unknown exception code: " + code
                + " msg " + msg);
    }
```

也是只识别了上面列出的几种异常。

## 结论

Android 异常的跨进程传递只能是下面 8 种异常：

 - SecurityException
 - BadParcelableException
 - IllegalArgumentException
 - NullPointerException
 - IllegalStateException
 - NetworkOnMainThreadException
 - UnsupportedOperationException
 - ServiceSpecificException

