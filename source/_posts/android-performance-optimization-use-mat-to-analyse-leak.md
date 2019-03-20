---
title: Android 性能优化工具篇 -- MAT分析内存泄漏
categories: Android性能优化
comments: true
tags: [Android性能优化, 开发工具]
description: 介绍Android分析内存泄漏工具的使用方法
date: 2016-2-20 10:00:00
---

MAT 是 Memory Analyzer Tool 的简称，它是一款强大的内存分析工具，使用它能帮助开发者快速分析内存泄漏以及优化内存的使用。
内存泄漏也是我们开发过程中经常碰到的问题，掌握了MAT工具，那么你就不会惧怕内存泄漏，使用它可以让内存泄漏无所遁形。

## MAT下载

进入[网址](https://www.eclipse.org/mat/)下载MAT工具，如果你使用 Eclipse 开发工具而且已经集成了插件，可以不用下载了。

## 场景准备

我们用下面的代码产生一个内存泄漏的场景：
MainActivity.java

```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if(mInnerClassInstance == null){
            mInnerClassInstance = new InnerClass();
        }
        mInnerClassInstance.testFunc();
    }


    public static InnerClass mInnerClassInstance;
    class InnerClass{
        public void testFunc(){
            Log.e("InnerClass", "InnerClass.testFunc()");
        }
    }
```

我们知道，在 Java 中，非静态内部类会默认隐性引用外部类对象。而上面的例子中的静态变量`mInnerClassInstance`在第一个`MainActivity`实例创建后便会一直存在，那么它就会一直持有`MainActivity`的一个引用，在`MainActivity`实例销毁后它是无法被回收的，因此便造成了内存泄漏。

## 内存泄漏的初步分析
首先可以用`adb shell dumpsys meminfo <包名>`先进行初步的分析。
我们按下手机的后退键回到桌面，这时会调用`MainActivity`的`onDestroy()`，正常情况下`MainActivity`实例会被回收。
```
$ adb shell dumpsys meminfo com.example.hq.testsomething

Applications Memory Usage (kB):
Uptime: 23692543 Realtime: 23692543

** MEMINFO in pid 29474 [com.example.hq.testsomething] **
                   Pss  Private  Private  Swapped     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap     5252     4568        0     9400    32768    30309     2458
  Dalvik Heap     1761     1692        0      320    11829     7872     3957
 Dalvik Other      467      396        0     2848
        Stack      200      200        0        0                           
       Ashmem        2        0        0        0
    Other dev        4        0        4        0
     .so mmap     1047      156       12     2184
    .apk mmap      160        0       72        0
    .ttf mmap       33        0       16        0
    .dex mmap      268        4      264        0
    .oat mmap     1172        0      388        4
    .art mmap     1434      492      424       80
   Other mmap      131        8       28        4
      Unknown      188      188        0       96
        TOTAL    12119     7704     1208    14936    44597    38181     6415
 
 App Summary
                       Pss(KB)
                        ------
           Java Heap:     2608
         Native Heap:     4568
                Code:      912
               Stack:      200
            Graphics:        0
       Private Other:      624
              System:     3207
 
               TOTAL:    12119      TOTAL SWAP (KB):    14936
 
 Objects
               Views:       26         ViewRootImpl:        0
         AppContexts:        2           Activities:        1
              Assets:        3        AssetManagers:        2
       Local Binders:       11        Proxy Binders:       15
       Parcel memory:        3         Parcel count:       12
    Death Recipients:        1      OpenSSL Sockets:        0
 
 Dalvik
         isLargeHeap:    false
 
 SQL
         MEMORY_USED:        0
  PAGECACHE_OVERFLOW:        0          MALLOC_SIZE:        0

```

我们通过这个结果查看发现`Objects`这一项里面的`Activities`为1,证明还是有个`Activity`的实例存在的，没有被正常回收。下面我们来借助于MAT工具来分析一下。

## 使用Android Monitor分析内存泄漏

Android Studio 中的 Android Monitor 里面有个 Memory 控制台，它提供了一个内存监视器，我们可以通过它方便地查看应用程序的性能和内存使用情况，从而也就可以找到需要释放对象，查找内存泄漏等。

点击图中的按钮可以触发一次GC

![效果图](/images/development-tool-mat-to-analyse-leak/android_monitor_init_gc.png)

点击图中的按钮可以生成hprof文件来进行分析。

![效果图](/images/development-tool-mat-to-analyse-leak/android_monitor_dump_heap.png)

这里我们不再详细介绍。

## 生成HPROF文件

### 使用Android Studio
首先我们借助Android Studio生成hprof文件，Tools -> Android -> Android Device Monitor 打开 DDMS：

![效果图](/images/development-tool-mat-to-analyse-leak/mat_open_device_monitor.png)

点击一下我们需要调试的进程，此时进程上访的一排工具处于可点击状态，点击一下 Update Heap 按钮可以在Heap视图里面现实当前堆内存的使用情况，点击 Cause GC 按钮可以触发一次GC操作。

![效果图](/images/development-tool-mat-to-analyse-leak/mat_update_heap.png)

按回退键回到桌面，然后点击 Dump HPROF file 按钮，如图：

![效果图](/images/development-tool-mat-to-analyse-leak/mat_dump_hprof_file.png)

会生成 hprof 文件，会弹框提示我们保存下来。
然后打开MAT工具，File --> Open Heap Dump...，选择刚刚生成的文件，如果出现下面的错误，用 `hprof-conv` 命令转换一下就可以了。这是因为MAT是用来分析java程序的hprof文件的，与Android导出的hprof有一定的格式区别，因此我们需要把导出的hprof文件转换一下。`hprof-conv` 是 Android SDK 提供的工具，它位于 Android SDK 的platform-tools目录下。

![效果图](/images/development-tool-mat-to-analyse-leak/mat_open_error.png)

```
android-sdk-linux/platform-tools/hprof-conv com.example.hq.testsomething.hprof test.hprof
```

### 使用Eclipse

如果你的 Eclipse 安装了MAT插件，那么安装上面使用Android Studio生成hprof文件的步骤，点击 Dump HPROF file 按钮后，就直接打开了该hprof文件。

### 使用代码

```java
android.os.Debug.dumpHprofData(hprofName);
```
生成的hprof文件也需要用`hprof-conv`转换一下才能用MAT工具打开。

## 通过MAT工具分析内存泄漏

### 介绍

打开hprof文件后首先进入主视图，下来看一下这个主视图：

![效果图](/images/development-tool-mat-to-analyse-leak/mat_overview.png)

MAT 提供了很多功能，但是最常用的只有 Histogram 和 Domintor Tree：

 - Histogram：通过 Histogram 可以直观地看出内存中不同类型的 buffer 的数量和占用内存的大小。
 - Domintor Tree：把内存中的对象按照从大到小的顺序进行排序，并且可以分析对象之间的引用关系

我们看到在这两个视图中还会发现有 Shallow size 和 Retained Size 两个属性：

 - Shallow size：对象自身占用的内存大小，不包括它引用的对象。针对非数组类型的对象，它的大小就是对象与它所有的成员变量大小的总和。
当然这里面还会包括一些java语言特性的数据存储单元。针对数组类型的对象，它的大小是数组元素对象的大小总和。
 - Retained Size：当前对象大小+当前对象可直接或间接引用到的对象的大小总和。(间接引用的含义：A->B->C, C就是间接引用)

一个对象到GC Roots的引用链被称为Path to GC Roots，通过分析Path to GC Roots可以找出JAVA的内存泄露问题。

### 内存泄漏分析

我们在主视图中选择`Actions`-->`Domintor Tree`或者`Histogram`也可以。
因为在前面我们已经初步分析了`MainActivity`是存在内存泄漏的，这里我们可以使用界面中的搜索功能，在输入框中输入`MainActivity`进行过滤，查看当前存在的`MainActivity`对象，我们发现当前有两个`MainActivity`对象，这是因为我们按back键退出应用再进来，系统都会重新创建一个新的`MainActivity`，但是第一次创建的老的对象却无法回收，所以就出现了两个`MainActivity`对象。

![效果图](/images/development-tool-mat-to-analyse-leak/mat_activity_leak.png)

我们在进一步分析一下是什么导致了内存泄漏，单击选中`MainActivity`，然后单击鼠标右键->Path To GC Roots->exclude all phontom/weak/soft etc. references，如图所示。

![效果图](/images/development-tool-mat-to-analyse-leak/mat_path_to_gc_root_exclude_phontom_soft_weak.png)

这里在 Path To GC Roots 之所以选择排除所有的虚引用、弱引用和软引用，是因为它们都有很大的几率被GC回收掉的，它们并不会构成内存泄漏。
可以看到是 `mInnerClassInstance` 引用了 `MainActivity` 导致了无法释放。

### Domintor Tree 视图分析

一般分析内存泄漏我们需要分析 Domintor Tree 视图，但是在视图中内存泄漏一般不会直接显示出来，这个时候我们一般要需要按照内存的从大到小去排查一遍。

![效果图](/images/development-tool-mat-to-analyse-leak/mat_domintor_tree_analysis.png)

在图中我们就发现第2列中就有个`BitmapDrawable`对象，同样我们选择`BitmapDrawable`对象，然后单击鼠标右键->Path To GC Roots->exclude all phontom/weak/soft etc. references，如图所示。

![效果图](/images/development-tool-mat-to-analyse-leak/mat_bitmap_leak_result.png)

可以看到同样是 `mInnerClassInstance` 引用了 `MainActivity` 导致了`BitmapDrawable`对象无法释放。


