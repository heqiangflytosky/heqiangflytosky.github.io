---
title: Android IPC 之 Messenger
categories: Android
comments: true
tags: [Android, Messenger]
description: 介绍Messenger的使用
date: 2015-2-2 10:00:00
---

## 概述
说起 Android 进程间通信，大家第一时间的想法可能是编写AIDL文件，但是Android上面还有另外一种简单方便的实现方式：`Messenger`，也就是这篇文章的主角。

## 实例
我们现在通过一个实例来介绍`Messenger`的使用方法。
实例中用到了两个APK，MessengerClient和MessengerServer。

### Server端
```java
public class MessengerService extends Service {

    private static final int MSG_ADD = 0;
    private final Messenger  mMessenger = new Messenger(new ServiceHandler());

    public MessengerService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        Log.e("Test","MessengerService onBind");
        return mMessenger.getBinder();
    }

    public static class ServiceHandler extends Handler{
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case MSG_ADD:
                    Message messageTo = Message.obtain();
                    messageTo.arg1 = msg.arg1 + msg.arg2;
                    try {
                        msg.replyTo.send(messageTo);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;

                default:
                    super.handleMessage(msg);
                    break;
            }
        }
    }
}
```
AndroidManifest.xml
```xml
        <service
            android:name=".MessengerService"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="com.android.hq.messenger"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
        </service>
```
可以看到Server端代码很简单，就是一个 `Service`，只需要去声明一个 `Messenger` 对象，然后 `onBind` 方法返回 `mMessenger.getBinder()`。

### Client端
```java
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";
    private static final int MSG_ADD = 0;

    private Messenger mService = null;
    // 缓存在bindService之前请求发送的消息
    private ArrayList<Runnable> mPendingTask = new ArrayList<>();
    private static final int STATUS_UNBIND = 0;
    private static final int STATUS_BINDING = 1;
    private static final int STATUS_BOUND = 2;
    private int mBindStatus = STATUS_UNBIND;
    private IBinder.DeathRecipient mDeathRecipient;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mDeathRecipient = new IBinder.DeathRecipient(){
            @Override
            public void binderDied() {
                Log.d(TAG,"binderDied");
                mServiceConnection.onServiceDisconnected(null);
                doBindService();
            }
        };
        doBindService();
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mServiceConnection);
    }

    private void doBindService(){
        if(mBindStatus == STATUS_UNBIND){
            mBindStatus = STATUS_BINDING;
            Intent intent = new Intent();
            intent.setAction("com.android.hq.messenger");
            intent.setPackage("com.android.hq.messengerserver");
//            intent.setClassName("com.android.hq.messengerserver", "com.android.hq.messengerserver.MessengerService");
            bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
        }

    }

    public void onAddClick(View v){
        Message message = Message.obtain(null, MSG_ADD, 100, 100);
        message.replyTo = mMessenger;
        sendMessage(message);
    }

    private void sendMessage(final Message message) {
        if(message == null)
            return;
        if (mService == null) {
            doBindService();
            synchronized (mPendingTask) {
                mPendingTask.add(new Runnable() {
                    @Override
                    public void run() {
                        sendMessage(message);
                    }
                });
            }
        } else {
            try {
                mService.send(message);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }

    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mBindStatus = STATUS_BOUND;
            mService = new Messenger(service);
            Toast.makeText(MainActivity.this, "onServiceConnected", Toast.LENGTH_SHORT).show();
            try {
                service.linkToDeath(mDeathRecipient, 0);
            } catch (RemoteException e) {
                Log.d(TAG, "linkToDeath", e);
            }
            synchronized (mPendingTask) {
                for (int i = 0; i < mPendingTask.size(); i++) {
                    mPendingTask.get(i).run();
                }
                mPendingTask.clear();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mService = null;
            mBindStatus = STATUS_UNBIND;
            Toast.makeText(MainActivity.this, "onServiceDisconnected", Toast.LENGTH_SHORT).show();
        }
    };

    private Messenger mMessenger = new Messenger(new Handler(){
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what){
                case MSG_ADD:
                    int result = msg.arg1;
                    Toast.makeText(MainActivity.this,"result = "+result, Toast.LENGTH_LONG).show();
                    break;
                default:
                    super.handleMessage(msg);
                    break;
            }
        }
    });

}
```

## 源码分析
看一下`Messenger`的源码，首先看一下它的构造函数：
```java
    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }

    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
```
我们先来分析第一个，看看这个`getIMessenger()`是怎么实现的，在Handler.java类中，它是这么实现的：
```java
    final IMessenger getIMessenger() {
        synchronized (mQueue) {
            if (mMessenger != null) {
                return mMessenger;
            }
            mMessenger = new MessengerImpl();
            return mMessenger;
        }
    }
```
那么这个`MessengerImpl()`类又是什么呢？
```java
    private final class MessengerImpl extends IMessenger.Stub {
        public void send(Message msg) {
            msg.sendingUid = Binder.getCallingUid();
            Handler.this.sendMessage(msg);
        }
    }
```
咦！`IMessenger.Stub`，似曾相识的感觉！是的，我们平时写aidl文件，就会生成一个`IXXX.Stub`的类，那么是不是也有个`IMessenger.aidl`的文件呢？哈，还真有，
frameworks/base/core/java/android/os/IMessenger.aidl：
```java
package android.os;

import android.os.Message;

/** @hide */
oneway interface IMessenger {
    void send(in Message msg);
}
```
到这里大家应该明白了，`Messenger`和我们用aidl文件的方式实现进程间通信其实是异曲同工的，它也是依赖aidl文件生成的类，继承了IMessenger.Stub类，实现了send方法，send方法中参数会通过客户端传递过来，最终发送给handler进行处理。
其实在它的另一个构造函数中已经体现出来了：
```java
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
```

## Messenger和AIDL区别
Messenger是以串行的方式处理客户端发来的消息，如果有大量的消息同时发送到服务端，服务端仍然只能一个一个的处理，如果有大量并发请求，Messenger就不太适用了。Messenger的作用主要是为了传递消息，很多时候我们可能需要跨进程调用服务端的方法，这种情形Messenger就无法做到了，这个时候可以直接使用AIDL来实现。

## 问题分析
在个别手机上会出现跨进程`bindService`失败的问题，可能是个别定制rom默认禁止掉了跨进程`bindService`这种方式，在原生系统里面是可以的。
或者是在手机管家里面设置允许应用后台运行即可。

## 参考资料 
http://blog.csdn.net/lmj623565791/article/details/47017485
