---
title: Android AMS 与 APP 进程通信
categories: Android源码分析
comments: true
tags: [Android源码分析, 进程通信]
description: 基于Android6.0来分析 AMS 与 APP 进程通信
date: 2016-6-8 10:00:00
---

## 概述

在 Android 中，APK 运行在 App 进程，而 AMS 运行在 system_server 进程，AMS 承担着对 Activity 的生命周期的管理等工作，而 Activity 生命周期函数的回调又是在 App 进程中进行的，App 进程需要频繁的和 AMS 进程进行通信。那么理解 AMS 进程和 App 进程之间的通信就对 Activity 的启动流程的理解有很好的铺垫作用。因此在介绍 Activity 的启动流程之前我们先来学习一下 AMS 进程和 App 进程之间是如何通信的。
提起到 Android 进程间通信，我们自然就想到了 Binder。是的，AMS 和 APP 之间肯定也是通过 Binder 进行通信的。下面进行详细的介绍。
本文涉及到的类包含：

```
frameworks/base/core/java/android/app/ActivityManagerNative.java
frameworks/base/core/java/android/app/ActivityManagerNative$ActivityManagerProxy.java
frameworks/base/core/java/android/app/ApplicationThreadNative.java
frameworks/base/core/java/android/app/ApplicationThreadNative$ApplicationThreadProxy.java
frameworks/base/core/java/android/app/ActivityThread.java
frameworks/base/core/java/android/app/ActivityThread$ApplicationThread.java
```

## 相关类关系图

![效果图](http://www.plantuml.com/plantuml/svg/jLHDRzD04BtlhnZrn8TIn9LQbb8EA1Af8j9xDRQtpGfxrzhTM4HGn8KJFI3K4wc4UiiLzOA8rFmPB8T_mHhRn8ut7nTkA-VDUs_UpEIKwP32GRS_XFJB5NG70_ZzuUklnMqs_P5-lCk-pzFf_G5HhncFKM84VeXAmLi2S2naGELp4GhfE6gYD8tE59K9bQuBGuel9ALy7OTnV1PBuLEb3AgF5vHh99T021UQ0YeuUKeSptNy7F-ieW4NbelFozkhz6QMtSsp-RVbOfhDFe7pv2yGt5fHoLglINzUPzUpLWtb0UIwX32kgJn7dqAlLpr9qUinuyP_7T7rDKkOdlIH6wdcJt4SCXyr4_nq92a613sb9VgwJAu5E37lX0AiPvD7_1ZYiMVWu0aHKkHWYYHoPUWUF3mYbsG3vq2ADnDeJoNdx40iMO8cx7F6COHUqHz4hsXaeZYgonQ8HB00F8EgwUJoTg3oHpGOX_GbZha_ggBQQdT7y__9HNE8GvCHCnEqXxOcOGOEIgFKTMAx4TGQZO6cvmqUobNOOM77BZGfgqrenxlQWsqRiAlJ_NlSDkhtbfs8pAVYcNDL7frtXuVLHVxxrEdOasmYx8T7oEhW2_RMURS1hICdwqK5qZQC_Q2bkRPKYRPU_FxmglSD2sX9jBmFikd_ovxnIg6qyyY6-WC0)

<!--   
@startuml
Title "AMS 和 APP 通信相关类图"
skinparam class {
  BorderColor<<system_server>> SeaGreen
  BorderColor<<app_process>> Magenta
} 

note as N1
<b><color:SeaGreen > 运行在系统进程 </color >
<b><color:Magenta > 运行在应用进程 </color >
end note

interface IInterface
class Binder
interface IActivityManager
interface IApplicationThread
abstract class ApplicationThreadNative  <<app_process>> {
  + public boolean onTransact();
}
class ApplicationThreadProxy <<system_server>> {
  - private final IBinder mRemote;
  + public final void bindApplication();
  + public final void scheduleLaunchActivity();
}
class ApplicationThread  <<app_process>> {
  + public final void bindApplication();
  + public final void scheduleLaunchActivity();
}
class ActivityManagerService <<system_server>> {
  + public final int startActivity();
  + public final void attachApplication();
}
class ActivityManagerNative <<system_server>> {
  + public boolean onTransact();
}
class ActivityManagerProxy <<app_process>> {
  - private IBinder mRemote;
  + public int startActivity();
  + public void attachApplication();
}
class ActivityThread  <<app_process>> {
  ~ ApplicationThread mAppThread;
}
IBinder <|.. Binder
Binder <|-- ActivityManagerNative
Binder <|-- ApplicationThreadNative

IInterface <|.. IActivityManager
IInterface <|.. IApplicationThread

IActivityManager <|.. ActivityManagerProxy
IActivityManager <|.. ActivityManagerNative
IApplicationThread <|.. ApplicationThreadProxy
IApplicationThread <|.. ApplicationThreadNative

ActivityManagerNative <|-- ActivityManagerService
ApplicationThreadNative <|-- ApplicationThread

ActivityThread *-- ApplicationThread
@enduml
-->


## 进程间通信

下面通过一个图来表示一下 AMS 和 APP 的通信，该图片摘自[Weishu Note](http://weishu.me/2016/01/12/binder-index-for-newer/)，描述的很好，直接拿来用来。

![效果图](/images/android-source-code-analysis-ams-app-ipc/ams-ipc-app.png)

在 App 进程侧，App 进程作为Binder的客户端当发起启动 `Activity` 时，通过作为服务端的代理对象的 `ActivityManagerProxy` 来发起远程调用，此时 system 进程作为服务端，`ActivityManagerNative` 作为 `Binder` 本地对象收到远程调用后，由它的实现类 `ActivityManagerService` 完成相应的生命周期管理以及任务栈管理后，会把控制权交给App进程，让App进程完成Activity类对象的创建，以及生命周期回调。
接下来的调用 system 进程作为客户端，App 进程作为服务端，通过system进程作为服务端的代理对象的 `ApplicationThreadProxy` 发起调用，`ApplicationThread` 作为服务端的 `Binder` 本地对象收到远程调用后，通过 `Handler` 发送消息由 `ActivityThread` (也就是UI线程 )处理。
了解了这些知识后再看[Android startActivity 流程分析](http://www.heqiangfly.com/2016/04/10/android-source-code-analysis-activity-start-process/)的流程图，会有助于理解。

<!--   
http://duanqz.github.io/2016-01-29-Activity-IPC
http://weishu.me/2016/03/21/understand-plugin-framework-activity-management/
http://weishu.me/2016/01/12/binder-index-for-newer/
-->
