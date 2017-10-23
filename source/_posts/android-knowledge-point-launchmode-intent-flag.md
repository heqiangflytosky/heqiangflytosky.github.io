
---
title: Android LaunchMode 与 Intent Flag 详解
categories: Android
comments: true
tags: [Android, LaunchMode, Intent Flag]
description: 介绍 Android LaunchMode 与 Intent Flag
date: 2015-12-8 10:00:00
---

## LaunchMode

### standard

如果不显式的指定 launchMode，Activity都会按照此种模式启动。这种模式下的 Activity 可以有多个实例，不同任务中可以有不同的 Activity 实例，同一个任务中也可以有多个 Activity 实例。
Android L(Android 5.0)前后对这种启动模式的处理方式是不同的：
 - Android L之前：每次以该模式启动的 Activity 都会被压入当前任务的顶部，启动 N 次，在当前任务就会出现 N 个 Activity 的实例，每次Back键就会销毁一个，直到按了 N 次Back键。
 - Android L之后：如果要以该模式启动 Activity 都是来自同一应用，那么还是会像之前一样，压入当前任务的顶部; 如果是来自不同应用，那么将会创建一个新的任务，然后将 Activity 的实例压入新的任务中。Android L做的优化主要是针对多任务的显示。

### singleTop

这种模式下仅有一个栈顶的 Activity 实例，如果当前任务的顶部就是待启动的 Activity 实例，那么并不会再创建一个新的 Activity 实例，而是仅仅调用已有实例的 `onNewIntent()` 方法，所以对于要以 singleTop 启动的Activity，需要处理 `onCreate()` 和 `onNewIntent()` 这两种情况下的启动参数。

### singleTask

具体启动 singleTask 会不会新建一个 Task，则和 TaskAffinity 有关。在我们不设置 `android:taskAffinity` 的情况下，同一个应用的 Activity 默认具有相同的 TaskAffinity，除非你自己设置了 Activity 的 `android:taskAffinity`。
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

该模式下，系统中只允许有一个存在该 Activity 的 Task 存在，而且该 Task 中只允许有该 Activity 一个实例存在。

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

 - standard 和 singleTop：如果栈顶已经存在待启动 Activity 的实例，singleTop 会利用已有的实例，而 standard 仍然会新建一个。 然而，singleTop 只对当前应用启动有效，对于跨应用启动的情况，singleTop 与 standard 模式没有区别。在跨应用启动这种情况下，Intent 并不会去寻找已有任务的 Activity，而是直接创建一个新的 Activity 实例：在 Android L 之前，将 Activity 实例置于发起者的任务顶，在 Android L 之后，将 Activity 实例置于新任务的根部。
 - singleTask 和 singleInstance 模式的 Activity 在系统中都只有一个实例（当然也就只有一个该 Activity 的Task），但是 singleTask 模式的 Task 中允许有别的 Activity 存在，而 singleInstance 模式的 Task 中只有该 Activity 一个实例，不允许有其他 Activity 存在。

## Intent Flag

### FLAG_ACTIVITY_NEW_TASK

当使用这个 Flag 时，
Task 中已经有A和B，现在B以 `FLAG_ACTIVITY_NEW_TASK` 的方式来启动A，先来看一下 `android:taskAffinity` 相同的情况下：
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
再来测试另外一种情况，Task中已经存在A，在另外的一个Task中的B以 `FLAG_ACTIVITY_NEW_TASK` 启动A：
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
因此可以得到下面的结论：FLAG_ACTIVITY_NEW_TASK 只会去关心该 `Activity` 的 Task 的个数，如果不存在就新建，存在就直接推到前台。而不去关心 `Activity` 的数量，也就是说该 Task 中可以有多个 Activity 存在，这个要和 `singleTask` 做区别。

<!--   
http://duanqz.github.io/2016-01-21-Activity-LaunchMode
http://www.jianshu.com/p/7f1c9fac2af2
http://www.jianshu.com/p/c1386015856a
https://www.2cto.com/kf/201508/436663.html
http://blog.csdn.net/vipzjyno1/article/details/25463457
http://www.jianshu.com/p/537aa221eec4
http://blog.csdn.net/lincyang/article/details/6802017

http://blog.csdn.net/qq_21763489/article/details/52875739
FLAG_ACTIVITY_PREVIOUS_IS_TOP

taskAffinity 和 FLAG_ACTIVITY_NEW_TASK
https://www.baidu.com/s?word=FLAG_ACTIVITY_NEW_TASK&tn=sitehao123&ie=utf-8
http://www.cnblogs.com/xiaoQLu/archive/2012/07/17/2595294.html
http://blog.csdn.net/zhangjg_blog/article/details/10923643


Intent.FLAG_ACTIVITY_TASK_ON_HOME
https://www.baidu.com/s?word=FLAG_ACTIVITY_TASK_ON_HOME&tn=50000021_hao_pg&ie=utf-8&sc=UWd1pgw-pA7EnHc1FMfqnHRsPWRvnWndn1DLriuW5y99U1Dznzu9m1YYPWnknHbLrHn&ssl_sample=s_4%2Cs_30&srcqid=1348961502141319706
http://blog.csdn.net/new_abc/article/details/13737331

http://blog.csdn.net/ljz2009y/article/details/26621815
blog.csdn.net/lincyang/article/details/6826021
http://blog.csdn.net/goodlixueyong/article/details/49620667
-->
