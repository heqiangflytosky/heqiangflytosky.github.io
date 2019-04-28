---
title: Android IPC 之 AIDL 异步调用
categories: Android
comments: true
tags: [Android, IPC, AIDL]
description: 介绍AIDL实现异步调用的方法
date: 2015-2-5 10:00:00
---
## 概述

上一篇博客 [Android IPC 之 AIDL ](http://www.heqiangfly.com/2015/02/01/android-ipc-aidl/) 中我们了解到 AIDL 是默认异步调用的，本文我们就探索如何实现 AIDL 的异步调用。
异步调用可以有两种方法实现：

 - 在 AILD 方法前添加 oneway 关键字
 - 使用 Callback 实现

## oneway 关键字

在上一篇的基础上我们在 aidl 文件中添加下面的方法：

```
interface StudentManager {
    ......

    oneway void testOneWay(String para);
}
```

在 Service 端添加：

```
    public static class MyStudentManager extends StudentManager.Stub{
        ......

        @Override
        public void testOneWay(String para) throws RemoteException {
            Log.d(TAG, "call testOneWay() para = "+para);
            Log.d(TAG, "start sleep");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.d(TAG, "sleep end !");
        }
    }
```

Client 端调用：

```
        Log.d(TAG, "client testOneway ");
        try {
            mStudentManager.testOneWay("TestOneway");
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        Log.d(TAG, "client testOneway end");
```

打印如下：

```
19:17:06.580 Client  : client testOneway 
19:17:06.580 Client  : client testOneway end
19:17:06.581 AIDLService: call testOneWay() para = TestOneway
19:17:06.581 AIDLService: start sleep
19:17:08.582 sleep end !
```

可见，已经实现了异步调用。

再来看一下 aidl 生成的 java 文件中：

```
@Override public void testOneWay(java.lang.String para) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeString(para);
mRemote.transact(Stub.TRANSACTION_testOneWay, _data, null, android.os.IBinder.FLAG_ONEWAY);
}
finally {
_data.recycle();
}
}
```

mRemote.transact 方法的最后一个参数变成了 android.os.IBinder.FLAG_ONEWAY。
**注意**：使用 oneway 关键字时参数不能设置为 out 或者 inout 类型。

## 回调函数方法

前面介绍的 oneway 关键字没办法获得执行的结果，那么下面再介绍一个用回调函数方法实现异步调用的方法。
增加一个 IMyAidlCallback.aidl 文件：

```
// IMyAidlCallback.aidl
package com.android.hq.aidldemo;

interface IMyAidlCallback {
    void onWorkDone(String action);
}
```

在 StudentManager.aidl 中增加一个方法：

```
// StudentManager.aidl
package com.android.hq.aidldemo;

import com.android.hq.aidldemo.Student;
import com.android.hq.aidldemo.IMyAidlCallback;

interface StudentManager {
    ......

    oneway void doWork(IMyAidlCallback callback);
}
```

在 service 端增加对该方法的处理：

```
    public static class MyStudentManager extends StudentManager.Stub{
        ......

        @Override
        public void doWork(IMyAidlCallback callback) throws RemoteException {
            Log.d(TAG, "call doWork()");
            Log.d(TAG, "start sleep");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.d(TAG, "sleep end !");
            callback.onWorkDone("Done");
        }
    }
```

client 端调用：

```
    public void testCallback(View view) {
        if(mStudentManager == null){
            return;
        }

        Log.d(TAG, "client doWork ");
        try {
            IMyAidlCallback callback = new IMyAidlCallback.Stub() {
                @Override
                public void onWorkDone(String action) throws RemoteException {
                    Log.e(TAG, "onWorkDone reply = "+action);
                }
            };
            mStudentManager.doWork(callback);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        Log.d(TAG, "client doWork end");
    }
```

打印测试结果：

```
19:57:33.153  Client  : client doWork 
19:57:33.154  Client  : client doWork end
19:57:33.154  AIDLService: call doWork()
19:57:33.154  AIDLService: start sleep
19:57:35.155  AIDLService: sleep end !
19:57:35.156  Client  : onWorkDone reply = Done
```

## RemoteCallbackList

RemoteCallbackList 是 Android 专门设计用来保存 Service 和 Clinet 端的回调函数列表的。比如我们设计一个观察者模式来监听 Service 端的变化来通知 Client 端，那么就 Service 就要维护一个 Client 端回调的列表，那么这个列表就可以用 RemoteCallbackList 来实现。具体用比如 ArrayList 和 Map 为什么不行，可以参考 《Android 开发艺术探索》这本书或者网上资料，这里不过多介绍。
