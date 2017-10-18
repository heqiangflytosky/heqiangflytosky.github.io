---
title: Android startActivity 流程分析
categories: Android源码分析
comments: true
tags: [Android源码分析, startActivity]
description: 基于Android6.0来分析 Activity 的启动过程
date: 2016-4-10 10:00:00
---

## 概述

Activity的启动方式有两种，一种是显式的，一种是隐式的。
而且，启动的 `Activity` 和原 `Activity` 的进程关系的不同又可以分为两种情况，一种是在同一个进程，另外一种情况是开启一个新的进程来启动 `Activity`。

## 相关类介绍

涉及到的类：

```java
frameworks/base/core/java/android/app/Activity.java
frameworks/base/core/java/android/app/ActivityThread.java
frameworks/base/core/java/android/app/Instrumentation.java
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java
frameworks/base/services/core/java/com/android/server/am/ActivityStack.java
```
 - ActivityThread：代表了应用的主线程，在Activity.attach()方法中初始化
 - TaskRecord：Activity的管理者
 - ActivityStack：Stack的管理者
 - ActivityStackSupervisor：ActivityStack的管理者，早期的Android版本是没有这个类的，直到Android 6.0才出现。

## 启动流程图

![效果图](http://www.plantuml.com/plantuml/svg/pLTDJnin4Btlht13hrL5Zf62590g8WLKiQ5LLLN8U0TYiTWhsqdXrgzJUu1J3-sv4ZTwwQc7_fa6-1draat8ElP-PMbxYPBn-BsPyRDdCg01e0FErJu_yJpzWHghKg5E5A6dWXEGie5MUlHmeDR38NWH5eeI6c6cVOYY8wfEKyOkaqeCZu4fh2XdrWrRcE5341h_v1HXQRNlhNh00dGNLMVBdqMKXRgjUKUGvU63655YT_4L9aV-C8fT91Tkd_HA58MKt2RS7mZ0mMqAHW9D_QjCMGjLgPaTHsgrCKMOhA77A6rukDcOvqaGPaayMfijjQJIkBFpn_4NhE1E4TCloBdf2HSx88UXojbjwAa59q1yExifFUFtV2nffUMbO-ZIRhR0JwJOqeSXC9CQrWcTYDPgjG0d2YuOzrJlTdDH-8xSbI1g836kM9fb2vy-nzHIAFDYEknPH08a3qVWhfV94S1zX97AjyV94GJUlVFiCZA0cAfkmXBff35pJahVspwF4WSAilQuJOwbObz4wp86egPot9ROZu3G0qhAAiV3eK8FY7xB5Mp3wwJxU0XLka4uz2tdZL1k61byBrVY7lXgpT56Mr9BDnY6qCD3hUBPzKM8SKqeuYQgVupFBd___3O-UtVpvltLv-ytlxvUNtoAxUUNuuyNSxkkltou-l5siXWtjnWbbO6zcICq_nmKyN5K0a89gDfvsDxd1C1z4iO3ZL3fFi1cOUu0uI8emMMfn_BnLCoxI2pvFbeoNf7Eu78flx0CQBbkLit5AD0o7iaSv7QOpc1p7kROPzbijNdjTd8DaRw7qx4SjTkfKNTolLl7cBB5jpOPaOsvUd4tLhgAWmhxqxD-WZFt-EJCgatxhDhy5xDgyTWhpwiB5AuUQH1bCTbRqZvbWJqLUsEmcwq49R0NOItQ6s0MXB3k14rBqN9HB2iBWUItnWf_30eVPot_3UoJJNMkYbDDdKegJL58KBtExlR6vbvdezzzrks_gFugJfJf3Auacgxy0HV71U6AwVmltOrwElF_qPgR6MLoB_q1)

<!-- 
------------------------------------------------------------------------------ FirstActivity 进程
├── Activity.startActivity()
     ├── Activity.startActivityForResult()
          ├── Instrumentation.execStartActivity()
               ├── ActivityManagerNative.ActivityManagerProxy.startActivity()
------------------------------------------------------------------------------ FirstActivity 进程
------------------------------------------------------------------------------ AMS 进程
                    ├── ActivityManagerNative.onTransact()
                         ├── ActivityManagerService.startActivity()
                              ├── ActivityManagerService.startActivityAsUser()
                                   ├── ActivityStackSupervisor.startActivityMayWait()
                                        ├── ActivityStackSupervisor.resolveActivity()
                                        ├── ActivityStackSupervisor.startActivityLocked()
                                             ├── ActivityStackSupervisor.startActivityUncheckedLocked()
                                                  ├── ActivityStack.startActivityLocked()
                                                       ├── ActivityStackSupervisor.resumeTopActivitiesLocked()
                                                            ├── ActivityStack.resumeTopActivityLocked()
                                                                 ├── ActivityStack.resumeTopActivityInnerLocked()
                                                                      ├── ActivityStackSupervisor.startSpecificActivityLocked()
                                                                           ├── ActivityStackSupervisor.realStartActivityLocked() 在本进程启动
                                                                               此方法的分析同新进程启动Activity中的realStartActivityLocked()，下面分析
                                                                                ├── ApplicationThreadNative.ApplicationThreadProxy.scheduleLaunchActivity()
------------------------------------------------------------------------------ AMS 进程
------------------------------------------------------------------------------ FirstActivity 进程
├── ActivityThread.H.handleMessage():LAUNCH_ACTIVITY
     ├── ActivityThread.handleLaunchActivity()
          ├── ActivityThread.performLaunchActivity()
------------------------------------------------------------------------------ FirstActivity 进程
------------------------------------------------------------------------------ AMS 进程
                                                                           ├── ActivityManagerService.startProcessLocked() 启动新的进程
                                                                                ├── ActivityManagerService.newProcessRecordLocked()
                                                                                ├── ActivityManagerService.startProcessLocked()
                                                                                     ├── Process.start()
------------------------------------------------------------------------------ AMS 进程
------------------------------------------------------------------------------ SecondActivity 进程
├── ActivityThread.main()
     ├── ActivityThread.attach()
          ├── ActivityManagerNative.ActivityManagerProxy.attachApplication()
------------------------------------------------------------------------------ SecondActivity 进程
------------------------------------------------------------------------------ AMS 进程
├── ActivityManagerNative.onTransact()
     ├── ActivityManagerService.attachApplication()
          ├── ActivityManagerService.attachApplicationLocked()
               ├── ApplicationThreadNative.ApplicationThreadProxy.bindApplication()
------------------------------------------------------------------------------ AMS 进程
------------------------------------------------------------------------------ SecondActivity 进程
                    ├── ApplicationThread.bindApplication()
                         ├── ActivityThread.handleBindApplication()
------------------------------------------------------------------------------ SecondActivity 进程
------------------------------------------------------------------------------ AMS 进程
               ├── ActivityStackSupervisor.attachApplicationLocked()
                    ├── ActivityStackSupervisor.realStartActivityLocked() 此处同上面的同一进程启动Activity
                         ├── ApplicationThreadNative.ApplicationThreadProxy.scheduleLaunchActivity()
------------------------------------------------------------------------------ AMS 进程
------------------------------------------------------------------------------ SecondActivity 进程
├── ApplicationThread.scheduleLaunchActivity()
   ├── ActivityThread.H.handleMessage():LAUNCH_ACTIVITY
        ├── ActivityThread.handleLaunchActivity()
             ├── ActivityThread.performLaunchActivity()
------------------------------------------------------------------------------ SecondActivity 进程

http://www.plantuml.com/plantuml/png/jLVTYjj65BxNKqnDBsn8qhIzMGmRyCffruuDnZRgPHbBi_OGQKQCHjdrdKCefOKa5oqfj9IIGjbUDIsqjFI7lapjZT-YvqYMFBBbzR9i3AlrdByvSxvlT8wZXro4LD601598Tw9am8XMCREYN1FfgS_WgRYhuy2t9jnZv4HAFP9dbWKschiyf4AJIZyMcWUiuMh-YEjfXT28z1j5c-FfI77FuUoq5OH-OdBij3RYG7Iq8E-GxElRnsaqfsZPZeOJnQW7bjdNbMLxMBHq3XAnSr0Kz-YOTQc0fqhqlMvHNpWBB3OJZJLJBN4Yq-mspk4qfHi7JEXqQrWLzI2mPH1AaPaqxGq3vahLeLFO9jLtBElmaBnxXzXIH13Q8wCfQG_8uQ5bzHlazZseTvr8yO1Dc_9KM1JJfveX3AaUYbqdOtb4tOThBI80Vuc_iws6glSTLBQ7THBAIACwL2oArZPGRu_jM86_iSBDJ6N3ijf3Y9w67vM6JWm0l3fXPyo5eXz9wJCg1gxYeQvMqAk7NeXdjLQfhRq1SUU0tCxYkwl2cIc0YKLzJTxAbGG-gwIm81CgF9zr8Jw6xmu-_FYeOJ_ezUyF1hN4jdZqSebP3aJnlIAA1XCJE1CNICgGtnapRSSA2EUnisgDt2Dt4pFyf03rkca5St6-AH0xa_MwcUbObXfkM685fC0yeCGsr6BCjjtWKCLKCgwa3WwF-CXd2JpIz_3_BN92_OZLjcEj2fQ3aCTAuTxlKA2A8xcNySJTIYQ4HylBSgcKP0Fio6H5pfH8ZKJsIf46cTYNCYHGbnnTm5QmqtoP6vQXqcU1VBGYxMMuR6FJPF1UbrCCKkvjP6uK5zszwxXkml91BC1SYch526TWi0k7tLZihloLDKR5cF1sNuQKsDuLhiAMJqjqAHTflohrtmY0hGDBhgaxiIVSznGJ9KJ51fD9SO6kJNk_8xITJDQqCwT8GoTg21vGH2Wt418S51zFT9qilZfVllnj-Uttyp-Up7-PBMUp-QT_pZ__yUBBl_PlakuYHRx5VU5TCJML41vUdXeQPzeU1_I4-TDB_Gs8tvGqGQ1OAaXK9GHu_Upt-VCNQRsMdpoz-EUd__vzkNZyQfChs9wPHhBWN572ZjRQevNsDGmVRZgOZ6wL4uhgQn9gd-CRHosDYkUcNZ9CLWDZLhf5xjQkS2sRdSjPda27HeiNFviyNtpsnVplsKwUTqfpxrYNlsgCYyTq70vXlXbHrH3UHrnuyXGdhh6IyPuxSYEytTE_RxPVbx-Ir_N7Zh6SVNtntGyuvow--NKDnlDNhyyVBsRdr-xzDz7kzy2wezyROcCmsvOo64foDSoCxnTdnkpN2GwCHVH0Wo_NesmkBkjJ-7EcPEiTujMXafQOF8m9_dy0

@startuml
hide footbox

box "1st App Process" #LightBlue
participant Activity
participant Instrumentation
participant ActivityManagerProxy as ActivityManagerProxy_1
end box

box "AMS Process"
participant ActivityManagerNative
participant ActivityManagerService
participant ActivityStackSupervisor
participant ActivityStack
participant ApplicationThreadProxy
end box

box "2nd App Process" #LightBlue
participant ActivityManagerProxy as ActivityManagerProxy_2
participant ApplicationThread
participant "ActivityThread / ActivityThread$H" as ActivityThread
end box

-> Activity:startActivity
activate Activity
Activity -> Activity:startActivityForResult
activate Activity
Activity -> Instrumentation:execStartActivity
activate Instrumentation
Instrumentation -> ActivityManagerProxy_1:startActivity
activate ActivityManagerProxy_1
ActivityManagerProxy_1 -> ActivityManagerNative:onTransact
activate ActivityManagerNative
ActivityManagerNative -> ActivityManagerService:startActivity
activate ActivityManagerService
ActivityManagerService -> ActivityManagerService:startActivityAsUser
activate ActivityManagerService
ActivityManagerService -> ActivityStackSupervisor:startActivityMayWait
activate ActivityStackSupervisor
ActivityStackSupervisor -> ActivityStackSupervisor:resolveActivity
activate ActivityStackSupervisor
deactivate ActivityStackSupervisor
ActivityStackSupervisor -> ActivityStackSupervisor:startActivityLocked
activate ActivityStackSupervisor
ActivityStackSupervisor -> ActivityStackSupervisor:startActivityUncheckedLocked
activate ActivityStackSupervisor
ActivityStackSupervisor -> ActivityStack:startActivityLocked
activate ActivityStack
ActivityStack -> ActivityStackSupervisor:resumeTopActivitiesLocked
activate ActivityStackSupervisor
ActivityStackSupervisor -> ActivityStack:resumeTopActivityLocked
activate ActivityStack
ActivityStack -> ActivityStack:resumeTopActivityInnerLocked
activate ActivityStack
ActivityStack -> ActivityStackSupervisor:startSpecificActivityLocked
activate ActivityStackSupervisor

alt !createNewProcess
  ActivityStackSupervisor -> ActivityStackSupervisor:realStartActivityLocked
  activate ActivityStackSupervisor
  ActivityStackSupervisor -[#Blue]> ApplicationThreadProxy:scheduleLaunchActivity
  note right
  可以参考新进程
  启动Activity
  的流程
  end note
  activate ApplicationThreadProxy
  deactivate ApplicationThreadProxy
  deactivate ActivityStackSupervisor
else createNewProcess
  ActivityStackSupervisor -> ActivityManagerService:startProcessLocked
  activate ActivityManagerService
  ActivityManagerService -> ActivityManagerService:newProcessRecordLocked
  activate ActivityManagerService
  deactivate ActivityManagerService
  ActivityManagerService -> ActivityManagerService:startProcessLocked
  activate ActivityManagerService
  deactivate ActivityManagerService
  deactivate ActivityManagerService
end

deactivate ActivityStackSupervisor
deactivate ActivityStack
deactivate ActivityStack
deactivate ActivityStackSupervisor
deactivate ActivityStack
deactivate ActivityStackSupervisor
deactivate ActivityStackSupervisor
deactivate ActivityStackSupervisor
deactivate ActivityManagerService
deactivate ActivityManagerService
deactivate ActivityManagerNative
deactivate ActivityManagerProxy_1
deactivate Instrumentation
deactivate Activity
deactivate Activity

== create New Process ==

-> ActivityThread:main
activate ActivityThread
ActivityThread -> ActivityThread:attach
activate ActivityThread
ActivityThread -> ActivityManagerProxy_2:attachApplication
activate ActivityManagerProxy_2
ActivityManagerProxy_2 -> ActivityManagerNative:onTransact
activate ActivityManagerNative
ActivityManagerNative -> ActivityManagerService:attachApplication
activate ActivityManagerService
ActivityManagerService -> ActivityManagerService:attachApplicationLocked
activate ActivityManagerService

ActivityManagerService -> ApplicationThreadProxy:bindApplication
activate ApplicationThreadProxy
ApplicationThreadProxy -> ApplicationThread:bindApplication
activate ApplicationThread
ApplicationThread -> ActivityThread:handleBindApplication
activate ActivityThread
deactivate ActivityThread
deactivate ApplicationThread
deactivate ApplicationThreadProxy

ActivityManagerService -> ActivityStackSupervisor:attachApplicationLocked
activate ActivityStackSupervisor
ActivityStackSupervisor -> ActivityStackSupervisor:realStartActivityLocked
activate ActivityStackSupervisor
ActivityStackSupervisor -[#Blue]> ApplicationThreadProxy:scheduleLaunchActivity
activate ApplicationThreadProxy
ApplicationThreadProxy -> ApplicationThread:scheduleLaunchActivity
activate ApplicationThread
ApplicationThread -> ActivityThread:LAUNCH_ACTIVITY
activate ActivityThread
ActivityThread -> ActivityThread:handleLaunchActivity
activate ActivityThread
ActivityThread -> ActivityThread:performLaunchActivity
activate ActivityThread
deactivate ActivityThread
deactivate ActivityThread
deactivate ActivityThread
deactivate ApplicationThread
deactivate ApplicationThreadProxy
deactivate ActivityStackSupervisor
deactivate ActivityStackSupervisor

deactivate ActivityManagerService
deactivate ActivityManagerService
deactivate ActivityManagerNative
deactivate ActivityManagerProxy_2
deactivate ActivityThread

deactivate ActivityThread
@enduml

-->

## 进程间的通信

## 启动流程

### 启动前期，当前进程

```java
    public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        // 一般情况下会走mParent == null分支
        if (mParent == null) {
            //调用Instrumentation.execStartActivity()启动新的Activity。mMainThread类型为ActivityThread, 在attach()函数被回调时被赋值。
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                // 如果activity之前已经启动，而且处于阻塞状态，execStartActivity函数直接返回要启动的activity的result或者null。
                //如果结果不为空，发送结果
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // 如果需要一个返回结果，那么赋值mStartedActivity为true，在结果返回来之前保持当前Activity不可见
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

这里注意到 `mParent` 变量，一般情况下这里 `mParent` 是为空的，除非是启动 `ActivityGroup` 等可以嵌套 `Activity` 的内部的 `Activity` 时才会不为空。

Instrumentation.execStartActivity

```java
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        // 判断mActivityMonitors是否为空，mActivityMonitors可以通过addMonitor()方法创建，一般在自动化测试时使用
        // 一般情况下不会走这个分支
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intent)) {
                        am.mHits++;
                        // 如果处于阻塞状态，直接返回
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess();
            // 这里的ActivityManagerNative.getDefault返回ActivityManagerService的远程接口，即ActivityManagerProxy接口
            // 由它通过进程间通信调用 AMS 的 startActivity 方法
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

`mActivityMonitors` 是 `ActivityMonitor` 的一个集合，该类用来监控应用中的单个活动，可监控一些指定的意图。一般在一些测试Case时使用。创建 `ActivityMonitor` 实例后，通过调用 `ActivityInstrumentationTestCase2` 的 `getInstrumentation()` 方法获取 `Instrumentation` 实例，然后通过 `addMonitor` 方法添加这个实例，当目标活动启动后，系统会匹配 `Instrumentation` 中的 `ActivityMonitor` 实例列表，如果匹配，就会累加计数器。


### ActivityManagerService的处理

#### ActivityManagerService.startActivityAsUser

```java
    @Override
    public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options, int userId) {
        enforceNotIsolatedCaller("startActivity");
        userId = handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(), userId,
                false, ALLOW_FULL_ONLY, "startActivity", null);
        return mStackSupervisor.startActivityMayWait(caller, -1, callingPackage, intent,
                resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
                profilerInfo, null, null, options, false, userId, null, null);
    }
```
从这里开始就在AMS进程中执行了，下面简单介绍一下这些参数代表的意义：

 - resultTo： 指调用者Activity的mToken，mToken对象保存它所处的ActivityRecord信息

#### ActivityStackSupervisor.startActivityMayWait()

```java
    final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult, Configuration config,
            Bundle options, boolean ignoreTargetSecurity, int userId,
            IActivityContainer iContainer, TaskRecord inTask) {
        // 不允许传递文件描述符
        if (intent != null && intent.hasFileDescriptors()) {
            throw new IllegalArgumentException("File descriptors passed in Intent");
        }
        boolean componentSpecified = intent.getComponent() != null;
        // 重新实例化一个Intent对象，防止对intent的修改
        intent = new Intent(intent);

        // 通过Intent解析Activity信息，最终会调用PackageManagerService.resolveIntent()来查询和选择相关的Activity
        // 如果有多个Activity匹配，那么就会弹框让用户来选择，这部分分析此处省略
        ActivityInfo aInfo =
                resolveActivity(intent, resolvedType, startFlags, profilerInfo, userId);

        ActivityContainer container = (ActivityContainer)iContainer;
        synchronized (mService) {
            // 一般情况下container 为null
            if (container != null && container.mParentActivity != null &&
                    container.mParentActivity.state != RESUMED) {
                return ActivityManager.START_CANCELED;
            }
            final int realCallingPid = Binder.getCallingPid();
            final int realCallingUid = Binder.getCallingUid();
            int callingPid;
            if (callingUid >= 0) {
                callingPid = -1;
            } else if (caller == null) {
                callingPid = realCallingPid;
                callingUid = realCallingUid;
            } else {
                callingPid = callingUid = -1;
            }

            final ActivityStack stack;
            if (container == null || container.mStack.isOnHomeDisplay()) {
                // 一般会走到这里，将当前的ActivityStack赋值给stack
                stack = mFocusedStack;
            } else {
                stack = container.mStack;
            }
            stack.mConfigWillChange = config != null && mService.mConfiguration.diff(config) != 0;

            final long origId = Binder.clearCallingIdentity();

            if (aInfo != null &&
                    (aInfo.applicationInfo.privateFlags
                            &ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
                // 处理heavy-weight process，即重量级进程，这种进程在内存不足时不会被kill掉
                // 一般不会走这里
                if (aInfo.processName.equals(aInfo.applicationInfo.packageName)) {
                    if (mService.mHeavyWeightProcess != null &&
                            (mService.mHeavyWeightProcess.info.uid != aInfo.applicationInfo.uid ||
                            !mService.mHeavyWeightProcess.processName.equals(aInfo.processName))) {
                        int appCallingUid = callingUid;
			…………//此处代码省略
                    }
                }
            }

            int res = startActivityLocked(caller, intent, resolvedType, aInfo,
                    voiceSession, voiceInteractor, resultTo, resultWho,
                    requestCode, callingPid, callingUid, callingPackage,
                    realCallingPid, realCallingUid, startFlags, options, ignoreTargetSecurity,
                    componentSpecified, null, container, inTask);

            Binder.restoreCallingIdentity(origId);

            if (stack.mConfigWillChange) {
                // 一般不会走这里
                …………
            }

            if (outResult != null) {
                // 一般不会走这里
                ……
            }

            return res;
        }
    }
```

#### ActivityStackSupervisor.startActivityLocked()

```java
    final int startActivityLocked(IApplicationThread caller,
            Intent intent, String resolvedType, ActivityInfo aInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode,
            int callingPid, int callingUid, String callingPackage,
            int realCallingPid, int realCallingUid, int startFlags, Bundle options,
            boolean ignoreTargetSecurity, boolean componentSpecified, ActivityRecord[] outActivity,
            ActivityContainer container, TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;

        ProcessRecord callerApp = null;
        if (caller != null) {
            // 获取调用者所在的ProcessRecord
            callerApp = mService.getRecordForAppLocked(caller);
            if (callerApp != null) {
                callingPid = callerApp.pid;
                callingUid = callerApp.info.uid;
            } else {
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }
        // 获取userId，如果是管理员用户该值为0
        final int userId = aInfo != null ? UserHandle.getUserId(aInfo.applicationInfo.uid) : 0;

        if (err == ActivityManager.START_SUCCESS) {
            // 打印一些信息
        }

        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        // resultTo为调用者Activity的mToken，保存了它所处的ActivityRecord信息
        if (resultTo != null) {
            // 获取调用者Activity所对用的ActivityRecord
            sourceRecord = isInAnyStackLocked(resultTo);
            if (sourceRecord != null) {
                // 如果需要返回结果，即requestCode不为-1，赋值resultRecord
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }

        final int launchFlags = intent.getFlags();

        if ((launchFlags & Intent.FLAG_ACTIVITY_FORWARD_RESULT) != 0 && sourceRecord != null) {
            // 对Intent.FLAG_ACTIVITY_FORWARD_RESULT处理，如果设置了这个flag，执行结果相当于透传
            // 比如Activity启动顺序A->B->C，那么C的结果将直接返回给A
            …………//此处代码忽略
        }

        if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
            err = ActivityManager.START_INTENT_NOT_RESOLVED;
        }

        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
            // 如果待启动的Activity没有在Manifestfest中显示声明，就会有这个error
            err = ActivityManager.START_CLASS_NOT_FOUND;
        }

        if (err == ActivityManager.START_SUCCESS
                && !isCurrentProfileLocked(userId)
                && (aInfo.flags & FLAG_SHOW_FOR_ALL_USERS) == 0) {
            err = ActivityManager.START_NOT_CURRENT_USER_ACTIVITY;
        }

        if (err == ActivityManager.START_SUCCESS && sourceRecord != null
                && sourceRecord.task.voiceSession != null) {
            ……
        }

        if (err == ActivityManager.START_SUCCESS && voiceSession != null) {
            ……
        }

        final ActivityStack resultStack = resultRecord == null ? null : resultRecord.task.stack;

        if (err != ActivityManager.START_SUCCESS) {
            if (resultRecord != null) {
                resultStack.sendActivityResultLocked(-1,
                    resultRecord, resultWho, requestCode,
                    Activity.RESULT_CANCELED, null);
            }
            ActivityOptions.abort(options);
            return err;
        }

        boolean abort = false;

        final int startAnyPerm = mService.checkPermission(
                START_ANY_ACTIVITY, callingPid, callingUid);

        执行一系列的权限检查
        …………

        abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
                callingPid, resolvedType, aInfo.applicationInfo);

        if (mService.mController != null) {
            try {
                // The Intent we give to the watcher has the extra data
                // stripped off, since it can contain private information.
                Intent watchIntent = intent.cloneFilter();
                abort |= !mService.mController.activityStarting(watchIntent,
                        aInfo.applicationInfo.packageName);
            } catch (RemoteException e) {
                mService.mController = null;
            }
        }

        if (abort) {
            // 权限检查不通过就退出
            if (resultRecord != null) {
                resultStack.sendActivityResultLocked(-1, resultRecord, resultWho, requestCode,
                        Activity.RESULT_CANCELED, null);
            }
            ActivityOptions.abort(options);
            return ActivityManager.START_SUCCESS;
        }

        // 新建一个ActivityRecord，对应将要启动的Activity
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingUid, callingPackage,
                intent, resolvedType, aInfo, mService.mConfiguration, resultRecord, resultWho,
                requestCode, componentSpecified, voiceSession != null, this, container, options);
        if (outActivity != null) {
            outActivity[0] = r;
        }

        if (r.appTimeTracker == null && sourceRecord != null) {
            r.appTimeTracker = sourceRecord.appTimeTracker;
        }

        // 当前的ActivityStack
        final ActivityStack stack = mFocusedStack;
        if (voiceSession == null && (stack.mResumedActivity == null
                || stack.mResumedActivity.info.applicationInfo.uid != callingUid)) {
            if (!mService.checkAppSwitchAllowedLocked(callingPid, callingUid,
                    realCallingPid, realCallingUid, "Activity start")) {
                PendingActivityLaunch pal =
                        new PendingActivityLaunch(r, sourceRecord, startFlags, stack);
                mPendingActivityLaunches.add(pal);
                ActivityOptions.abort(options);
                return ActivityManager.START_SWITCHES_CANCELED;
            }
        }

        if (mService.mDidAppSwitch) {
            mService.mAppSwitchesAllowedTime = 0;
        } else {
            mService.mDidAppSwitch = true;
        }

        doPendingActivityLaunchesLocked(false);

        err = startActivityUncheckedLocked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, true, options, inTask);

        if (err < 0) {
            notifyActivityDrawnForKeyguard();
        }
        return err;
    }
```

#### ActivityStackSupervisor.startActivityUncheckedLocked()

参数中 r 代表即将要启动的 `Activity` 对应的 `ActivityRecord`，`sourceRecord` 代表调用者。

```java
    final int startActivityUncheckedLocked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor, int startFlags,
            boolean doResume, Bundle options, TaskRecord inTask) {
        final Intent intent = r.intent;
        final int callingUid = r.launchedFromUid;

        // 如果指定的inTask不在Recents列表中，认为inTask是无效的
        // inTask不为空，表示调用者希望在指定的task里面启动activity
        // 一般情况下为null
        if (inTask != null && !inTask.inRecents) {
            inTask = null;
        }

        // 被启动Activity的启动模式
        final boolean launchSingleTop = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP;
        final boolean launchSingleInstance = r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE;
        final boolean launchSingleTask = r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK;

        int launchFlags = intent.getFlags();
        if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0 &&
                (launchSingleInstance || launchSingleTask)) {
            //当Inent中定义的launch flag和manifest中定义的有冲突的话优先使用manifest中定义的
            launchFlags &=
                    ~(Intent.FLAG_ACTIVITY_NEW_DOCUMENT | Intent.FLAG_ACTIVITY_MULTIPLE_TASK);
        } else {
            switch (r.info.documentLaunchMode) {
                case ActivityInfo.DOCUMENT_LAUNCH_NONE:
                    break;
                case ActivityInfo.DOCUMENT_LAUNCH_INTO_EXISTING:
                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_DOCUMENT;
                    break;
                case ActivityInfo.DOCUMENT_LAUNCH_ALWAYS:
                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_DOCUMENT;
                    break;
                case ActivityInfo.DOCUMENT_LAUNCH_NEVER:
                    launchFlags &= ~Intent.FLAG_ACTIVITY_MULTIPLE_TASK;
                    break;
            }
        }
        // 是否为后台启动任务的标志位
        final boolean launchTaskBehind = r.mLaunchTaskBehind
                && !launchSingleTask && !launchSingleInstance
                && (launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0;

        if (r.resultTo != null && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0
                && r.resultTo.task.stack != null) {
            // 如果需要返回结果，但是此时的launchFlags有FLAG_ACTIVITY_NEW_TASK，则取消返回结果
            r.resultTo.task.stack.sendActivityResultLocked(-1,
                    r.resultTo, r.resultWho, r.requestCode,
                    Activity.RESULT_CANCELED, null);
            r.resultTo = null;
        }

        if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_DOCUMENT) != 0 && r.resultTo == null) {
            // 赋予FLAG_ACTIVITY_NEW_TASK标志
            launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
        }

        if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            // 如果设置了android:documentLaunchMode为always，那么设置FLAG_ACTIVITY_MULTIPLE_TASK标志
            // 即使在其他task里面有该Activity以及启动，仍然新建Task再次启动一个Activity
            if (launchTaskBehind
                    || r.info.documentLaunchMode == ActivityInfo.DOCUMENT_LAUNCH_ALWAYS) {
                launchFlags |= Intent.FLAG_ACTIVITY_MULTIPLE_TASK;
            }
        }

        // 判断一下是否是用户的行为，比如用户按下Home键等等
        // 如果是用户的行为的话Activity跳转的时候是要调用onUserLeaveHint()方法的
        mUserLeaving = (launchFlags & Intent.FLAG_ACTIVITY_NO_USER_ACTION) == 0;

        // 如果设置了不需要resume，那么设置延迟resume标志
        if (!doResume) {
            r.delayedResume = true;
        }

        // FLAG_ACTIVITY_PREVIOUS_IS_TOP标志来决定待启动Activity会不会被作为栈顶，
        // 如果调用者希望待启动的Activity马上销毁，就会使用该启动参数。
        // 通常情况下notTop为null
        ActivityRecord notTop =
                (launchFlags & Intent.FLAG_ACTIVITY_PREVIOUS_IS_TOP) != 0 ? r : null;

        if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
            ActivityRecord checkedCaller = sourceRecord;
            if (checkedCaller == null) {
                checkedCaller = mFocusedStack.topRunningNonDelayedActivityLocked(notTop);
            }
            if (!checkedCaller.realActivity.equals(r.realActivity)) {
                // 如果调用者和被调用者不是同一个Activity，去掉这个标志，因为这个时候都需要启动
                startFlags &= ~ActivityManager.START_FLAG_ONLY_IF_NEEDED;
            }
        }

        boolean addingToTask = false;//表示是否需要在inTask中启动activity
        TaskRecord reuseTask = null;//表示可以复用的task
        //当调用者不是来自activity，而是明确指定task的情况。
        if (sourceRecord == null && inTask != null && inTask.stack != null) {
            final Intent baseIntent = inTask.getBaseIntent();
            final ActivityRecord root = inTask.getRootActivity();
            
            ……

            if (root == null) {
                // 如果inTask是个空的task，从baseIntent里面挑选出一些flag设置到launchFlags
                final int flagsOfInterest = Intent.FLAG_ACTIVITY_NEW_TASK
                        | Intent.FLAG_ACTIVITY_MULTIPLE_TASK | Intent.FLAG_ACTIVITY_NEW_DOCUMENT
                        | Intent.FLAG_ACTIVITY_RETAIN_IN_RECENTS;
                launchFlags = (launchFlags&~flagsOfInterest)
                        | (baseIntent.getFlags()&flagsOfInterest);
                intent.setFlags(launchFlags);
                inTask.setIntent(r);
                addingToTask = true;
            } else if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
                // 如果inTask非空，launchFlags要求在新task里面启动那么就不加入到这个task里面去
                addingToTask = false;
            } else {
                // 其他情况就加入到inTask
                addingToTask = true;
            }
            // 如果inTask不为空，就作为复用的task
            reuseTask = inTask;
        } else {
            inTask = null;
        }

        if (inTask == null) {
            // 当没有指定task
            if (sourceRecord == null) {
                // 调用者不是一个activity，那么就在一个新的task里面启动activity
                if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) == 0 && inTask == null) {
                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
                }
            } else if (sourceRecord.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
                // 如果调用者是一个activity，而且调用者的启动模式是LAUNCH_SINGLE_INSTANCE，
                // 那么也需要在新的task里面启动
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
            } else if (launchSingleInstance || launchSingleTask) {
                // 启动标志位里面是SingleInstance或者launchSingleTask，也需要在新task里面启动
                launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
            }
        }

        ActivityInfo newTaskInfo = null;
        Intent newTaskIntent = null;
        ActivityStack sourceStack;
        if (sourceRecord != null) {
            if (sourceRecord.finishing) {
                // 如果调用者activity处于正在销毁的状态，那么就需要在新的task里启动
                // 并且保存一些调用者的信息，并把sourceRecord和sourceStack置空
                if ((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
                    launchFlags |= Intent.FLAG_ACTIVITY_NEW_TASK;
                    newTaskInfo = sourceRecord.info;
                    newTaskIntent = sourceRecord.task.intent;
                }
                sourceRecord = null;
                sourceStack = null;
            } else {
                // 如果不在销毁状态，把其所在的栈赋于sourceStack
                sourceStack = sourceRecord.task.stack;
            }
        } else {
            sourceStack = null;
        }

        boolean movedHome = false;
        ActivityStack targetStack;
        // 至此，启动参数的修正工作结束，设置给intent
        intent.setFlags(launchFlags);
        // 看是否需要动画
        final boolean noAnimation = (launchFlags & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0;

        // FLAG_ACTIVITY_NEW_TASK && FLAG_ACTIVITY_MULTIPLE_TASK 表示不是在当前任务中启动，
        // 但是要查看是否有目标Activity已经存在的任务，同launchSingleInstance和launchSingleTask
        if (((launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0 &&
                (launchFlags & Intent.FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
                || launchSingleInstance || launchSingleTask) {

            if (inTask == null && r.resultTo == null) {
                // 在没有指定task且不要返回结果的情况下，去查找是否有匹配的activity
                // findTaskLocked是找到匹配的目标ActivityRecord所在的任务栈(TaskRecord)，包含taskAffinity的操作
                // 这个方法后面会详细介绍
                // 从栈顶至栈底遍历ActivityStack中的所有Activity，根据Intent和ActivityInfo这
                // 两个参数得到的Activity的包名
                ActivityRecord intentActivity = !launchSingleInstance ?
                        findTaskLocked(r) : findActivityLocked(intent, r.info);
                if (intentActivity != null) {
                    // intentActivity不为空表示找到目标activity，那么目标Activity所在的任务(TaskRecord)
                    // 和任务所在的栈(ActivityRecord)就是宿主任务和宿主栈
                    // 这里通过isLockTaskModeViolation锁定屏幕的那些Task
                    if (isLockTaskModeViolation(intentActivity.task,
                            (launchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                            == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))) {
                        // 如果不是锁定的task，就结束启动流程
                        // lock task mode 是Android 5.0 Lollipop添加的功能
                        showLockTaskToast();
                        return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
                    }
                    if (r.task == null) {
                        r.task = intentActivity.task;
                    }
                    if (intentActivity.task.intent == null) {
                        intentActivity.task.setIntent(r);
                    }
                    targetStack = intentActivity.task.stack;
                    targetStack.mLastPausedActivity = null;

                    // 如果目标task不在前台就需要把它移至前台
                    // 这里首先获取焦点所在的栈
                    final ActivityStack focusStack = getFocusedStack();
                    //从焦点栈里面找到当前可见的ActivityRecord
                    ActivityRecord curTop = (focusStack == null)
                            ? null : focusStack.topRunningNonDelayedActivityLocked(notTop);
                    boolean movedToFront = false;
                    if (curTop != null && (curTop.task != intentActivity.task ||
                            curTop.task != focusStack.topTask())) {
                        // 满足上面的条件表示待启动的activity和当前可见activity不在同一个task
                        // 那么就需要将待启动的activity所在的栈移动到前台
                        r.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT);
                        if (sourceRecord == null || (sourceStack.topActivity() != null &&
                                sourceStack.topActivity().task == sourceRecord.task)) {
                            if (launchTaskBehind && sourceRecord != null) {
                                intentActivity.setTaskToAffiliateWith(sourceRecord.task);
                            }
                            movedHome = true;
                            // 将目标任务移动到栈顶
                            targetStack.moveTaskToFrontLocked(intentActivity.task, noAnimation,
                                    options, r.appTimeTracker, "bringingFoundTaskToFront");
                            movedToFront = true;
                            if ((launchFlags &
                                    (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME))
                                    == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                                //将toReturnTo设置为home
                                intentActivity.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE);
                            }
                            options = null;
                        }
                    }
                    if (!movedToFront) {
                        // 将当前栈移动到前台
                        targetStack.moveToFront("intentActivityFound");
                    }

                    // 根据需要重置activity所在的task
                    if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                        intentActivity = targetStack.resetTaskIfNeededLocked(intentActivity, r);
                    }
                    if ((startFlags & ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                        // 如果不需要启动一个新的activity，结束启动流程
                        if (doResume) {
                            resumeTopActivitiesLocked(targetStack, null, options);
                            // 如果没有移动到前台，通知Keyguard
                            if (!movedToFront) {
                                notifyActivityDrawnForKeyguard();
                            }
                        } else {
                            ActivityOptions.abort(options);
                        }
                        return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                    }
                    if ((launchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                            == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK)) {
                        // 这里需要移除目标task中所有关联的其他activity，即要完全替代目标task
                        // 将该启动activity变为task中的根activity
                        reuseTask = intentActivity.task;
                        reuseTask.performClearTaskLocked();
                        reuseTask.setIntent(r);
                    } else if ((launchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                            || launchSingleInstance || launchSingleTask) {
                        // 清除task中目标activity上面的所有activity，返回目标activity
                        ActivityRecord top =
                                intentActivity.task.performClearTaskLocked(r, launchFlags);
                        if (top != null) {
                            if (top.frontOfTask) {
                                top.task.setIntent(r);
                            }
                            //触发调用onNewIntent()
                            top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                        } else {
                            // A special case: we need to start the activity because it is not
                            // currently running, and the caller has asked to clear the current
                            // task to have this activity at the top.
                            addingToTask = true;
                            // Now pretend like this activity is being started by the top of its
                            // task, so it is put in the right place.
                            sourceRecord = intentActivity;
                            TaskRecord task = sourceRecord.task;
                            if (task != null && task.stack == null) {
                                // Target stack got cleared when we all activities were removed
                                // above. Go ahead and reset it.
                                targetStack = computeStackFocus(sourceRecord, false /* newTask */);
                                targetStack.addTask(
                                        task, !launchTaskBehind /* toTop */, false /* moving */);
                            }

                        }
                    } else if (r.realActivity.equals(intentActivity.task.realActivity)) {
                        // In this case the top activity on the task is the
                        // same as the one being launched, so we take that
                        // as a request to bring the task to the foreground.
                        // If the top activity in the task is the root
                        // activity, deliver this new intent to it if it
                        // desires.
                        if (((launchFlags&Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0 || launchSingleTop)
                                && intentActivity.realActivity.equals(r.realActivity)) {
                            ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r,
                                    intentActivity.task);
                            if (intentActivity.frontOfTask) {
                                intentActivity.task.setIntent(r);
                            }
                            intentActivity.deliverNewIntentLocked(callingUid, r.intent,
                                    r.launchedFromPackage);
                        } else if (!r.intent.filterEquals(intentActivity.task.intent)) {
                            // In this case we are launching the root activity
                            // of the task, but with a different intent.  We
                            // should start a new instance on top.
                            addingToTask = true;
                            sourceRecord = intentActivity;
                        }
                    } else if ((launchFlags&Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) == 0) {
                        // In this case an activity is being launched in to an
                        // existing task, without resetting that task.  This
                        // is typically the situation of launching an activity
                        // from a notification or shortcut.  We want to place
                        // the new activity on top of the current task.
                        addingToTask = true;
                        sourceRecord = intentActivity;
                    } else if (!intentActivity.task.rootWasReset) {
                        // In this case we are launching in to an existing task
                        // that has not yet been started from its front door.
                        // The current task has been brought to the front.
                        // Ideally, we'd probably like to place this new task
                        // at the bottom of its stack, but that's a little hard
                        // to do with the current organization of the code so
                        // for now we'll just drop it.
                        intentActivity.task.setIntent(r);
                    }
                    if (!addingToTask && reuseTask == null) {
                        // 如果不需要加入指定的task，且不用复用task，那么直接把找到的目标activity推到前台
                        // 并结束启动流程
                        if (doResume) {
                            targetStack.resumeTopActivityLocked(null, options);
                            if (!movedToFront) {
                                notifyActivityDrawnForKeyguard();
                            }
                        } else {
                            ActivityOptions.abort(options);
                        }
                        return ActivityManager.START_TASK_TO_FRONT;
                    }
                }//end of (intentActivity != null)
            }//end of (inTask == null && r.resultTo == null)
        }

        if (r.packageName != null) {
            // 待启动activity包名不为空
            ActivityStack topStack = mFocusedStack;
            ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(notTop);
            if (top != null && r.resultTo == null) {
                if (top.realActivity.equals(r.realActivity) && top.userId == r.userId) {
                    // 待启动的activity和前台显示的activity是同一个activity的情况
                    if (top.app != null && top.app.thread != null) {
                        if ((launchFlags & Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                            || launchSingleTop || launchSingleTask) {
                            ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, top,
                                    top.task);
                            topStack.mLastPausedActivity = null;
                            if (doResume) {
                                resumeTopActivitiesLocked();
                            }
                            ActivityOptions.abort(options);
                            if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                                return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                            }
                            top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                            return ActivityManager.START_DELIVERED_TO_TOP;
                        }
                    }
                }
            }

        } else {
            //如果目标activity包名为空，表示找不到待启动的activity，结束启动流程
            if (r.resultTo != null && r.resultTo.task.stack != null) {
                r.resultTo.task.stack.sendActivityResultLocked(-1, r.resultTo, r.resultWho,
                        r.requestCode, Activity.RESULT_CANCELED, null);
            }
            ActivityOptions.abort(options);
            return ActivityManager.START_CLASS_NOT_FOUND;
        }
```

目标activity不存在的情况：

```java
        // 接下来处理目标activity不存在的情况

        boolean newTask = false;
        boolean keepCurTransition = false;

        TaskRecord taskToAffiliate = launchTaskBehind && sourceRecord != null ?
                sourceRecord.task : null;

        if (r.resultTo == null && inTask == null && !addingToTask
                && (launchFlags & Intent.FLAG_ACTIVITY_NEW_TASK) != 0) {
            // 这种情况下要在一个新的task中启动activity
            newTask = true;
            targetStack = computeStackFocus(r, newTask);
            targetStack.moveToFront("startingNewTask");

            if (reuseTask == null) {
                // 如果没有可以复用的就新建一个
                r.setTask(targetStack.createTaskRecord(getNextTaskId(),
                        newTaskInfo != null ? newTaskInfo : r.info,
                        newTaskIntent != null ? newTaskIntent : intent,
                        voiceSession, voiceInteractor, !launchTaskBehind /* toTop */),
                        taskToAffiliate);
            } else {
                // 复用task
                r.setTask(reuseTask, taskToAffiliate);
            }
            if (isLockTaskModeViolation(r.task)) {
                return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
            if (!movedHome) {
                if ((launchFlags &
                        (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME))
                        == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_TASK_ON_HOME)) {
                    // 把当前新启动的任务置于Home任务之上，也就是按back键从这个任务返回的时候
                    // 会回到home，即使这个不是他们最后看见的activity
                    r.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE);
                }
            }
        } else if (sourceRecord != null) {
            // 不用在新task中启动activity
            final TaskRecord sourceTask = sourceRecord.task;
            if (isLockTaskModeViolation(sourceTask)) {
                return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
            // 那么目标activity的栈也就是调用者所在的栈
            targetStack = sourceTask.stack;
            targetStack.moveToFront("sourceStackToFront");
            final TaskRecord topTask = targetStack.topTask();
            if (topTask != sourceTask) {
                // 移动task到前台
                targetStack.moveTaskToFrontLocked(sourceTask, noAnimation, options,
                        r.appTimeTracker, "sourceTaskToFront");
            }
            if (!addingToTask && (launchFlags&Intent.FLAG_ACTIVITY_CLEAR_TOP) != 0) {
                // 清除activity
                ActivityRecord top = sourceTask.performClearTaskLocked(r, launchFlags);
                keepCurTransition = true;
                if (top != null) {
                    ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r, top.task);
                    top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                    targetStack.mLastPausedActivity = null;
                    if (doResume) {
                        targetStack.resumeTopActivityLocked(null);
                    }
                    ActivityOptions.abort(options);
                    return ActivityManager.START_DELIVERED_TO_TOP;
                }
            } else if (!addingToTask &&
                    (launchFlags&Intent.FLAG_ACTIVITY_REORDER_TO_FRONT) != 0) {
                final ActivityRecord top = sourceTask.findActivityInHistoryLocked(r);
                if (top != null) {
                    final TaskRecord task = top.task;
                    task.moveActivityToFrontLocked(top);
                    ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, r, task);
                    top.updateOptionsLocked(options);
                    top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                    targetStack.mLastPausedActivity = null;
                    if (doResume) {
                        targetStack.resumeTopActivityLocked(null);
                    }
                    return ActivityManager.START_DELIVERED_TO_TOP;
                }
            }
            r.setTask(sourceTask, null);
        } else if (inTask != null) {
            // 在指定的task中启动activity
            if (isLockTaskModeViolation(inTask)) {
                return ActivityManager.START_RETURN_LOCK_TASK_MODE_VIOLATION;
            }
            targetStack = inTask.stack;
            targetStack.moveTaskToFrontLocked(inTask, noAnimation, options, r.appTimeTracker,
                    "inTaskToFront");

            // check一下是在这个task中启动一个新的activity或者是复用当前栈顶的activity
            ActivityRecord top = inTask.getTopActivity();
            if (top != null && top.realActivity.equals(r.realActivity) && top.userId == r.userId) {
                if ((launchFlags & Intent.FLAG_ACTIVITY_SINGLE_TOP) != 0
                        || launchSingleTop || launchSingleTask) {
                    ActivityStack.logStartActivity(EventLogTags.AM_NEW_INTENT, top, top.task);
                    if ((startFlags&ActivityManager.START_FLAG_ONLY_IF_NEEDED) != 0) {
                        return ActivityManager.START_RETURN_INTENT_TO_CALLER;
                    }
                    top.deliverNewIntentLocked(callingUid, r.intent, r.launchedFromPackage);
                    return ActivityManager.START_DELIVERED_TO_TOP;
                }
            }

            if (!addingToTask) {
                ActivityOptions.abort(options);
                return ActivityManager.START_TASK_TO_FRONT;
            }

            r.setTask(inTask, null);
        } else {
            // 既不启动一个新的activity，也不在指定的task中启动
            // 目前不会走到这个分支中
            targetStack = computeStackFocus(r, newTask);
            targetStack.moveToFront("addingToTopTask");
            ActivityRecord prev = targetStack.topActivity();
            r.setTask(prev != null ? prev.task : targetStack.createTaskRecord(getNextTaskId(),
                            r.info, intent, null, null, true), null);
            mWindowManager.moveTaskToTop(r.task.taskId);
        }

        mService.grantUriPermissionFromIntentLocked(callingUid, r.packageName,
                intent, r.getUriPermissionsLocked(), r.userId);

        if (sourceRecord != null && sourceRecord.isRecentsActivity()) {
            r.task.setTaskToReturnTo(RECENTS_ACTIVITY_TYPE);
        }
        if (newTask) {
            EventLog.writeEvent(EventLogTags.AM_CREATE_TASK, r.userId, r.task.taskId);
        }
        ActivityStack.logStartActivity(EventLogTags.AM_CREATE_ACTIVITY, r, r.task);
        targetStack.mLastPausedActivity = null;
        // 进行下一步流程，启动activity
        targetStack.startActivityLocked(r, newTask, doResume, keepCurTransition, options);
        if (!launchTaskBehind) {
            // 如果不是在后台启动就把焦点设置在当前activity
            mService.setFocusedActivityLocked(r, "startedActivity");
        }
        return ActivityManager.START_SUCCESS;
    }
```

可以看到这个方法首先是修正了一些启动参数，接着根据目标Activity是否存在分别进行了处理，然后启动activity。

#### ActivityStack.startActivityLocked()

```java
    final void startActivityLocked(ActivityRecord r, boolean newTask,
            boolean doResume, boolean keepCurTransition, Bundle options) {
        TaskRecord rTask = r.task;
        final int taskId = rTask.taskId;
        if (!r.mLaunchTaskBehind && (taskForIdLocked(taskId) == null || newTask)) {
            // 历史activity被清除了或者是明确要求在新的task启动activity
            // 将task插入stack栈顶
            insertTaskAtTop(rTask, r);
            // 调用WindowManagerService 
            mWindowManager.moveTaskToTop(taskId);
        }
        TaskRecord task = null;
        if (!newTask) {
            // 在一个已经存在的task中启动activity
            boolean startIt = true;// 是否需要启动activity的标志位
            for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
                task = mTaskHistory.get(taskNdx);
                if (task.getTopActivity() == null) {
                    //  该task中已经没有activity了
                    continue;
                }
                if (task == r.task) {
                    // 匹配到目标task
                    if (!startIt) {
                        // 如果不需要启动，就先添加到栈顶
                        task.addActivityToTop(r);
                        r.putInHistory();
                        // 调用WindowManagerService 添加apptoken
                        mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                                r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                                (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0,
                                r.userId, r.info.configChanges, task.voiceSession != null,
                                r.mLaunchTaskBehind);
                        if (VALIDATE_TOKENS) {
                            validateAppTokensLocked();
                        }
                        ActivityOptions.abort(options);
                        return;
                    }
                    break;
                } else if (task.numFullscreen > 0) {
                    // 如果在这个任务中有全屏显示的activity，就先不显示
                    startIt = false;
                }
            }
        }

        // 走到这里已经把目标activity所在的task推到了stack的栈顶

        // 如果不需要在新的activity中启动，而且上面查到了目标task，且task在栈顶，就不需要调用onUserLeaving方法
        if (task == r.task && mTaskHistory.indexOf(task) != (mTaskHistory.size() - 1)) {
            mStackSupervisor.mUserLeaving = false;
        }

        task = r.task;

        // 将activity放到task的栈顶
        task.addActivityToTop(r);
        // activity移动或者销毁之后调用setFrontOfTask方法
        task.setFrontOfTask();

        r.putInHistory();
        if (!isHomeStack() || numActivities() > 0) {
            // 进入这个分支的条件当前stack不是桌面stack，也即当前是否是桌面。或者当前stack中已有task

            // 一些切换动画相关的操作
            boolean showStartingIcon = newTask;
            ProcessRecord proc = r.app;
            if (proc == null) {
                proc = mService.mProcessNames.get(r.processName, r.info.applicationInfo.uid);
            }
            if (proc == null || proc.thread == null) {
                showStartingIcon = true;
            }
            if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_NO_ANIMATION) != 0) {
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, keepCurTransition);
                mNoAnimActivities.add(r);
            } else {
                mWindowManager.prepareAppTransition(newTask
                        ? r.mLaunchTaskBehind
                                ? AppTransition.TRANSIT_TASK_OPEN_BEHIND
                                : AppTransition.TRANSIT_TASK_OPEN
                        : AppTransition.TRANSIT_ACTIVITY_OPEN, keepCurTransition);
                mNoAnimActivities.remove(r);
            }
            // 调用WindowManagerService 添加apptoken
            mWindowManager.addAppToken(task.mActivities.indexOf(r),
                    r.appToken, r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                    (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId,
                    r.info.configChanges, task.voiceSession != null, r.mLaunchTaskBehind);
            boolean doShow = true;
            // 一些显示StartingWindow的操作
            if (newTask) {
                if ((r.intent.getFlags() & Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) != 0) {
                    resetTaskIfNeededLocked(r, r);
                    doShow = topRunningNonDelayedActivityLocked(null) == r;
                }
            } else if (options != null && new ActivityOptions(options).getAnimationType()
                    == ActivityOptions.ANIM_SCENE_TRANSITION) {
                doShow = false;
            }
            if (r.mLaunchTaskBehind) {
                mWindowManager.setAppVisibility(r.appToken, true);
                ensureActivitiesVisibleLocked(null, 0);
            } else if (SHOW_APP_STARTING_PREVIEW && doShow) {
                ActivityRecord prev = mResumedActivity;
                // prev 表示当前在resumed状态的activity
                if (prev != null) {
                    // 如果它和目标activity不在同一个task
                    if (prev.task != r.task) {
                        prev = null;
                    }
                    // 在同一个task且已经显示
                    else if (prev.nowVisible) {
                        prev = null;
                    }
                }
                mWindowManager.setAppStartingWindow(
                        r.appToken, r.packageName, r.theme,
                        mService.compatibilityInfoForPackageLocked(
                                r.info.applicationInfo), r.nonLocalizedLabel,
                        r.labelRes, r.icon, r.logo, r.windowFlags,
                        prev != null ? prev.appToken : null, showStartingIcon);
                r.mStartingWindowShown = true;
            }
        } else {
            // 进入这个分支条件：目标stack中还没有任何activity，那就不要做任何动画
            // 调用addAppToken绑定窗口
            mWindowManager.addAppToken(task.mActivities.indexOf(r), r.appToken,
                    r.task.taskId, mStackId, r.info.screenOrientation, r.fullscreen,
                    (r.info.flags & ActivityInfo.FLAG_SHOW_FOR_ALL_USERS) != 0, r.userId,
                    r.info.configChanges, task.voiceSession != null, r.mLaunchTaskBehind);
            ActivityOptions.abort(options);
            options = null;
        }
        if (VALIDATE_TOKENS) {
            validateAppTokensLocked();
        }
        // 显示activity
        if (doResume) {
            mStackSupervisor.resumeTopActivitiesLocked(this, r, options);
        }
    }
```

#### ActivityStackSupervisor.resumeTopActivitiesLocked()

```java
    boolean resumeTopActivitiesLocked(ActivityStack targetStack, ActivityRecord target,
            Bundle targetOptions) {
        if (targetStack == null) {
            targetStack = mFocusedStack;
        }
        // Do targetStack first.
        boolean result = false;
        if (isFrontStack(targetStack)) {
            result = targetStack.resumeTopActivityLocked(target, targetOptions);
        }
        // 遍历所有显示设备上的ActivityStack
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (stack == targetStack) {
                    // 上面已经处理过
                    continue;
                }
                if (isFrontStack(stack)) {
                    // 由其他显示设备进行处理
                    stack.resumeTopActivityLocked(null);
                }
            }
        }
        return result;
    }
```

#### ActivityStack.resumeTopActivityLocked()

```java
    final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // 防止递归调用
            mStackSupervisor.inResumeTopActivity = true;
            if (mService.mLockScreenShown == ActivityManagerService.LOCK_SCREEN_LEAVING) {
                mService.mLockScreenShown = ActivityManagerService.LOCK_SCREEN_HIDDEN;
                mService.updateSleepIfNeededLocked();
            }
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }
        return result;
    }
```

#### ActivityStack.resumeTopActivityInnerLocked()

这里的参数 `prev` 表示待启动的 `ActivityRecord`。

```java
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, Bundle options) {

        if (!mService.mBooting && !mService.mBooted) {
            // 如果系统还没有启动结束，这里结束启动流程
            return false;
        }

        ActivityRecord parent = mActivityContainer.mParentActivity;
        if ((parent != null && parent.state != ActivityState.RESUMED) ||
                !mActivityContainer.isAttachedLocked()) {
            // 如果存在ParentActivity且ParentActivity还没有resume，那么结束启动流程
            return false;
        }
        // 在当前activity下面的activity如果有处于INITIALIZING而且正在做窗口动画
        // 那么就取消这些窗口动画
        cancelInitializingActivities();

        // 找到当前栈顶正在运行的activity，一般表示将要显示的activity
        final ActivityRecord next = topRunningActivityLocked(null);

        final boolean userLeaving = mStackSupervisor.mUserLeaving;
        mStackSupervisor.mUserLeaving = false;

        final TaskRecord prevTask = prev != null ? prev.task : null;
        if (next == null) {
            // 没有将要显示的activity
            final String reason = "noMoreActivities";
            if (!mFullscreen) {
                final ActivityStack stack = getNextVisibleStackLocked();
                if (adjustFocusToNextVisibleStackLocked(stack, reason)) {
                    return mStackSupervisor.resumeTopActivitiesLocked(stack, prev, null);
                }
            }
            // 显示桌面，结束启动流程
            ActivityOptions.abort(options);
            final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                    HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
            return isOnHomeDisplay() &&
                    mStackSupervisor.resumeHomeStackTask(returnTaskType, prev, reason);
        }

        next.delayedResume = false;

        if (mResumedActivity == next && next.state == ActivityState.RESUMED &&
                    mStackSupervisor.allResumedActivitiesComplete()) {
            // 当前正在显示的activity就是下一个将要显示的activity，那么结束启动流程
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            return false;
        }

        final TaskRecord nextTask = next.task;
        if (prevTask != null && prevTask.stack == this &&
                prevTask.isOverHomeStack() && prev.finishing && prev.frontOfTask) {
            if (prevTask == nextTask) {
                prevTask.setFrontOfTask();
            } else if (prevTask != topTask()) {
                final int taskNdx = mTaskHistory.indexOf(prevTask) + 1;
                mTaskHistory.get(taskNdx).setTaskToReturnTo(HOME_ACTIVITY_TYPE);
            } else if (!isOnHomeDisplay()) {
                return false;
            } else if (!isHomeStack()){
                final int returnTaskType = prevTask == null || !prevTask.isOverHomeStack() ?
                        HOME_ACTIVITY_TYPE : prevTask.getTaskToReturnTo();
                return isOnHomeDisplay() &&
                        mStackSupervisor.resumeHomeStackTask(returnTaskType, prev, "prevFinished");
            }
        }

        // 如果系统在休眠状态且栈顶activity在pause状态，结束启动流程
        if (mService.isSleepingOrShuttingDown()
                && mLastPausedActivity == next
                && mStackSupervisor.allPausedActivitiesComplete()) {
            mWindowManager.executeAppTransition();
            mNoAnimActivities.clear();
            ActivityOptions.abort(options);
            return false;
        }

        // 找不到待启动activity的user，结束启动流程
        if (mService.mStartedUsers.get(next.userId) == null) {
            return false;
        }

        // 从Stopping状态进入休眠状态和待显示状态列表中删除待启动activity
        // 待启动activity有可能处于这些数组中
        mStackSupervisor.mStoppingActivities.remove(next);
        mStackSupervisor.mGoingToSleepActivities.remove(next);
        next.sleeping = false;
        mStackSupervisor.mWaitingVisibleActivities.remove(next);

        // 如果当前正在pause一些activity，那么就要把这些工作做完
        // 这种情况下也会结束启动流程
        if (!mStackSupervisor.allPausedActivitiesComplete()) {
            return false;
        }
```

接下来就开始进行一些activity的切换工作，首先要pause当前activity，将待显示的activity进行resume。

```java
        // 设置WakeLock，保证在启动过程中系统不会休眠
        mStackSupervisor.setLaunchSource(next.info.applicationInfo.uid);

        // 开始把当前activity进入pause状态，这样才能把待显示activity进入resume状态
        // 如果启动参数里面有FLAG_RESUME_WHILE_PAUSING，表示当前的activity正在pause时，可以执行待显示
        // activity的resume动作
        boolean dontWaitForPause = (next.info.flags&ActivityInfo.FLAG_RESUME_WHILE_PAUSING) != 0;
        // 开始pause stack里面的activity
        boolean pausing = mStackSupervisor.pauseBackStacks(userLeaving, true, dontWaitForPause);
        if (mResumedActivity != null) {
            // 把当前正在显示的activity进入pasue状态
            pausing |= startPausingLocked(userLeaving, false, true, dontWaitForPause);
        }
        if (pausing) {
            // pausing为true表示当前有正在pause的activity，那么这里先返回，等完全pause再执行后面的操作
            // 一般第一次会走这里，但是在startPausingLocked中发送的消息执行以后还会再次执行这个函数
            // ActivityStack.activityPausedLocked->ActivityStack.completePauseLocked->
            // ActivityStackSupervisor.resumeTopActivitiesLocked->ActivityStack.resumeTopActivityLocked()
            if (next.app != null && next.app.thread != null) {
                mService.updateLruProcessLocked(next.app, true, null);
            }
            return true;
        }

        if (mService.isSleeping() && mLastNoHistoryActivity != null &&
                !mLastNoHistoryActivity.finishing) {
            requestFinishActivityLocked(mLastNoHistoryActivity.appToken, Activity.RESULT_CANCELED,
                    null, "resume-no-history", false);
            mLastNoHistoryActivity = null;
        }

        if (prev != null && prev != next) {
            if (!mStackSupervisor.mWaitingVisibleActivities.contains(prev)
                    && next != null && !next.nowVisible) {
                mStackSupervisor.mWaitingVisibleActivities.add(prev);
            } else {
                if (prev.finishing) {
                    mWindowManager.setAppVisibility(prev.appToken, false);
                } else {
                }
            }
        }

        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    next.packageName, false, next.userId); /* TODO: Verify if correct userid */
        } catch (RemoteException e1) {
        } catch (IllegalArgumentException e) {}

        boolean anim = true;
        if (prev != null) {
            if (prev.finishing) {
                if (mNoAnimActivities.contains(prev)) {
                    anim = false;
                    mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
                } else {
                    mWindowManager.prepareAppTransition(prev.task == next.task
                            ? AppTransition.TRANSIT_ACTIVITY_CLOSE
                            : AppTransition.TRANSIT_TASK_CLOSE, false);
                }
                mWindowManager.setAppWillBeHidden(prev.appToken);
                mWindowManager.setAppVisibility(prev.appToken, false);
            } else {
                if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                        "Prepare open transition: prev=" + prev);
                if (mNoAnimActivities.contains(next)) {
                    anim = false;
                    mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
                } else {
                    mWindowManager.prepareAppTransition(prev.task == next.task
                            ? AppTransition.TRANSIT_ACTIVITY_OPEN
                            : next.mLaunchTaskBehind
                                    ? AppTransition.TRANSIT_TASK_OPEN_BEHIND
                                    : AppTransition.TRANSIT_TASK_OPEN, false);
                }
            }
            if (false) {
                mWindowManager.setAppWillBeHidden(prev.appToken);
                mWindowManager.setAppVisibility(prev.appToken, false);
            }
        } else {
            if (mNoAnimActivities.contains(next)) {
                anim = false;
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_NONE, false);
            } else {
                mWindowManager.prepareAppTransition(AppTransition.TRANSIT_ACTIVITY_OPEN, false);
            }
        }

        Bundle resumeAnimOptions = null;
        if (anim) {
            ActivityOptions opts = next.getOptionsForTargetActivityLocked();
            if (opts != null) {
                resumeAnimOptions = opts.toBundle();
            }
            next.applyOptionsLocked();
        } else {
            next.clearOptionsLocked();
        }

        ActivityStack lastStack = mStackSupervisor.getLastStack();
        if (next.app != null && next.app.thread != null) {
            // 走到这个分支说明待启动activity所在的进程已经存在了？
            mWindowManager.setAppVisibility(next.appToken, true);

            // schedule launch ticks to collect information about slow apps.
            next.startLaunchTickingLocked();

            ActivityRecord lastResumedActivity =
                    lastStack == null ? null :lastStack.mResumedActivity;
            ActivityState lastState = next.state;

            mService.updateCpuStats();
            // 待启动activity状态已经是RESUMED
            next.state = ActivityState.RESUMED;
            mResumedActivity = next;
            next.task.touchActiveTime();
            mRecentTasks.addLocked(next.task);
            mService.updateLruProcessLocked(next.app, true, null);
            updateLRUListLocked(next);
            mService.updateOomAdjLocked();

            boolean notUpdated = true;
            if (mStackSupervisor.isFrontStack(this)) {
                Configuration config = mWindowManager.updateOrientationFromAppTokens(
                        mService.mConfiguration,
                        next.mayFreezeScreenLocked(next.app) ? next.appToken : null);
                if (config != null) {
                    next.frozenBeforeDestroy = true;
                }
                notUpdated = !mService.updateConfigurationLocked(config, next, false, false);
            }

            if (notUpdated) {
                ActivityRecord nextNext = topRunningActivityLocked(null);
                if (nextNext != next) {
                    mStackSupervisor.scheduleResumeTopActivities();
                }
                if (mStackSupervisor.reportResumedActivityLocked(next)) {
                    mNoAnimActivities.clear();
                    return true;
                }
                return false;
            }

            try {
                ArrayList<ResultInfo> a = next.results;
                if (a != null) {
                    final int N = a.size();
                    if (!next.finishing && N > 0) {
                        next.app.thread.scheduleSendResult(next.appToken, a);
                    }
                }

                if (next.newIntents != null) {
                    next.app.thread.scheduleNewIntent(next.newIntents, next.appToken);
                }

                EventLog.writeEvent(EventLogTags.AM_RESUME_ACTIVITY, next.userId,
                        System.identityHashCode(next), next.task.taskId, next.shortComponentName);

                next.sleeping = false;
                mService.showAskCompatModeDialogLocked(next);
                next.app.pendingUiClean = true;
                next.app.forceProcessStateUpTo(mService.mTopProcessState);
                next.clearOptionsLocked();
                next.app.thread.scheduleResumeActivity(next.appToken, next.app.repProcState,
                        mService.isNextTransitionForward(), resumeAnimOptions);

                mStackSupervisor.checkReadyForSleepLocked();
            } catch (Exception e) {
                // Whoops, need to restart this activity!
                next.state = lastState;
                if (lastStack != null) {
                    lastStack.mResumedActivity = lastResumedActivity;
                }
                Slog.i(TAG, "Restarting because process died: " + next);
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else  if (SHOW_APP_STARTING_PREVIEW && lastStack != null &&
                        mStackSupervisor.isFrontStack(lastStack)) {
                    mWindowManager.setAppStartingWindow(
                            next.appToken, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(next.info.applicationInfo),
                            next.nonLocalizedLabel, next.labelRes, next.icon, next.logo,
                            next.windowFlags, null, true);
                }
                mStackSupervisor.startSpecificActivityLocked(next, true, false);
                return true;
            }

            try {
                next.visible = true;
                completeResumeLocked(next);
            } catch (Exception e) {
                requestFinishActivityLocked(next.appToken, Activity.RESULT_CANCELED, null,
                        "resume-exception", true);
                return true;
            }
            next.stopped = false;

        } else {
            // 走到这个分支说明待启动activity所在的进程没有指定
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    mWindowManager.setAppStartingWindow(
                            next.appToken, next.packageName, next.theme,
                            mService.compatibilityInfoForPackageLocked(
                                    next.info.applicationInfo),
                            next.nonLocalizedLabel,
                            next.labelRes, next.icon, next.logo, next.windowFlags,
                            null, true);
                }
            }
            mStackSupervisor.startSpecificActivityLocked(next, true, true);
        }
        return true;
    }
```
此方法中把待启动activity推到栈顶，并把activity的状态置为resume状态。

#### ActivityStackSupervisor.startSpecificActivityLocked()

```java
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // 这里获取一下activity所运行的进程
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.task.stack.setLaunchTime(r);
        // 这里会走两个分支，一个是进程已经存在，一个是不存在
        // 已经存在进程的case下面走的流程已经包含在创建进程case的流程里面
        // 这里就不再单独介绍
        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {}

            // 如果抛出异常，那么重启发起启动activity进程
        }
        //启动一个新进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```

### AMS中启动新进程

#### ActivityManagerService.startProcessLocked()

这里先介绍一些什么是bad进程：如果一个进程在一分钟内crash两次，那么就会被认为是bad进程而被添加到`mBadProcesses`列表，`BadProcessInfo`里面保存了添加的时间。如果如果用户重新启动进程，那么就把该进程重列表中删除，直到下次crash时。

```java
    final ProcessRecord startProcessLocked(String processName, ApplicationInfo info,
            boolean knownToBeDead, int intentFlags, String hostingType, ComponentName hostingName,
            boolean allowWhileBooting, boolean isolated, int isolatedUid, boolean keepIfLarge,
            String abiOverride, String entryPoint, String[] entryPointArgs, Runnable crashHandler) {
        long startTime = SystemClock.elapsedRealtime();
        ProcessRecord app;
        // 代表是否是完全隔绝的沙箱进程，完全隔绝的沙箱进程每次启动都是独立的，不能复用已有的进程信息。
        if (!isolated) {
            app = getProcessRecordLocked(processName, info.uid, keepIfLarge);
            checkTime(startTime, "startProcess: after getProcessRecord");

            if ((intentFlags & Intent.FLAG_FROM_BACKGROUND) != 0) {
                // 如果是后台进程要判断一下是否是bad进程，如果是bad进程，直接退出
                if (mBadProcesses.get(info.processName, info.uid) != null) {
                    return null;
                }
            } else {
                // 如果是前台启动，清除bad进程标志
                mProcessCrashTimes.remove(info.processName, info.uid);
                if (mBadProcesses.get(info.processName, info.uid) != null) {
                    EventLog.writeEvent(EventLogTags.AM_PROC_GOOD,
                            UserHandle.getUserId(info.uid), info.uid,
                            info.processName);
                    mBadProcesses.remove(info.processName, info.uid);
                    if (app != null) {
                        app.bad = false;
                    }
                }
            }
        } else {
            // 如果是完全隔绝的沙箱进程，那么就不能复用一些已经存在的进程的信息。
            app = null;
        }

        synchronized(ActivityManagerService.this) {
            nativeMigrateToBoost();
            mIsBoosted = true;
            mBoostStartTime = SystemClock.uptimeMillis();
            Message msg = mHandler.obtainMessage(APP_BOOST_DEACTIVATE_MSG);
            mHandler.sendMessageDelayed(msg, APP_BOOST_MESSAGE_DELAY);
        }

        // ProcessRecord已经存在，且已经分配了pid
        if (app != null && app.pid > 0) {
            if (!knownToBeDead || app.thread == null) {
                // 调用者不认为该进程已死亡或者没有thread对象attached到该进程.则不应该清理该进程
                app.addPackage(info.packageName, info.versionCode, mProcessStats);
                checkTime(startTime, "startProcess: done, added package to proc");
                return app;
            }

            // 有thread实例attach到该进程，那么杀死并清理该进程
            checkTime(startTime, "startProcess: bad proc running, killing");
            killProcessGroup(app.info.uid, app.pid);
            handleAppDiedLocked(app, true, true);
            checkTime(startTime, "startProcess: done killing old proc");
        }

        String hostingNameStr = hostingName != null
                ? hostingName.flattenToShortString() : null;

        if (app == null) {
            checkTime(startTime, "startProcess: creating new process record");
            // 创建新的ProcessRecord对象
            app = newProcessRecordLocked(info, processName, isolated, isolatedUid);
            if (app == null) {
                // 创建失败
                return null;
            }
            app.crashHandler = crashHandler;
            checkTime(startTime, "startProcess: done creating new process record");
        } else {
            //如果这是进程中新package，则添加到列表
            app.addPackage(info.packageName, info.versionCode, mProcessStats);
            checkTime(startTime, "startProcess: added package to existing proc");
        }

        //当系统未准备完毕，则将当前进程加入到mProcessesOnHold
        if (!mProcessesReady
                && !isAllowedWhileBooting(info)
                && !allowWhileBooting) {
            if (!mProcessesOnHold.contains(app)) {
                mProcessesOnHold.add(app);
            }
            checkTime(startTime, "startProcess: returning with proc on hold");
            return app;
        }

        checkTime(startTime, "startProcess: stepping in to startProcess");
        // 启动进程
        startProcessLocked(
                app, hostingType, hostingNameStr, abiOverride, entryPoint, entryPointArgs);
        checkTime(startTime, "startProcess: done starting proc!");
        return (app.pid != 0) ? app : null;
    }
```

#### ActivityManagerService.newProcessRecordLocked()

创建新的ProcessRecord对象

```java
    final ProcessRecord newProcessRecordLocked(ApplicationInfo info, String customProcess,
            boolean isolated, int isolatedUid) {
        String proc = customProcess != null ? customProcess : info.processName;
        // 电量统计
        BatteryStatsImpl stats = mBatteryStatsService.getActiveStatistics();
        final int userId = UserHandle.getUserId(info.uid);
        int uid = info.uid;
        if (isolated) {
            // 隔离进程相关
            ......
        }
        final ProcessRecord r = new ProcessRecord(stats, info, proc, uid);
        if (!mBooted && !mBooting
                && userId == UserHandle.USER_OWNER
                && (info.flags & PERSISTENT_MASK) == PERSISTENT_MASK) {
            r.persistent = true;
        }
        addProcessNameLocked(r);
        return r;
    }
```

#### ActivityManagerService.startProcessLocked()

```java
    private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {
        long startTime = SystemClock.elapsedRealtime();
        if (app.pid > 0 && app.pid != MY_PID) {
            checkTime(startTime, "startProcess: removing from pids map");
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.remove(app.pid);
                mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);
            }
            checkTime(startTime, "startProcess: done removing from pids map");
            app.setPid(0);
        }

        // 从HoldProcesses列表中移除该ProcessRecord
        mProcessesOnHold.remove(app);

        checkTime(startTime, "startProcess: starting to update cpu stats");
        updateCpuStats();
        checkTime(startTime, "startProcess: done updating cpu stats");

        try {
            try {
                if (AppGlobals.getPackageManager().isPackageFrozen(app.info.packageName)) {
                    // This is caught below as if we had failed to fork zygote
                    throw new RuntimeException("Package " + app.info.packageName + " is frozen!");
                }
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }

            int uid = app.uid;
            int[] gids = null;
            int mountExternal = Zygote.MOUNT_EXTERNAL_NONE;
            if (!app.isolated) {
                int[] permGids = null;
                try {
                    checkTime(startTime, "startProcess: getting gids from package manager");
                    final IPackageManager pm = AppGlobals.getPackageManager();
                    permGids = pm.getPackageGids(app.info.packageName, app.userId);
                    MountServiceInternal mountServiceInternal = LocalServices.getService(
                            MountServiceInternal.class);
                    mountExternal = mountServiceInternal.getExternalStorageMountMode(uid,
                            app.info.packageName);
                } catch (RemoteException e) {
                    throw e.rethrowAsRuntimeException();
                }

                if (ArrayUtils.isEmpty(permGids)) {
                    gids = new int[2];
                } else {
                    gids = new int[permGids.length + 2];
                    System.arraycopy(permGids, 0, gids, 2, permGids.length);
                }
                gids[0] = UserHandle.getSharedAppGid(UserHandle.getAppId(uid));
                gids[1] = UserHandle.getUserGid(UserHandle.getUserId(uid));
            }
            checkTime(startTime, "startProcess: building args");
            if (mFactoryTest != FactoryTest.FACTORY_TEST_OFF) {
                if (mFactoryTest == FactoryTest.FACTORY_TEST_LOW_LEVEL
                        && mTopComponent != null
                        && app.processName.equals(mTopComponent.getPackageName())) {
                    uid = 0;
                }
                if (mFactoryTest == FactoryTest.FACTORY_TEST_HIGH_LEVEL
                        && (app.info.flags&ApplicationInfo.FLAG_FACTORY_TEST) != 0) {
                    uid = 0;
                }
            }

            // 设置一些debug参数
            int debugFlags = 0;
            ......

            String requiredAbi = (abiOverride != null) ? abiOverride : app.info.primaryCpuAbi;
            if (requiredAbi == null) {
                requiredAbi = Build.SUPPORTED_ABIS[0];
            }

            String instructionSet = null;
            if (app.info.primaryCpuAbi != null) {
                instructionSet = VMRuntime.getInstructionSet(app.info.primaryCpuAbi);
            }

            app.gids = gids;
            app.requiredAbi = requiredAbi;
            app.instructionSet = instructionSet;

            // 创建一个进程
            boolean isActivityProcess = (entryPoint == null);
            if (entryPoint == null) entryPoint = "android.app.ActivityThread";
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "Start proc: " +
                    app.processName);
            checkTime(startTime, "startProcess: asking zygote to start proc");
            Process.ProcessStartResult startResult = Process.start(entryPoint,
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, requiredAbi, instructionSet,
                    app.info.dataDir, entryPointArgs);
            checkTime(startTime, "startProcess: returned from zygote!");
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

            if (app.isolated) {
                mBatteryStatsService.addIsolatedUid(app.uid, app.info.uid);
            }
            mBatteryStatsService.noteProcessStart(app.processName, app.info.uid);
            checkTime(startTime, "startProcess: done updating battery stats");

            EventLog.writeEvent(EventLogTags.AM_PROC_START,
                    UserHandle.getUserId(uid), startResult.pid, uid,
                    app.processName, hostingType,
                    hostingNameStr != null ? hostingNameStr : "");

            if (app.persistent) {
                Watchdog.getInstance().processStarted(app.processName, startResult.pid);
            }
            //设置一些属性变量
            checkTime(startTime, "startProcess: building log message");
            app.setPid(startResult.pid);
            app.usingWrapper = startResult.usingWrapper;
            app.removed = false;
            app.killed = false;
            app.killedByAm = false;
            checkTime(startTime, "startProcess: starting to update pids map");
            //将新创建的进程加入到mPidsSelfLocked
            synchronized (mPidsSelfLocked) {
                this.mPidsSelfLocked.put(startResult.pid, app);
                if (isActivityProcess) {
                    Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
                    msg.obj = app;
                    mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                            ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
                }
            }
            checkTime(startTime, "startProcess: done updating pids map");
        } catch (RuntimeException e) {
            app.setPid(0);
            mBatteryStatsService.noteProcessFinish(app.processName, app.info.uid);
            if (app.isolated) {
                mBatteryStatsService.removeIsolatedUid(app.uid, app.info.uid);
            }
        }
    }
```

### 新进程

`Process.start()` 通过socket通信告知Zygote创建fork子进程，创建新进程后将 `ActivityThread` 类加载到新进程，并调用 `ActivityThread.main()` 方法。

#### ActivityThread.main()

```java
    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        //启动性能统计
        SamplingProfilerIntegration.start();
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();
        EventLogger.setReporter(new EventLoggingReporter());
        AndroidKeyStoreProvider.install();
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");
        // 初始化主线程消息队列
        Looper.prepareMainLooper();
        // 创建ActivityThread对象
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
        // 主线程Handler对象mH
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        // 消息循环开始运行
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

#### ActivityThread.attach()

```java
    private void attach(boolean system) {
        sCurrentActivityThread = this;
        // 是否是系统进程
        mSystemThread = system;
        if (!system) {
            ViewRootImpl.addFirstDrawHandler(new Runnable() {
                @Override
                public void run() {
                    ensureJitEnabled();
                }
            });
            // 暂时设置进程的名字为<pre-initialized>
            android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
            RuntimeInit.setApplicationObject(mAppThread.asBinder());
            // 调用AMS
            final IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {}
            // GC检测
            BinderInternal.addGcWatcher(new Runnable() {
                @Override public void run() {
                    if (!mSomeActivitiesChanged) {
                        return;
                    }
                    Runtime runtime = Runtime.getRuntime();
                    long dalvikMax = runtime.maxMemory();
                    long dalvikUsed = runtime.totalMemory() - runtime.freeMemory();
                    if (dalvikUsed > ((3*dalvikMax)/4)) {
                        //当已用内存超过最大内存的3/4,则请求释放内存空间
                        mSomeActivitiesChanged = false;
                        try {
                            mgr.releaseSomeActivities(mAppThread);
                        } catch (RemoteException e) {
                        }
                    }
                }
            });
        } else {
            // 如果是系统进程，设置名称为system_process
            android.ddm.DdmHandleAppName.setAppName("system_process",
                    UserHandle.myUserId());
            // 初始化mInstrumentation和mInitialApplication
            // 非系统进程这些对象的初始化在handleBindApplication()中进行
            try {
                mInstrumentation = new Instrumentation();
                ContextImpl context = ContextImpl.createAppContext(
                        this, getSystemContext().mPackageInfo);
                mInitialApplication = context.mPackageInfo.makeApplication(true, null);
                mInitialApplication.onCreate();
            } catch (Exception e) {...}
        }

        DropBox.setReporter(new DropBoxReporter());
        // 这里要快速的设置Config回调
        ViewRootImpl.addConfigCallback(new ComponentCallbacks2() {
            @Override
            public void onConfigurationChanged(Configuration newConfig) {
                synchronized (mResourcesManager) {
                    // 快速响应onConfigurationChanged
                    if (mResourcesManager.applyConfigurationToResourcesLocked(newConfig, null)) {
                        if (mPendingConfiguration == null ||
                                mPendingConfiguration.isOtherSeqNewer(newConfig)) {
                            mPendingConfiguration = newConfig;

                            sendMessage(H.CONFIGURATION_CHANGED, newConfig);
                        }
                    }
                }
            }
            @Override
            public void onLowMemory() {
            }
            @Override
            public void onTrimMemory(int level) {
            }
        });
    }
```

接下来又会转到 `ActivityManagerService` 进程去执行。

#### ActivityManagerService.attachApplication()

```java
    @Override
    public final void attachApplication(IApplicationThread thread) {
        synchronized (this) {
            int callingPid = Binder.getCallingPid();
            final long origId = Binder.clearCallingIdentity();
            // thread是ApplicationThreadProxy对象，用于和应用进程的ApplicationThread通信
            attachApplicationLocked(thread, callingPid);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

#### ActivityManagerService.attachApplicationLocked()

```java
    private final boolean attachApplicationLocked(IApplicationThread thread,
            int pid) {

        // 根据pid获取应用的ProcessRecord
        ProcessRecord app;
        if (pid != MY_PID && pid >= 0) {
            synchronized (mPidsSelfLocked) {
                app = mPidsSelfLocked.get(pid);
            }
        } else {
            app = null;
        }

        if (app == null) {
            // 获取ProcessRecord失败，退出
            EventLog.writeEvent(EventLogTags.AM_DROP_PROCESS, pid);
            if (pid > 0 && pid != MY_PID) {
                Process.killProcessQuiet(pid);
            } else {
                try {
                    thread.scheduleExit();
                } catch (Exception e) {}
            }
            return false;
        }

        // 如果ProcessRecord还和之前的进程绑定，就清除之前绑定的进程信息
        if (app.thread != null) {
            handleAppDiedLocked(app, true, true);
        }

        // 绑定应用进程的DeathRecipient，当应用进程崩溃时，系统进程可以收到通知
        final String processName = app.processName;
        try {
            AppDeathRecipient adr = new AppDeathRecipient(
                    app, pid, thread);
            thread.asBinder().linkToDeath(adr, 0);
            app.deathRecipient = adr;
        } catch (RemoteException e) {
            app.resetPackageList(mProcessStats);
            // 绑定失败，这里会重启进程
            startProcessLocked(app, "link fail", processName);
            return false;
        }

        EventLog.writeEvent(EventLogTags.AM_PROC_BOUND, app.userId, app.pid, app.processName);
        // 激活ProcessRecord，设置进程的一些信息
        app.makeActive(thread, mProcessStats);
        app.curAdj = app.setAdj = -100;
        app.curSchedGroup = app.setSchedGroup = Process.THREAD_GROUP_DEFAULT;
        app.forcingToForeground = null;
        updateProcessForegroundLocked(app, false, false);
        app.hasShownUi = false;
        app.debugging = false;
        app.cached = false;
        app.killedByAm = false;
        // 移除超时标志
        mHandler.removeMessages(PROC_START_TIMEOUT_MSG, app);

        boolean normalMode = mProcessesReady || isAllowedWhileBooting(app.info);
        // 获取应用所有的provider
        List<ProviderInfo> providers = normalMode ? generateApplicationProvidersLocked(app) : null;
        // 如果有provider在mLaunchingProviders列表中，那么发消息移除它们
        if (providers != null && checkAppInLaunchingProvidersLocked(app)) {
            Message msg = mHandler.obtainMessage(CONTENT_PROVIDER_PUBLISH_TIMEOUT_MSG);
            msg.obj = app;
            mHandler.sendMessageDelayed(msg, CONTENT_PROVIDER_PUBLISH_TIMEOUT);
        }

        try {
            int testMode = IApplicationThread.DEBUG_OFF;
            if (mDebugApp != null && mDebugApp.equals(processName)) {
                testMode = mWaitForDebugger
                    ? IApplicationThread.DEBUG_WAIT
                    : IApplicationThread.DEBUG_ON;
                app.debugging = true;
                if (mDebugTransient) {
                    mDebugApp = mOrigDebugApp;
                    mWaitForDebugger = mOrigWaitForDebugger;
                }
            }
            String profileFile = app.instrumentationProfileFile;
            ParcelFileDescriptor profileFd = null;
            int samplingInterval = 0;
            boolean profileAutoStop = false;
            if (mProfileApp != null && mProfileApp.equals(processName)) {
                mProfileProc = app;
                profileFile = mProfileFile;
                profileFd = mProfileFd;
                samplingInterval = mSamplingInterval;
                profileAutoStop = mAutoStopProfiler;
            }
            boolean enableOpenGlTrace = false;
            if (mOpenGlTraceApp != null && mOpenGlTraceApp.equals(processName)) {
                enableOpenGlTrace = true;
                mOpenGlTraceApp = null;
            }

            boolean isRestrictedBackupMode = false;
            if (mBackupTarget != null && mBackupAppName.equals(processName)) {
                isRestrictedBackupMode = (mBackupTarget.backupMode == BackupRecord.RESTORE)
                        || (mBackupTarget.backupMode == BackupRecord.RESTORE_FULL)
                        || (mBackupTarget.backupMode == BackupRecord.BACKUP_FULL);
            }

            ensurePackageDexOpt(app.instrumentationInfo != null
                    ? app.instrumentationInfo.packageName
                    : app.info.packageName);
            if (app.instrumentationClass != null) {
                ensurePackageDexOpt(app.instrumentationClass.getPackageName());
            }
            ApplicationInfo appInfo = app.instrumentationInfo != null
                    ? app.instrumentationInfo : app.info;
            app.compat = compatibilityInfoForPackageLocked(appInfo);
            if (profileFd != null) {
                profileFd = profileFd.dup();
            }
            ProfilerInfo profilerInfo = profileFile == null ? null
                    : new ProfilerInfo(profileFile, profileFd, samplingInterval, profileAutoStop);
            // 发起跨进程调用。绑定应用进程，发送一些参数给应用进程
            thread.bindApplication(processName, appInfo, providers, app.instrumentationClass,
                    profilerInfo, app.instrumentationArguments, app.instrumentationWatcher,
                    app.instrumentationUiAutomationConnection, testMode, enableOpenGlTrace,
                    isRestrictedBackupMode || !normalMode, app.persistent,
                    new Configuration(mConfiguration), app.compat,
                    getCommonServicesLocked(app.isolated),
                    mCoreSettingsObserver.getCoreSettingsLocked());
            updateLruProcessLocked(app, false, null);
            app.lastRequestedGc = app.lastLowMemory = SystemClock.uptimeMillis();
        } catch (Exception e) {
            // 绑定失败这里会重启进程
            app.resetPackageList(mProcessStats);
            app.unlinkDeathRecipient();
            startProcessLocked(app, "bind fail", processName);
            return false;
        }

        // 把进程从待启动进程列表中移除
        mPersistentStartingProcesses.remove(app);
        mProcessesOnHold.remove(app);

        boolean badApp = false;
        boolean didSomething = false;

        // 处理Activity
        if (normalMode) {
            try {
                if (mStackSupervisor.attachApplicationLocked(app)) {
                    didSomething = true;
                }
            } catch (Exception e) {
                badApp = true;
            }
        }

        // 处理Service
        if (!badApp) {
            try {
                didSomething |= mServices.attachApplicationLocked(app, processName);
            } catch (Exception e) {
                badApp = true;
            }
        }

        // 处理Broadcast
        if (!badApp && isPendingBroadcastProcessLocked(pid)) {
            try {
                didSomething |= sendPendingBroadcastsLocked(app);
            } catch (Exception e) {
                badApp = true;
            }
        }

        if (!badApp && mBackupTarget != null && mBackupTarget.appInfo.uid == app.uid) {
            ensurePackageDexOpt(mBackupTarget.appInfo.packageName);
            try {
                thread.scheduleCreateBackupAgent(mBackupTarget.appInfo,
                        compatibilityInfoForPackageLocked(mBackupTarget.appInfo),
                        mBackupTarget.backupMode);
            } catch (Exception e) {
                badApp = true;
            }
        }
        // 杀掉坏的进程
        if (badApp) {
            app.kill("error during init", true);
            handleAppDiedLocked(app, false, true);
            return false;
        }

        if (!didSomething) {
            updateOomAdjLocked();
        }

        return true;
    }
```

#### ActivityThread.handleBindApplication()

这里又转到应用进程来处理进程绑定的工作。

```java
    private void handleBindApplication(AppBindData data) {
        mBoundApplication = data;
        mConfiguration = new Configuration(data.config);
        mCompatConfiguration = new Configuration(data.config);

        mProfiler = new Profiler();
        if (data.initProfilerInfo != null) {
            mProfiler.profileFile = data.initProfilerInfo.profileFile;
            mProfiler.profileFd = data.initProfilerInfo.profileFd;
            mProfiler.samplingInterval = data.initProfilerInfo.samplingInterval;
            mProfiler.autoStopProfiler = data.initProfilerInfo.autoStopProfiler;
        }

        // 设置进程的名字
        Process.setArgV0(data.processName);
        android.ddm.DdmHandleAppName.setAppName(data.processName,
                                                UserHandle.myUserId());

        if (data.persistent) {
            // 对persistent进程的处理，在低内存设备上不用硬件加速
            if (!ActivityManager.isHighEndGfx()) {
                HardwareRenderer.disable(false);
            }
        }

        if (mProfiler.profileFd != null) {
            mProfiler.startProfiling();
        }

        if (data.appInfo.targetSdkVersion <= android.os.Build.VERSION_CODES.HONEYCOMB_MR1) {
            AsyncTask.setDefaultExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
        }

        Message.updateCheckRecycle(data.appInfo.targetSdkVersion);

        //设置时区
        TimeZone.setDefault(null);
        Locale.setDefault(data.config.locale);

        // 更新系统配置
        mResourcesManager.applyConfigurationToResourcesLocked(data.config, data.compatInfo);
        mCurDefaultDisplayDpi = data.config.densityDpi;
        applyCompatConfiguration(mCurDefaultDisplayDpi);

        data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);

        if ((data.appInfo.flags&ApplicationInfo.FLAG_SUPPORTS_SCREEN_DENSITIES)
                == 0) {
            mDensityCompatMode = true;
            Bitmap.setDefaultDensity(DisplayMetrics.DENSITY_DEFAULT);
        }
        updateDefaultDensity();

        // 创建Context上下文
        final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
        if (!Process.isIsolated()) {
            ......
        }

        // 设置时区，debug模式，StrictMode
        final boolean is24Hr = "24".equals(mCoreSettings.getString(Settings.System.TIME_12_24));
        DateFormat.set24HourTimePref(is24Hr);

        View.mDebugViewAttributes =
                mCoreSettings.getInt(Settings.Global.DEBUG_VIEW_ATTRIBUTES, 0) != 0;
        if ((data.appInfo.flags &
             (ApplicationInfo.FLAG_SYSTEM |
              ApplicationInfo.FLAG_UPDATED_SYSTEM_APP)) != 0) {
            StrictMode.conditionallyEnableDebugLogging();
        }
        if (data.appInfo.targetSdkVersion > 9) {
            StrictMode.enableDeathOnNetwork();
        }

        NetworkSecurityPolicy.getInstance().setCleartextTrafficPermitted(
                (data.appInfo.flags & ApplicationInfo.FLAG_USES_CLEARTEXT_TRAFFIC) != 0);

        if (data.debugMode != IApplicationThread.DEBUG_OFF) {
            Debug.changeDebugPort(8100);
            if (data.debugMode == IApplicationThread.DEBUG_WAIT) {
                IActivityManager mgr = ActivityManagerNative.getDefault();
                try {
                    mgr.showWaitingForDebugger(mAppThread, true);
                } catch (RemoteException ex) {}

                Debug.waitForDebugger();

                try {
                    mgr.showWaitingForDebugger(mAppThread, false);
                } catch (RemoteException ex) {}

            } else { }
        }

        if (data.enableOpenGlTrace) {
            GLUtils.setTracingLevel(1);
        }

        // 如果是调试模式生成systrace 信息
        boolean appTracingAllowed = (data.appInfo.flags&ApplicationInfo.FLAG_DEBUGGABLE) != 0;
        Trace.setAppTracingAllowed(appTracingAllowed);

        // 初始化默认http代理
        IBinder b = ServiceManager.getService(Context.CONNECTIVITY_SERVICE);
        if (b != null) {
            IConnectivityManager service = IConnectivityManager.Stub.asInterface(b);
            try {
                final ProxyInfo proxyInfo = service.getProxyForNetwork(null);
                Proxy.setHttpProxySystemProperty(proxyInfo);
            } catch (RemoteException e) {}
        }

        if (data.instrumentationName != null) {
            InstrumentationInfo ii = null;
            try {
                ii = appContext.getPackageManager().
                    getInstrumentationInfo(data.instrumentationName, 0);
            } catch (PackageManager.NameNotFoundException e) {
            }
            if (ii == null) {
                throw new RuntimeException(
                    "Unable to find instrumentation info for: "
                    + data.instrumentationName);
            }

            mInstrumentationPackageName = ii.packageName;
            mInstrumentationAppDir = ii.sourceDir;
            mInstrumentationSplitAppDirs = ii.splitSourceDirs;
            mInstrumentationLibDir = ii.nativeLibraryDir;
            mInstrumentedAppDir = data.info.getAppDir();
            mInstrumentedSplitAppDirs = data.info.getSplitAppDirs();
            mInstrumentedLibDir = data.info.getLibDir();

            ApplicationInfo instrApp = new ApplicationInfo();
            instrApp.packageName = ii.packageName;
            instrApp.sourceDir = ii.sourceDir;
            instrApp.publicSourceDir = ii.publicSourceDir;
            instrApp.splitSourceDirs = ii.splitSourceDirs;
            instrApp.splitPublicSourceDirs = ii.splitPublicSourceDirs;
            instrApp.dataDir = ii.dataDir;
            instrApp.nativeLibraryDir = ii.nativeLibraryDir;
            LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                    appContext.getClassLoader(), false, true, false);
            ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

            try {
                java.lang.ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            } catch (Exception e) {
                throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            mInstrumentation.init(this, instrContext, appContext,
                   new ComponentName(ii.packageName, ii.name), data.instrumentationWatcher,
                   data.instrumentationUiAutomationConnection);

            if (mProfiler.profileFile != null && !ii.handleProfiling
                    && mProfiler.profileFd == null) {
                mProfiler.handlingProfiling = true;
                File file = new File(mProfiler.profileFile);
                file.getParentFile().mkdirs();
                Debug.startMethodTracing(file.toString(), 8 * 1024 * 1024);
            }

        } else {
            // 初始化mInstrumentation
            mInstrumentation = new Instrumentation();
        }

        if ((data.appInfo.flags&ApplicationInfo.FLAG_LARGE_HEAP) != 0) {
            // 如果设置了largeHeap，清楚内存上限
            dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
        } else {
            // 设置内存上限
            dalvik.system.VMRuntime.getRuntime().clampGrowthLimit();
        }

        final StrictMode.ThreadPolicy savedPolicy = StrictMode.allowThreadDiskWrites();
        try {
            // 创建Application对象
            Application app = data.info.makeApplication(data.restrictedBackupMode, null);
            mInitialApplication = app;

            // 装载Providers
            if (!data.restrictedBackupMode) {
                List<ProviderInfo> providers = data.providers;
                if (providers != null) {
                    installContentProviders(app, providers);
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }

            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            try {
                //调用Application.onCreate()
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        } finally {
            StrictMode.setThreadPolicy(savedPolicy);
        }
    }
```

#### ActivityStackSupervisor.attachApplicationLocked()

接下来会继续处理 `ActivityManagerService.attachApplication()` 中的 `ActivityStackSupervisor.attachApplicationLocked()` 方法。将 `ActivityStackSupervisor` 绑定到应用进程。

```java
    boolean attachApplicationLocked(ProcessRecord app) throws RemoteException {
        final String processName = app.processName;
        boolean didSomething = false;
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                if (!isFrontStack(stack)) {
                    continue;
                }
                ActivityRecord hr = stack.topRunningActivityLocked(null);
                if (hr != null) {
                    if (hr.app == null && app.uid == hr.info.applicationInfo.uid
                            && processName.equals(hr.processName)) {
                        try {
                            // 真正的启动activity，调用Activity.onCreate，从这里开始的流程
                            // 就和进程已经存在时启动activity流程一样的了
                            if (realStartActivityLocked(hr, app, true, true)) {
                                didSomething = true;
                            }
                        } catch (RemoteException e) {
                            throw e;
                        }
                    }
                }
            }
        }
        if (!didSomething) {
            ensureActivitiesVisibleLocked(null, 0);
        }
        return didSomething;
    }
```

#### ActivityStackSupervisor.realStartActivityLocked()

开始执行真正的Activity启动操作，从这里开始的流程就和进程已经存在时启动activity流程一样的了。

```java
    final boolean realStartActivityLocked(ActivityRecord r,
            ProcessRecord app, boolean andResume, boolean checkConfig)
            throws RemoteException {

        if (andResume) {
            // 开始冻屏
            r.startFreezingScreenLocked(app, 0);
            mWindowManager.setAppVisibility(r.appToken, true);

            r.startLaunchTickingLocked();
        }

        if (checkConfig) {
            Configuration config = mWindowManager.updateOrientationFromAppTokens(
                    mService.mConfiguration,
                    r.mayFreezeScreenLocked(app) ? r.appToken : null);
            mService.updateConfigurationLocked(config, r, false, false);
        }

        r.app = app;
        app.waitingToKill = null;
        r.launchCount++;
        r.lastLaunchTime = SystemClock.uptimeMillis();

        int idx = app.activities.indexOf(r);
        if (idx < 0) {
            app.activities.add(r);
        }
        mService.updateLruProcessLocked(app, true, null);
        mService.updateOomAdjLocked();

        final TaskRecord task = r.task;
        if (task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE ||
                task.mLockTaskAuth == LOCK_TASK_AUTH_LAUNCHABLE_PRIV) {
            setLockTaskModeLocked(task, LOCK_TASK_MODE_LOCKED, "mLockTaskAuth==LAUNCHABLE", false);
        }

        final ActivityStack stack = task.stack;
        try {
            if (app.thread == null) {
                throw new RemoteException();
            }
            List<ResultInfo> results = null;
            List<ReferrerIntent> newIntents = null;
            if (andResume) {
                results = r.results;
                newIntents = r.newIntents;
            }
            if (andResume) {
                EventLog.writeEvent(EventLogTags.AM_RESTART_ACTIVITY,
                        r.userId, System.identityHashCode(r),
                        task.taskId, r.shortComponentName);
            }
            if (r.isHomeActivity() && r.isNotResolverActivity()) {
                // Home process 是task中的第一个进程
                mService.mHomeProcess = task.mActivities.get(0).app;
            }
            mService.ensurePackageDexOpt(r.intent.getComponent().getPackageName());
            r.sleeping = false;
            r.forceNewConfig = false;
            mService.showAskCompatModeDialogLocked(r);
            r.compat = mService.compatibilityInfoForPackageLocked(r.info.applicationInfo);
            ProfilerInfo profilerInfo = null;
            if (mService.mProfileApp != null && mService.mProfileApp.equals(app.processName)) {
                if (mService.mProfileProc == null || mService.mProfileProc == app) {
                    mService.mProfileProc = app;
                    final String profileFile = mService.mProfileFile;
                    if (profileFile != null) {
                        ParcelFileDescriptor profileFd = mService.mProfileFd;
                        if (profileFd != null) {
                            try {
                                profileFd = profileFd.dup();
                            } catch (IOException e) {
                                if (profileFd != null) {
                                    try {
                                        profileFd.close();
                                    } catch (IOException o) {
                                    }
                                    profileFd = null;
                                }
                            }
                        }

                        profilerInfo = new ProfilerInfo(profileFile, profileFd,
                                mService.mSamplingInterval, mService.mAutoStopProfiler);
                    }
                }
            }

            if (andResume) {
                app.hasShownUi = true;
                app.pendingUiClean = true;
            }
            app.forceProcessStateUpTo(mService.mTopProcessState);
            // 调用ApplicationThreadNative.ApplicationThreadProxy.scheduleLaunchActivity()，跳到应用进程
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info, new Configuration(mService.mConfiguration),
                    new Configuration(stack.mOverrideConfig), r.compat, r.launchedFromPackage,
                    task.voiceInteractor, app.repProcState, r.icicle, r.persistentState, results,
                    newIntents, !andResume, mService.isNextTransitionForward(), profilerInfo);

            if ((app.info.privateFlags&ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0) {
                if (app.processName.equals(app.info.packageName)) {
                    if (mService.mHeavyWeightProcess != null
                            && mService.mHeavyWeightProcess != app) {
                        ...
                    }
                    mService.mHeavyWeightProcess = app;
                    Message msg = mService.mHandler.obtainMessage(
                            ActivityManagerService.POST_HEAVY_NOTIFICATION_MSG);
                    msg.obj = r;
                    mService.mHandler.sendMessage(msg);
                }
            }

        } catch (RemoteException e) {
            if (r.launchFailed) {
                mService.appDiedLocked(app);
                stack.requestFinishActivityLocked(r.appToken, Activity.RESULT_CANCELED, null,
                        "2nd-crash", false);
                return false;
            }

            app.activities.remove(r);
            throw e;
        }

        r.launchFailed = false;
        ...

        if (andResume) {
            stack.minimalResumeActivityLocked(r);
        } else {
            r.state = STOPPED;
            r.stopped = true;
        }

        if (isFrontStack(stack)) {
            mService.startSetupActivityLocked();
        }

        mService.mServices.updateServiceConnectionActivitiesLocked(r.app);

        return true;
    }

```

#### ActivityThread.H.handleMessage():LAUNCH_ACTIVITY

跳转到应用进程来处理，从这里开始就开始Activity的生命周期的运转了。

```java
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
        }
```

#### ActivityThread.handleLaunchActivity()

```java
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        unscheduleGcIdler();
        mSomeActivitiesChanged = true;

        if (r.profilerInfo != null) {
            mProfiler.setProfiler(r.profilerInfo);
            mProfiler.startProfiling();
        }

        handleConfigurationChanged(null, null);

        // 初始化WMS
        WindowManagerGlobal.initialize();
        // 创建activity并调用onCreate，onStart等生命周期函数
        Activity a = performLaunchActivity(r, customIntent);

        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            Bundle oldState = r.state;
            // 调用onResume方法
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed);

            if (!r.activity.mFinished && r.startsNotResumed) {
                try {
                    r.activity.mCalled = false;
                    mInstrumentation.callActivityOnPause(r.activity);
                    if (r.isPreHoneycomb()) {
                        r.state = oldState;
                    }
                    if (!r.activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPause()");
                    }

                } catch (SuperNotCalledException e) {
                    throw e;
                } catch (Exception e) {
                    if (!mInstrumentation.onException(r.activity, e)) {
                        throw new RuntimeException(
                                "Unable to pause activity "
                                + r.intent.getComponent().toShortString()
                                + ": " + e.toString(), e);
                    }
                }
                r.paused = true;
            }
        } else {
            try {
                ActivityManagerNative.getDefault()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);
            } catch (RemoteException ex) {}
        }
    }
```

#### ActivityThread.performLaunchActivity()

```java
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // 收集要启动的Activity的package信息
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
        // 收集要启动的Activity的component信息
        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        // 新建一个Activity对象，通过ClassLoader将要启动的Activity类加载进来
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            // 获取Application对象
            // 不知道大家有没有印象，在ActivityThread.handleBindApplication方法中已经调用过makeApplication
            // 为什么还要make呢？追踪源码会发现，如果已经创建就返回以前创建的对象
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                // 创建Context对象，并绑定到Activity
                Context appContext = createBaseContextForActivity(r, activity);
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                // 调用Activity.onCreate方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    // 调用Activity.onStart方法
                    activity.performStart();
                    r.stopped = false;
                }
                if (!r.activity.mFinished) {
                    // 调用Activity.onRestoreInstanceState方法
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
                if (!r.activity.mFinished) {
                    activity.mCalled = false;
                    // 调用Activity.onPostCreate方法
                    if (r.isPersistable()) {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state,
                                r.persistentState);
                    } else {
                        mInstrumentation.callActivityOnPostCreate(activity, r.state);
                    }
                    if (!activity.mCalled) {
                        throw new SuperNotCalledException(
                            "Activity " + r.intent.getComponent().toShortString() +
                            " did not call through to super.onPostCreate()");
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }
        return activity;
    }
```

从 `performLaunchActivity` 方法开始，正式进入了 `Activity` 的 `onCreate()`、`onStart()`、`onResume()`等生命周期过程。
至此，`Activity` 的启动流程基本介绍完了。

## 方法详解

### findTaskLocked()

### findActivityLocked()

### resolveActivity()

## 参考文档

<!--     
http://duanqz.github.io/2016-07-29-Activity-LaunchProcess-Part1#10-amsnewprocessrecordlocked
http://duanqz.github.io/2016-10-23-Activity-LaunchProcess-Part2#3-amsattachapplicationlocked
http://gityuan.com/2016/03/12/start-activity/

http://blog.csdn.net/luoshengyang/article/details/6689748
http://blog.csdn.net/Luoshengyang/article/details/6703247
http://duanqz.github.io/2016-02-01-Activity-Maintenance
http://duanqz.github.io/2016-07-29-Activity-LaunchProcess-Part1
http://gityuan.com/2016/03/12/start-activity/
http://gityuan.com/2016/10/09/app-process-create-2/
http://www.cnblogs.com/franksunny/archive/2012/04/17/2453403.html
http://blog.csdn.net/shan987/article/details/50511439
http://blog.csdn.net/yunbin_7/article/details/46003509
http://www.jianshu.com/p/2d52fa0bce90
-->
