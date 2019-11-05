---
title: Android  应该了解的知识系列 - 关于广播
categories: Android
comments: true
tags: [Android]
description: 介绍广播的一些基础知识
date: 2015-1-10 10:00:00
---

## 广播的类型

广播从发送方式上分可以分为下面三类：

 1. 普通广播
 2. 有序广播
 3. 粘性广播

从广播接收器的注册方式上分可以分为两类：

 1. 静态广播：通过 AndroidManifest.xml 的标签来申明的 BroadcastReceiver。
 2. 动态广播：通过 registerReceiver() 方式注册的 BroadcastReceiver，动态注册更为灵活，可在不需要时通过 unregisterReceiver() 取消注册。

从发送广播的 Intent 类型上面可以分为：

 - 显式广播：通过显式 Intent 发送的广播。
 - 隐式广播：通过隐式 Intent 发送的广播。

从发送广播的广播队列的不同可以分为：

 - 前台广播
 - 后台广播

### 普通广播

这类广播我们经常使用，是一种无序的广播机制，理论上，所有的接受者几乎同时会获得该 intent 的消息，接受者之间不存在先后顺序，不能截断和修改 intent 的数据。
发送方式：`sendBroadcast(Intent intent)`、`sendBroadcast(Intent intent, String receiverPermission)`、`sendBroadcastMultiplePermissions(Intent intent,String[] receiverPermissions)` 等。

### 有序广播

所有接受者都可以设置 priority ， 按照 priority 的先后顺序进行传递， 上一个优先级的接受者，可以截断（abortBroadcast()）和修改 intent 里面的数据。 同时，也可以设置一个最后接收者（总是在最后一个接收到这个 intent，用来做一些特定的功能）。
发送方式：`sendOrderedBroadcast(intent)`
对于接收同一个广播，在相同优先级的情况下，动态注册优先级别高于静态注册。在动态注册中，最早动态注册优先级别最高；在静态注册中，最早安装的程序，静态注册优先级别最高（安装 APK 会解析 manifest.xml,把其加入队列）。

### 粘性广播

粘性广播没有周期限制， 一般的 intent 只能发送给当前已经注册了这个监听的 receiver，一旦发送完毕就会失去作用周期，而粘性广播没有这个限制，即便后来注册的 intent 也可以收到这个广播。 需要注意的一点是 这种发送方式不会导致 ANR， 因为它没有发送时间的限制。
发送方式：`sendStickyBroadcast(intent)`

### 前台广播和后台广播

Android系统关于广播有两个广播队列——前台广播队列mFgBroadcastQueue和后台广播队列mBgBroadcastQueue。在发送方发送广播时，默认是放置在后台广播队列的。但是发送方在给intent设置FLAG_RECEIVER_FOREGROUND标记后，发送的广播就会被放置在前台广播队列。

```
Intent intent = new Intent(); 
intent.setAction("xxxxxx"); 
intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND);//前台广播（默认是后台广播） 
sendBroadcast(intent);
```

前台广播和后台广播的区别在于它们的超时时间不同：

 - 前台广播超时时间为10秒
 - 后台广播超时时间为60秒

一旦出现超时，就会出现我们熟知的 ANR。

## 应用内广播

App应用内广播可以理解成一种局部广播的形式，广播的发送者和接收者都同属于一个App。实际的业务需求中，App应用内广播确实可能需要用到。同时，之所以使用应用内广播时，而不是使用全局广播的形式，更多的考虑到的是Android广播机制中的安全性问题以及效率问题。
确切地说，应该叫进程内广播，它的发送和接收只能在同一个进程内进行。如果一个应用是多进程的应用，那么进程间也是无法使用的。
相比于全局广播，App应用内广播优势体现在：

 1. 安全性更高；
 2. 更加高效。

```
//注册应用内广播接收器
localBroadcastManager = LocalBroadcastManager.getInstance(this);
localBroadcastManager.registerReceiver(mBroadcastReceiver, intentFilter);
        
//取消注册应用内广播接收器
localBroadcastManager.unregisterReceiver(mBroadcastReceiver);

//发送应用内广播
localBroadcastManager.sendBroadcast(intent);
```

##  sendBroadcastAsUser

android 4.2 之后加入了多用户:

 - UserHandle.ALL
 - UserHandle.CURRENT
 - UserHandle.CURRENT_OR_SELF
 - UserHandle.OWNER

这就造就了另外一个发送函数 `sendBroadcastAsUser()` 用来区分不同的用户。

## FLAG_INCLUDE_STOPPED_PACKAGES 和 FLAG_EXCLUDE_STOPPED_PACKAGES

从Android 3.1开始，给Intent定义了两个新的Flag，分别为FLAG_INCLUDE_STOPPED_PACKAGES和FLAG_EXCLUDE_STOPPED_PACKAGES，用来控制Intent是否要对处于停止状态的App起作用，顾名思义：

 - FLAG_INCLUDE_STOPPED_PACKAGES：表示包含未启动的App
 - FLAG_EXCLUDE_STOPPED_PACKAGES：表示不包含未启动的App

```
        Intent intent = new Intent("android.hq.action.TEST");
        intent.addFlags(Intent.FLAG_INCLUDE_STOPPED_PACKAGES);
        intent.putExtra("test", "TEST");
        sendBroadcast(intent);
```

值得注意的是，Android 3.1开始，系统向所有Intent的广播添加了 `FLAG_EXCLUDE_STOPPED_PACKAGES` 标志。这样做是为了防止广播无意或不必要地开启未启动App的后台服务。如果要强制调起未启动的App，后台服务或应用程序可以通过向广播Intent添加 `FLAG_INCLUDE_STOPPED_PACKAGES` 标志来唤醒。
还需要注意的是，在一些手机厂商的ROM中需要设置允许应用后台允许这个设置才能生效。

## 广播安全和限制

在不加限制的情况下，只要BroadcaseReceiver指定的action和sendBroadcast action 一致就可以接收广播。但是在有些场景情况下，我们不允许所有的应用都可以接收广播。那么我们可以采用下的方法。

### 局部广播

上面已经介绍。
优点：

 - 数据在进程内传播，不用担心数据泄漏
 - 相比全局广播，更加高效

缺点：

 - 只能动态注册和取消
 - 无法满足跨进程传播的场景

### 指定某个应用允许接收

可以通过 intent 指定包名 Intent.setPackage 设置广播仅对相同包名的有效。

优点：

 - 支持跨进程
 - 接收器可以静态注册也可以动态注册
 - 指定包名的应用才能接收到数据，安全性高

缺点：

 - 只能满足一个应用接收场景，不能够同时指定多个


```
        Intent intent = new Intent();
        intent.setPackage("包名");
        intent.setAction("Action");
        sendBroadcast(intent);
```

### 指定某个接收器

优点：

 - 可以指定到具体某个接收器，安全性更高
 - 接收器可以动态注册，也可以静态注册

缺点：

 - 只能指定一个接收器，局限性较大


```
        Intent intent = new Intent();
        intent. setComponent(new ComponentName("包名", "Receiver类名"));
        intent.setAction("Action");
        sendBroadcast(intent);
```

### 指定接收权限

通过 `sendBroadcast(Intent intent, String receiverPermission)`、`sendBroadcastMultiplePermissions(Intent intent,String[] receiverPermissions)` 发送的广播，它的接收者必须具备指定的权限才可以接收。

## Android O 上对广播的限制

### 限制内容

[Android 官网对广播限制的解释](https://developer.android.google.cn/guide/components/broadcast-exceptions)
先来了解一下显式广播和隐式广播。
显式广播就是通过显式 Intent 发送的广播。
隐式广播是通过隐式 Intent 发送的广播。
在  Android O 以后，除了有限的例外之外（比如开机广播），应用无法使用清单注册（静态注册）的方式来接收隐式广播。但对于这些隐式广播，可以通过运行时注册（动态注册）的方式注册。对于显式广播，则依然可以通过清单注册（静态注册）的方式监听。
为什么要做这个限制呢？对于一个隐式广播的接收，手机中可能存在很多这样的App都静态注册了接收，如果发出了这样一个广播，不管这些应用是否在运行，都会被唤醒并执行任务。这样会有耗电等问题出现。
因为动态注册的广播只有在动态注册的代码执行了之后才会注册，这样就保证了只有活跃的应用才会收到广播。避免了前面提到的拉起一大片应用的问题。
系统对下面的广播做了例外，允许静态注册的广播接收隐式广播：

```
// Android 8.0 上不限制的隐式广播
/**
开机广播
 Intent.ACTION_LOCKED_BOOT_COMPLETED
 Intent.ACTION_BOOT_COMPLETED
*/
"保留原因：这些广播只在首次启动时发送一次，并且许多应用都需要接收此广播以便进行作业、闹铃等事项的安排。"

/**
增删用户
Intent.ACTION_USER_INITIALIZE
"android.intent.action.USER_ADDED"
"android.intent.action.USER_REMOVED"
*/
"保留原因：这些广播只有拥有特定系统权限的app才能监听，因此大多数正常应用都无法接收它们。"
    
/**
时区、ALARM变化
"android.intent.action.TIME_SET"
Intent.ACTION_TIMEZONE_CHANGED
AlarmManager.ACTION_NEXT_ALARM_CLOCK_CHANGED
*/
"保留原因：时钟应用可能需要接收这些广播，以便在时间或时区变化时更新闹铃"

/**
语言区域变化
Intent.ACTION_LOCALE_CHANGED
*/
"保留原因：只在语言区域发生变化时发送，并不频繁。 应用可能需要在语言区域发生变化时更新其数据。"

/**
Usb相关
UsbManager.ACTION_USB_ACCESSORY_ATTACHED
UsbManager.ACTION_USB_ACCESSORY_DETACHED
UsbManager.ACTION_USB_DEVICE_ATTACHED
UsbManager.ACTION_USB_DEVICE_DETACHED
*/
"保留原因：如果应用需要了解这些 USB 相关事件的信息，目前尚未找到能够替代注册广播的可行方案"

/**
蓝牙状态相关
BluetoothHeadset.ACTION_CONNECTION_STATE_CHANGED
BluetoothA2dp.ACTION_CONNECTION_STATE_CHANGED
BluetoothDevice.ACTION_ACL_CONNECTED
BluetoothDevice.ACTION_ACL_DISCONNECTED
*/
"保留原因：应用接收这些蓝牙事件的广播时不太可能会影响用户体验"

/**
Telephony相关
CarrierConfigManager.ACTION_CARRIER_CONFIG_CHANGED
TelephonyIntents.ACTION_*_SUBSCRIPTION_CHANGED
TelephonyIntents.SECRET_CODE_ACTION
TelephonyManager.ACTION_PHONE_STATE_CHANGED
TelecomManager.ACTION_PHONE_ACCOUNT_REGISTERED
TelecomManager.ACTION_PHONE_ACCOUNT_UNREGISTERED
*/
"保留原因：设备制造商 (OEM) 电话应用可能需要接收这些广播"

/**
账号相关
AccountManager.LOGIN_ACCOUNTS_CHANGED_ACTION
*/
"保留原因：一些应用需要了解登录帐号的变化，以便为新帐号和变化的帐号设置计划操作"

/**
应用数据清除
Intent.ACTION_PACKAGE_DATA_CLEARED
*/
"保留原因：只在用户显式地从 Settings 清除其数据时发送，因此广播接收器不太可能严重影响用户体验"
    
/**
软件包被移除
Intent.ACTION_PACKAGE_FULLY_REMOVED
*/
"保留原因：一些应用可能需要在另一软件包被移除时更新其存储的数据；对于这些应用，尚未找到能够替代注册此广播的可行方案"

/**
外拨电话
Intent.ACTION_NEW_OUTGOING_CALL
*/
"保留原因：执行操作来响应用户打电话行为的应用需要接收此广播"
    
/**
当设备所有者被设置、改变或清除时发出
DevicePolicyManager.ACTION_DEVICE_OWNER_CHANGED
*/
"保留原因：此广播发送得不是很频繁；一些应用需要接收它，以便知晓设备的安全状态发生了变化"
    
/**
日历相关
CalendarContract.ACTION_EVENT_REMINDER
*/
"保留原因：由日历provider发送，用于向日历应用发布事件提醒。因为日历provider不清楚日历应用是什么，所以此广播必须是隐式广播。"
    
/**
安装或移除存储相关广播
Intent.ACTION_MEDIA_MOUNTED
Intent.ACTION_MEDIA_CHECKING
Intent.ACTION_MEDIA_EJECT
Intent.ACTION_MEDIA_UNMOUNTED
Intent.ACTION_MEDIA_UNMOUNTABLE
Intent.ACTION_MEDIA_REMOVED
Intent.ACTION_MEDIA_BAD_REMOVAL
*/
"保留原因：这些广播是作为用户与设备进行物理交互的结果：安装或移除存储卷或当启动初始化时（当可用卷被装载）的一部分发送的，因此它们不是很常见，并且通常是在用户的掌控下"

/**
短信、WAP PUSH相关
Telephony.Sms.Intents.SMS_RECEIVED_ACTION
Telephony.Sms.Intents.WAP_PUSH_RECEIVED_ACTION

注意：需要申请以下权限才可以接收
"android.permission.RECEIVE_SMS"
"android.permission.RECEIVE_WAP_PUSH"
*/
"保留原因：SMS短信应用需要接收这些广播"
```

### 如何应对

对于应对这些受限制的广播来满足我们各种奇葩的需求呢？

 1.将静态注册修改为动态注册
 2.将隐式广播改为显式广播
 3.通过queryBroadcastReceivers检索接收intent的接收者，然后通过显式广播一一发送。
 4.这里的Android O并不是运行的Android版本，而是在AndroidManifest文件中定义的targetSdkVersion的值，因此如果我们不强依赖Android O，也可以把项目文件中的targetSdkVersion设置为25及以下的版本号。
 5.如果必须以一对多的方式发送广播，并且接收者无法动态注册的话，可以给Intent增加一个FLAG_RECEIVER_INCLUDE_BACKGROUND的Flag，不过这个标志位在源码中被hide掉了，直接用他的属性值，这样也是有效的，intent.addFlags(0x01000000)，谨慎使用，如果Android后面版本策略变更，这个也许会失效。

## 一些广播

 - android.intent.action.PACKAGE_ADDED、android.intent.action.PACKAGE_REPLACED、android.intent.action.PACKAGE_CHANGED、android.intent.action.PACKAGE_REMOVED、android.intent.action.PACKAGE_FULLY_REMOVED：当有应用被安装、更新、改变、卸载时系统会发出这些广播。这些广播都是隐式广播，符合Android O的限制条件。只有动态注册的应用启动后才能收到。
 - android.intent.action.MY_PACKAGE_REPLACED：本应用更新时系统会发出这个广播，这个是显式广播，静态注册也可以把自身拉起并收到广播。
 - Intent.ACTION_LOCKED_BOOT_COMPLETED 和 Intent.ACTION_BOOT_COMPLETED：https://www.jianshu.com/p/6198568d56a7

## 注意事项

在《阿里巴巴Android开发手册》中对广播的使用提到：

1.避免在 `BroadcastReceiver#onReceive()` 中执行耗时操作，如果有耗时工作， 应该创建 `IntentService` 完成，而不应该在 `BroadcastReceiver` 内创建子线程去做。
说明:
由于该方法是在主线程执行，如果执行耗时操作会导致 UI不流畅。可以使用 `IntentService` 、 创 建 `HandlerThread` 或 者 调 用 `Context#registerReceiver(BroadcastReceiver, IntentFilter, String, Handler)` 方法等方式，在其他 Wroker 线程执行 `onReceive` 方法。`BroadcastReceiver#onReceive()` 方法耗时超过 10 秒钟，可能会被系统杀死。

2.避免使用隐式 `Intent` 广播敏感信息，信息可能被其他注册了对应 `BroadcastReceiver 的 App` 接收。
说明:
通过 `Context#sendBroadcast()` 发送的隐式广播会被所有感兴趣的 receiver 接收，恶 意应用注册监听该广播的 receiver 可能会获取到 Intent 中传递的敏感信息，并进行其他危险操作。如果发送的广播为使用`Context#sendOrderedBroadcast()` 方法发送的有序广播，优先级较高的恶意 receiver 可能直接丢弃该广播，造成服务不可用，或者向广播结果塞入恶意数据。
如果广播仅限于应用内，则可以使用`LocalBroadcastManager#sendBroadcast()` 实现，避免敏感信息外泄和 Intent 拦截的风险。

<!--  
https://www.cnblogs.com/lwbqqyumidi/p/4168017.html
-->

