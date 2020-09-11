---
title: Android 性能优化实战篇 -- 内存优化
categories: Android性能优化
comments: true
tags: [内存优化]
description: 介绍内存优化的一些方法
date: 2018-10-8 10:00:00
---

## 概述

内存在任何软件开发环境中都是及其宝贵的资源，在物理内存有限的移动设备上更是如此。虽然 Java 的虚拟机扮演了常规的垃圾回收角色，但这并不意味着可以忽视内存的使用。内存的不合理使用对 UI 卡顿、耗电、稳定性都有不同程度的影响。因此内存优化在我们开发过程中一直是性能优化的重点。
关于内存的优化应该贯穿于我们软件开发的始终，不应该是做完一次之后不再关注，这样才能开发中高性能的软件。
内存的优化也是有一定的规律可循的，包括Android官方和Android的开发者都总结出了一些技巧和建议，只要我们在编程过程中注意这些问题或者遵循这些技巧，就能开发出在内存方面表现不错的软件。

[Android 官方文档：Manage your app's memory](https://developer.android.com/topic/performance/memory)

## 内存优化的意义

Java 虚拟机已经有了完善的垃圾回收体系，那为什么我们还要在内存上下功夫呢？
应用在使用过程中随着内存的增加，当达到一定的条件后，GC开始工作，为新的对象释放出更多的内存。但是在GC的过程中，任何其他在工作的线程会暂停，包括负责绘制的UI线程。如果GC过于频繁而且持续时间过长，就会导致卡顿，影响应用的流畅性。
如果在内存峰值是申请一块过大的内存，可能会导致堆内存不足而导致OOM，影响系统稳定性。
另外，内存泄漏也会导致内存的增加，从而可能导致OOM。
在应用程序后台运行时，特别是服务进程，即使通过一系列的方法提高进程的优先级，但是如果内存占用过高，在系统资源紧张的情况下，还是会被系统kill。
综上，总结出内存优化的意义：

 - 减少卡顿，提高应用流畅度
 - 减少 OOM，提高应用稳定性
 - 减少内存占用，提供应用后台运行存活率。
 - 减少内存泄漏，提高流畅度和稳定性

## 内存优化步骤

 - 检查：通过一系列的工具来掌握应用的内存使用情况。编码过程中通过运行lint检查工具发现可能影响内存性能的地方。运行过程中使用 Memory Monitor和 HeapViewer 可以很好地帮助我们查看程序的内存使用情况，可以发现内存峰值和内存抖动，一旦出现这些问题，可以进一步使用 Allocation Tracker 和 MAT 详细分析内存，找到内存占用大的对象，然后进行优化。
 - 监控：可以借助一些监控工具来实时监测内存使用异常，比如内存泄漏监测工具 LeakCanary。
 - 优化：优化分两个方法，一个是平时我们编码是要养成良好的素养和习惯。二是发现问题后的问题优化。主要包括节省内存开销和避免内存泄漏两个目标。

运行 lint 结果中关注 `Android Lint:Performance` 中的警告项可以帮助我们进一步优化我们的代码。Android Studio 中菜单项 Analyze -> Inspect Code 可以启动静态代码检查。
另外介绍一个查看内存使用情况的命令行：

```
adb shell dumpsys meminfo -a <进程名称>
```

## 内存优化实践

注意从三个方面来进行内存优化：

 - 避免内存泄漏
 - 提高代码质量，优化内存空间
 - 优化内存使用策略

### 1.避免内存泄漏

关于内存泄漏的问题，会在另外一篇博客中专门介绍。

### 2.优化内存空间

#### a.合理使用对象引用

针对强引用、软引用、弱引用和虚引用，根据业务需求合理使用不同引用，以提高内存的使用效率。

#### b.减少不必要的内存开销

##### 自动装箱

当原始类型自动转换为包装类型时，都会产生一个新的对象，这些对象比基础数据类型对象要大，这样就会产生更多的内存和性能开销。比如一个基础整型（int）对象只有4个字节，而Integer对象有16字节，因此在内存和时间性能上都有额外的开销。
因此，在写程序时需要避免这里情况。特别是HashMap这类的容器。

##### 内存复用

在 Android 系统中，有些数据是可以复用的，通过内存复用可以有效减少应用的内存开销。

 - 有效利用系统自带的资源： Android 系统本身内置了大量资源，比如一些通用的字符串、颜色定义、常用图片，还有些动画和页面样式以及简单布局，这些资源应用可以直接引用。这样不仅可以在一定程度上较少内存开销，还可以有效减少APK大小。
 - 视图复用：如果出现了大量重复子组件，而子组件是大量重复的，可以使用 ViewHolder 实现 ConvertView 复用，这也基本是 ListView 和 GridView 的做法。
 - 对象池：通过程序实现一些复用逻辑，对相同数据类型使用同一块内存区域，也可以利用系统框架的具有复用特性的组件减少对象的重复创建，从而减少内存的分配和回收。比如线程池。
 - Bitmap 对象复用：利用 Bitmap 中的 inBitmap 高级特性，提高 Android 系统在 Bitmap 的分配与释放效率。

##### StringBuilder

字符串拼接时使用 StringBuilder 来代替频繁的 “+”。

#### c.使用最优的数据类型

Android 应用开发是基于 Java 语言编程，但是在 Java 中有很多的数据结构和类型，未必就是最省内存的数据结构，需要根据实际的开发需求，选择最好的数据类型或类类型。Android自身也做了一定的努力，针对移动开发提出了一系列的数据类容器结构优化。

##### ArrayMap、SparseArray 和 HashMap

HashMap 是我们经常用到的容器，但它并不是最节约内存的容器。
我们可以使用 Android 提供的 ArrayMap 来代替 HashMap 的使用。
ArrayMap 在执行插入或者删除操作时，从性能角度来讲是比 HashMap 稍微差一点。但是如果涉及很小的对象数，比如 1000 以下，就不需要担心这个问题了。ArrayMap 在循环遍历时会比 HashMap 更加高效。
SparseArray仅仅能存储key为int类型的数据。同一时候，SparseArray在存储和读取数据时候，使用的是二分查找法。
SparseArray比HashMap更省内存，在某些条件下性能更好，主要是由于它避免了对key的自己主动装箱（int转为Integer类型），它内部则是通过两个数组来进行数据存储的。一个存储key，另外一个存储value，为了优化性能，它内部对数据还採取了压缩的方式来表示稀疏数组的数据，从而节约内存空间。

##### 枚举类型

使用枚举类型会占用额外的内存空间，是直接定义常量的三倍以上，我可以使用注解来代替枚举。
参考 [Java 基础 -- enum ](http://www.heqiangfly.com/2013/06/26/java-basic-enum/)


##### LruCache

Android 在 support v4 包中提供了 LruCache 来保存需要缓存的对象，可以翻译为最近最少使用缓存。它内部维护一个双向链表，不支持线程安全，当其中一个值被访问时，它被放到队列的尾部，当缓存将满时，队列头部的值（也就是最近最少被访问的）被丢弃，之后可以被垃圾回收。
Android 官方也一致推荐使用 LruCache最为图片内存缓存。

#### d.图片内存优化

应用中一般有很多图片需要显示，图片占用的内存在整个应用中是非常突出的。而且图片需要申请连续的内存，如果剩余的内存碎片化比较严重，虽然剩余内存大于图片所需的内存，仍然会导致位图内存不够引发OOM。
在Android中，一张图片解码成位图后，占用的内存=图片长度 * 图片宽度 * 单位像素占用的字节数。所以我们可以从这个几个方面考虑优化。
在 Android 默认情况下，当图片解码时，会按照每个像素占32bit来处理。即红色、绿色、蓝色和透明通道各占8bit，即使是没有透明通道的JPEG格式。并且这些图片在被渲染之前，首先要被作为纹理传送到GPU，这意味着每一张图片会同时占用 CPU 和 GPU 内存。
BitmapFactory.Options 提供了一些选项，来供我们解码图片时进行优化设置。

##### 设置位图规格

Android 提供了多种位图格式，可以根据产品需求来选择。

| 格式 | 每个像素占用位数 |
|:-------------:|:-------------:|
| ARGB_8888 | 32 |
| RGB_565 | 16 |
| ARGB_4444 | 16 |
| ALPHA_8 | 8 |

最高的是 ARGB_8888，也是系统默认的位图格式，其他几种都减小了位图通道位，可以减少内存开销并提升图片显示性能。
当然这种空间的节省是要付出视觉质量受损代价的，可以根据不同的场景选择不同的规格。
Android 中通过 BitmapFactory.Options 的inPreferredConfig 属性来选择位图规格。

##### inSampleSize

当内存中图片大于屏幕显示区域或者大于指定显示区域的大小时，会造成内存的浪费。这时就可以使用 inSampleSize 实现图片的压缩功能。
当 inSampleSize 为1时就和原始图片一样大小，为2时获得只有原始大小1/2的图片，同理为4时是原始1/4大小的图片。
inSampleSize的默认值和最小值为1（当小于1时，解码器将该值当做1来处理），且在大于1时，该值只能为2的幂（当不为2的幂时，解码器会取与该值最接近的2的幂）。

##### inScaled、inDensity和inTargetDensity

虽然 inSampleSize 可以缩小图片，但是比率只能是2的幂，无法实现更加精细的操作。如果想要更细地缩放图片，就需要使用 inScaled、inDensity和inTargetDensity。
当 inScaled 为 true 时，系统会根据现有的密度来划分目标密度，会重设图片大小。
但是这些属性会使用过多的算法，导致图片显示过程需要更多的时间开销，如果图片很多的话会影响图片显示效果。
因此最佳实践是两种方案的结合：先使用 inSampleSize 处理图片，转换为接近目标的 2 次幂，然后用 inDensity和inTargetDensity 生成最终想要的准确大小。

##### inJustDecodeBounds

如果仅仅是想要得到图片的宽高信息，可以设置这个属性为true。解码 Bitmap 时可以只返回其高、宽和Mime类型，而不必为其申请内存，从而节省了内存空间。

##### inBitmap

这个属性是 Android 3.0 引进的，如果设置该属性，解码图片是会尝试重用一个已经存在的位图。这就意味着位图内存已经被重用了，从而改善性能，并且没有内存的分配和释放过程。

##### 使用更小的图片

在符合产品需求的情况下，使用小图，节省内存。

#### e.优化布局，减少内存消耗

避免过度绘制，布局越简单扁平，占用内存就越少，效率就越高。

### 3.内存使用策略优化

#### a.谨慎使用 large heap

我们知道，我们可以通过在 manifest 的 application 标签下添加 `largeHeap=true` 的属性来为应用声明一个比默认值更大的heap空间。但是，我们声明 large heap 是为了给一部分需要消耗大量 ram 的应用开辟的特殊通道（比如图库应用），只有我们明确了哪里会使用大量内存而且这些大内存是必须保留时才这样做。它并不是我们解决OOM的方法。
因此，请谨慎使用large heap属性。使用额外的内存空间会影响系统整体的用户体验，并且会使得每次gc的运行时间更长。在任务切换时，系统的性能会大打折扣。
另外, large heap并不一定能够获取到更大的heap。在某些有严格限制的机器上，large heap的大小和通常的heap size是一样的。

#### b.使用缓存时综合考虑容量

比如使用 LruCache 时，这个容量既不能太大，太大会造成其他可用内存变小，容易导致 OOM，但是也不能太小，否则起不到缓存的作用。
Google 官方文档给出以下意见作为参考：

 - 分配 LruCache 大小时考虑应用剩余内存有多大
 - 一次屏幕显示多少张图片，有多少张图片是缓存起来准备显示的
 - 考虑设备的分辨率和尺寸，缓存相同的图片数，手机dpi越大，需要内存也大
 - 图片分辨率和像素质量决定了占用内存的大小
 - 图片访问的频繁程度是多少，如果存在多个不同要求的图片类型，可以考虑用多个 LruCache 来做缓存，按照访问的频率分配到不同的 LruCache 中。

#### c.内存紧张时释放内存

在系统内存紧张时，Android 系统提供了回调方法（onLowMemory()与onTrimMemory()）来通知应用清理自己的不必要内存，来避免被 kill 掉。

 - onLowMemory()：当系统内存不足时，所有的后台应用（优先级为background的进程，不是指后台运行的进程）被kill之后，应用会收到这个回调。
 - onTrimMemory()：Android 4.0 (API level 14)加的回调。系统会根据不同的内存状态来回调。OnTrimMemory 有个int型的参数，代表不同的内存状态： TRIM_MEMORY_RUNNING_MODERATE、TRIM_MEMORY_RUNNING_LOW、TRIM_MEMORY_RUNNING_CRITICAL、TRIM_MEMORY_UI_HIDDEN、TRIM_MEMORY_BACKGROUND、TRIM_MEMORY_MODERATE 和 TRIM_MEMORY_COMPLETE。前三个状态是应用程序正在运行时收到的级别，TRIM_MEMORY_UI_HIDDEN 是应用不可见时的级别，后面三个是应用程序被缓存时（后台进程）时的级别。

关于 onLowMemory() 和 onTrimMemory() 系统在 Application、Activity、Fragement、Service 和 ContentProvider中都提供了回调，只需要重写这些方法做处理就行，另外还提供了API ComponentCallbacks（onLowMemory） 和 ComponentCallbacks2 （onTrimMemory），通过Context.registerComponentCallbacks ()在合适的时候注册回调就可以了。

通常在我们代码实现了onTrimMemory后很难复显这种内存消耗场景，Android 系统为我们提供了命令来模拟触发该水平内存释放，如下命令：

```
adb shell dumpsys gfxinfo <package_name> -cmd trim <value>
```

packagename为包名或者进程id，value为ComponentCallbacks2.java里面定义的值，可以为80、60、40、20、5等，我们模拟触发其中的等级即可。

```
    static final int TRIM_MEMORY_COMPLETE = 80;
    static final int TRIM_MEMORY_MODERATE = 60;
    static final int TRIM_MEMORY_BACKGROUND = 40;
    static final int TRIM_MEMORY_UI_HIDDEN = 20;
    static final int TRIM_MEMORY_RUNNING_CRITICAL = 15;
    static final int TRIM_MEMORY_RUNNING_LOW = 10;
    static final int TRIM_MEMORY_RUNNING_MODERATE = 5;
```

 - TRIM_MEMORY_RUNNING_MODERATE：内存不足(后台进程超过5个)，该进程是前台运行进程，并且该进程优先级比较高，需要清理内存。
 - TRIM_MEMORY_RUNNING_LOW：内存不足(后台进程不足5个)，该进程是前台运行进程，并且该进程优先级比较高，需要清理内存。
 - TRIM_MEMORY_RUNNING_CRITICAL：内存不足(后台进程不足3个)，该进程是前台运行进程，并且该进程优先级比较高，需要清理内存
 - TRIM_MEMORY_UI_HIDDEN：内存不足，并且该进程的UI已经不可见了。这个是只有当我们程序中的所有UI组件全部不可见的时候才会触发，这和Activity onStop() 方法还是有很大区别的，因为 onStop() 方法只是当一个Activity完全不可见的时候就会调用，比如说用户打开了我们程序中的另一个Activity。因此，我们可以在onStop()方法中去释放一些Activity相关的资源，比如说取消网络连接或者注销广播接收器等，但是像UI相关的资源应该一直要等到onTrimMemory(TRIM_MEMORY_UI_HIDDEN)这个回调之后才去释放，这样可以保证如果用户只是从我们程序的一个Activity回到了另外一个Activity，界面相关的资源都不需要重新加载，从而提升响应速度。
 - TRIM_MEMORY_BACKGROUND：内存不足，并且该进程是后台进程，而且在后台进程的列表LRU缓存的最近位置，近期可能不会被清理，但是这时我们应该去释放一些比较容易恢复的资源以便让手机内存更加充足，防止内存条件不足的问题进一步恶化，这样我们的应用程序也就可以更久地保存在缓存中。
 - TRIM_MEMORY_MODERATE：内存不足，并且该进程在后台进程列表的中部。如果手机内存得不到进一步的释放，我们的应用很快就会有被杀掉的风险。
 - TRIM_MEMORY_COMPLETE：内存不足，并且该进程在后台进程列表最后一个，马上就要被清理。这时应该尽可能的释放一些内存保证我们的应用程序不被杀掉。

#### d.珍惜Services资源

如果你的应用需要在后台使用service，除非它被触发并执行一个任务，否则其他时候service都应该是停止状态。另外需要注意当这个service完成任务之后因为停止service失败而引起的内存泄漏。
当你启动一个service，系统会倾向为了保留这个service而一直保留service所在的进程。这使得进程的运行代价很高，因为系统没有办法把service所占用的RAM空间腾出来让给其他组件，另外service还不能被paged out。这减少了系统能够存放到LRU缓存当中的进程数量，它会影响app之间的切换效率。它甚至会导致系统内存使用不稳定，从而无法继续保持住所有目前正在运行的service。
限制你的service的最好办法是使用IntentService， 它会在处理完交代给它的intent任务之后尽快结束自己。更多信息，请阅读[Running in a Background Service](developer.android.com/training/run-background-service/index.html)。
当一个Service已经不再需要的时候还继续保留它，这对Android应用的内存管理来说是最糟糕的错误之一。因此千万不要贪婪的使得一个Service持续保留。不仅仅是因为它会使得你的应用因为RAM空间的不足而性能糟糕，还会使得用户发现那些有着常驻后台行为的应用并且可能卸载它。
另外还可以使用 JobScheduler 来 避免持久 Service 的使用。[ Background Optimizations](https://developer.android.com/topic/performance/background-optimization.html)

#### e.谨慎使用多进程

这个技术必须谨慎使用，大多数app都不应该运行在多个进程中。因为如果使用不当，它会显著增加内存的使用，而不是减少。当你的app需要在后台运行与前台一样的大量的任务的时候，可以考虑使用这个技术。

## 其他

### 使用ProGuard来剔除不需要的代码

ProGuard能够通过移除不需要的代码，重命名类，域与方法等等对代码进行压缩，优化与混淆。使用ProGuard可以使得你的代码更加紧凑，这样能够减少mapping代码所需要的内存空间。

### 谨慎使用依赖注入框架

使用类似Guice、RoboGuice、Dagger和Butterknife或者等框架注入代码，在某种程度上可以简化你的代码。但是这些框架为了要搜寻代码中的注解，通常都需要经历较长的初始化过程，并且还可能将一些你用不到的对象也一并加载到内存当中。这些用不到的对象会一直占用着内存空间，可能要过很久之后才会得到释放。
因此我们需要在使用方便和内存之间做个平衡，因此，多写几行看似繁琐的代码也许是更好的选择。

### 谨慎使用第三方libraries

很多开源的library代码都不是为移动网络环境而编写的，如果运用在移动设备上，并不一定适合。即使是针对Android而设计的library，也需要特别谨慎，特别是在你不知道引入的library具体做了什么事情的时候。例如，其中一个library使用的是nano protobufs, 而另外一个使用的是micro protobufs。这样一来，在你的应用里面就有2种protobuf的实现方式。这样类似的冲突还可能发生在输出日志，加载图片，缓存等等模块里面。另外不要为了1个或者2个功能而导入整个library，如果没有一个合适的库与你的需求相吻合，你应该考虑自己去实现，而不是导入一个大而全的解决方案。

## 内存使用情况分析

[Android 官方文档：使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.com/studio/profile/memory-profiler)

我们可以借助一些工具在应用程序的整个生命周期内观察内存的使用情况，发现问题并进行优化。

## 参考文章

《Android应用性能优化最佳实践》
http://hukai.me/android-performance-oom/

<!--  
https://developer.android.com/topic/performance/memory
http://hukai.me/android-performance-oom/
https://blog.csdn.net/u010127332/article/details/82424472
https://www.cnblogs.com/ldq2016/p/6635774.html
https://developer.android.com/topic/performance/memory
-->