
---
title: Android LaunchMode taskAffinity 与 Intent Flag 详解
categories: Android
comments: true
tags: [Android, LaunchMode, Intent Flag]
description: 介绍 Android LaunchMode 与 Intent Flag
date: 2015-12-8 10:00:00
---

## taskAffinity

我们可以在 AndroidManifest.xml 中通过 `android:taskAffinity` 属性为 Activity 指定 taskAffinity。
那么什么是 taskAffinity 呢？
taskAffinity 可以为 Activity 指定归属于那个 Task。
一般情况下，如果应用中的 Activity 都没有特殊的显式指定 taskAffinity，那么它的这个属性就等于 Application 指明的 taskAffinity，如果 Application 也没有指明，那么该 taskAffinity 的值就等于应用的包名。
taskAffinity 可以为任意字符串，但是非空的话必须至少包含一个“.”，否则会编译报错。但是你可以指定taskAffinity为空字符串，这时它就不属于任何 Task。
Activity 的启动都会在它通过 taskAffinity 指定的 Task 里面启动。

## LaunchMode

### standard

标准模式。如果不显式的指定 launchMode，Activity都会按照此种模式启动。这种模式下的 Activity 可以有多个实例，不同任务中可以有不同的 Activity 实例，同一个任务中也可以有多个 Activity 实例。这种模式下启动Activity总是会重新创建Activity，对应调用 `onCreate()` 方法。
Android L(Android 5.0)前后对这种启动模式的处理方式是不同的：

 - Android L之前：每次以该模式启动的 Activity 都会被压入当前任务的顶部，启动 N 次，在当前任务就会出现 N 个 Activity 的实例，每次Back键就会销毁一个，直到按了 N 次Back键。
 - Android L之后：如果要以该模式启动 Activity 都是来自同一应用，那么还是会像之前一样，压入当前任务的顶部; 如果是来自不同应用，那么将会创建一个新的任务，然后将 Activity 的实例压入新的任务中。Android L做的优化主要是针对多任务的显示。

### singleTop

栈顶复用模式。这种模式下栈顶仅有一个 Activity 实例，也就是说不会有多个相同的 Activity 叠加在栈顶。如果当前任务的顶部就是待启动的 Activity 实例，那么并不会再创建一个新的 Activity 实例，而是仅仅调用已有实例的 `onNewIntent()` 方法，所以对于要以 singleTop 启动的Activity，需要处理 `onCreate()` 和 `onNewIntent()` 这两种情况下的启动参数。
但是如果任务中已经有这个 Activity 实例，但是不在栈顶，还是会重新创建一个新的 Activity 实例。

### singleTask

栈内复用模式，也是单实例模式。该模式下该 Activity 实例仅有一个 Task 存在。但是该 Task 中可能会有多个 Activity。
系统在该类型的 Task 不存在时创建一个新的 Task，并将该 Activity 放入 Task 底部。具体启动 singleTask 会不会新建一个 Task，则和 TaskAffinity 有关。在我们不设置 `android:taskAffinity` 的情况下，同一个应用的 Activity 默认具有相同的 TaskAffinity，除非你自己设置了 Activity 的 `android:taskAffinity`。
以 A 启动 B（singleTask）为例，下面我们分集中情况来介绍：

 - A 和 B 的 TaskAffinity 相同，该 Task 里面没有 B：那么就直接在 A 的 Task 里面启动 B。
 - A 和 B 的 TaskAffinity 相同，该 Task 里面存在 B：清除该 Activity 上面的所有对象，把该 Activity 推到前台。
 - A 和 B 的 TaskAffinity 不同，且 B 不存在：则创建一个新的 Task，并将该 Activity 放入 Task 底部。
 - A 和 B 的 TaskAffinity 不同，且 B 存在：在该 Task 中启动该 Activity，并把该 Task 拉回前台，如果该 Activity 上面有其他的对象，则清除该 Activity 上面的所有对象，把该 Activity 推到前台。

也就是说，启动模式为 singleTask 的 Activity 在系统中只会存在一个实例。
如果在栈中已经有该Activity的实例，就重用该实例(会调用实例的 `onNewIntent()`)。重用时，会让该实例回到栈顶，因此在它上面的实例将会被移除栈。
但是要注意，系统可能会随时杀掉后台运行的 Activity，如果这一切发生，那么系统就会调用 `onCreate()` 方法，而不调用 `onNewIntent()` 方法。，因此，如果有需要在 `Activity` 启动时处理的方法最好在 `onCreate() `和 `onNewIntent()` 方法中都要调用一下。
我们其实也注意到，Intent Flag 中 有 FLAG_ACTIVITY_NEW_TASK，singleTask 的作用相当于 FLAG_ACTIVITY_CLEAR_TOP + FLAG_ACTIVITY_NEW_TASK。

### singleInstance

单实例模式，也是增强型 singleTask 模式。该模式下，系统中只允许有一个存在该 Activity 的 Task 存在，而且该 Task 中只允许有该 Activity 一个实例存在。

 - Activity 不存在：总是新建一个 Task，并且创建一个 Activity。
 - Activity 存在：则会重用该 Activity。

这里可能会有个疑问，如果 A 和 B （singleInstance）的 TaskAffinity 相同，A 启动 B 时的情况是怎么样的呢？我们来做一个测试：
看一下 `adb shell dumpsys activity` 的打印：
启动 B 前：

```
    Task id #356
      TaskRecord{bbe4f0d #356 A=com.example.heqiang.testsomething U=0 sz=2}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #1: ActivityRecord{eeae154 u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t356}
          Intent { cmp=com.example.heqiang.testsomething/.launchFlag.ActivityA }
          ProcessRecord{66c150e 15760:com.example.heqiang.testsomething/u0a36}
        Hist #0: ActivityRecord{d6950cc u0 com.example.heqiang.testsomething/.MainActivity t356}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{66c150e 15760:com.example.heqiang.testsomething/u0a36}
```

该 Task 里面有 MainActivity 和 A。
A 启动 B：

```
    Task id #361
      TaskRecord{378cfdb #361 A=com.example.heqiang.testsomething U=0 sz=1}
      Intent { flg=0x10000000 cmp=com.example.heqiang.testsomething/.launchFlag.ActivityB }
        Hist #0: ActivityRecord{9e32cdf u0 com.example.heqiang.testsomething/.launchFlag.ActivityB t361}
          Intent { flg=0x10000000 cmp=com.example.heqiang.testsomething/.launchFlag.ActivityB }
          ProcessRecord{f103a35 16820:com.example.heqiang.testsomething/u0a36}
    Task id #360
      TaskRecord{99a446c #360 A=com.example.heqiang.testsomething U=0 sz=2}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #1: ActivityRecord{c51aa5 u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t360}
          Intent { cmp=com.example.heqiang.testsomething/.launchFlag.ActivityA }
          ProcessRecord{f103a35 16820:com.example.heqiang.testsomething/u0a36}
        Hist #0: ActivityRecord{d2364a7 u0 com.example.heqiang.testsomething/.MainActivity t360}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{f103a35 16820:com.example.heqiang.testsomething/u0a36}
```

可以看到，新建了一个 Task 用来管理 ActivityB。印证了上面的总结，Activity 不存在时总是新建一个 Task，并且创建一个 Activity。

### 区别

 - standard 和 singleTop：这两种模式下 Activity 在系统中可以有多个实例。如果栈顶已经存在待启动 Activity 的实例，singleTop 会利用已有的实例，而 standard 仍然会新建一个。 然而，singleTop 只对当前应用启动有效，对于跨应用启动的情况，singleTop 与 standard 模式没有区别。在跨应用启动这种情况下，Intent 并不会去寻找已有任务的 Activity，而是直接创建一个新的 Activity 实例：在 Android L 之前，将 Activity 实例置于发起者的任务顶，在 Android L 之后，将 Activity 实例置于新任务的根部。
 - singleTask 和 singleInstance 模式的 Activity 在系统中都只有一个实例（当然也就只有一个该 Activity 的Task），但是 singleTask 模式的 Task 中允许有别的 Activity 存在，而 singleInstance 模式的 Task 中只有该 Activity 一个实例，不允许有其他 Activity 存在。

## LaunchMode 和 taskAffinity 的关系

### standard 模式下指定 taskAffinity

A (MainActivity) 以 standard 模式启动 B(OtherTestActivity)，为 B 指定一个不同于 A 的 taskAffinity。

```
    Task id #21330
    mFullscreen=true
    mBounds=null
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
      TaskRecord{ff958fa #21330 A=com.example.heqiang.testsomething U=0 StackId=1 sz=2}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #1: ActivityRecord{967b73e u0 com.example.heqiang.testsomething/.commontest.OtherTestActivity t21330}
          Intent { cmp=com.example.heqiang.testsomething/.commontest.OtherTestActivity }
          ProcessRecord{1a33eab 14060:com.example.heqiang.testsomething/u0a172}
        Hist #0: ActivityRecord{daf4dea u0 com.example.heqiang.testsomething/.MainActivity t21330}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{1a33eab 14060:com.example.heqiang.testsomething/u0a172}

```

可以看到，他们仍然在同一个 Task 中，这个 Task 为 A Activity 所在的 Task。
因此在 standard 模式下设置 taskAffinity 好像没什么作用。

### singleTask 模式下指定 taskAffinity

这种模式下启动和 taskAffinity 的关系上面已经介绍了一些普通的情况，这里不再介绍。
下面来看一个不同的 Activity 指定同一个 taskAffinity 的情况。
我们先来看一下同一个应用中的情况。首先，这两个 Activity 没有指定特殊进程，他们运行在一个进程中。
在 A（MainActivity） 中以 singleTask 启动同一个 taskAffinity （但不同于A）的 B（OtherTestActivity） 和 C（EventActivity） 。

```
    Task id #21332
    mFullscreen=true
    mBounds=null
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
      TaskRecord{a989fd7 #21332 A=com.hq.test U=0 StackId=1 sz=2}
      Intent { flg=0x10000000 cmp=com.example.heqiang.testsomething/.commontest.OtherTestActivity }
        Hist #1: ActivityRecord{6c49b46 u0 com.example.heqiang.testsomething/.event.EventActivity t21332}
          Intent { flg=0x10400000 cmp=com.example.heqiang.testsomething/.event.EventActivity }
          ProcessRecord{64a0ee2 22782:com.example.heqiang.testsomething/u0a172}
        Hist #0: ActivityRecord{ce7925e u0 com.example.heqiang.testsomething/.commontest.OtherTestActivity t21332}
          Intent { flg=0x10000000 cmp=com.example.heqiang.testsomething/.commontest.OtherTestActivity }
          ProcessRecord{64a0ee2 22782:com.example.heqiang.testsomething/u0a172}
    Task id #21331
    mFullscreen=true
    mBounds=null
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
      TaskRecord{aafbad #21331 A=com.example.heqiang.testsomething U=0 StackId=1 sz=1}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #0: ActivityRecord{f944c08 u0 com.example.heqiang.testsomething/.MainActivity t21331}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{64a0ee2 22782:com.example.heqiang.testsomething/u0a172}

```

可以看到，这种情况下，B 和 C 在同一个 Task中，而且在同一个进程中。
不在同一个进程的情况这里就不再测试了， B 和 C 这时候仍在 在同一个 Task 中。也就是说 Task 和 进程没有联系。

下面来说明一种情况，就是 A 应用和 B 应用依赖了SDK包，他们以 singleTask 模式启动 SDK 中的同一个 Activity C，这又该什么情况呢？
其实这种情况只要你理解了下面的事实，那么 LaunchMode 、taskAffinity 以及 Task 的关系就很容易理解了。
也就是虽然 SDK 中的 C 从代码层面上讲是同一个，但是对于A 应用和 B 应用来说，调用的确实两个不同的 Activity ，因为它们的包名不同，比如：

 - `com.hq.test.sdkdemo/com.hq.sdk.TestActivity`
 - `com.example.heqiang.testsomething/com.hq.sdk.TestActivity`

可以下面的信息：

```
    Task id #21313
    mFullscreen=true
    mBounds=null
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
      TaskRecord{6cf314a #21313 A=com.hq.test U=0 StackId=1 sz=2}
      Intent { flg=0x10000000 cmp=com.example.heqiang.testsomething/com.hq.sdk.TestActivity }
        Hist #1: ActivityRecord{88936eb u0 com.hq.test.sdkdemo/com.hq.sdk.TestActivity t21313}
          Intent { flg=0x10400000 cmp=com.hq.test.sdkdemo/com.hq.sdk.TestActivity }
          ProcessRecord{7571816 5028:com.hq.test.sdkdemo/u0a137}
        Hist #0: ActivityRecord{dc4f387 u0 com.example.heqiang.testsomething/com.hq.sdk.TestActivity t21313}
          Intent { flg=0x10000000 cmp=com.example.heqiang.testsomething/com.hq.sdk.TestActivity }
          ProcessRecord{702597 3847:com.example.heqiang.testsomething/u0a172}
    Task id #21315
    mFullscreen=true
    mBounds=null
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
      TaskRecord{751d1d8 #21315 A=com.hq.test.sdkdemo U=0 StackId=1 sz=1}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.hq.test.sdkdemo/.MainActivity }
        Hist #0: ActivityRecord{7b3c6a6 u0 com.hq.test.sdkdemo/.MainActivity t21315}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.hq.test.sdkdemo/.MainActivity }
          ProcessRecord{7571816 5028:com.hq.test.sdkdemo/u0a137}
    Task id #21312
    mFullscreen=true
    mBounds=null
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
      TaskRecord{d65931 #21312 A=com.example.heqiang.testsomething U=0 StackId=1 sz=1}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #0: ActivityRecord{85b37c0 u0 com.example.heqiang.testsomething/.MainActivity t21312}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{702597 3847:com.example.heqiang.testsomething/u0a172}

```

在 com.hq.test Task中，有两个 `TestActivity`，他们属于不同进程。


## Intent Flag

### FLAG_ACTIVITY_NEW_TASK

当使用这个 Flag 时，
Task 中已经有A（standard）和B，现在B以 `FLAG_ACTIVITY_NEW_TASK` 的方式来启动A（standard），先来看一下 `android:taskAffinity` 相同的情况下：
启动前：
```
  Stack #2:
    Task id #454
      TaskRecord{97815d0 #454 A=com.example.heqiang.testsomething U=0 sz=3}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #2: ActivityRecord{196a583 u0 com.example.heqiang.testsomething/.launchFlag.ActivityB t454}
          Intent { cmp=com.example.heqiang.testsomething/.launchFlag.ActivityB }
          ProcessRecord{da6d9c9 6104:com.example.heqiang.testsomething/u0a123}
        Hist #1: ActivityRecord{d3a4985 u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t454}
          Intent { cmp=com.example.heqiang.testsomething/.launchFlag.ActivityA }
          ProcessRecord{da6d9c9 6104:com.example.heqiang.testsomething/u0a123}
        Hist #0: ActivityRecord{4ce90ca u0 com.example.heqiang.testsomething/.MainActivity t454}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{da6d9c9 6104:com.example.heqiang.testsomething/u0a123}

    Running activities (most recent first):
      TaskRecord{97815d0 #454 A=com.example.heqiang.testsomething U=0 sz=3}
        Run #2: ActivityRecord{196a583 u0 com.example.heqiang.testsomething/.launchFlag.ActivityB t454}
        Run #1: ActivityRecord{d3a4985 u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t454}
        Run #0: ActivityRecord{4ce90ca u0 com.example.heqiang.testsomething/.MainActivity t454}

    mResumedActivity: ActivityRecord{196a583 u0 com.example.heqiang.testsomething/.launchFlag.ActivityB t454}
```
启动后：
```
  Stack #2:
    Task id #454
      TaskRecord{97815d0 #454 A=com.example.heqiang.testsomething U=0 sz=4}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #3: ActivityRecord{367178a u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t454}
          Intent { flg=0x10000000 cmp=com.example.heqiang.testsomething/.launchFlag.ActivityA }
          ProcessRecord{da6d9c9 6104:com.example.heqiang.testsomething/u0a123}
        Hist #2: ActivityRecord{196a583 u0 com.example.heqiang.testsomething/.launchFlag.ActivityB t454}
          Intent { cmp=com.example.heqiang.testsomething/.launchFlag.ActivityB }
          ProcessRecord{da6d9c9 6104:com.example.heqiang.testsomething/u0a123}
        Hist #1: ActivityRecord{d3a4985 u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t454}
          Intent { cmp=com.example.heqiang.testsomething/.launchFlag.ActivityA }
          ProcessRecord{da6d9c9 6104:com.example.heqiang.testsomething/u0a123}
        Hist #0: ActivityRecord{4ce90ca u0 com.example.heqiang.testsomething/.MainActivity t454}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{da6d9c9 6104:com.example.heqiang.testsomething/u0a123}

    Running activities (most recent first):
      TaskRecord{97815d0 #454 A=com.example.heqiang.testsomething U=0 sz=4}
        Run #3: ActivityRecord{367178a u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t454}
        Run #2: ActivityRecord{196a583 u0 com.example.heqiang.testsomething/.launchFlag.ActivityB t454}
        Run #1: ActivityRecord{d3a4985 u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t454}
        Run #0: ActivityRecord{4ce90ca u0 com.example.heqiang.testsomething/.MainActivity t454}

    mResumedActivity: ActivityRecord{367178a u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t454}

```

我们发现会在B上面直接创建A。
再来测试另外一种情况，Task中已经存在A（standard），在另外的一个Task中的B以 `FLAG_ACTIVITY_NEW_TASK` 启动A（standard）：
启动前：

```
    Task id #458
      TaskRecord{5d0631a #458 A=com.example.heqiang.testsomething U=0 sz=2}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #1: ActivityRecord{4112d5f u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t458}
          Intent { cmp=com.example.heqiang.testsomething/.launchFlag.ActivityA }
          ProcessRecord{8084e4b 6497:com.example.heqiang.testsomething/u0a123}
        Hist #0: ActivityRecord{3a7589b u0 com.example.heqiang.testsomething/.MainActivity t458}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{8084e4b 6497:com.example.heqiang.testsomething/u0a123}

```
启动后：

```
    Task id #458
      TaskRecord{5d0631a #458 A=com.example.heqiang.testsomething U=0 sz=3}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
        Hist #2: ActivityRecord{569c2ed u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t458}
          Intent { flg=0x10400000 cmp=com.example.heqiang.testsomething/.launchFlag.ActivityA }
          ProcessRecord{8084e4b 6497:com.example.heqiang.testsomething/u0a123}
        Hist #1: ActivityRecord{4112d5f u0 com.example.heqiang.testsomething/.launchFlag.ActivityA t458}
          Intent { cmp=com.example.heqiang.testsomething/.launchFlag.ActivityA }
          ProcessRecord{8084e4b 6497:com.example.heqiang.testsomething/u0a123}
        Hist #0: ActivityRecord{3a7589b u0 com.example.heqiang.testsomething/.MainActivity t458}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.example.heqiang.testsomething/.MainActivity }
          ProcessRecord{8084e4b 6497:com.example.heqiang.testsomething/u0a123}

```
可以看到在A上面又新创建了个A。
当然这里只是以A是 standard 的启动模式来做介绍，具体其他的启动模式情况肯定是不一样的。
因此可以得到下面的结论：FLAG_ACTIVITY_NEW_TASK 只会去关心该 `Activity` 的 Task 的个数，如果不存在就新建，存在就直接推到前台。而不去关心 `Activity` 的个数（和启动模式有关），也就是说该 Task 中可以有多个 Activity 存在，这个要和 `singleTask` 做区别。

### FLAG_ACTIVITY_CLEAR_TOP

启动模式不同，具体操作也不一样。但是都会清除该 Activity 上面的所有对象。
standard 模式：系统收到新的 Intent 时，如果 Task 中存在该 Activity 则弹出 Activity 和 Activity 上面的所有对象，并重新初始化一个新的 Activity 对象放入栈顶，以处理 Intent 请求。因为 standard 总是初始化新的 Activity 来处理 Intent请求。
其他模式：则会将 Activity 上面的对象弹出，使该 Activity 在栈顶部，以处理 Intent请求。

### FLAG_ACTIVITY_CLEAR_TASK

启动 Activity 时如果加入了这个标志，那么如果存放该 Activity 的 Task 已经存在，会先把该 Task 里面所有的 Activity 清空，然后再启动该 Activity。那么该 Activity 就成了该 Task 里面唯一的一个 Activity。
应用场景：如果在同一 Task 里面 A 启动 B，B 启动 C，但是在 C 里面后退的话会默认返回 B，然后再返回 A，如果不想返回 B 和 A，那么就可以用该 Flag，启动 C 时把 A 和 B 清除掉，比如在登陆界面跳转到到主界面。从主界面后退肯定不想返回登陆界面的。

### FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS

通过这个 Flag 启动的 Activity 将不会添加到最近使用应用列表中，我们从任务管理器中无法找到这个 Activity。

### FLAG_ACTIVITY_NO_HISTORY

通过这个 Flag 启动的 Activity 一旦退出后，就不会存在栈中。比如 A 添加这个 Flag 启动 B，B启动 C，这是栈中就只有 A、C。

### FLAG_ACTIVITY_LAUNCHED_FROM_HISTORY

当从手机任务管理器启动的Activity Intent 的 Flag 会带有这个属性，我们可以用它来区分是否从任务管理器里面启动的。

## LaunchMode 与 StartActivityForResult 的关系

<!--   

-->
