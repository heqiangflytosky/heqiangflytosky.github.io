---
title: Android 基础 -- 关于 Service
categories: Android
comments: true
tags: [Android]
description: 介绍 Service 的一些基础知识
date: 2015-12-20 10:00:00
---

## 概述

Service 是Android的四大组件之一，特点是可以提供一些在后台运行的没有UI界面的服务，甚至在程序退出的情况下仍然需要运行的服务。
**注意：** Service 是运行在后台，没有可视化的页面，我们很多时候会把耗时的操作放在Service中执行，但是注意，Service是运行在主线程的，不是在子线程中，
Service和Thread没有半点的关系，所以如果在Service中执行耗时操作，一样是需要开起线程，否则会引起ANR，这个需要区别开来。

## Service 分类

按照和客户端的进程关系可以分为：

 - 本地 Service：和客户端在同一个进程
 - 远程 Service：和客户端在不同的进程

按照Service运行的方式来分：

 - 前台 Service：在通知栏可以显示通知。前台Service优先级较高。
 - 后台 Service：处于后台，用户无法感知。后台Service优先级较低，当系统出现内存不足情况时，很有可能会被回收

## 启动 Service 方法

 - bindService：绑定服务，对应的解绑方法 unbindService()
 - startService：启动服务，对应停止方法：stopService()

### bindService

流程：

客户端首次bindService -> onCreate -> onBind -> 可以进行其他客户端的绑定 -> 所有客户端都 unbindService 后 -> onUnbind->onDestroy
这里要注意，Service 启动调用 onBind 之后，如果还有其他客户端继续调用 bindService 绑定服务，那么 onBind 不会再调用的，除非系统杀掉进行或者 crash。也就是说，如果Service存在的话，onBind 只会走一次。

### startService

流程：

客户端首次 startService -> onCreate -> onStartCommand -> onStart-> 可以多次 startService，此时只调用 onStartCommand 和 onStart -> stopService ->onDestroy

多次调用startService，该Service只能被创建一次，即该Service的onCreate方法只会被调用一次。但是每次调用startService，onStartCommand方法都会被调用。Service的onStart方法在API 5时被废弃，替代它的是onStartCommand方法。

### 先 startService 然后 bindService

流程：

startService  -> onCreate -> onStartCommand -> onStart ->(bindService) -> onBind -> (onServiceConnected) -> (unbindService) -> onUnbind -> (stopService) -> onDestroy

startService  -> onCreate -> onStartCommand -> onStart ->(bindService) -> onBind -> (onServiceConnected) ->  stopService（此时不会调用 onDestroy，因为还有其他绑定） -> (unbindService) -> onUnbind -> onDestroy

**注意：** 这里就要注意当调用 stopService 时的不同表现，如果当前有其他客户端绑定 Service，调用 stopService 是不会调用 onDestroy 的，除非解除所有绑定。

## 生命周期方法

### onCreate()

创建 Service 时调用，开始 Service 的声明周期。 

### onBind

客户端绑定 Service 时调动，Service 第一次被绑定时 onBind 方法会被调用，但是后面客户端再执行 bindService 时， onBind 方法并不会被多次调用，即并不会多次绑定服务。

### onStartCommand

每次执行 startService() 启动 Service 时都会调用一次 onStartCommond()。
4种返回值所代表的意义：

 - START_STICKY：如果service进程被kill掉，保留service的状态为开始状态，但不保留递送的intent对象。随后系统会尝试重新创建service，由于服务状态为开始状态，所以创建服务后一定会调用onStartCommand(Intent,int,int)方法。如果在此期间没有任何启动命令被传递到service，那么参数Intent将为null。
 - START_NOT_STICKY：“非粘性的”。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统将会把它置为started状态，系统不会自动重启该服务，直到startService(Intent intent)方法再次被调用;。
 - START_REDELIVER_INTENT：重传Intent。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统会自动重启该服务，并将Intent的值传入。
 - START_STICKY_COMPATIBILITY：START_STICKY的兼容版本，但不保证服务被kill后一定能重启。

### onUnbind

解除所有绑定时调用。比如在通过 unbindService 解绑后，此时没有和该 Service 有连接的客户端时，或者调用者Context不存在（如Activity被finish了）时，会调用此方法。

### onRebind

举例说明：如果在 Activity 中调用 content.bindService，注意这里的 content 是 Activity 对象。当我们没有调用 unbindService 解除绑定，而退出 Activity 时，也就是结束 Activity 的声明周期，这是会调用已经绑定的 Service 的 onUnbind 方法。
如果在上面介绍的 onUnbind 方法中返回 true，如果这个 Service 一直存活，那么下次再启动 Activity 进行绑定时，就会调用这个 onRebind 方法。

### onDestory

结束 Service 的生命周期。注意这里并不代表回收 Service 对象。

## 客户端和服务端通信

启动或者绑定 Service 之后，避免不了的要和 Service进行通信，下面介绍几种它们之间交互的几种方法。

### bindService

 1. bindService 方法会传递一个 intent 参数，当 Service 是第一次绑定时会回调 onBind 方法，这个 intent 会传递给 intent。也就是可以用intent传递一些信息。
 2. 绑定后，onBind 会返回一个 Binder 对象给客户端，客户端在 ServiceConnection 对象的 onServiceConnected 回调方法中收到这个 Binder 对象，后面的交互可以通过这个 Binder 对象来实现。具体可以参考后台的基本应用一节。
 3. 客户端和服务端在同一个进程时，还可以考虑扩展 Binder 的方式来实现交互。具体参考后面的介绍。
 4. 使用广播
 5. 客户端和服务端在同一个进程时，可以考虑使用一些全局变量或者对象来实现。

### startService

启动 Service 后，和客户端的通信限制比较大。

 1. 使用 startService  传递 intent 给 Service，在 Service 的 onStartCommand 方法中处理 Intent。
 2. 使用广播
 3. 客户端和服务端在同一个进程时，可以考虑使用一些全局变量或者对象来实现。

 
## 基本应用

来介绍一下 bindService 的基本应用。
还可以参考我的博客[ Android IPC 之 Messenger ](http://www.heqiangfly.com/2015/02/02/android-ipc-messenger/)。

服务端：

```
public class RemoteService extends Service{
    private final static String TAG = "RemoteService";

    public static final int CODE_1 = 1;

    private final Messenger mMessenger = new Messenger(new ServiceHandler());
    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        Log.d(TAG, "RemoteService onBind");
        // 返回一个实现 IBinder 接口的 Binder 对象。
        return mMessenger.getBinder();
    }

    private static class ServiceHandler extends Handler {
        @Override
        public void handleMessage(final Message msg) {
            switch (msg.what) {
                case CODE_1:
                    Log.d(TAG, "handleMessage CODE_1");
                    break;
                default:

                    break;
            }
        }
    }
}

```

客户端：

```
public class ServiceHelper {
    private final static String TAG = "ServiceHelper";

    private static volatile ServiceHelper sInstance;
    private static Context sContext;
    private ArrayList<Runnable> mPendingTask = new ArrayList<>();

    private static final int STATUS_UNBIND = 0;
    private static final int STATUS_BINDING = 1;
    private static final int STATUS_BOUND = 2;

    private int mBindStatus = STATUS_UNBIND;
    private final IBinder.DeathRecipient mDeathRecipient;
    private Messenger mService = null;
    private IBinder mBinder;

    private ServiceHelper(Context context) {
        sContext = context.getApplicationContext();
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

    public static void init(Context context) {
        if(sInstance == null) {
            synchronized (ServiceHelper.class) {
                if(sInstance == null) {
                    sInstance = new ServiceHelper(context);
                }
            }
        }
    }

    public static ServiceHelper getInstance() {
        return sInstance;
    }

    public void onDestroy() {
        if (mBindStatus == STATUS_BOUND && sContext != null && mServiceConnection != null) {
            sContext.unbindService(mServiceConnection);
            mBinder.unlinkToDeath(mDeathRecipient, 0);  
            sContext = null;
        }
    }

    private void doBindService(){
        Log.d(TAG,"ServiceConnection doBindService");
        if(mBindStatus == STATUS_UNBIND){
            mBindStatus = STATUS_BINDING;
            Intent intent = new Intent(sContext, RemoteService.class);
            sContext.bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
        }
    }

    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 得到 Service  服务的组件信息 name
            // 得到 onBind 方法返回的 Binder 对象 service
            Log.d(TAG,"ServiceConnection onServiceConnected");
            mBindStatus = STATUS_BOUND;
            mBinder = service;
            mService = new Messenger(service);

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
            Log.d(TAG,"ServiceConnection onServiceDisconnected");
            mBindStatus = STATUS_UNBIND;
            mService = null;
        }
    };

    public void sendTestMessage() {
        Bundle data = new Bundle();
        data.putString("KEY1", "Test");
        sendMessage(RemoteService.CODE_1, data);
    }

    private void sendMessage(final int flag, final Bundle data) {
        if (mService == null) {
            doBindService();
            synchronized (mPendingTask) {
                mPendingTask.add(new Runnable() {
                    @Override
                    public void run() {
                        sendMessage(flag, data);
                    }
                });
            }
        } else {
            Message message = Message.obtain(null, flag, 0, 0);
            message.setData(data);
            //message.replyTo = mMessenger;
            try {
                mService.send(message);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
}

```

## 进程内 bindService 扩展 binder

如果客户端和服务端在同一个进程，我们来看一下用来通信的 IBinder 的特点。
我们先来观察 Service 的 onBind 方法返回的 IBinder 引用对象：

![效果图](/images/react-native-image/image1.png)

再来对比客户端的 onServiceConnected 方法回调的参数 IBinder 引用对象：

![效果图](/images/react-native-image/image1.png)

它们居然是同一个指向同一个对象。
这就为我们进行服务端和客户端的通信添加了另外一个思路：扩展 IBinder。
继承 Binder 类（不能直接实现 IBinder 接口），添加自己要扩展的方法：

```
public class MyBinder extends Binder {

    public void testMethod() {
        Log.e("Test","testMethod()");
    }
}
```

然后在 Service 的 onBind 方法返回 MyBinder 的一个实例：

```
    private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.d(TAG,"ServiceConnection onServiceConnected");

            if (service instanceof MyBinder) {
                ((MyBinder) service).testMethod();
                return;
            }
            ......
        }
        ......
    };
```

结果 testMethod() 方法成功调用。
所以我们如果在同一个进程 bindService 的时候，如果调用了 unbindService，那么要及时将 onServiceConnected 获取的 Binder 引用置空，否则可能会虽然 Service 已经调用 onDestory，生命周期已经走完，但是客户端一直持有 Service 对象而导致对象无法回收，导致内存泄漏。

## 关于unbindService未调用onServiceDisconnected

 - onServiceDisconnected() 在连接正常关闭的情况下是不会被调用的。
 - 该方法只在Service 所在进程 Crash 了或者被杀死的时候调用。例如, 系统资源不足, 要关闭一些Services, 刚好连接绑定的 Service 是被关闭者之一,  这个时候onServiceDisconnected() 就会被调用