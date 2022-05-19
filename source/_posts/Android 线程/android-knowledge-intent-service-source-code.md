---
title: Android 线程 -- IntentService 使用及源码分析
categories: Android 线程
comments: true
tags: [Android 线程]
description: 介绍 IntentService 使用及源码分析
date: 2016-9-2 10:00:00
---

## 概述

本文来介绍一下 `IntentService`，但从名字上看，应该是和 `Service` 有关，的确是这样，`IntentService` 是一种特殊的 `Service`，它继承了 `Service` 并且是一个抽象类，因此必须是它的子类才能使用 `IntentService`，`IntentService` 可用于执行后台耗时的任务，当任务执行完后它会自动停止，同时，由于 `IntentService` 是服务的原因，它的优先级比单纯的线程要高，因此可以用于执行一些高优先级的后台任务。

## 源码分析

在实现上，`IntentService` 封装了 `HandlerThread` 和 `Handler`。
我们先来看一下它的构造函数：

```
    public IntentService(String name) {
        super();
        mName = name;
    }
```

构造函数很简单，调用父类的构造函数，name 参数是赋给工作线程的名字。
我们再来看一下它的 `onCreate()` 方法：

```
    @Override
    public void onCreate() {
        startService(Context, Intent)

        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
```

`onCreate()` 方法会在 `IntentService` 第一次启动时被调用。
首先会构造一个 `HandlerThread`，并且根据构造函数的 name 参数线程命名。
然后获取 `HandlerThread` 的 `Looper` 对象构造一个 `Handler` 对象。
再来看一下 `onStartCommand` 方法：

```
    public void setIntentRedelivery(boolean enabled) {
        mRedelivery = enabled;
    }

    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
```

每次启动 `IntentService` ，它的 `onStartCommand` 方法就会被调用一次，`onStartCommand` 调用 `onStart` 来处理 `Intent`，然后根据 `mRedelivery` 变量来返回 `START_REDELIVER_INTENT` 或者 `START_NOT_STICKY`，`mRedelivery` 变量是通过 `setIntentRedelivery` 方法来赋值的。
下面来温习一下 `Service` 中 `onStartCommand` 方法4种返回值所代表的意义：

 1. START_STICKY：如果service进程被kill掉，保留service的状态为开始状态，但不保留递送的intent对象。随后系统会尝试重新创建service，由于服务状态为开始状态，所以创建服务后一定会调用onStartCommand(Intent,int,int)方法。如果在此期间没有任何启动命令被传递到service，那么参数Intent将为null。
 2. START_NOT_STICKY：“非粘性的”。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统将会把它置为started状态，系统不会自动重启该服务，直到startService(Intent intent)方法再次被调用。
 3. START_REDELIVER_INTENT：重传Intent。使用这个返回值时，如果在执行完onStartCommand后，服务被异常kill掉，系统会自动重启该服务，并将Intent的值传入。
 4. START_STICKY_COMPATIBILITY：START_STICKY的兼容版本，但不保证服务被kill后一定能重启。

`onStart` 方法对 `Intent` 的处理其实也就是通过 `mServiceHandler` 发送消息，然后在 `Handler` 中对消息进行处理。

```
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }
```

`Handler` 首先会在工作线程调用 `onHandleIntent` 来处理 `Intent`，这个方法是我们在继承 `IntentService` 时需要实现的方法。
在 `onHandleIntent` 方法执行结束后，会调用 `stopSelf(int startId)` 来尝试停止服务。这里之所以调用这个方法而不是 `stopSelf()` 方法是因为 `stopSelf()` 会立刻停止服务，而这个时候消息队列中可能还有其它消息未处理完。`stopSelf(int startId)` 则会等待所有的消息都处理完才会终止服务。
`stopSelf(int startId)` 方法会在尝试停止服务的时候判断最近启动服务的次数是否和 `startId` 相等，如果相等就立即停止，如果不相等则不停止服务。
因此，如果目前只存在一个后台任务，那么该任务一旦执行完，`IntentService` 就会停止，如果存在多个后台任务，那么当 `onHandleIntent` 执行完最后一个任务时才会停止服务。
如果 `IntentService` 停止后再次发送任务请求，那么就会创建新的 `IntentService` 对象来执行任务。

```
    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    }

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```

在 `IntentService` 销毁的时候会退出 `Looper` 循环。

## 使用方法

那么下面我们就带着上面分析源码的一些结论来验证一下：
因为 `IntentService` 是一个 `Service` ，那么就不要忘记在 AndriodManifest 里面添加声明。

```
    public void clickTestIntentService1(View view) {
        Intent intent = new Intent(this, MyIntentService.class);
        intent.putExtra("action","ACTION_ONE");
        startService(intent);
    }

    public void clickTestIntentService2(View view) {
        Intent intent = new Intent(this, MyIntentService.class);
        intent.putExtra("action","ACTION_TWO");
        startService(intent);
    }

    public void clickTestStopIntentService(View view) {
        Intent intent = new Intent(this, MyIntentService.class);
        stopService(intent);
    }

    public static class MyIntentService extends IntentService{

        public MyIntentService() {
            super("MyIntentService");
            Log.e("Test","MyIntentService Constructor " +this);
        }

        @Override
        public void onCreate() {
            Log.e("Test","MyIntentService onCreate ");
            super.onCreate();
        }

        @Override
        public void onStart(@Nullable Intent intent, int startId) {
            Log.e("Test","MyIntentService onStart ");
            super.onStart(intent, startId);
        }

        @Override
        public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
            Log.e("Test","MyIntentService onStartCommand ");
            return super.onStartCommand(intent, flags, startId);
        }

        @Override
        public void onDestroy() {
            Log.e("Test","MyIntentService onDestroy ");
            super.onDestroy();
        }

        @Override
        protected void onHandleIntent(@Nullable Intent intent) {
            Log.e("Test","MyIntentService onHandleIntent " +Thread.currentThread());
            String action = intent.getStringExtra("action");
            Log.e("Test","MyIntentService onHandleIntent action " +action+" start");
            switch (action) {
                case "ACTION_ONE":
                    try {
                        Thread.sleep(5000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    break;
                case "ACTION_TWO":
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    break;
            }

            Log.e("Test","MyIntentService onHandleIntent action " +action+" end");
        }
    }

```

### 正常执行一个任务

```
Test: MyIntentService Constructor com.example.heqiang.testsomething.MainActivity$MyIntentService@5020dde
Test: MyIntentService onCreate 
Test: MyIntentService onStartCommand 
Test: MyIntentService onStart 
Test: MyIntentService onHandleIntent Thread[IntentService[MyIntentService],5,main]
Test: MyIntentService onHandleIntent action ACTION_ONE start
Test: MyIntentService onHandleIntent action ACTION_ONE end
Test: MyIntentService onDestroy 
```

### 第一个任务执行结束后启动第二个任务

```
Test: MyIntentService Constructor com.example.heqiang.testsomething.MainActivity$MyIntentService@90cfdd5
Test: MyIntentService onCreate 
Test: MyIntentService onStartCommand 
Test: MyIntentService onStart 
Test: MyIntentService onHandleIntent Thread[IntentService[MyIntentService],5,main]
Test: MyIntentService onHandleIntent action ACTION_ONE start
Test: MyIntentService onHandleIntent action ACTION_ONE end
Test: MyIntentService onDestroy 

Test: MyIntentService Constructor com.example.heqiang.testsomething.MainActivity$MyIntentService@89d0b78
Test: MyIntentService onCreate 
Test: MyIntentService onStartCommand 
Test: MyIntentService onStart 
Test: MyIntentService onHandleIntent Thread[IntentService[MyIntentService],5,main]
Test: MyIntentService onHandleIntent action ACTION_TWO start
Test: MyIntentService onHandleIntent action ACTION_TWO end
Test: MyIntentService onDestroy 
```

和上面的分析一致，第一个任务执行完后启动第二个任务，会重新创建了一个 `IntentService` 对象。

### 第一个任务正在执行时启动第二个任务

```
Test: MyIntentService Constructor com.example.heqiang.testsomething.MainActivity$MyIntentService@c1163b7
Test: MyIntentService onCreate 
Test: MyIntentService onStartCommand 
Test: MyIntentService onStart 
Test: MyIntentService onHandleIntent Thread[IntentService[MyIntentService],5,main]
Test: MyIntentService onHandleIntent action ACTION_ONE start

Test: MyIntentService onStartCommand 
Test: MyIntentService onStart 

Test: MyIntentService onHandleIntent action ACTION_ONE end

Test: MyIntentService onHandleIntent Thread[IntentService[MyIntentService],5,main]
Test: MyIntentService onHandleIntent action ACTION_TWO start
Test: MyIntentService onHandleIntent action ACTION_TWO end

Test: MyIntentService onDestroy 
```

和上面的分析一致，启动第二个任务时，调用了 `IntentService` 的 `IntentService`，两个任务执行完毕后，`IntentService` 退出。

### 第一个任务正在执行的时候停止服务

```
Test: MyIntentService Constructor com.example.heqiang.testsomething.MainActivity$MyIntentService@90cfdd5
Test: MyIntentService onCreate 
Test: MyIntentService onStartCommand 
Test: MyIntentService onStart 
Test: MyIntentService onHandleIntent Thread[IntentService[MyIntentService],5,main]
Test: MyIntentService onHandleIntent action ACTION_ONE start

Test: MyIntentService onDestroy 

Test: MyIntentService onHandleIntent action ACTION_ONE end
```

任务执行过程中停止服务，正在执行的任务还是可以继续执行完的。

### 第一个任务正在执行的时候停止服务，再启动第二个任务

```
Test: MyIntentService Constructor com.example.heqiang.testsomething.MainActivity$MyIntentService@b2ca251
Test: MyIntentService onCreate 
Test: MyIntentService onStartCommand 
Test: MyIntentService onStart 
Test: MyIntentService onHandleIntent Thread[IntentService[MyIntentService],5,main]
Test: MyIntentService onHandleIntent action ACTION_ONE start

Test: MyIntentService onDestroy 

Test: MyIntentService Constructor com.example.heqiang.testsomething.MainActivity$MyIntentService@4c8128d
Test: MyIntentService onCreate 
Test: MyIntentService onStartCommand 
Test: MyIntentService onStart 
Test: MyIntentService onHandleIntent Thread[IntentService[MyIntentService],5,main]
Test: MyIntentService onHandleIntent action ACTION_TWO start
Test: MyIntentService onHandleIntent action ACTION_TWO end
Test: MyIntentService onDestroy 

Test: MyIntentService onHandleIntent action ACTION_ONE end
```

启动第二个任务时，会立即创建一个新的服务，两个任务在两个线程中运行，所以其执行完的先后顺序不确定。