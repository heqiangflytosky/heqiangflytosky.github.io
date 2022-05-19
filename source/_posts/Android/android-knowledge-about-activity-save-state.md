---
title: Android -- Activity 之 onSaveInstanceState 深入理解
categories: Android
comments: true
tags: [onSaveInstanceState]
description: 基于Android6.0来分析 onSaveInstanceState 数据保存源码
date: 2018-8-12 10:00:00
---

## 概述

我们在 Android 开发过程中应该都用过 onSaveInstanceState() 方法和 onRestoreInstanceState()，它们是 Android 系统为我们提供的保存和恢复 Activity 数据的方法。当 Activity 屏幕旋转导致重建或者被意外杀掉后，我们可以使用上面的方法来进行数据的保存和恢复工作，这样重新打开 Activity 时，可以恢复成原来的状态。
onSaveInstanceState() 只应该被用来保存 Activity 暂时的状态（例如，类成员变量的值关联着UI），而不应该用来保存持久化数据。持久化数据应该当用户离开当前的 Activity 时，在 onPause() 或者 onResume 中保存（比如，保存到数据库）。
一般情况下，通过 onSaveInstanceState 保存的数据是保存在 system 进程中，当 Activity 意外杀掉后可以做数据恢复，但是如果手机重启的话，这些数据是会丢失的。那么这个时候可以用 PersistableBundle 来做数据恢复，为此 Android 也重载的 onCreate、onSaveInstanceState 和 onRestoreInstanceState 来做相关操作。

## 相关方法介绍

 - onCreate(@Nullable Bundle savedInstanceState)
 - onCreate(@Nullable Bundle savedInstanceState,@Nullable PersistableBundle persistentState)
 - onSaveInstanceState(Bundle outState)
 - onSaveInstanceState(Bundle outState, PersistableBundle outPersistentState)
 - onRestoreInstanceState(Bundle savedInstanceState)
 - onRestoreInstanceState(Bundle savedInstanceState,PersistableBundle persistentState)

上面的 onSaveInstanceState 用于存储我们需要保存的数据，onCreate 和 onRestoreInstanceState 用户恢复数据。
onCreate 和 onRestoreInstanceState 都可以恢复数据，但他们也有些区别：

 - onRestoreInstanceState 是和 onSaveInstanceState 一一对应的方法。
 - onCreate 中的 Bundle 可能为 null，而 onRestoreInstanceState 中的 Bundle 不会为 null。
 - onRestoreInstanceState 不保证一定会被调用，除非上次Activity意外退出。

正常打开应用时 onCreate 中的 Bundle 为空，此时不会调用 onRestoreInstanceState。如果出现了意外退出，下次启动 Activity 时，onCreate 中的 Bundle 不为空，会调用 onRestoreInstanceState，且其参数 Bundle 不会为空。
上面的方法都有一个重载的方法，多了个 PersistableBundle 参数。这个方法后面会介绍。

## 使用方法

```
    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        outState.putString("Test1","Test1");
    }
```

在 onSaveInstanceState 中添加我们需要保存的数据。
然后在 onCreate 或者 onRestoreInstanceState 中取出数据。

```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_other_test);
        Intent intent = getIntent();
        Bundle bundle = intent.getExtras();
        if(bundle != null){
            Log.e("Test","Test = "+bundle.getString("Test"));
        }

        // savedInstanceState 可能为空
        if (savedInstanceState != null) {
            Log.e("Test","get savedInstanceState"+savedInstanceState);
            Log.e("Test","Test1 = "+savedInstanceState.getString("Test1"));
        }
    }
    
    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        super.onRestoreInstanceState(savedInstanceState);
        Log.e("Test","Test1 = "+savedInstanceState.getString("Test1"));
    }
```


## 关于 PersistableBundle

PersistableBundle 是 API 21为 Activity 新增加的属性，用于保存异常状况下Activity中的数据。
要想使用 PersistableBundle ，可以在 Manifest 中为 Activity 设置属性：

```
android:persistableMode="persistRootOnly"
```

和这个属性相关的方法有三个：

```
public void onSaveInstanceState(Bundle outState, PersistableBundle outPersistentState)
public void onRestoreInstanceState(Bundle savedInstanceState, PersistableBundle persistentState) 
public void onCreate(Bundle savedInstanceState, PersistableBundle persistentState)
```

persistableMode 可以有三种不同的配置：

 - persistNever：不起作用，不会调用上面方法做相关状态保存操作。
 - persistRootOnly：默认值。仅仅会作用在根activity或者task中。
 - persistAcrossReboots：重启设备，会持久化页面的数据或者状态。

## onSaveInstanceState 调用时机

系统不会保证onSaveInstanceState()在activity被销毁前总被调用（因为有一些不需要保存状态的情况，比如，用户用 BACK 键关闭此activity。如果onSaveInstanceState()被调用，那么一定是在onStop()之前，或者可能在onPause()之前。
如果你在onSaveInstanceState()不做啥事(不重载它)，有些 Activity 的状态也会被 Activity 默认的 onSaveInstanceState() 保存。比如，通常 EditText 会保存自己的状态，原因就是 Activity 默认的 onSaveInstanceState() 会调用其 layout 里面每一个 View 的 onSaveInstanceState()，当然也包括此 EditView 。你只需要为这些view提供唯一的ID（android:id属性），他们就会保存自己的状态，不用你操心了。这个自动保存可以通过 android:saveEnabled 属性或者 setSaveEnabled() 方法来禁用。

APP 进程调用栈如下所示：

```
├── ActivityThread.H.handleMessage:STOP_ACTIVITY_HIDE
    ├── ActivityThread.handleStopActivity
        ├── ActivityThread.performStopActivityInner
            ├── ActivityThread.callCallActivityOnSaveInstanceState
                ├── Instrumentation.callActivityOnSaveInstanceState
                    ├── Activity.performSaveInstanceState
                        ├── MyActivity.onSaveInstanceState

```

当我们按返回键退出当前 Activity 时，是不会调用 onSaveInstanceState 的。

我们来看一下 Activity 的 onSaveInstanceState 方法：

```
    protected void onSaveInstanceState(Bundle outState) {
	// 保存 View 的状态
        outState.putBundle(WINDOW_HIERARCHY_TAG, mWindow.saveHierarchyState());

        outState.putInt(LAST_AUTOFILL_ID, mLastAutofillId);
	// 保存 Fragment 的状态
        Parcelable p = mFragments.saveAllState();
        if (p != null) {
            outState.putParcelable(FRAGMENTS_TAG, p);
        }
        if (mAutoFillResetNeeded) {
            outState.putBoolean(AUTOFILL_RESET_NEEDED, true);
            getAutofillManager().onSaveInstanceState(outState);
        }
	// 通知一些声明周期的监听器来执行状态保存
        getApplication().dispatchActivitySaveInstanceState(this, outState);
    }
```

## 数据保存

那么是如何把我们需要保存的数据保存到 system 进程中的呢？

先来看 **App 进程**部分：

```
├── ApplicationThreadNative.onTransact:SCHEDULE_STOP_ACTIVITY_TRANSACTION
    ├── ActivityThread.ApplicationThread.scheduleStopActivity
        ├── ActivityThread.handleStopActivity
            ├── StopInfo.run
                ├── ActivityManagerProxy.activityStopped
                    ├── mRemote.transact(ACTIVITY_STOPPED_TRANSACTION, data, reply, IBinder.FLAG_ONEWAY)
```

可以看到，数据的保存是伴随着 onStop 回调方法的调用而进行的。

```
    private void handleStopActivity(IBinder token, boolean show, int configChanges, int seq) {
        //从 mActivities 中 取出 ActivityClientRecord 实例
        ActivityClientRecord r = mActivities.get(token);
        if (!checkAndUpdateLifecycleSeq(seq, r, "stopActivity")) {
            return;
        }
        // FLYME:huangjinhai@BugFixAOSP Check this to avoid a NullPointerException {@
        if (r == null) {
            Log.w(TAG, "handleStopActivity: no activity for token " + token);
            return;
        }
        // @}
        StopInfo info = new StopInfo();
        performStopActivityInner(r, info, show, true, "handleStopActivity");

        if (localLOGV) Slog.v(
            TAG, "Finishing stop of " + r + ": show=" + show
            + " win=" + r.window);

        updateVisibility(r, show);

        // Make sure any pending writes are now committed.
        if (!r.isPreHoneycomb()) {
            QueuedWork.waitToFinish();
        }

        // Schedule the call to tell the activity manager we have
        // stopped.  We don't do this immediately, because we want to
        // have a chance for any other pending work (in particular memory
        // trim requests) to complete before you tell the activity
        // manager to proceed and allow us to go fully into the background.
        // 赋值，传送到 system进程
        info.activity = r;
        info.state = r.state;
        info.persistentState = r.persistentState;
        mH.post(info);
        mSomeActivitiesChanged = true;
    }
```

```
    private static class StopInfo implements Runnable {
        ActivityClientRecord activity;
        Bundle state;
        PersistableBundle persistentState;
        CharSequence description;

        @Override public void run() {
            // Tell activity manager we have been stopped.
            try {
                if (DEBUG_MEMORY_TRIM) Slog.v(TAG, "Reporting activity stopped: " + activity);
                ActivityManagerNative.getDefault().activityStopped(
                    activity.token, state, persistentState, description);
            } catch (RemoteException ex) {
                if (ex instanceof TransactionTooLargeException
                        && activity.packageInfo.getTargetSdkVersion() < Build.VERSION_CODES.N) {
                    Log.e(TAG, "App sent too much data in instance state, so it was ignored", ex);
                    return;
                }
                throw ex.rethrowFromSystemServer();
            }
        }
    }
```

在 StopInfo 的run方法中，把state等数据发送给system进程。

后面会把数据传送到 system 进程。
再来看一下 **system 进程**部分：

```
├── ActivityManagerNative.onTransact:ACTIVITY_STOPPED_TRANSACTION
    ├── ActivityManagerService.activityStopped
        ├── ActivityStack.activityStoppedLocked
```

保存 state 操作是在 ActivityStack.activityStoppedLocked进行的。

```
    final void activityStoppedLocked(ActivityRecord r, Bundle icicle,
            PersistableBundle persistentState, CharSequence description) {
        if (r.state != ActivityState.STOPPING) {
            Slog.i(TAG, "Activity reported stop, but no longer stopping: " + r);
            mHandler.removeMessages(STOP_TIMEOUT_MSG, r);
            return;
        }
        // 把 persistentState  保存到 system 进程
        if (persistentState != null) {
            r.persistentState = persistentState;
            mService.notifyTaskPersisterLocked(r.task, false);
        }
        if (DEBUG_SAVED_STATE) Slog.i(TAG_SAVED_STATE, "Saving icicle of " + r + ": " + icicle);
        if (icicle != null) {
            // If icicle is null, this is happening due to a timeout, so we
            // haven't really saved the state.
            // 把 icicle  保存到 system 进程
            r.icicle = icicle;
            r.haveState = true;
            r.launchCount = 0;
            r.updateThumbnailLocked(null, description);
        }
        ......
        }
    }
```


## 数据恢复过程

我们从 App 进程再到 system 进程来反向追踪这个调用流程。
APP 进程调用栈如下所示：

```
├── ApplicationThreadNative.onTransact:SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION
    ├── ActivityThread.ApplicationThread.scheduleLaunchActivity
        ├── ActivityThread.H.handleMessage:LAUNCH_ACTIVITY
            ├── ActivityThread.handleLaunchActivity
                ├── ActivityThread.performLaunchActivity
                    ├── Instrumentation.callActivityOnCreate
                        ├── Activity.performCreate
                            ├── MyActivity.onCreate
                    ├── Instrumentation.callActivityOnRestoreInstanceState
                        ├── Activity.performRestoreInstanceState
                            ├── MyActivity.onRestoreInstanceState
```

先来看一下 ActivityThread.performLaunchActivity 这部分代码：

```
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ......
                activity.mCalled = false;
                // 调用 callActivityOnCreate
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
                    activity.performStart();
                    r.stopped = false;
                }
                // 根据 r.state 来决定是否调用 callActivityOnRestoreInstanceState
                // 因此，onRestoreInstanceState 回调在 Activity的生命周期中不是必调用的
                if (!r.activity.mFinished) {
                    if (r.isPersistable()) {
                        if (r.state != null || r.persistentState != null) {
                            mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,
                                    r.persistentState);
                        }
                    } else if (r.state != null) {
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);
                    }
                }
            }
            r.paused = true;

            mActivities.put(r.token, r);
        ......
    }
```

根据上面的代码我们发现，Acitivity 的 OnCreate 的 Bundle 参数是否为空以及是否回调 onRestoreInstanceState 是根据 r.state 是否为空来决定的。
那么决定这一切的 ActivityClientRecord 是什么呢？
ActivityClientRecord 是 ActivityThread 的一个内部类，它是 App 进程中一个 Activity 的代表，和 Activity 有一一对应的关系，它的 activity 成员引用真正的 Activity 组件。
那么 ActivityClientRecord 又是从 哪里传递过来的呢？
现在我们来看一下 App 进程调用的源头：ActivityThread.ApplicationThread.scheduleLaunchActivity：

```
        @Override
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);

            ActivityClientRecord r = new ActivityClientRecord();

            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            r.persistentState = persistentState;

            r.pendingResults = pendingResults;
            r.pendingIntents = pendingNewIntents;

            r.startsNotResumed = notResumed;
            r.isForward = isForward;

            r.profilerInfo = profilerInfo;

            r.overrideConfig = overrideConfig;
            updatePendingConfiguration(curConfig);

            sendMessage(H.LAUNCH_ACTIVITY, r);
```

这里创建了一个 ActivityClientRecord 对象，然后对我们关心的 r.state 进行赋值。state 是从 system_server 通过进程间调用传递过来的。

那么我们现在就开始分析 **system_server 进程**部分：

在 ApplicationThreadProxy.scheduleLaunchActivity 里面，state 被写入然后传递到客户端：

```
    public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
            ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
            CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
            int procState, Bundle state, PersistableBundle persistentState,
            List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
            boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) throws RemoteException {
        Parcel data = Parcel.obtain();
        ......
        data.writeInt(procState);
        data.writeBundle(state);
        data.writePersistableBundle(persistentState);
        ......
        mRemote.transact(SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }

```

那么现在我们就追踪一下，state的来源：

```
├── ActivityStack.resumeTopActivityInnerLocked
    ├── ActivityStackSupervisor.startSpecificActivityLocked
        ├── ActivityStackSupervisor.realStartActivityLocked
            ├── ApplicationThreadProxy.scheduleLaunchActivity
```

在 ActivityStackSupervisor.realStartActivityLocked 方法中，把参数 ActivityRecord 的 icicle 属性作为参数传递给ActivityStack.resumeTopActivityInnerLocked。
ActivityRecord 对应着 App 进程中的一个Acitivty实例，保存了 Activity 的所有信息。
那么 ActivityRecord 参数又是从哪里得到的呢？
在 ActivityStack.resumeTopActivityInnerLocked 方法中：

```
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ......
    final ActivityRecord next = topRunningActivityLocked();
    ......
    mStackSupervisor.startSpecificActivityLocked(next, true, true);
    ......
}
```

通过追踪源码发现：

```
    final ActivityRecord topRunningActivityLocked() {
        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
            ActivityRecord r = mTaskHistory.get(taskNdx).topRunningActivityLocked();
            if (r != null) {
                return r;
            }
        }
        return null;
    }
```

mTaskHistory 中存储了对应与 App 进程中的任务栈。
因此，当 App 进程中的 Activity 意外退出时，system 进程还是保存着 Activity 的信息的，以便于下次启动时恢复数据。

## 数据清理

system_server 进程

强制停止：当调用 am fore-stop 或者 设置里面的强制停止时，调用流程如下：

```
├── ActivityManagerService.forceStopPackage
    ├── ActivityManagerService.forceStopPackageLocked
        ├── ActivityManagerService.forceStopPackageLocked
            ├── ActivityStackSupervisor.finishDisabledPackageActivitiesLocked
                ├── ActivityStack.finishDisabledPackageActivitiesLocked
                    ├── ActivityStack.finishActivityLocked
                        ├── ActivityStack.finishActivityResultsLocked
```

onDestory：当Activity 走正常销毁逻辑时：

```
├── ActivityManagerService.activityDestroyed
    ├── ActivityStack.activityDestroyedLocked
        ├── ActivityStack.removeActivityFromHistoryLocked
            ├── ActivityStack.finishActivityResultsLocked
```

都调用了 ActivityStack.finishActivityResultsLocked，在这个方法中，会对 icicle 进行置空操作：

```
    final void finishActivityResultsLocked(ActivityRecord r, int resultCode, Intent resultData) {
        // send the result
        ActivityRecord resultTo = r.resultTo;
        if (resultTo != null) {
            if (DEBUG_RESULTS) Slog.v(TAG_RESULTS, "Adding result to " + resultTo
                    + " who=" + r.resultWho + " req=" + r.requestCode
                    + " res=" + resultCode + " data=" + resultData);
            if (resultTo.userId != r.userId) {
                if (resultData != null) {
                    resultData.prepareToLeaveUser(r.userId);
                }
            }
            if (r.info.applicationInfo.uid > 0) {
                mService.grantUriPermissionFromIntentLocked(r.info.applicationInfo.uid,
                        resultTo.packageName, resultData,
                        resultTo.getUriPermissionsLocked(), resultTo.userId);
            }
            resultTo.addResultLocked(r, r.resultWho, r.requestCode, resultCode,
                                     resultData);
            r.resultTo = null;
        }
        else if (DEBUG_RESULTS) Slog.v(TAG_RESULTS, "No result destination from " + r);

        // Make sure this HistoryRecord is not holding on to other resources,
        // because clients have remote IPC references to this object so we
        // can't assume that will go away and want to avoid circular IPC refs.
        r.results = null;
        r.pendingResults = null;
        r.newIntents = null;
        r.icicle = null;
    }
```