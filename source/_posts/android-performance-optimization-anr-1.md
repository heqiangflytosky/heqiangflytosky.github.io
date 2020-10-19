

---
title: Android 性能优化实战篇 -- ANR 问题分析1
categories: Android性能优化
comments: true
tags: [ANR]
description: 分析内存过低一起的 ANR
date: 2018-01-10 10:00:00
---


## 概述

本次分析 ANR 的问题原因是由于应用内存过低，主线程等待 GC 线程完成而导致的 ANR。

## 分析

从 /data/anr/ 目录下面拉取到 trace 文件，首先看看 main 线程的情况：

```
"main" prio=5 tid=1 WaitingForGcToComplete
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x73c16330 self=0xf0d48000
  | sysTid=23755 nice=-10 cgrp=default sched=0/0 handle=0xf4ead494
  | state=S schedstat=( 66207538885 3504506943 19134 ) utm=6386 stm=234 core=3 HZ=100
  | stack=0xff710000-0xff712000 stackSize=8MB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23755/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 0017ba95  /system/lib/libart.so (art::gc::Heap::WaitForGcToCompleteLocked(art::gc::GcCause, art::Thread*)+244)
  native: #03 pc 00184b53  /system/lib/libart.so (art::gc::Heap::WaitForGcToComplete(art::gc::GcCause, art::Thread*)+186)
  native: #04 pc 0018272f  /system/lib/libart.so (art::gc::Heap::AllocateInternalWithGc(art::Thread*, art::gc::AllocatorType, bool, unsigned int, unsigned int*, unsigned int*, unsigned int*, art::ObjPtr<art::mirror::Class>*)+74)
  native: #05 pc 003ccb8b  /system/lib/libart.so (artAllocObjectFromCodeInitializedRegionTLAB+246)
  native: #06 pc 00412d47  /system/lib/libart.so (art_quick_alloc_object_initialized_region_tlab+70)
  native: #07 pc 0008c6f5  [anon:dalvik-jit-code-cache] (Java_org_hapjs_component_view_b_a_a__Ljava_lang_String_2Landroid_view_MotionEvent_2Z+196)
  at org.hapjs.component.view.b.a.a(SourceFile:375)
  at org.hapjs.component.view.b.a.a(SourceFile:359)
  at org.hapjs.component.view.b.a.a(SourceFile:335)
  at org.hapjs.component.view.a.a.onTouchEvent(SourceFile:164)
  at android.view.View.dispatchTouchEvent(View.java:12560)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3024)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2705)
  ......
```

主线程正在 WaitingForGcToComplete，等待 GC 结束。
看一下内存使用情况：

```
Heap: 0% free, 255MB/256MB; 8277953 objects
......
Total number of allocations 52646254
Total bytes allocated 1GB
Total bytes freed 1GB
Free memory 488KB
Free memory until GC 488KB
Free memory until OOME 488KB
Total memory 256MB
Max memory 256MB
```

目前内存只有 488K 的剩余，已经接近 256M 的内存限制，内存对象有5千多万，也就是说GC过程需要扫描这些对象的巨大部分，导致耗时很久，另外内存距离OOM只有 488K ，说明有内存泄漏，或内存使用不合理。

另外查看线程信息时发现，此时有大量的线程在运行，仅其中一种类型的线程 [io]-XX竟然有100多个：

```
"[io]-1564" prio=5 tid=79 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15806e28 self=0xd0c0f600
  | sysTid=26683 nice=0 cgrp=default sched=0/0 handle=0xc4c3f970
  | state=S schedstat=( 31151045 39657190 904 ) utm=0 stm=3 core=1 HZ=100
  | stack=0xc4b3c000-0xc4b3e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  ...

......

"[io]-1582" prio=5 tid=98 WaitingForGcToComplete
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15808e90 self=0xcb826800
  | sysTid=26701 nice=0 cgrp=default sched=0/0 handle=0xbe0ed970
  | state=S schedstat=( 682866530 34270257 465 ) utm=66 stm=1 core=1 HZ=100
  | stack=0xbdfea000-0xbdfec000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26701/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 0017ba95  /system/lib/libart.so (art::gc::Heap::WaitForGcToCompleteLocked(art::gc::GcCause, art::Thread*)+244)
  native: #03 pc 00184b53  /system/lib/libart.so (art::gc::Heap::WaitForGcToComplete(art::gc::GcCause, art::Thread*)+186)
  native: #04 pc 0018272f  /system/lib/libart.so (art::gc::Heap::AllocateInternalWithGc(art::Thread*, art::gc::AllocatorType, bool, unsigned int, unsigned int*, unsigned int*, unsigned int*, art::ObjPtr<art::mirror::Class>*)+74)
  native: #05 pc 003ccb8b  /system/lib/libart.so (artAllocObjectFromCodeInitializedRegionTLAB+246)
  native: #06 pc 00412d47  /system/lib/libart.so (art_quick_alloc_object_initialized_region_tlab+70)
  native: #07 pc 00016e8d  [anon:dalvik-jit-code-cache] (Java_org_hapjs_render_a_a_a__ILjava_lang_String_2+276)
  at org.hapjs.render.a.a.a(SourceFile:42)
  - locked <0x007aea59> (a org.hapjs.render.a.a)
  at org.hapjs.render.a.d.b(SourceFile:291)
  ...
 
......

"[io]-1721" prio=5 tid=246 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x134418e8 self=0xc7206000
  | sysTid=26841 nice=0 cgrp=default sched=0/0 handle=0xb474d970
  | state=S schedstat=( 4215470 903438 14 ) utm=0 stm=0 core=4 HZ=100
  | stack=0xb464a000-0xb464c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  ...
```

[io]-1582 也是在等待 GC 完成状态，由于该方法正在执行一个 synchronized 方法，竟然导致有 100 多个线程在等待锁的释放。
根据这个线索，发现执行这些线程是因为前端页面正在创建一个800多个item的列表，复现这个步骤时虽然不会每次都发生 anr，但是界面有明显的卡顿现象。采用分页加载后解决了这个问题。

完成的 trace 日志见注释内容：

<!--
```

Cmd line: com.meizu.flyme.directservice:Launcher4
Build fingerprint: 'meizu/meizu_16s_CN/16s:9/PKQ1.190202.001/1594089253:user/release-keys'
ABI: 'arm'
Build type: optimized
Zygote loaded classes=10672 post zygote classes=2913
Intern table: 83663 strong; 367 weak
JNI: CheckJNI is off; globals=704 (plus 3269 weak)
Libraries: /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libc++_shared.so /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libgifimage.so /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libimagepipeline.so /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libinspector.so /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libmmkv.so /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libyoga.so /system/lib/libandroid.so /system/lib/libcompiler_rt.so /system/lib/libjavacrypto.so /system/lib/libjnigraphics.so /system/lib/libmedia_jni.so /system/lib/libqti_performance.so /system/lib/libsoundpool.so /system/lib/libwebviewchromium_loader.so libjavacore.so libopenjdk.so (17)
Heap: 0% free, 255MB/256MB; 8277953 objects
Dumping cumulative Gc timings
Start Dumping histograms for 133 iterations for concurrent copying
ProcessMarkStack:	Sum: 63.611s 99% C.I. 8.512ms-2546.176ms Avg: 478.284ms Max: 3822.117ms
ScanImmuneSpaces:	Sum: 1.023s 99% C.I. 3.318ms-32.860ms Avg: 7.694ms Max: 56.225ms
ClearFromSpace:	Sum: 597.910ms 99% C.I. 0.117ms-52.584ms Avg: 4.495ms Max: 72.389ms
SweepLargeObjects:	Sum: 378.361ms 99% C.I. 0.016ms-10.945ms Avg: 2.844ms Max: 15.585ms
VisitConcurrentRoots:	Sum: 375.651ms 99% C.I. 1.465ms-6.436ms Avg: 2.824ms Max: 7.646ms
FlipOtherThreads:	Sum: 327.661ms 99% C.I. 0.080ms-16.135ms Avg: 2.463ms Max: 16.758ms
ProcessReferences:	Sum: 147.846ms 99% C.I. 0.485us-3417.500us Avg: 555.812us Max: 4049us
SweepSystemWeaks:	Sum: 145.051ms 99% C.I. 0.087ms-7.082ms Avg: 1.090ms Max: 13.157ms
EmptyRBMarkBitStack:	Sum: 74.686ms 99% C.I. 61us-11102us Avg: 561.548us Max: 24839us
GrayAllDirtyImmuneObjects:	Sum: 60.059ms 99% C.I. 183us-2573us Avg: 451.571us Max: 5015us
InitializePhase:	Sum: 43.302ms 99% C.I. 67us-1150.250us Avg: 325.578us Max: 1243us
EnqueueFinalizerReferences:	Sum: 41.584ms 99% C.I. 14us-3605us Avg: 312.661us Max: 5516us
ThreadListFlip:	Sum: 41.578ms 99% C.I. 17us-2567.500us Avg: 312.616us Max: 2857us
FlipThreadRoots:	Sum: 35.988ms 99% C.I. 0.563us-8736.500us Avg: 270.586us Max: 9922us
MarkingPhase:	Sum: 31.311ms 99% C.I. 12us-2501.500us Avg: 235.421us Max: 3059us
ResumeOtherThreads:	Sum: 28.391ms 99% C.I. 1.072us-11277us Avg: 213.466us Max: 15261us
ForwardSoftReferences:	Sum: 17.198ms 99% C.I. 10us-533.750us Avg: 129.308us Max: 693us
MarkZygoteLargeObjects:	Sum: 15.667ms 99% C.I. 16us-516.750us Avg: 117.796us Max: 547us
ReclaimPhase:	Sum: 11.960ms 99% C.I. 3us-1802.500us Avg: 89.924us Max: 2777us
VisitNonThreadRoots:	Sum: 11.459ms 99% C.I. 35us-433.750us Avg: 86.157us Max: 558us
ClearRegionSpaceCards:	Sum: 10.714ms 99% C.I. 19us-1372us Avg: 80.556us Max: 3470us
MarkStackAsLive:	Sum: 4.699ms 99% C.I. 3us-954.250us Avg: 35.330us Max: 2600us
ResumeRunnableThreads:	Sum: 4.377ms 99% C.I. 2us-169us Avg: 32.909us Max: 169us
(Paused)GrayAllNewlyDirtyImmuneObjects:	Sum: 4.091ms 99% C.I. 12us-183.500us Avg: 30.759us Max: 218us
RecordFree:	Sum: 3.396ms 99% C.I. 13us-69us Avg: 25.533us Max: 69us
(Paused)SetFromSpace:	Sum: 1.984ms 99% C.I. 0.255us-85us Avg: 14.917us Max: 85us
SweepAllocSpace:	Sum: 970us 99% C.I. 2us-51us Avg: 7.293us Max: 51us
(Paused)ClearCards:	Sum: 845us 99% C.I. 250ns-6000ns Avg: 235ns Max: 6000ns
SwapBitmaps:	Sum: 802us 99% C.I. 2us-59us Avg: 6.030us Max: 59us
(Paused)FlipCallback:	Sum: 491us 99% C.I. 1us-15us Avg: 3.691us Max: 15us
Sweep:	Sum: 348us 99% C.I. 1us-19us Avg: 2.616us Max: 19us
UnBindBitmaps:	Sum: 41us 99% C.I. 250ns-3000ns Avg: 308ns Max: 3000ns
Done Dumping histograms
concurrent copying paused:	Sum: 49.600ms 99% C.I. 45us-2667.500us Avg: 372.932us Max: 2903us
concurrent copying total time: 67.053s mean time: 504.162ms
concurrent copying freed: 44382975 objects with total size 3GB
concurrent copying throughput: 661909/s / 54MB/s
Cumulative bytes moved 895062736
Cumulative objects moved 28340055
Peak regions allocated 1024 (256MB) / 1024 (256MB)
Total time spent in GC: 67.053s
Mean GC size throughput: 20MB/s
Mean GC object throughput: 661684 objects/s
Total number of allocations 52646254
Total bytes allocated 1GB
Total bytes freed 1GB
Free memory 488KB
Free memory until GC 488KB
Free memory until OOME 488KB
Total memory 256MB
Max memory 256MB
Zygote space size 1456KB
Total mutator paused time: 49.600ms
Total time waiting for GC to complete: 138.167s
Total GC count: 133
Total GC time: 67.053s
Total blocking GC count: 45
Total blocking GC time: 25.422s
Histogram of GC count per 10000 ms: 0:24,1:14,2:13,3:4,4:3,5:5,9:1,11:1,14:1
Histogram of blocking GC count per 10000 ms: 0:56,1:2,2:4,3:2,5:1,14:1
Registered native bytes allocated: 28920285
/data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/oat/arm/base.odex: speed-profile
/system/framework/oat/arm/org.apache.http.legacy.boot.odex: speed-profile
Current JIT code cache size: 1522KB
Current JIT data cache size: 974KB
Current JIT mini-debug-info size: 0B
Current JIT capacity: 3MB
Current number of JIT JNI stub entries: 0
Current number of JIT code cache entries: 1530
Total number of JIT compilations: 1681
Total number of JIT compilations for on stack replacement: 6
Total number of JIT code cache collections: 10
Memory used for stack maps: Avg: 501B Max: 9KB Min: 24B
Memory used for compiled code: Avg: 1052B Max: 11KB Min: 2B
Memory used for profiling info: Avg: 136B Max: 3KB Min: 16B
Start Dumping histograms for 1691 iterations for JIT timings
Compiling:	Sum: 2.347s 99% C.I. 0.048ms-13.318ms Avg: 1.396ms Max: 33.136ms
TrimMaps:	Sum: 102.356ms 99% C.I. 12us-489.874us Avg: 60.889us Max: 1607us
Code cache collection:	Sum: 15.921ms 99% C.I. 0.229ms-8.810ms Avg: 1.592ms Max: 9.164ms
Done Dumping histograms
Memory used for compilation: Avg: 101KB Max: 1478KB Min: 11KB
ProfileSaver total_bytes_written=74001
ProfileSaver total_number_of_writes=2
ProfileSaver total_number_of_code_cache_queries=10
ProfileSaver total_number_of_skipped_writes=8
ProfileSaver total_number_of_failed_writes=0
ProfileSaver total_ms_of_sleep=634295
ProfileSaver total_ms_of_work=441
ProfileSaver max_number_profile_entries_cached=0
ProfileSaver total_number_of_hot_spikes=43
ProfileSaver total_number_of_wake_ups=42
Number of JIT inline cache deoptimizations: 58
Number of JIT same target deoptimizations: 8
Number of class hierarchy analysis deoptimizations: 1

suspend all histogram:	Sum: 42.258ms 99% C.I. 3us-2490.879us Avg: 278.013us Max: 2847us
DALVIK THREADS (245):
"Signal Catcher" daemon prio=5 tid=3 Runnable
  | group="system" sCount=0 dsCount=0 flags=0 obj=0x1de00258 self=0xf0d48c00
  | sysTid=23761 nice=0 cgrp=default sched=0/0 handle=0xec188970
  | state=R schedstat=( 35883074 3276560 35 ) utm=1 stm=2 core=4 HZ=100
  | stack=0xec08d000-0xec08f000 stackSize=1010KB
  | held mutexes= "mutator lock"(shared held)
  native: #00 pc 002daa3f  /system/lib/libart.so (art::DumpNativeStack(std::__1::basic_ostream<char, std::__1::char_traits<char>>&, int, BacktraceMap*, char const*, art::ArtMethod*, void*, bool)+134)
  native: #01 pc 0037082b  /system/lib/libart.so (art::Thread::DumpStack(std::__1::basic_ostream<char, std::__1::char_traits<char>>&, bool, BacktraceMap*, bool) const+210)
  native: #02 pc 0036cfe3  /system/lib/libart.so (art::Thread::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char>>&, bool, BacktraceMap*, bool) const+34)
  native: #03 pc 00385c99  /system/lib/libart.so (art::DumpCheckpoint::Run(art::Thread*)+624)
  native: #04 pc 0037ff7f  /system/lib/libart.so (art::ThreadList::RunCheckpoint(art::Closure*, art::Closure*)+314)
  native: #05 pc 0037f677  /system/lib/libart.so (art::ThreadList::Dump(std::__1::basic_ostream<char, std::__1::char_traits<char>>&, bool)+758)
  native: #06 pc 0037f2af  /system/lib/libart.so (art::ThreadList::DumpForSigQuit(std::__1::basic_ostream<char, std::__1::char_traits<char>>&)+614)
  native: #07 pc 00358fe9  /system/lib/libart.so (art::Runtime::DumpForSigQuit(std::__1::basic_ostream<char, std::__1::char_traits<char>>&)+120)
  native: #08 pc 00362599  /system/lib/libart.so (art::SignalCatcher::HandleSigQuit()+1040)
  native: #09 pc 0036172b  /system/lib/libart.so (art::SignalCatcher::Run(void*)+246)
  native: #10 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #11 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)
  (no managed stack frames)

"main" prio=5 tid=1 WaitingForGcToComplete
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x73c16330 self=0xf0d48000
  | sysTid=23755 nice=-10 cgrp=default sched=0/0 handle=0xf4ead494
  | state=S schedstat=( 66207538885 3504506943 19134 ) utm=6386 stm=234 core=3 HZ=100
  | stack=0xff710000-0xff712000 stackSize=8MB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23755/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 0017ba95  /system/lib/libart.so (art::gc::Heap::WaitForGcToCompleteLocked(art::gc::GcCause, art::Thread*)+244)
  native: #03 pc 00184b53  /system/lib/libart.so (art::gc::Heap::WaitForGcToComplete(art::gc::GcCause, art::Thread*)+186)
  native: #04 pc 0018272f  /system/lib/libart.so (art::gc::Heap::AllocateInternalWithGc(art::Thread*, art::gc::AllocatorType, bool, unsigned int, unsigned int*, unsigned int*, unsigned int*, art::ObjPtr<art::mirror::Class>*)+74)
  native: #05 pc 003ccb8b  /system/lib/libart.so (artAllocObjectFromCodeInitializedRegionTLAB+246)
  native: #06 pc 00412d47  /system/lib/libart.so (art_quick_alloc_object_initialized_region_tlab+70)
  native: #07 pc 0008c6f5  [anon:dalvik-jit-code-cache] (Java_org_hapjs_component_view_b_a_a__Ljava_lang_String_2Landroid_view_MotionEvent_2Z+196)
  at org.hapjs.component.view.b.a.a(SourceFile:375)
  at org.hapjs.component.view.b.a.a(SourceFile:359)
  at org.hapjs.component.view.b.a.a(SourceFile:335)
  at org.hapjs.component.view.a.a.onTouchEvent(SourceFile:164)
  at android.view.View.dispatchTouchEvent(View.java:12560)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3024)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2705)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at org.hapjs.render.RootView.dispatchTouchEvent(SourceFile:1570)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3030)
  at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2719)
  at com.android.internal.policy.DecorView.superDispatchTouchEvent(DecorView.java:472)
  at com.android.internal.policy.PhoneWindow.superDispatchTouchEvent(PhoneWindow.java:1844)
  at android.app.Activity.dispatchTouchEvent(Activity.java:3433)
  at androidx.appcompat.view.WindowCallbackWrapper.dispatchTouchEvent(SourceFile:69)
  at com.android.internal.policy.DecorView.dispatchTouchEvent(DecorView.java:428)
  at android.view.View.dispatchPointerEvent(View.java:12799)
  at android.view.ViewRootImpl$ViewPostImeInputStage.processPointerEvent(ViewRootImpl.java:5352)
  at android.view.ViewRootImpl$ViewPostImeInputStage.onProcess(ViewRootImpl.java:5136)
  at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4619)
  at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:4672)
  at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:4638)
  at android.view.ViewRootImpl$AsyncInputStage.forward(ViewRootImpl.java:4778)
  at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:4646)
  at android.view.ViewRootImpl$AsyncInputStage.apply(ViewRootImpl.java:4835)
  at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4619)
  at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:4672)
  at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:4638)
  at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:4646)
  at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:4619)
  at android.view.ViewRootImpl.deliverInputEvent(ViewRootImpl.java:7352)
  at android.view.ViewRootImpl.doProcessInputEvents(ViewRootImpl.java:7321)
  at android.view.ViewRootImpl.enqueueInputEvent(ViewRootImpl.java:7282)
  at android.view.ViewRootImpl$WindowInputEventReceiver.onInputEvent(ViewRootImpl.java:7473)
  at android.view.InputEventReceiver.dispatchInputEvent(InputEventReceiver.java:187)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.app.ActivityThread.main(ActivityThread.java:7035)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:937)

"Jit thread pool worker thread 0" daemon prio=5 tid=2 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de001d0 self=0xebf88000
  | sysTid=23760 nice=9 cgrp=default sched=0/0 handle=0xec289970
  | state=S schedstat=( 2540856294 1767005270 4734 ) utm=192 stm=61 core=5 HZ=100
  | stack=0xec18b000-0xec18d000 stackSize=1022KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23760/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 00386e63  /system/lib/libart.so (art::ThreadPool::GetTask(art::Thread*)+174)
  native: #03 pc 00386727  /system/lib/libart.so (art::ThreadPoolWorker::Run()+62)
  native: #04 pc 0038635b  /system/lib/libart.so (art::ThreadPoolWorker::Callback(void*)+94)
  native: #05 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #06 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)
  (no managed stack frames)

"ReferenceQueueDaemon" daemon prio=5 tid=4 Waiting
  | group="system" sCount=1 dsCount=0 flags=1 obj=0x1de002e0 self=0xf0d4c800
  | sysTid=23762 nice=4 cgrp=default sched=0/0 handle=0xd58dd970
  | state=S schedstat=( 253053419 171350994 1038 ) utm=18 stm=6 core=3 HZ=100
  | stack=0xd57da000-0xd57dc000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0376be09> (a java.lang.Class<java.lang.ref.ReferenceQueue>)
  at java.lang.Daemons$ReferenceQueueDaemon.runInternal(Daemons.java:178)
  - locked <0x0376be09> (a java.lang.Class<java.lang.ref.ReferenceQueue>)
  at java.lang.Daemons$Daemon.run(Daemons.java:103)
  at java.lang.Thread.run(Thread.java:764)

"FinalizerDaemon" daemon prio=5 tid=5 Waiting
  | group="system" sCount=1 dsCount=0 flags=1 obj=0x1de00368 self=0xf0d4ce00
  | sysTid=23763 nice=4 cgrp=default sched=0/0 handle=0xd57d7970
  | state=S schedstat=( 69947188 14912135 184 ) utm=5 stm=1 core=1 HZ=100
  | stack=0xd56d4000-0xd56d6000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0b6bd80e> (a java.lang.Object)
  at java.lang.Object.wait(Object.java:422)
  at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:188)
  - locked <0x0b6bd80e> (a java.lang.Object)
  at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:209)
  at java.lang.Daemons$FinalizerDaemon.runInternal(Daemons.java:232)
  at java.lang.Daemons$Daemon.run(Daemons.java:103)
  at java.lang.Thread.run(Thread.java:764)

"FinalizerWatchdogDaemon" daemon prio=5 tid=6 Sleeping
  | group="system" sCount=1 dsCount=0 flags=1 obj=0x1de003f0 self=0xf0134600
  | sysTid=23764 nice=4 cgrp=default sched=0/0 handle=0xd56d1970
  | state=S schedstat=( 2113387 1467137 21 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xd55ce000-0xd55d0000 stackSize=1042KB
  | held mutexes=
  at java.lang.Thread.sleep(Native method)
  - sleeping on <0x0193722f> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:373)
  - locked <0x0193722f> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:314)
  at java.lang.Daemons$FinalizerWatchdogDaemon.sleepFor(Daemons.java:342)
  at java.lang.Daemons$FinalizerWatchdogDaemon.waitForFinalization(Daemons.java:364)
  at java.lang.Daemons$FinalizerWatchdogDaemon.runInternal(Daemons.java:281)
  at java.lang.Daemons$Daemon.run(Daemons.java:103)
  at java.lang.Thread.run(Thread.java:764)

"HeapTaskDaemon" daemon prio=5 tid=7 WaitingForCheckPointsToRun
  | group="system" sCount=1 dsCount=0 flags=1 obj=0x1de05b40 self=0xf0134c00
  | sysTid=23765 nice=4 cgrp=default sched=0/0 handle=0xd55cb970
  | state=S schedstat=( 49488998500 1735699135 7105 ) utm=4789 stm=159 core=2 HZ=100
  | stack=0xd54c8000-0xd54ca000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23765/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 000a305b  /system/lib/libart.so (_ZN3art7Barrier9IncrementILNS0_12LockHandlingE1EEEvPNS_6ThreadEi+46)
  native: #03 pc 0015b811  /system/lib/libart.so (art::gc::collector::ConcurrentCopying::RevokeThreadLocalMarkStacks(bool, art::Closure*)+524)
  native: #04 pc 0015c289  /system/lib/libart.so (art::gc::collector::ConcurrentCopying::ProcessThreadLocalMarkStacks(bool, art::Closure*)+20)
  native: #05 pc 0015bf41  /system/lib/libart.so (art::gc::collector::ConcurrentCopying::ProcessMarkStackOnce()+488)
  native: #06 pc 0015bd45  /system/lib/libart.so (art::gc::collector::ConcurrentCopying::ProcessMarkStack()+8)
  native: #07 pc 00156b1f  /system/lib/libart.so (art::gc::collector::ConcurrentCopying::MarkingPhase()+1318)
  native: #08 pc 00155665  /system/lib/libart.so (art::gc::collector::ConcurrentCopying::RunPhases()+856)
  native: #09 pc 00166295  /system/lib/libart.so (art::gc::collector::GarbageCollector::Run(art::gc::GcCause, bool)+252)
  native: #10 pc 0017ff05  /system/lib/libart.so (art::gc::Heap::CollectGarbageInternal(art::gc::collector::GcType, art::gc::GcCause, bool)+2420)
  native: #11 pc 0018d721  /system/lib/libart.so (art::gc::Heap::ConcurrentGC(art::Thread*, art::gc::GcCause, bool)+68)
  native: #12 pc 00191551  /system/lib/libart.so (art::gc::Heap::ConcurrentGCTask::Run(art::Thread*)+20)
  native: #13 pc 001aa39b  /system/lib/libart.so (art::gc::TaskProcessor::RunAllTasks(art::Thread*)+34)
  at dalvik.system.VMRuntime.runHeapTasks(Native method)
  at java.lang.Daemons$HeapTaskDaemon.runInternal(Daemons.java:475)
  at java.lang.Daemons$Daemon.run(Daemons.java:103)
  at java.lang.Thread.run(Thread.java:764)

"Binder:23755_1" prio=5 tid=8 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00480 self=0xebf8bc00
  | sysTid=23766 nice=0 cgrp=default sched=0/0 handle=0xd53c7970
  | state=S schedstat=( 10268906 27881616 119 ) utm=1 stm=0 core=3 HZ=100
  | stack=0xd52cc000-0xd52ce000 stackSize=1010KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23766/stack)
  native: #00 pc 00060be8  /system/lib/libc.so (__ioctl+8)
  native: #01 pc 00022d6f  /system/lib/libc.so (ioctl+30)
  native: #02 pc 0004036d  /system/lib/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+204)
  native: #03 pc 000404c9  /system/lib/libbinder.so (android::IPCThreadState::getAndExecuteCommand()+8)
  native: #04 pc 00040a3d  /system/lib/libbinder.so (android::IPCThreadState::joinThreadPool(bool)+40)
  native: #05 pc 000581b5  /system/lib/libbinder.so (android::PoolThread::threadLoop()+12)
  native: #06 pc 0000c6af  /system/lib/libutils.so (android::Thread::_threadLoop(void*)+202)
  native: #07 pc 00070369  /system/lib/libandroid_runtime.so (android::AndroidRuntime::javaThreadShell(void*)+88)
  native: #08 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #09 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)
  (no managed stack frames)

"Binder:23755_2" prio=5 tid=9 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00508 self=0xebe4a600
  | sysTid=23767 nice=0 cgrp=default sched=0/0 handle=0xd52c9970
  | state=S schedstat=( 20190115 25331456 162 ) utm=1 stm=0 core=3 HZ=100
  | stack=0xd51ce000-0xd51d0000 stackSize=1010KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23767/stack)
  native: #00 pc 00060be8  /system/lib/libc.so (__ioctl+8)
  native: #01 pc 00022d6f  /system/lib/libc.so (ioctl+30)
  native: #02 pc 0004036d  /system/lib/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+204)
  native: #03 pc 000404c9  /system/lib/libbinder.so (android::IPCThreadState::getAndExecuteCommand()+8)
  native: #04 pc 00040a5d  /system/lib/libbinder.so (android::IPCThreadState::joinThreadPool(bool)+72)
  native: #05 pc 000581b5  /system/lib/libbinder.so (android::PoolThread::threadLoop()+12)
  native: #06 pc 0000c6af  /system/lib/libutils.so (android::Thread::_threadLoop(void*)+202)
  native: #07 pc 00070369  /system/lib/libandroid_runtime.so (android::AndroidRuntime::javaThreadShell(void*)+88)
  native: #08 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #09 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)
  (no managed stack frames)

"Profile Saver" daemon prio=5 tid=10 Native
  | group="system" sCount=1 dsCount=0 flags=1 obj=0x1de00590 self=0xebf9a800
  | sysTid=23768 nice=9 cgrp=default sched=0/0 handle=0xd340d970
  | state=S schedstat=( 439121514 34705102 110 ) utm=42 stm=1 core=5 HZ=100
  | stack=0xd3312000-0xd3314000 stackSize=1010KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23768/stack)
  native: #00 pc 00019eb4  /system/lib/libc.so (syscall+32)
  native: #01 pc 000a6ecf  /system/lib/libart.so (art::ConditionVariable::TimedWait(art::Thread*, long long, int)+98)
  native: #02 pc 0025e9a5  /system/lib/libart.so (art::ProfileSaver::Run()+716)
  native: #03 pc 002611dd  /system/lib/libart.so (art::ProfileSaver::RunProfileSaverThread(void*)+52)
  native: #04 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #05 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)
  (no managed stack frames)

"[single]-322" prio=5 tid=15 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00618 self=0xebf18c00
  | sysTid=23782 nice=0 cgrp=default sched=0/0 handle=0xd19ff970
  | state=S schedstat=( 233439 1330520 9 ) utm=0 stm=0 core=7 HZ=100
  | stack=0xd18fc000-0xd18fe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x07ec203c> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x07ec203c> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1120)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:849)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"RxSchedulerPurge-1" daemon prio=5 tid=16 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00788 self=0xd1c19a00
  | sysTid=23785 nice=0 cgrp=default sched=0/0 handle=0xd16ff970
  | state=S schedstat=( 258661893 90105370 715 ) utm=20 stm=5 core=2 HZ=100
  | stack=0xd15fc000-0xd15fe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0bc431c5> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0bc431c5> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2101)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1132)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:849)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"RxCachedWorkerPoolEvictor-1" daemon prio=5 tid=17 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de008f8 self=0xd1714000
  | sysTid=23786 nice=0 cgrp=default sched=0/0 handle=0xd15f9970
  | state=S schedstat=( 7647657 1198022 13 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xd14f6000-0xd14f8000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0d11671a> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0d11671a> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2101)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1132)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:849)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"RenderThread" daemon prio=7 tid=21 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00a68 self=0xd1716a00
  | sysTid=23790 nice=-10 cgrp=default sched=0/0 handle=0xd10be970
  | state=S schedstat=( 4325673132 980299293 7007 ) utm=339 stm=92 core=1 HZ=100
  | stack=0xd0fc3000-0xd0fc5000 stackSize=1010KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23790/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 0022350f  /system/lib/libhwui.so (???)
  native: #05 pc 0023e3f3  /system/lib/libhwui.so (android::uirenderer::renderthread::RenderThread::threadLoop()+306)
  native: #06 pc 0000c6af  /system/lib/libutils.so (android::Thread::_threadLoop(void*)+202)
  native: #07 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #08 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)
  (no managed stack frames)

"queued-work-looper" prio=5 tid=23 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00af0 self=0xd1716400
  | sysTid=23812 nice=-2 cgrp=default sched=0/0 handle=0xd0739970
  | state=S schedstat=( 14859114 11225887 83 ) utm=0 stm=1 core=4 HZ=100
  | stack=0xd0636000-0xd0638000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23812/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"[io-thread]-333" prio=5 tid=24 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00be0 self=0xd1715e00
  | sysTid=23813 nice=0 cgrp=default sched=0/0 handle=0xd0633970
  | state=S schedstat=( 4408646 9836143 88 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xd0530000-0xd0532000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x05ef024b> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x05ef024b> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io-thread]-334" prio=5 tid=25 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00d68 self=0xd0db4e00
  | sysTid=23814 nice=0 cgrp=default sched=0/0 handle=0xd052d970
  | state=S schedstat=( 4023803 11013803 40 ) utm=0 stm=0 core=7 HZ=100
  | stack=0xd042a000-0xd042c000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0a790428> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0a790428> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"v8Inspector_sender" prio=5 tid=26 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00e48 self=0xd0db5400
  | sysTid=23815 nice=0 cgrp=default sched=0/0 handle=0xd01fe970
  | state=S schedstat=( 11045920736 1982500426 22611 ) utm=1010 stm=94 core=6 HZ=100
  | stack=0xd00fb000-0xd00fd000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23815/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"StethoListener-main" prio=5 tid=27 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de00f38 self=0xd0db5a00
  | sysTid=23816 nice=0 cgrp=default sched=0/0 handle=0xcffff970
  | state=S schedstat=( 2392029 1251979 13 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xcfefc000-0xcfefe000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23816/stack)
  native: #00 pc 00060c84  /system/lib/libc.so (__ppoll+20)
  native: #01 pc 000245fb  /system/lib/libc.so (poll+54)
  native: #02 pc 0001f9cb  /system/lib/libjavacore.so (Linux_poll(_JNIEnv*, _jobject*, _jobjectArray*, int)+578)
  at libcore.io.Linux.poll(Native method)
  at libcore.io.BlockGuardOs.poll(BlockGuardOs.java:219)
  at android.system.Os.poll(Os.java:374)
  at libcore.io.IoBridge.poll(IoBridge.java:712)
  at java.net.PlainSocketImpl.socketAccept(PlainSocketImpl.java:189)
  at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:451)
  at java.net.ServerSocket.implAccept(ServerSocket.java:547)
  at java.net.ServerSocket.accept(ServerSocket.java:515)
  at com.facebook.stetho.server.AbsServerSocket$ServerSocketImpl.accept(SourceFile:36)
  at com.facebook.stetho.server.LocalSocketServer.listenOnAddress(SourceFile:93)
  at com.facebook.stetho.server.LocalSocketServer.run(SourceFile:75)
  at com.facebook.stetho.server.ServerManager$1.run(SourceFile:39)

"Binder:23755_3" prio=5 tid=28 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de014c0 self=0xd1369000
  | sysTid=23817 nice=0 cgrp=default sched=0/0 handle=0xcfef9970
  | state=S schedstat=( 3614324 2681406 34 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xcfdfe000-0xcfe00000 stackSize=1010KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23817/stack)
  native: #00 pc 00060be8  /system/lib/libc.so (__ioctl+8)
  native: #01 pc 00022d6f  /system/lib/libc.so (ioctl+30)
  native: #02 pc 0004036d  /system/lib/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+204)
  native: #03 pc 000404c9  /system/lib/libbinder.so (android::IPCThreadState::getAndExecuteCommand()+8)
  native: #04 pc 00040a5d  /system/lib/libbinder.so (android::IPCThreadState::joinThreadPool(bool)+72)
  native: #05 pc 000581b5  /system/lib/libbinder.so (android::PoolThread::threadLoop()+12)
  native: #06 pc 0000c6af  /system/lib/libutils.so (android::Thread::_threadLoop(void*)+202)
  native: #07 pc 00070369  /system/lib/libandroid_runtime.so (android::AndroidRuntime::javaThreadShell(void*)+88)
  native: #08 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #09 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)
  (no managed stack frames)

"OkHttp ConnectionPool" daemon prio=5 tid=29 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de01548 self=0xd136d200
  | sysTid=23819 nice=0 cgrp=default sched=0/0 handle=0xcfcf5970
  | state=S schedstat=( 1661981 5092499 6 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xcfbf2000-0xcfbf4000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x03bb2541> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x03bb2541> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2101)
  at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"TcmReceiver" prio=5 tid=30 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de01680 self=0xd136cc00
  | sysTid=23818 nice=0 cgrp=default sched=0/0 handle=0xcfdfb970
  | state=S schedstat=( 350260 3318438 2 ) utm=0 stm=0 core=7 HZ=100
  | stack=0xcfcf8000-0xcfcfa000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23818/stack)
  native: #00 pc 00061b60  /system/lib/libc.so (recvmsg+8)
  native: #01 pc 000c3ffd  /system/lib/libandroid_runtime.so (android::socket_read_all(_JNIEnv*, _jobject*, int, void*, unsigned int)+88)
  native: #02 pc 000c3dab  /system/lib/libandroid_runtime.so (android::socket_readba(_JNIEnv*, _jobject*, _jbyteArray*, int, int, _jobject*)+158)
  at android.net.LocalSocketImpl.readba_native(Native method)
  at android.net.LocalSocketImpl.access$300(LocalSocketImpl.java:36)
  at android.net.LocalSocketImpl$SocketInputStream.read(LocalSocketImpl.java:110)
  - locked <0x0d4edae6> (a java.lang.Object)
  at com.qti.tcmclient.DpmTcmClient$TcmReceiver.run(DpmTcmClient.java:146)
  at java.lang.Thread.run(Thread.java:764)

"UsageStats_Logger" prio=5 tid=31 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de017a8 self=0xd0db7000
  | sysTid=23820 nice=0 cgrp=default sched=0/0 handle=0xcfa7f970
  | state=S schedstat=( 176406 0 1 ) utm=0 stm=0 core=7 HZ=100
  | stack=0xcf97c000-0xcf97e000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23820/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"com.meizu.statsrpk.apiWorker" prio=5 tid=32 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de01898 self=0xd0db7600
  | sysTid=23822 nice=5 cgrp=default sched=0/0 handle=0xcf979970
  | state=S schedstat=( 72425828 193152389 527 ) utm=6 stm=1 core=5 HZ=100
  | stack=0xcf876000-0xcf878000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23822/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"rpk.ConfigControllerWorker" prio=5 tid=33 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de01988 self=0xd1370e00
  | sysTid=23829 nice=5 cgrp=default sched=0/0 handle=0xcf7f2970
  | state=S schedstat=( 683543 2620313 4 ) utm=0 stm=0 core=7 HZ=100
  | stack=0xcf6ef000-0xcf6f1000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23829/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"[single]-344" prio=5 tid=34 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de01a78 self=0xebf19200
  | sysTid=23847 nice=0 cgrp=default sched=0/0 handle=0xcf6ec970
  | state=S schedstat=( 171501415 66380987 228 ) utm=11 stm=5 core=6 HZ=100
  | stack=0xcf5e9000-0xcf5eb000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x05297827> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x05297827> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1120)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:849)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"FrescoLightWeightBackgroundExecutor-1" prio=5 tid=36 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de01be8 self=0xd0db8e00
  | sysTid=23850 nice=10 cgrp=default sched=0/0 handle=0xca4ef970
  | state=S schedstat=( 117493230 219943968 386 ) utm=11 stm=0 core=4 HZ=100
  | stack=0xca3ec000-0xca3ee000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0a6e3ad4> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0a6e3ad4> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"magnifier pixel copy result handler" prio=5 tid=37 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de01e30 self=0xcf148e00
  | sysTid=23851 nice=0 cgrp=default sched=0/0 handle=0xca3e9970
  | state=S schedstat=( 403647 788125 1 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xca2e6000-0xca2e8000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23851/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"FrescoDecodeExecutor-1" prio=5 tid=41 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de01f20 self=0xd1754e00
  | sysTid=23856 nice=10 cgrp=default sched=0/0 handle=0xc9f4e970
  | state=S schedstat=( 120790524 104473435 158 ) utm=10 stm=1 core=5 HZ=100
  | stack=0xc9e4b000-0xc9e4d000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0ad3d47d> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0ad3d47d> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoBackgroundExecutor-1" prio=5 tid=42 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de020b8 self=0xccdaa600
  | sysTid=23858 nice=10 cgrp=default sched=0/0 handle=0xc9d42970
  | state=S schedstat=( 2630313 352500 4 ) utm=0 stm=0 core=5 HZ=100
  | stack=0xc9c3f000-0xc9c41000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0c9dbf72> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0c9dbf72> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoIoBoundExecutor-1" prio=5 tid=52 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de02250 self=0xccd7ae00
  | sysTid=23867 nice=10 cgrp=default sched=0/0 handle=0xc92b9970
  | state=S schedstat=( 217099739 222106204 480 ) utm=16 stm=4 core=6 HZ=100
  | stack=0xc91b6000-0xc91b8000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x03ab6fc3> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x03ab6fc3> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoIoBoundExecutor-2" prio=5 tid=53 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de023e8 self=0xccd7b400
  | sysTid=23868 nice=10 cgrp=default sched=0/0 handle=0xc91b3970
  | state=S schedstat=( 174234016 224904064 678 ) utm=11 stm=6 core=5 HZ=100
  | stack=0xc90b0000-0xc90b2000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0fde3040> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0fde3040> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoDecodeExecutor-2" prio=5 tid=59 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de024d8 self=0xf015ac00
  | sysTid=23874 nice=10 cgrp=default sched=0/0 handle=0xc8b8f970
  | state=S schedstat=( 149592605 111761105 244 ) utm=13 stm=1 core=3 HZ=100
  | stack=0xc8a8c000-0xc8a8e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x05ce3b79> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x05ce3b79> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoDecodeExecutor-3" prio=5 tid=60 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de025c8 self=0xf015b200
  | sysTid=23875 nice=10 cgrp=default sched=0/0 handle=0xc89ff970
  | state=S schedstat=( 152765152 99181567 198 ) utm=15 stm=0 core=0 HZ=100
  | stack=0xc88fc000-0xc88fe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x078460be> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x078460be> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoDecodeExecutor-4" prio=5 tid=61 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de026b8 self=0xf015b800
  | sysTid=23876 nice=10 cgrp=default sched=0/0 handle=0xc88f9970
  | state=S schedstat=( 151596934 125156135 204 ) utm=12 stm=2 core=6 HZ=100
  | stack=0xc87f6000-0xc87f8000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x09cb451f> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x09cb451f> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoDecodeExecutor-5" prio=5 tid=62 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de027a8 self=0xf015be00
  | sysTid=23877 nice=10 cgrp=default sched=0/0 handle=0xc87f3970
  | state=S schedstat=( 149726818 150505056 210 ) utm=14 stm=0 core=6 HZ=100
  | stack=0xc86f0000-0xc86f2000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0f96106c> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0f96106c> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoDecodeExecutor-6" prio=5 tid=64 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de02898 self=0xc979ea00
  | sysTid=23879 nice=10 cgrp=default sched=0/0 handle=0xc84f9970
  | state=S schedstat=( 141159746 121890523 215 ) utm=11 stm=2 core=5 HZ=100
  | stack=0xc83f6000-0xc83f8000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0b6e1635> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0b6e1635> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoDecodeExecutor-7" prio=5 tid=74 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de02988 self=0xc8664000
  | sysTid=23889 nice=10 cgrp=default sched=0/0 handle=0xc777f970
  | state=S schedstat=( 151943020 120844014 353 ) utm=11 stm=3 core=6 HZ=100
  | stack=0xc767c000-0xc767e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0aa1caca> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0aa1caca> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoDecodeExecutor-8" prio=5 tid=75 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de02a78 self=0xf015c400
  | sysTid=23890 nice=10 cgrp=default sched=0/0 handle=0xc74ff970
  | state=S schedstat=( 162992661 116541403 206 ) utm=15 stm=0 core=5 HZ=100
  | stack=0xc73fc000-0xc73fe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x04e9143b> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x04e9143b> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"OkHttp ConnectionPool" daemon prio=5 tid=110 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de02b68 self=0xc727da00
  | sysTid=23928 nice=0 cgrp=default sched=0/0 handle=0xc3d49970
  | state=S schedstat=( 24473228 41233334 205 ) utm=1 stm=0 core=2 HZ=100
  | stack=0xc3c46000-0xc3c48000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0c09c758> (a okhttp3.ConnectionPool)
  at okhttp3.ConnectionPool$1.run(SourceFile:67)
  - locked <0x0c09c758> (a okhttp3.ConnectionPool)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"Screencast Thread" daemon prio=5 tid=116 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de02d50 self=0xc61f4000
  | sysTid=23946 nice=0 cgrp=default sched=0/0 handle=0xce8ff970
  | state=S schedstat=( 236614 82969 6 ) utm=0 stm=0 core=5 HZ=100
  | stack=0xce7fc000-0xce7fe000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23946/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"Screencast Thread" daemon prio=5 tid=117 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de02e40 self=0xc61f4600
  | sysTid=23948 nice=0 cgrp=default sched=0/0 handle=0xcdaff970
  | state=S schedstat=( 467344 469531 3 ) utm=0 stm=0 core=5 HZ=100
  | stack=0xcd9fc000-0xcd9fe000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23948/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"Screencast Thread" daemon prio=5 tid=119 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de02f30 self=0xc61f4c00
  | sysTid=23956 nice=0 cgrp=default sched=0/0 handle=0xcca7f970
  | state=S schedstat=( 921083194 432469418 989 ) utm=89 stm=3 core=5 HZ=100
  | stack=0xcc97c000-0xcc97e000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23956/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"ContentCapture" prio=5 tid=120 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03020 self=0xf0cf9200
  | sysTid=23958 nice=0 cgrp=default sched=0/0 handle=0xcbdde970
  | state=S schedstat=( 30804626 23640892 159 ) utm=1 stm=1 core=0 HZ=100
  | stack=0xcbcdb000-0xcbcdd000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/23958/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"[io-thread]-431" prio=5 tid=121 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03110 self=0xf0cfa400
  | sysTid=24031 nice=0 cgrp=default sched=0/0 handle=0xcbeff970
  | state=S schedstat=( 4306824 55691508 23 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xcbdfc000-0xcbdfe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x00f6e0b1> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x00f6e0b1> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"FrescoBackgroundExecutor-2" prio=5 tid=123 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de031f0 self=0xd0dd6a00
  | sysTid=24097 nice=10 cgrp=default sched=0/0 handle=0xcbb2c970
  | state=S schedstat=( 2098022 2290259 6 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xcba29000-0xcba2b000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0b79c996> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0b79c996> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"[io-thread]-435" prio=5 tid=81 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de032e0 self=0xd1c45000
  | sysTid=24131 nice=0 cgrp=default sched=0/0 handle=0xc67f3970
  | state=S schedstat=( 2255779 2311772 13 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xc66f0000-0xc66f2000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0e4ab917> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0e4ab917> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"StethoListener-main" prio=5 tid=13 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de033c0 self=0xd1c45c00
  | sysTid=24152 nice=0 cgrp=default sched=0/0 handle=0xcb07f970
  | state=S schedstat=( 2639687 247866 11 ) utm=0 stm=0 core=4 HZ=100
  | stack=0xcaf7c000-0xcaf7e000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/24152/stack)
  native: #00 pc 00060c84  /system/lib/libc.so (__ppoll+20)
  native: #01 pc 000245fb  /system/lib/libc.so (poll+54)
  native: #02 pc 0001f9cb  /system/lib/libjavacore.so (Linux_poll(_JNIEnv*, _jobject*, _jobjectArray*, int)+578)
  at libcore.io.Linux.poll(Native method)
  at libcore.io.BlockGuardOs.poll(BlockGuardOs.java:219)
  at android.system.Os.poll(Os.java:374)
  at libcore.io.IoBridge.poll(IoBridge.java:712)
  at java.net.PlainSocketImpl.socketAccept(PlainSocketImpl.java:189)
  at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:451)
  at java.net.ServerSocket.implAccept(ServerSocket.java:547)
  at java.net.ServerSocket.accept(ServerSocket.java:515)
  at com.facebook.stetho.server.AbsServerSocket$ServerSocketImpl.accept(SourceFile:36)
  at com.facebook.stetho.server.LocalSocketServer.listenOnAddress(SourceFile:93)
  at com.facebook.stetho.server.LocalSocketServer.run(SourceFile:75)
  at com.facebook.stetho.server.ServerManager$1.run(SourceFile:39)

"Binder:23755_4" prio=5 tid=35 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03690 self=0xd136d800
  | sysTid=24159 nice=0 cgrp=default sched=0/0 handle=0xc6fff970
  | state=S schedstat=( 6912292 9753229 64 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xc6f04000-0xc6f06000 stackSize=1010KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/24159/stack)
  native: #00 pc 00060be8  /system/lib/libc.so (__ioctl+8)
  native: #01 pc 00022d6f  /system/lib/libc.so (ioctl+30)
  native: #02 pc 0004036d  /system/lib/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+204)
  native: #03 pc 000404c9  /system/lib/libbinder.so (android::IPCThreadState::getAndExecuteCommand()+8)
  native: #04 pc 00040a5d  /system/lib/libbinder.so (android::IPCThreadState::joinThreadPool(bool)+72)
  native: #05 pc 000581b5  /system/lib/libbinder.so (android::PoolThread::threadLoop()+12)
  native: #06 pc 0000c6af  /system/lib/libutils.so (android::Thread::_threadLoop(void*)+202)
  native: #07 pc 00070369  /system/lib/libandroid_runtime.so (android::AndroidRuntime::javaThreadShell(void*)+88)
  native: #08 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #09 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)
  (no managed stack frames)

"FrescoBackgroundExecutor-3" prio=5 tid=43 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03718 self=0xebfa9800
  | sysTid=24162 nice=10 cgrp=default sched=0/0 handle=0xc4dff970
  | state=S schedstat=( 1285990 3389687 11 ) utm=0 stm=0 core=7 HZ=100
  | stack=0xc4cfc000-0xc4cfe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x09000104> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x09000104> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"Screencast Thread" daemon prio=5 tid=50 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03808 self=0xcdb88400
  | sysTid=24184 nice=0 cgrp=default sched=0/0 handle=0xc9a30970
  | state=S schedstat=( 265208 120729 2 ) utm=0 stm=0 core=5 HZ=100
  | stack=0xc992d000-0xc992f000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/24184/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"Screencast Thread" daemon prio=5 tid=55 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de038f8 self=0xcdb88a00
  | sysTid=24186 nice=0 cgrp=default sched=0/0 handle=0xc8fa7970
  | state=S schedstat=( 11347917 15990 3 ) utm=1 stm=0 core=6 HZ=100
  | stack=0xc8ea4000-0xc8ea6000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/24186/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"Screencast Thread" daemon prio=5 tid=58 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de039e8 self=0xcdbfb000
  | sysTid=24189 nice=0 cgrp=default sched=0/0 handle=0xc79ee970
  | state=S schedstat=( 36634064875 2171318833 6105 ) utm=3369 stm=294 core=5 HZ=100
  | stack=0xc78eb000-0xc78ed000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/24189/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"FrescoBackgroundExecutor-4" prio=5 tid=14 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03ad8 self=0xd1a2ec00
  | sysTid=24485 nice=10 cgrp=default sched=0/0 handle=0xc547f970
  | state=S schedstat=( 2728229 1575782 6 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xc537c000-0xc537e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0267d6ed> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0267d6ed> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoBackgroundExecutor-5" prio=5 tid=11 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03bc8 self=0xf0cf9e00
  | sysTid=24521 nice=10 cgrp=default sched=0/0 handle=0xcd6ef970
  | state=S schedstat=( 2060053 1547083 7 ) utm=0 stm=0 core=4 HZ=100
  | stack=0xcd5ec000-0xcd5ee000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0200e922> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0200e922> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoBackgroundExecutor-6" prio=5 tid=45 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03cb8 self=0xd1a32800
  | sysTid=24563 nice=10 cgrp=default sched=0/0 handle=0xca97f970
  | state=S schedstat=( 782499 974116 5 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xca87c000-0xca87e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x03b7cfb3> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x03b7cfb3> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoBackgroundExecutor-7" prio=5 tid=47 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03da8 self=0xd1a2f200
  | sysTid=24576 nice=10 cgrp=default sched=0/0 handle=0xcb2bf970
  | state=S schedstat=( 1753543 637709 12 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xcb1bc000-0xcb1be000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0dfe2970> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0dfe2970> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"FrescoBackgroundExecutor-8" prio=5 tid=51 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03e98 self=0xcc3df000
  | sysTid=24577 nice=10 cgrp=default sched=0/0 handle=0xc6c79970
  | state=S schedstat=( 2203282 1117552 5 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xc6b76000-0xc6b78000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0277f4e9> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0277f4e9> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"ImageAttachExecutor-1" prio=5 tid=54 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de03f88 self=0xc8664600
  | sysTid=24582 nice=10 cgrp=default sched=0/0 handle=0xc192f970
  | state=S schedstat=( 1726199 7124479 10 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xc182c000-0xc182e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0ee8756e> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0ee8756e> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"ImageAttachExecutor-2" prio=5 tid=63 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de04120 self=0xc8664c00
  | sysTid=24583 nice=10 cgrp=default sched=0/0 handle=0xc1829970
  | state=S schedstat=( 1399738 5215155 17 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xc1726000-0xc1728000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0dd5b40f> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0dd5b40f> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"ImageAttachExecutor-3" prio=5 tid=65 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de04210 self=0xc8665200
  | sysTid=24584 nice=10 cgrp=default sched=0/0 handle=0xc1723970
  | state=S schedstat=( 1915417 2260418 11 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xc1620000-0xc1622000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x03746c9c> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x03746c9c> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"ImageAttachExecutor-4" prio=5 tid=66 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de04300 self=0xc8665800
  | sysTid=24585 nice=10 cgrp=default sched=0/0 handle=0xc161d970
  | state=S schedstat=( 529166 7751095 10 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xc151a000-0xc151c000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0e51f6a5> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0e51f6a5> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at com.facebook.imagepipeline.core.PriorityThreadFactory$1.run(SourceFile:53)
  at java.lang.Thread.run(Thread.java:764)

"StethoListener-main" prio=5 tid=40 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de04c08 self=0xd1c7ea00
  | sysTid=25509 nice=0 cgrp=default sched=0/0 handle=0xc9e48970
  | state=S schedstat=( 1588228 4403230 3 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xc9d45000-0xc9d47000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/25509/stack)
  native: #00 pc 00060c84  /system/lib/libc.so (__ppoll+20)
  native: #01 pc 000245fb  /system/lib/libc.so (poll+54)
  native: #02 pc 0001f9cb  /system/lib/libjavacore.so (Linux_poll(_JNIEnv*, _jobject*, _jobjectArray*, int)+578)
  at libcore.io.Linux.poll(Native method)
  at libcore.io.BlockGuardOs.poll(BlockGuardOs.java:219)
  at android.system.Os.poll(Os.java:374)
  at libcore.io.IoBridge.poll(IoBridge.java:712)
  at java.net.PlainSocketImpl.socketAccept(PlainSocketImpl.java:189)
  at java.net.AbstractPlainSocketImpl.accept(AbstractPlainSocketImpl.java:451)
  at java.net.ServerSocket.implAccept(ServerSocket.java:547)
  at java.net.ServerSocket.accept(ServerSocket.java:515)
  at com.facebook.stetho.server.AbsServerSocket$ServerSocketImpl.accept(SourceFile:36)
  at com.facebook.stetho.server.LocalSocketServer.listenOnAddress(SourceFile:93)
  at com.facebook.stetho.server.LocalSocketServer.run(SourceFile:75)
  at com.facebook.stetho.server.ServerManager$1.run(SourceFile:39)

"OkHttp ConnectionPool" daemon prio=5 tid=48 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de05078 self=0xcdb87e00
  | sysTid=25515 nice=0 cgrp=default sched=0/0 handle=0xc577f970
  | state=S schedstat=( 1161459 1464583 3 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xc567c000-0xc567e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0aaa7a7a> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0aaa7a7a> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:461)
  at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
  at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:937)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1282" prio=5 tid=86 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de05818 self=0xcdeefc00
  | sysTid=25553 nice=0 cgrp=default sched=0/0 handle=0xc0b7f970
  | state=S schedstat=( 317374347 280244750 4488 ) utm=17 stm=14 core=5 HZ=100
  | stack=0xc0a7c000-0xc0a7e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0043822b> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0043822b> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:868)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1021)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1328)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:232)
  at org.hapjs.render.a.b$a.a(SourceFile:300)
  at org.hapjs.render.a.b$a.a(SourceFile:286)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"Binder:23755_5" prio=5 tid=38 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1de05ab8 self=0xf0162800
  | sysTid=25712 nice=0 cgrp=default sched=0/0 handle=0xcb6bf970
  | state=S schedstat=( 6157553 11372551 33 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xcb5c4000-0xcb5c6000 stackSize=1010KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/25712/stack)
  native: #00 pc 00060be8  /system/lib/libc.so (__ioctl+8)
  native: #01 pc 00022d6f  /system/lib/libc.so (ioctl+30)
  native: #02 pc 0004036d  /system/lib/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+204)
  native: #03 pc 000404c9  /system/lib/libbinder.so (android::IPCThreadState::getAndExecuteCommand()+8)
  native: #04 pc 00040a5d  /system/lib/libbinder.so (android::IPCThreadState::joinThreadPool(bool)+72)
  native: #05 pc 000581b5  /system/lib/libbinder.so (android::PoolThread::threadLoop()+12)
  native: #06 pc 0000c6af  /system/lib/libutils.so (android::Thread::_threadLoop(void*)+202)
  native: #07 pc 00070369  /system/lib/libandroid_runtime.so (android::AndroidRuntime::javaThreadShell(void*)+88)
  native: #08 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #09 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)
  (no managed stack frames)

"RxCachedThreadScheduler-4" daemon prio=5 tid=12 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14d41568 self=0xf015ee00
  | sysTid=26444 nice=0 cgrp=default sched=0/0 handle=0xd14f3970
  | state=S schedstat=( 1982029 265834 11 ) utm=0 stm=0 core=5 HZ=100
  | stack=0xd13f0000-0xd13f2000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x06a9b688> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x06a9b688> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2059)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:1120)
  at java.util.concurrent.ScheduledThreadPoolExecutor$DelayedWorkQueue.take(ScheduledThreadPoolExecutor.java:849)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1092)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1433" prio=5 tid=22 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15e402e8 self=0xd1c18200
  | sysTid=26455 nice=0 cgrp=default sched=0/0 handle=0xcbcd8970
  | state=S schedstat=( 234225209 151253512 2678 ) utm=18 stm=4 core=6 HZ=100
  | stack=0xcbbd5000-0xcbbd7000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x01105821> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x01105821> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:868)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1021)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1328)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:232)
  at org.hapjs.render.a.b$a.a(SourceFile:300)
  at org.hapjs.render.a.b$a.a(SourceFile:286)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"OkHttp Dispatcher" prio=5 tid=44 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15e404a8 self=0xc6db9c00
  | sysTid=26458 nice=0 cgrp=default sched=0/0 handle=0xcb3d0970
  | state=S schedstat=( 198441104 35716663 601 ) utm=12 stm=6 core=1 HZ=100
  | stack=0xcb2cd000-0xcb2cf000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0955c446> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0955c446> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:461)
  at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
  at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:937)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"Okio Watchdog" daemon prio=5 tid=46 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15e40588 self=0xd1c7cc00
  | sysTid=26459 nice=0 cgrp=default sched=0/0 handle=0xcad7f970
  | state=S schedstat=( 22695986 3580314 405 ) utm=2 stm=0 core=0 HZ=100
  | stack=0xcac7c000-0xcac7e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0f761607> (a java.lang.Class<okio.AsyncTimeout>)
  at okio.AsyncTimeout.awaitTimeout(SourceFile:361)
  at okio.AsyncTimeout$Watchdog.run(SourceFile:312)
  - locked <0x0f761607> (a java.lang.Class<okio.AsyncTimeout>)

"OkHttp Dispatcher" prio=5 tid=68 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15e406f0 self=0xcd7fac00
  | sysTid=26468 nice=0 cgrp=default sched=0/0 handle=0xc93f9970
  | state=S schedstat=( 77451155 16130834 239 ) utm=2 stm=5 core=6 HZ=100
  | stack=0xc92f6000-0xc92f8000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0067b334> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0067b334> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:461)
  at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
  at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:937)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"OkHttp Dispatcher" prio=5 tid=69 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15e407d0 self=0xce023c00
  | sysTid=26470 nice=0 cgrp=default sched=0/0 handle=0xc8ea1970
  | state=S schedstat=( 29062344 10294008 127 ) utm=2 stm=0 core=1 HZ=100
  | stack=0xc8d9e000-0xc8da0000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x02f9555d> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x02f9555d> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:461)
  at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
  at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:937)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"OkHttp Dispatcher" prio=5 tid=70 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15e408b0 self=0xc5c54200
  | sysTid=26471 nice=0 cgrp=default sched=0/0 handle=0xc8d9b970
  | state=S schedstat=( 42561610 19505828 147 ) utm=4 stm=0 core=0 HZ=100
  | stack=0xc8c98000-0xc8c9a000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0f19ded2> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0f19ded2> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:461)
  at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
  at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:937)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"OkHttp Dispatcher" prio=5 tid=71 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15e40990 self=0xc4951400
  | sysTid=26472 nice=0 cgrp=default sched=0/0 handle=0xc6d7f970
  | state=S schedstat=( 36656931 20397813 129 ) utm=2 stm=1 core=6 HZ=100
  | stack=0xc6c7c000-0xc6c7e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x08540ba3> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x08540ba3> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:461)
  at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
  at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:937)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"OkHttp DiskLruCache" daemon prio=5 tid=72 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15e40a50 self=0xd1c7d200
  | sysTid=26485 nice=0 cgrp=default sched=0/0 handle=0xcaaff970
  | state=S schedstat=( 3053646 144792 4 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xca9fc000-0xca9fe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0a66cea0> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0a66cea0> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2101)
  at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1453" prio=5 tid=84 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14d42a98 self=0xcc472400
  | sysTid=26493 nice=0 cgrp=default sched=0/0 handle=0xc3c43970
  | state=S schedstat=( 133422850 245117400 3235 ) utm=10 stm=3 core=3 HZ=100
  | stack=0xc3b40000-0xc3b42000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.b(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.b$a.a(SourceFile:325)
  at org.hapjs.render.a.b$a.a(SourceFile:309)
  at org.hapjs.render.a.b$a.a(SourceFile:286)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1552" prio=5 tid=39 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14400540 self=0xcdef0200
  | sysTid=26621 nice=0 cgrp=default sched=0/0 handle=0xc40ff970
  | state=S schedstat=( 592375960 41933908 289 ) utm=55 stm=4 core=5 HZ=100
  | stack=0xc3ffc000-0xc3ffe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0193161e> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0193161e> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:868)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1021)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1328)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:232)
  at org.hapjs.render.a.b$a.a(SourceFile:300)
  at org.hapjs.render.a.b$a.a(SourceFile:286)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"OkHttp Dispatcher" prio=5 tid=73 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12d402d0 self=0xebfdde00
  | sysTid=26643 nice=0 cgrp=default sched=0/0 handle=0xbfeff970
  | state=S schedstat=( 37385259 11444273 87 ) utm=3 stm=0 core=6 HZ=100
  | stack=0xbfdfc000-0xbfdfe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0591beff> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0591beff> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.SynchronousQueue$TransferStack.awaitFulfill(SynchronousQueue.java:461)
  at java.util.concurrent.SynchronousQueue$TransferStack.transfer(SynchronousQueue.java:362)
  at java.util.concurrent.SynchronousQueue.poll(SynchronousQueue.java:937)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"JsThread" prio=5 tid=20 WaitingForGcToComplete
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12d40390 self=0xebfa9e00
  | sysTid=26652 nice=0 cgrp=default sched=0/0 handle=0xcd9f9970
  | state=S schedstat=( 1208851206 114013230 873 ) utm=112 stm=8 core=5 HZ=100
  | stack=0xcd8f6000-0xcd8f8000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26652/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 0017ba95  /system/lib/libart.so (art::gc::Heap::WaitForGcToCompleteLocked(art::gc::GcCause, art::Thread*)+244)
  native: #03 pc 00184b53  /system/lib/libart.so (art::gc::Heap::WaitForGcToComplete(art::gc::GcCause, art::Thread*)+186)
  native: #04 pc 0018272f  /system/lib/libart.so (art::gc::Heap::AllocateInternalWithGc(art::Thread*, art::gc::AllocatorType, bool, unsigned int, unsigned int*, unsigned int*, unsigned int*, art::ObjPtr<art::mirror::Class>*)+74)
  native: #05 pc 000f5b29  /system/lib/libart.so (art::mirror::Object* art::gc::Heap::AllocObjectWithAllocator<true, false, art::VoidFunctor>(art::Thread*, art::ObjPtr<art::gc::Heap::AllocObjectWithAllocator<true, false, art::VoidFunctor>::Class>, unsigned int, art::gc::AllocatorType, art::VoidFunctor const&)+656)
  native: #06 pc 002691ef  /system/lib/libart.so (art::JNI::NewObjectV(_JNIEnv*, _jclass*, _jmethodID*, std::__va_list)+558)
  native: #07 pc 00181961  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #08 pc 00186a81  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #09 pc 00186b67  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #10 pc 001f49d7  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #11 pc 001f43bb  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #12 pc 001f3e17  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #13 pc 007adf20  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  at com.eclipsesource.v8.V8._executeVoidFunction(Native method)
  at com.eclipsesource.v8.V8.executeVoidFunction(SourceFile:1154)
  at com.eclipsesource.v8.V8Object.executeVoidFunction(SourceFile:435)
  at org.hapjs.render.jsruntime.JsThread$a.handleMessage(SourceFile:244)
  at android.os.Handler.dispatchMessage(Handler.java:106)
  at android.os.Looper.loop(Looper.java:223)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"RenderActionThread" prio=5 tid=49 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12d405a0 self=0xebfdcc00
  | sysTid=26653 nice=0 cgrp=default sched=0/0 handle=0xcd37f970
  | state=S schedstat=( 167239 15105 1 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xcd27c000-0xcd27e000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26653/stack)
  native: #00 pc 00060aac  /system/lib/libc.so (__epoll_pwait+20)
  native: #01 pc 00027aa5  /system/lib/libc.so (epoll_wait+16)
  native: #02 pc 0000f9e9  /system/lib/libutils.so (android::Looper::pollInner(int)+112)
  native: #03 pc 0000f8fb  /system/lib/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+26)
  native: #04 pc 000bf417  /system/lib/libandroid_runtime.so (android::android_os_MessageQueue_nativePollOnce(_JNIEnv*, _jobject*, long long, int)+24)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:326)
  at android.os.Looper.loop(Looper.java:167)
  at android.os.HandlerThread.run(HandlerThread.java:65)

"[io]-1556" prio=5 tid=18 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12ec0e78 self=0xccd15200
  | sysTid=26658 nice=0 cgrp=default sched=0/0 handle=0xcb5c1970
  | state=S schedstat=( 29889422 22851094 252 ) utm=2 stm=0 core=5 HZ=100
  | stack=0xcb4be000-0xcb4c0000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0f7a34cc> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0f7a34cc> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:868)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1021)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1328)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:232)
  at org.hapjs.render.a.b$a.a(SourceFile:300)
  at org.hapjs.render.a.b$a.a(SourceFile:286)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1557" prio=5 tid=19 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12ec0f58 self=0xccd16400
  | sysTid=26659 nice=0 cgrp=default sched=0/0 handle=0xc9b43970
  | state=S schedstat=( 43973794 34217037 383 ) utm=4 stm=0 core=5 HZ=100
  | stack=0xc9a40000-0xc9a42000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x04e6d315> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x04e6d315> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:868)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1021)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1328)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:232)
  at org.hapjs.render.a.b$a.a(SourceFile:300)
  at org.hapjs.render.a.b$a.a(SourceFile:286)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1558" prio=5 tid=56 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12ec1038 self=0xccd17000
  | sysTid=26660 nice=0 cgrp=default sched=0/0 handle=0xc94ff970
  | state=S schedstat=( 29945998 23218079 503 ) utm=2 stm=0 core=6 HZ=100
  | stack=0xc93fc000-0xc93fe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x01d6762a> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x01d6762a> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:868)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1021)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1328)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:232)
  at org.hapjs.render.a.b$a.a(SourceFile:300)
  at org.hapjs.render.a.b$a.a(SourceFile:286)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1560" prio=5 tid=57 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12ec1118 self=0xf0cf9800
  | sysTid=26663 nice=0 cgrp=default sched=0/0 handle=0xc5379970
  | state=S schedstat=( 29581613 8099066 47 ) utm=2 stm=0 core=4 HZ=100
  | stack=0xc5276000-0xc5278000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0ecd4c1b> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0ecd4c1b> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:868)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1021)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1328)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:232)
  at org.hapjs.render.a.b$a.a(SourceFile:300)
  at org.hapjs.render.a.b$a.a(SourceFile:286)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1559" prio=5 tid=67 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x12ec11f8 self=0xccd17c00
  | sysTid=26662 nice=0 cgrp=default sched=0/0 handle=0xc7c7f970
  | state=S schedstat=( 46628527 53200583 861 ) utm=0 stm=4 core=4 HZ=100
  | stack=0xc7b7c000-0xc7b7e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x027bd1b8> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x027bd1b8> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:190)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.parkAndCheckInterrupt(AbstractQueuedSynchronizer.java:868)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1021)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireSharedInterruptibly(AbstractQueuedSynchronizer.java:1328)
  at java.util.concurrent.CountDownLatch.await(CountDownLatch.java:232)
  at org.hapjs.render.a.b$a.a(SourceFile:300)
  at org.hapjs.render.a.b$a.a(SourceFile:286)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"AsyncTask #19" prio=5 tid=76 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x135c11c0 self=0xcfaaec00
  | sysTid=26665 nice=0 cgrp=default sched=0/0 handle=0xc087f970
  | state=S schedstat=( 510677 65573 2 ) utm=0 stm=0 core=4 HZ=100
  | stack=0xc077c000-0xc077e000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x04ee8b91> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x04ee8b91> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2101)
  at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"AsyncTask #20" prio=5 tid=77 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x135c12a0 self=0xcfaaf800
  | sysTid=26666 nice=0 cgrp=default sched=0/0 handle=0xbe3ff970
  | state=S schedstat=( 452291 61459 2 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xbe2fc000-0xbe2fe000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0b3dcaf6> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0b3dcaf6> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2101)
  at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"AsyncTask #21" prio=5 tid=78 TimedWaiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x135c1380 self=0xebf89200
  | sysTid=26667 nice=0 cgrp=default sched=0/0 handle=0xbe2f9970
  | state=S schedstat=( 345625 65000 2 ) utm=0 stm=0 core=4 HZ=100
  | stack=0xbe1f6000-0xbe1f8000 stackSize=1042KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0d6a8ef7> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:2137)
  - locked <0x0d6a8ef7> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:358)
  at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:230)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.awaitNanos(AbstractQueuedSynchronizer.java:2101)
  at java.util.concurrent.LinkedBlockingQueue.poll(LinkedBlockingQueue.java:467)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1091)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1152)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1564" prio=5 tid=79 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15806e28 self=0xd0c0f600
  | sysTid=26683 nice=0 cgrp=default sched=0/0 handle=0xc4c3f970
  | state=S schedstat=( 31151045 39657190 904 ) utm=0 stm=3 core=1 HZ=100
  | stack=0xc4b3c000-0xc4b3e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1565" prio=5 tid=80 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15807008 self=0xc97d2a00
  | sysTid=26684 nice=0 cgrp=default sched=0/0 handle=0xc127f970
  | state=S schedstat=( 29764867 47861355 837 ) utm=2 stm=0 core=6 HZ=100
  | stack=0xc117c000-0xc117e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1566" prio=5 tid=82 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x158071b0 self=0xcc038a00
  | sysTid=26685 nice=0 cgrp=default sched=0/0 handle=0xc1179970
  | state=S schedstat=( 32567707 39153744 826 ) utm=0 stm=3 core=3 HZ=100
  | stack=0xc1076000-0xc1078000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1568" prio=5 tid=83 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x158073a0 self=0xcc038400
  | sysTid=26687 nice=0 cgrp=default sched=0/0 handle=0xc0a79970
  | state=S schedstat=( 28302184 51583802 627 ) utm=0 stm=2 core=0 HZ=100
  | stack=0xc0976000-0xc0978000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1569" prio=5 tid=85 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15807628 self=0xccd21000
  | sysTid=26688 nice=0 cgrp=default sched=0/0 handle=0xbfdf9970
  | state=S schedstat=( 22640883 58473647 649 ) utm=0 stm=2 core=0 HZ=100
  | stack=0xbfcf6000-0xbfcf8000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1571" prio=5 tid=87 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x158077d0 self=0xccd25e00
  | sysTid=26690 nice=0 cgrp=default sched=0/0 handle=0xbfbed970
  | state=S schedstat=( 658162623 53357922 676 ) utm=64 stm=1 core=2 HZ=100
  | stack=0xbfaea000-0xbfaec000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1567" prio=5 tid=88 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15807978 self=0xccd7a800
  | sysTid=26686 nice=0 cgrp=default sched=0/0 handle=0xc1073970
  | state=S schedstat=( 26908959 43301782 706 ) utm=1 stm=1 core=2 HZ=100
  | stack=0xc0f70000-0xc0f72000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1570" prio=5 tid=89 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15807b20 self=0xccd19a00
  | sysTid=26689 nice=0 cgrp=default sched=0/0 handle=0xbfcf3970
  | state=S schedstat=( 678995730 42601288 658 ) utm=66 stm=1 core=5 HZ=100
  | stack=0xbfbf0000-0xbfbf2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1573" prio=5 tid=90 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15807d28 self=0xc9696400
  | sysTid=26692 nice=0 cgrp=default sched=0/0 handle=0xbf9e1970
  | state=S schedstat=( 21444385 41857176 632 ) utm=1 stm=1 core=3 HZ=100
  | stack=0xbf8de000-0xbf8e0000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1574" prio=5 tid=91 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15807fe0 self=0xc6267600
  | sysTid=26693 nice=0 cgrp=default sched=0/0 handle=0xbf8db970
  | state=S schedstat=( 25042329 36320631 691 ) utm=0 stm=2 core=1 HZ=100
  | stack=0xbf7d8000-0xbf7da000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1572" prio=5 tid=92 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15808288 self=0xc6267000
  | sysTid=26691 nice=0 cgrp=default sched=0/0 handle=0xbfae7970
  | state=S schedstat=( 20702135 57783440 598 ) utm=1 stm=1 core=0 HZ=100
  | stack=0xbf9e4000-0xbf9e6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.a(SourceFile:315)
  at org.hapjs.render.css.l.<init>(SourceFile:24)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1575" prio=5 tid=93 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x158084c8 self=0xc6267c00
  | sysTid=26694 nice=0 cgrp=default sched=0/0 handle=0xbf7d5970
  | state=S schedstat=( 23394108 38419474 572 ) utm=0 stm=2 core=2 HZ=100
  | stack=0xbf6d2000-0xbf6d4000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1576" prio=5 tid=94 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15808670 self=0xc6268200
  | sysTid=26695 nice=0 cgrp=default sched=0/0 handle=0xbf6cf970
  | state=S schedstat=( 585796856 61248242 530 ) utm=58 stm=0 core=0 HZ=100
  | stack=0xbf5cc000-0xbf5ce000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1577" prio=5 tid=95 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x158088b8 self=0xc6268e00
  | sysTid=26696 nice=0 cgrp=default sched=0/0 handle=0xbf5c9970
  | state=S schedstat=( 20489695 31623898 527 ) utm=0 stm=2 core=2 HZ=100
  | stack=0xbf4c6000-0xbf4c8000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.a(SourceFile:315)
  at org.hapjs.render.css.l.<init>(SourceFile:24)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1578" prio=5 tid=96 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15808aa8 self=0xcc036c00
  | sysTid=26697 nice=0 cgrp=default sched=0/0 handle=0xbf4c3970
  | state=S schedstat=( 20541097 44590205 569 ) utm=2 stm=0 core=6 HZ=100
  | stack=0xbf3c0000-0xbf3c2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1579" prio=5 tid=97 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15808c88 self=0xcb825600
  | sysTid=26698 nice=0 cgrp=default sched=0/0 handle=0xbf3bd970
  | state=S schedstat=( 18996101 39148233 481 ) utm=1 stm=0 core=0 HZ=100
  | stack=0xbf2ba000-0xbf2bc000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1582" prio=5 tid=98 WaitingForGcToComplete
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15808e90 self=0xcb826800
  | sysTid=26701 nice=0 cgrp=default sched=0/0 handle=0xbe0ed970
  | state=S schedstat=( 682866530 34270257 465 ) utm=66 stm=1 core=1 HZ=100
  | stack=0xbdfea000-0xbdfec000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26701/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 0017ba95  /system/lib/libart.so (art::gc::Heap::WaitForGcToCompleteLocked(art::gc::GcCause, art::Thread*)+244)
  native: #03 pc 00184b53  /system/lib/libart.so (art::gc::Heap::WaitForGcToComplete(art::gc::GcCause, art::Thread*)+186)
  native: #04 pc 0018272f  /system/lib/libart.so (art::gc::Heap::AllocateInternalWithGc(art::Thread*, art::gc::AllocatorType, bool, unsigned int, unsigned int*, unsigned int*, unsigned int*, art::ObjPtr<art::mirror::Class>*)+74)
  native: #05 pc 003ccb8b  /system/lib/libart.so (artAllocObjectFromCodeInitializedRegionTLAB+246)
  native: #06 pc 00412d47  /system/lib/libart.so (art_quick_alloc_object_initialized_region_tlab+70)
  native: #07 pc 00016e8d  [anon:dalvik-jit-code-cache] (Java_org_hapjs_render_a_a_a__ILjava_lang_String_2+276)
  at org.hapjs.render.a.a.a(SourceFile:42)
  - locked <0x007aea59> (a org.hapjs.render.a.a)
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1584" prio=5 tid=99 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15809118 self=0xcb827400
  | sysTid=26703 nice=0 cgrp=default sched=0/0 handle=0xbdee1970
  | state=S schedstat=( 21666148 21036208 468 ) utm=0 stm=2 core=0 HZ=100
  | stack=0xbddde000-0xbdde0000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1580" prio=5 tid=100 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x158092f8 self=0xcb825c00
  | sysTid=26699 nice=0 cgrp=default sched=0/0 handle=0xbf2b7970
  | state=S schedstat=( 21143965 50154375 488 ) utm=2 stm=0 core=1 HZ=100
  | stack=0xbf1b4000-0xbf1b6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1583" prio=5 tid=101 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15809580 self=0xcb826e00
  | sysTid=26702 nice=0 cgrp=default sched=0/0 handle=0xbdfe7970
  | state=S schedstat=( 612054038 37088751 451 ) utm=57 stm=3 core=2 HZ=100
  | stack=0xbdee4000-0xbdee6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1581" prio=5 tid=102 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15809770 self=0xcb826200
  | sysTid=26700 nice=0 cgrp=default sched=0/0 handle=0xbe1f3970
  | state=S schedstat=( 18824460 55582663 441 ) utm=0 stm=1 core=3 HZ=100
  | stack=0xbe0f0000-0xbe0f2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.a(SourceFile:315)
  at org.hapjs.render.css.l.<init>(SourceFile:24)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1586" prio=5 tid=103 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15809950 self=0xcb712000
  | sysTid=26705 nice=0 cgrp=default sched=0/0 handle=0xbdcd5970
  | state=S schedstat=( 16884383 34278181 428 ) utm=0 stm=1 core=0 HZ=100
  | stack=0xbdbd2000-0xbdbd4000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1587" prio=5 tid=104 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15800000 self=0xcb712600
  | sysTid=26706 nice=0 cgrp=default sched=0/0 handle=0xbdbcf970
  | state=S schedstat=( 19216081 41231830 516 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xbdacc000-0xbdace000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1588" prio=5 tid=105 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15809af8 self=0xcb712c00
  | sysTid=26707 nice=0 cgrp=default sched=0/0 handle=0xbdac9970
  | state=S schedstat=( 15324221 37788496 422 ) utm=1 stm=0 core=1 HZ=100
  | stack=0xbd9c6000-0xbd9c8000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1585" prio=5 tid=106 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15809cd8 self=0xcb827a00
  | sysTid=26704 nice=0 cgrp=default sched=0/0 handle=0xbdddb970
  | state=S schedstat=( 17140586 43480559 419 ) utm=0 stm=1 core=5 HZ=100
  | stack=0xbdcd8000-0xbdcda000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1590" prio=5 tid=107 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15809ec8 self=0xcb713800
  | sysTid=26709 nice=0 cgrp=default sched=0/0 handle=0xbd8bd970
  | state=S schedstat=( 645920325 27802127 395 ) utm=64 stm=0 core=6 HZ=100
  | stack=0xbd7ba000-0xbd7bc000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1591" prio=5 tid=108 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580a0b8 self=0xcb713e00
  | sysTid=26710 nice=0 cgrp=default sched=0/0 handle=0xbd7b7970
  | state=S schedstat=( 15078538 34966624 392 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xbd6b4000-0xbd6b6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1592" prio=5 tid=109 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580a2a8 self=0xcb714400
  | sysTid=26711 nice=0 cgrp=default sched=0/0 handle=0xbd6b1970
  | state=S schedstat=( 20451453 33366771 428 ) utm=0 stm=2 core=3 HZ=100
  | stack=0xbd5ae000-0xbd5b0000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1589" prio=5 tid=111 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580a488 self=0xcb713200
  | sysTid=26708 nice=0 cgrp=default sched=0/0 handle=0xbd9c3970
  | state=S schedstat=( 19069848 49674169 407 ) utm=0 stm=1 core=4 HZ=100
  | stack=0xbd8c0000-0xbd8c2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1594" prio=5 tid=112 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580a6d0 self=0xc98a9c00
  | sysTid=26713 nice=0 cgrp=default sched=0/0 handle=0xbd4a5970
  | state=S schedstat=( 14833956 20320220 363 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xbd3a2000-0xbd3a4000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1596" prio=5 tid=113 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580a950 self=0xc5da1000
  | sysTid=26715 nice=0 cgrp=default sched=0/0 handle=0xbc8af970
  | state=S schedstat=( 13823753 30666969 407 ) utm=1 stm=0 core=0 HZ=100
  | stack=0xbc7ac000-0xbc7ae000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1597" prio=5 tid=114 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580abd8 self=0xc581f200
  | sysTid=26716 nice=0 cgrp=default sched=0/0 handle=0xbc7a9970
  | state=S schedstat=( 14805002 36074741 436 ) utm=0 stm=1 core=1 HZ=100
  | stack=0xbc6a6000-0xbc6a8000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1595" prio=5 tid=115 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580ad80 self=0xc98aa200
  | sysTid=26714 nice=0 cgrp=default sched=0/0 handle=0xbd39f970
  | state=S schedstat=( 16201569 34851446 432 ) utm=0 stm=1 core=2 HZ=100
  | stack=0xbd29c000-0xbd29e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1593" prio=5 tid=118 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580b008 self=0xcb714a00
  | sysTid=26712 nice=0 cgrp=default sched=0/0 handle=0xbd5ab970
  | state=S schedstat=( 575270509 37748394 357 ) utm=56 stm=0 core=1 HZ=100
  | stack=0xbd4a8000-0xbd4aa000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1598" prio=5 tid=122 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580b1e8 self=0xc5da1600
  | sysTid=26717 nice=0 cgrp=default sched=0/0 handle=0xbc6a3970
  | state=S schedstat=( 20997338 16040474 318 ) utm=2 stm=0 core=1 HZ=100
  | stack=0xbc5a0000-0xbc5a2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1601" prio=5 tid=124 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580b3d8 self=0xca7f9000
  | sysTid=26720 nice=0 cgrp=default sched=0/0 handle=0xbc391970
  | state=S schedstat=( 13598432 31275687 456 ) utm=0 stm=1 core=3 HZ=100
  | stack=0xbc28e000-0xbc290000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1600" prio=5 tid=125 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580b660 self=0xced7fa00
  | sysTid=26719 nice=0 cgrp=default sched=0/0 handle=0xbc497970
  | state=S schedstat=( 14936931 26142812 360 ) utm=1 stm=0 core=1 HZ=100
  | stack=0xbc394000-0xbc396000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.a(SourceFile:315)
  at org.hapjs.render.css.l.<init>(SourceFile:24)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1599" prio=5 tid=126 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580b8e8 self=0xced7f400
  | sysTid=26718 nice=0 cgrp=default sched=0/0 handle=0xbc59d970
  | state=S schedstat=( 13836044 22686932 310 ) utm=0 stm=1 core=2 HZ=100
  | stack=0xbc49a000-0xbc49c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1603" prio=5 tid=127 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580bb70 self=0xc8c5f400
  | sysTid=26722 nice=0 cgrp=default sched=0/0 handle=0xbc185970
  | state=S schedstat=( 11591921 26126150 355 ) utm=0 stm=1 core=1 HZ=100
  | stack=0xbc082000-0xbc084000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1604" prio=5 tid=128 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580bd60 self=0xc8c5fa00
  | sysTid=26723 nice=0 cgrp=default sched=0/0 handle=0xbc07f970
  | state=S schedstat=( 580593268 27658448 356 ) utm=55 stm=2 core=0 HZ=100
  | stack=0xbbf7c000-0xbbf7e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.a(SourceFile:315)
  at org.hapjs.render.css.l.<init>(SourceFile:24)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1602" prio=5 tid=129 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580bf50 self=0xc8c5ee00
  | sysTid=26721 nice=0 cgrp=default sched=0/0 handle=0xbc28b970
  | state=S schedstat=( 666850135 35978116 338 ) utm=64 stm=1 core=0 HZ=100
  | stack=0xbc188000-0xbc18a000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1605" prio=5 tid=130 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580c178 self=0xc85df000
  | sysTid=26724 nice=0 cgrp=default sched=0/0 handle=0xbbf79970
  | state=S schedstat=( 678696707 28762760 374 ) utm=65 stm=1 core=1 HZ=100
  | stack=0xbbe76000-0xbbe78000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1606" prio=5 tid=131 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580c358 self=0xc85df600
  | sysTid=26725 nice=0 cgrp=default sched=0/0 handle=0xbbe73970
  | state=S schedstat=( 13574957 33995112 320 ) utm=0 stm=1 core=1 HZ=100
  | stack=0xbbd70000-0xbbd72000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1607" prio=5 tid=132 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580c560 self=0xc85e3800
  | sysTid=26726 nice=0 cgrp=default sched=0/0 handle=0xbbd6d970
  | state=S schedstat=( 15186304 24506831 355 ) utm=0 stm=1 core=1 HZ=100
  | stack=0xbbc6a000-0xbbc6c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1609" prio=5 tid=133 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580c788 self=0xc6137c00
  | sysTid=26728 nice=0 cgrp=default sched=0/0 handle=0xbbb61970
  | state=S schedstat=( 11537391 37398968 421 ) utm=1 stm=0 core=2 HZ=100
  | stack=0xbba5e000-0xbba60000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1608" prio=5 tid=134 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580c968 self=0xc85e3e00
  | sysTid=26727 nice=0 cgrp=default sched=0/0 handle=0xbbc67970
  | state=S schedstat=( 656639583 30322493 305 ) utm=63 stm=1 core=0 HZ=100
  | stack=0xbbb64000-0xbbb66000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1610" prio=5 tid=135 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580cbf0 self=0xc6138200
  | sysTid=26729 nice=0 cgrp=default sched=0/0 handle=0xbba5b970
  | state=S schedstat=( 11574425 27518756 342 ) utm=1 stm=0 core=2 HZ=100
  | stack=0xbb958000-0xbb95a000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1611" prio=5 tid=136 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580cde0 self=0xc51ec400
  | sysTid=26730 nice=0 cgrp=default sched=0/0 handle=0xbb955970
  | state=S schedstat=( 17332338 41700979 412 ) utm=0 stm=1 core=6 HZ=100
  | stack=0xbb852000-0xbb854000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1612" prio=5 tid=137 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580cfc0 self=0xc51eca00
  | sysTid=26731 nice=0 cgrp=default sched=0/0 handle=0xbb84f970
  | state=S schedstat=( 14067975 25214161 359 ) utm=0 stm=1 core=1 HZ=100
  | stack=0xbb74c000-0xbb74e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1613" prio=5 tid=138 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580d1a0 self=0xc51f0c00
  | sysTid=26732 nice=0 cgrp=default sched=0/0 handle=0xbb749970
  | state=S schedstat=( 677507382 27162866 325 ) utm=65 stm=2 core=0 HZ=100
  | stack=0xbb646000-0xbb648000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1614" prio=5 tid=139 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580d390 self=0xc51f1200
  | sysTid=26733 nice=0 cgrp=default sched=0/0 handle=0xbb643970
  | state=S schedstat=( 12156356 28424419 255 ) utm=1 stm=0 core=3 HZ=100
  | stack=0xbb540000-0xbb542000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.a(SourceFile:315)
  at org.hapjs.render.css.l.<init>(SourceFile:24)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1615" prio=5 tid=140 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580d580 self=0xcc078400
  | sysTid=26734 nice=0 cgrp=default sched=0/0 handle=0xbb53d970
  | state=S schedstat=( 11916193 24854163 264 ) utm=0 stm=1 core=0 HZ=100
  | stack=0xbb43a000-0xbb43c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.a(SourceFile:315)
  at org.hapjs.render.css.l.<init>(SourceFile:24)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1617" prio=5 tid=141 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580d7a8 self=0xcc07cc00
  | sysTid=26736 nice=0 cgrp=default sched=0/0 handle=0xbb331970
  | state=S schedstat=( 15147712 28417552 274 ) utm=0 stm=1 core=3 HZ=100
  | stack=0xbb22e000-0xbb230000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1618" prio=5 tid=142 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580d998 self=0xcc07d200
  | sysTid=26737 nice=0 cgrp=default sched=0/0 handle=0xbb22b970
  | state=S schedstat=( 12329428 23640161 274 ) utm=1 stm=0 core=3 HZ=100
  | stack=0xbb128000-0xbb12a000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1620" prio=5 tid=143 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580db40 self=0xcc07de00
  | sysTid=26739 nice=0 cgrp=default sched=0/0 handle=0xbb01f970
  | state=S schedstat=( 12153170 27542245 288 ) utm=1 stm=0 core=1 HZ=100
  | stack=0xbaf1c000-0xbaf1e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1619" prio=5 tid=144 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580dce8 self=0xcc07d800
  | sysTid=26738 nice=0 cgrp=default sched=0/0 handle=0xbb125970
  | state=S schedstat=( 16254231 37213751 295 ) utm=0 stm=1 core=3 HZ=100
  | stack=0xbb022000-0xbb024000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1616" prio=5 tid=145 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580df70 self=0xcc078a00
  | sysTid=26735 nice=0 cgrp=default sched=0/0 handle=0xbb437970
  | state=S schedstat=( 10976556 26018128 247 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xbb334000-0xbb336000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1623" prio=5 tid=146 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15806290 self=0xcb86a800
  | sysTid=26742 nice=0 cgrp=default sched=0/0 handle=0xbad0d970
  | state=S schedstat=( 14277861 23772559 315 ) utm=0 stm=1 core=3 HZ=100
  | stack=0xbac0a000-0xbac0c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1622" prio=5 tid=147 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580e118 self=0xcb8c3600
  | sysTid=26741 nice=0 cgrp=default sched=0/0 handle=0xbae13970
  | state=S schedstat=( 11724271 14466978 273 ) utm=1 stm=0 core=0 HZ=100
  | stack=0xbad10000-0xbad12000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1624" prio=5 tid=148 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580e308 self=0xcb86ae00
  | sysTid=26743 nice=0 cgrp=default sched=0/0 handle=0xbac07970
  | state=S schedstat=( 11678801 14162969 220 ) utm=1 stm=0 core=1 HZ=100
  | stack=0xbab04000-0xbab06000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1621" prio=5 tid=149 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580e590 self=0xcb8c3000
  | sysTid=26740 nice=0 cgrp=default sched=0/0 handle=0xbaf19970
  | state=S schedstat=( 659168196 22026508 272 ) utm=65 stm=0 core=6 HZ=100
  | stack=0xbae16000-0xbae18000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1626" prio=5 tid=150 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580e780 self=0xcb78d200
  | sysTid=26745 nice=0 cgrp=default sched=0/0 handle=0xba9fb970
  | state=S schedstat=( 11372865 20710887 229 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xba8f8000-0xba8fa000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1627" prio=5 tid=151 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580e970 self=0xcb78d800
  | sysTid=26746 nice=0 cgrp=default sched=0/0 handle=0xba8f5970
  | state=S schedstat=( 12568225 18978894 255 ) utm=0 stm=1 core=1 HZ=100
  | stack=0xba7f2000-0xba7f4000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1628" prio=5 tid=152 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580ebf8 self=0xcb78de00
  | sysTid=26747 nice=0 cgrp=default sched=0/0 handle=0xba7ef970
  | state=S schedstat=( 11434831 29553386 270 ) utm=0 stm=1 core=1 HZ=100
  | stack=0xba6ec000-0xba6ee000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1625" prio=5 tid=153 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580eda0 self=0xcb78cc00
  | sysTid=26744 nice=0 cgrp=default sched=0/0 handle=0xbab01970
  | state=S schedstat=( 11911712 26330680 269 ) utm=1 stm=0 core=2 HZ=100
  | stack=0xba9fe000-0xbaa00000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1629" prio=5 tid=154 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580ef90 self=0xcb78e400
  | sysTid=26748 nice=0 cgrp=default sched=0/0 handle=0xba6e9970
  | state=S schedstat=( 10222493 28007969 283 ) utm=1 stm=0 core=1 HZ=100
  | stack=0xba5e6000-0xba5e8000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1631" prio=5 tid=155 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580f180 self=0xcb792c00
  | sysTid=26750 nice=0 cgrp=default sched=0/0 handle=0xba4dd970
  | state=S schedstat=( 671198414 25386612 282 ) utm=63 stm=4 core=2 HZ=100
  | stack=0xba3da000-0xba3dc000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1630" prio=5 tid=156 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580f328 self=0xcb78ea00
  | sysTid=26749 nice=0 cgrp=default sched=0/0 handle=0xba5e3970
  | state=S schedstat=( 654276976 22170103 230 ) utm=63 stm=2 core=2 HZ=100
  | stack=0xba4e0000-0xba4e2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1632" prio=5 tid=157 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580f530 self=0xcb793200
  | sysTid=26751 nice=0 cgrp=default sched=0/0 handle=0xba3d7970
  | state=S schedstat=( 9819751 36845252 287 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xba2d4000-0xba2d6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1633" prio=5 tid=158 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580f710 self=0xcb793800
  | sysTid=26752 nice=0 cgrp=default sched=0/0 handle=0xba2d1970
  | state=S schedstat=( 13727034 28320726 229 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xba1ce000-0xba1d0000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1634" prio=5 tid=159 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580f918 self=0xcb793e00
  | sysTid=26753 nice=0 cgrp=default sched=0/0 handle=0xba1cb970
  | state=S schedstat=( 11465420 19124221 238 ) utm=1 stm=0 core=1 HZ=100
  | stack=0xba0c8000-0xba0ca000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1635" prio=5 tid=160 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580fba0 self=0xcb794400
  | sysTid=26754 nice=0 cgrp=default sched=0/0 handle=0xba095970
  | state=S schedstat=( 11522552 19286816 299 ) utm=0 stm=1 core=0 HZ=100
  | stack=0xb9f92000-0xb9f94000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1636" prio=5 tid=161 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x1580fe28 self=0xcb794a00
  | sysTid=26755 nice=0 cgrp=default sched=0/0 handle=0xb9f8f970
  | state=S schedstat=( 12691304 21003852 221 ) utm=1 stm=0 core=0 HZ=100
  | stack=0xb9e8c000-0xb9e8e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1637" prio=5 tid=162 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15810018 self=0xcb148c00
  | sysTid=26756 nice=0 cgrp=default sched=0/0 handle=0xb9e89970
  | state=S schedstat=( 13975214 26427647 232 ) utm=1 stm=0 core=0 HZ=100
  | stack=0xb9d86000-0xb9d88000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1638" prio=5 tid=163 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x158102a0 self=0xcb149200
  | sysTid=26757 nice=0 cgrp=default sched=0/0 handle=0xb9d83970
  | state=S schedstat=( 14399067 25110982 271 ) utm=0 stm=1 core=0 HZ=100
  | stack=0xb9c80000-0xb9c82000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1639" prio=5 tid=164 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15810480 self=0xcb149800
  | sysTid=26758 nice=0 cgrp=default sched=0/0 handle=0xb9c7d970
  | state=S schedstat=( 10042855 20721457 199 ) utm=0 stm=1 core=3 HZ=100
  | stack=0xb9b7a000-0xb9b7c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1641" prio=5 tid=165 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15810690 self=0xcb14a400
  | sysTid=26760 nice=0 cgrp=default sched=0/0 handle=0xb99f9970
  | state=S schedstat=( 11370321 21098586 191 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb98f6000-0xb98f8000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1642" prio=5 tid=166 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x158108d8 self=0xcb14aa00
  | sysTid=26761 nice=0 cgrp=default sched=0/0 handle=0xb98f3970
  | state=S schedstat=( 8901086 13903228 157 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb97f0000-0xb97f2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1643" prio=5 tid=167 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15810b78 self=0xd1310c00
  | sysTid=26762 nice=0 cgrp=default sched=0/0 handle=0xb97ed970
  | state=S schedstat=( 13326716 25241938 221 ) utm=1 stm=0 core=1 HZ=100
  | stack=0xb96ea000-0xb96ec000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1644" prio=5 tid=168 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15806480 self=0xd1311200
  | sysTid=26763 nice=0 cgrp=default sched=0/0 handle=0xb96e7970
  | state=S schedstat=( 11765676 28499685 203 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb95e4000-0xb95e6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1640" prio=5 tid=169 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x15806578 self=0xcb149e00
  | sysTid=26759 nice=0 cgrp=default sched=0/0 handle=0xb9aff970
  | state=S schedstat=( 9029847 21424474 172 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb99fc000-0xb99fe000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1645" prio=5 tid=170 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x143022d0 self=0xebf8b000
  | sysTid=26764 nice=0 cgrp=default sched=0/0 handle=0xb95e1970
  | state=S schedstat=( 13833543 16039426 168 ) utm=0 stm=1 core=0 HZ=100
  | stack=0xb94de000-0xb94e0000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1646" prio=5 tid=171 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x143024b0 self=0xc9696a00
  | sysTid=26765 nice=0 cgrp=default sched=0/0 handle=0xb94db970
  | state=S schedstat=( 8972861 30160209 191 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb93d8000-0xb93da000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1648" prio=5 tid=172 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14302658 self=0xebfdd200
  | sysTid=26767 nice=0 cgrp=default sched=0/0 handle=0xb92cf970
  | state=S schedstat=( 14780736 28953589 178 ) utm=1 stm=0 core=0 HZ=100
  | stack=0xb91cc000-0xb91ce000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1647" prio=5 tid=173 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14302c98 self=0xebf8b600
  | sysTid=26766 nice=0 cgrp=default sched=0/0 handle=0xb93d5970
  | state=S schedstat=( 10846567 25865726 179 ) utm=1 stm=0 core=0 HZ=100
  | stack=0xb92d2000-0xb92d4000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1649" prio=5 tid=174 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14302838 self=0xebfdd800
  | sysTid=26768 nice=0 cgrp=default sched=0/0 handle=0xb91c9970
  | state=S schedstat=( 7739485 19726717 161 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb90c6000-0xb90c8000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1650" prio=5 tid=175 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14302a18 self=0xc8669a00
  | sysTid=26769 nice=0 cgrp=default sched=0/0 handle=0xb90c3970
  | state=S schedstat=( 8678690 24029274 178 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb8fc0000-0xb8fc2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1651" prio=5 tid=176 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14302bc0 self=0xc6268800
  | sysTid=26770 nice=0 cgrp=default sched=0/0 handle=0xb8fbd970
  | state=S schedstat=( 10795667 18113808 138 ) utm=0 stm=1 core=2 HZ=100
  | stack=0xb8eba000-0xb8ebc000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.a(SourceFile:315)
  at org.hapjs.render.css.l.<init>(SourceFile:24)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1652" prio=5 tid=177 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14181730 self=0xcb86b400
  | sysTid=26771 nice=0 cgrp=default sched=0/0 handle=0xb8eb7970
  | state=S schedstat=( 11094731 22814224 195 ) utm=1 stm=0 core=0 HZ=100
  | stack=0xb8db4000-0xb8db6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1653" prio=5 tid=178 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x141819f8 self=0xcb86f000
  | sysTid=26772 nice=0 cgrp=default sched=0/0 handle=0xb8db1970
  | state=S schedstat=( 570725788 15401981 156 ) utm=56 stm=1 core=2 HZ=100
  | stack=0xb8cae000-0xb8cb0000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1654" prio=5 tid=179 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14181920 self=0xc8c5b200
  | sysTid=26773 nice=0 cgrp=default sched=0/0 handle=0xb8cab970
  | state=S schedstat=( 7457081 27315669 168 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb8ba8000-0xb8baa000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1655" prio=5 tid=180 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x141418b8 self=0xc8c5b800
  | sysTid=26774 nice=0 cgrp=default sched=0/0 handle=0xb8ba5970
  | state=S schedstat=( 644304691 23283276 163 ) utm=64 stm=0 core=1 HZ=100
  | stack=0xb8aa2000-0xb8aa4000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1656" prio=5 tid=181 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14141b40 self=0xc209d400
  | sysTid=26775 nice=0 cgrp=default sched=0/0 handle=0xb8a9f970
  | state=S schedstat=( 6347917 16137086 164 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb899c000-0xb899e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1657" prio=5 tid=182 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14101680 self=0xc209da00
  | sysTid=26776 nice=0 cgrp=default sched=0/0 handle=0xb8999970
  | state=S schedstat=( 7527294 14444171 155 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb8896000-0xb8898000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1658" prio=5 tid=183 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14101828 self=0xcec38c00
  | sysTid=26777 nice=0 cgrp=default sched=0/0 handle=0xb8893970
  | state=S schedstat=( 10968080 21165467 168 ) utm=1 stm=0 core=0 HZ=100
  | stack=0xb8790000-0xb8792000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1659" prio=5 tid=184 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x140c0098 self=0xcec39200
  | sysTid=26778 nice=0 cgrp=default sched=0/0 handle=0xb878d970
  | state=S schedstat=( 9937655 19267711 137 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb868a000-0xb868c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1660" prio=5 tid=185 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x140812e0 self=0xcfb64200
  | sysTid=26779 nice=0 cgrp=default sched=0/0 handle=0xb8687970
  | state=S schedstat=( 8068695 17036098 108 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb8584000-0xb8586000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:308)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1661" prio=5 tid=186 WaitingForGcToComplete
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x140814d0 self=0xcfb64800
  | sysTid=26780 nice=0 cgrp=default sched=0/0 handle=0xb8581970
  | state=S schedstat=( 11506820 11696248 128 ) utm=1 stm=0 core=3 HZ=100
  | stack=0xb847e000-0xb8480000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26780/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 0017ba95  /system/lib/libart.so (art::gc::Heap::WaitForGcToCompleteLocked(art::gc::GcCause, art::Thread*)+244)
  native: #03 pc 00184b53  /system/lib/libart.so (art::gc::Heap::WaitForGcToComplete(art::gc::GcCause, art::Thread*)+186)
  native: #04 pc 0018272f  /system/lib/libart.so (art::gc::Heap::AllocateInternalWithGc(art::Thread*, art::gc::AllocatorType, bool, unsigned int, unsigned int*, unsigned int*, unsigned int*, art::ObjPtr<art::mirror::Class>*)+74)
  native: #05 pc 003ccb8b  /system/lib/libart.so (artAllocObjectFromCodeInitializedRegionTLAB+246)
  native: #06 pc 00412d47  /system/lib/libart.so (art_quick_alloc_object_initialized_region_tlab+70)
  native: #07 pc 00018b13  [anon:dalvik-jit-code-cache] (Java_org_hapjs_render_css_a_a__Lorg_hapjs_render_css_n_2+1506)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1662" prio=5 tid=187 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x140816c0 self=0xcfb64e00
  | sysTid=26781 nice=0 cgrp=default sched=0/0 handle=0xb847b970
  | state=S schedstat=( 9910675 25879582 99 ) utm=0 stm=0 core=5 HZ=100
  | stack=0xb8378000-0xb837a000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1663" prio=5 tid=188 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14001730 self=0xcfb65400
  | sysTid=26782 nice=0 cgrp=default sched=0/0 handle=0xb8375970
  | state=S schedstat=( 12490829 10020834 89 ) utm=0 stm=1 core=2 HZ=100
  | stack=0xb8272000-0xb8274000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1664" prio=5 tid=189 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14001920 self=0xc7215600
  | sysTid=26783 nice=0 cgrp=default sched=0/0 handle=0xb826f970
  | state=S schedstat=( 7303071 3509846 76 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb816c000-0xb816e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1665" prio=5 tid=190 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14001ac8 self=0xc7215c00
  | sysTid=26784 nice=0 cgrp=default sched=0/0 handle=0xb8169970
  | state=S schedstat=( 7003599 9288587 72 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb8066000-0xb8068000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1666" prio=5 tid=191 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14001ca8 self=0xc7216200
  | sysTid=26785 nice=0 cgrp=default sched=0/0 handle=0xb8063970
  | state=S schedstat=( 6205990 5916720 57 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb7f60000-0xb7f62000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1667" prio=5 tid=192 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x14001e88 self=0xc7216800
  | sysTid=26786 nice=0 cgrp=default sched=0/0 handle=0xb7f5d970
  | state=S schedstat=( 636406255 6385111 94 ) utm=61 stm=1 core=0 HZ=100
  | stack=0xb7e5a000-0xb7e5c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1668" prio=5 tid=193 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13f80ee0 self=0xd130d000
  | sysTid=26787 nice=0 cgrp=default sched=0/0 handle=0xb7e57970
  | state=S schedstat=( 5623283 2681558 67 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb7d54000-0xb7d56000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1669" prio=5 tid=194 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13f810d0 self=0xc6134600
  | sysTid=26788 nice=0 cgrp=default sched=0/0 handle=0xb7d51970
  | state=S schedstat=( 6943594 9056822 73 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb7c4e000-0xb7c50000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1670" prio=5 tid=195 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13f812b0 self=0xc51df000
  | sysTid=26789 nice=0 cgrp=default sched=0/0 handle=0xb7bff970
  | state=S schedstat=( 4987708 9689949 79 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb7afc000-0xb7afe000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1671" prio=5 tid=196 WaitingForGcToComplete
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13f81490 self=0xcc07ea00
  | sysTid=26790 nice=0 cgrp=default sched=0/0 handle=0xb7af9970
  | state=S schedstat=( 7595941 7193906 71 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb79f6000-0xb79f8000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26790/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 0017ba95  /system/lib/libart.so (art::gc::Heap::WaitForGcToCompleteLocked(art::gc::GcCause, art::Thread*)+244)
  native: #03 pc 00184b53  /system/lib/libart.so (art::gc::Heap::WaitForGcToComplete(art::gc::GcCause, art::Thread*)+186)
  native: #04 pc 0018272f  /system/lib/libart.so (art::gc::Heap::AllocateInternalWithGc(art::Thread*, art::gc::AllocatorType, bool, unsigned int, unsigned int*, unsigned int*, unsigned int*, art::ObjPtr<art::mirror::Class>*)+74)
  native: #05 pc 003ccb8b  /system/lib/libart.so (artAllocObjectFromCodeInitializedRegionTLAB+246)
  native: #06 pc 00412d47  /system/lib/libart.so (art_quick_alloc_object_initialized_region_tlab+70)
  native: #07 pc 00019e47  [anon:dalvik-jit-code-cache] (Java_org_hapjs_render_VDomChangeAction__0003cinit_0003e__+430)
  at org.hapjs.render.VDomChangeAction.<init>(SourceFile:53)
  at org.hapjs.render.a.d.a(SourceFile:35)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1672" prio=5 tid=197 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13f81638 self=0xc581f800
  | sysTid=26791 nice=0 cgrp=default sched=0/0 handle=0xb79f3970
  | state=S schedstat=( 4408328 2598957 54 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb78f0000-0xb78f2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1673" prio=5 tid=198 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13f81828 self=0xc2091a00
  | sysTid=26792 nice=0 cgrp=default sched=0/0 handle=0xb78ed970
  | state=S schedstat=( 4967391 4931248 66 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb77ea000-0xb77ec000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1674" prio=5 tid=199 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13f819d0 self=0xc85ae800
  | sysTid=26793 nice=0 cgrp=default sched=0/0 handle=0xb77e7970
  | state=S schedstat=( 6578591 11740573 70 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb76e4000-0xb76e6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.a(SourceFile:315)
  at org.hapjs.render.css.l.<init>(SourceFile:24)
  at org.hapjs.render.css.a.a(SourceFile:275)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1675" prio=5 tid=200 WaitingForGcToComplete
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13f81b78 self=0xccd7ba00
  | sysTid=26794 nice=0 cgrp=default sched=0/0 handle=0xb76e1970
  | state=S schedstat=( 8396405 17210051 74 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb75de000-0xb75e0000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26794/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 0017ba95  /system/lib/libart.so (art::gc::Heap::WaitForGcToCompleteLocked(art::gc::GcCause, art::Thread*)+244)
  native: #03 pc 00184b53  /system/lib/libart.so (art::gc::Heap::WaitForGcToComplete(art::gc::GcCause, art::Thread*)+186)
  native: #04 pc 0018272f  /system/lib/libart.so (art::gc::Heap::AllocateInternalWithGc(art::Thread*, art::gc::AllocatorType, bool, unsigned int, unsigned int*, unsigned int*, unsigned int*, art::ObjPtr<art::mirror::Class>*)+74)
  native: #05 pc 003ccb8b  /system/lib/libart.so (artAllocObjectFromCodeInitializedRegionTLAB+246)
  native: #06 pc 00412d47  /system/lib/libart.so (art_quick_alloc_object_initialized_region_tlab+70)
  native: #07 pc 00042fb3  [anon:dalvik-jit-code-cache] (???)
  at java.util.concurrent.ConcurrentSkipListMap$EntrySet.iterator(ConcurrentSkipListMap.java:2488)
  at org.hapjs.render.css.a.a(SourceFile:90)
  at org.hapjs.render.css.a.a(SourceFile:73)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1676" prio=5 tid=201 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13ec10a8 self=0xc85aee00
  | sysTid=26795 nice=0 cgrp=default sched=0/0 handle=0xb75db970
  | state=S schedstat=( 6232913 7231146 75 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb74d8000-0xb74da000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1677" prio=5 tid=202 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13ec1250 self=0xcc1fc000
  | sysTid=26796 nice=0 cgrp=default sched=0/0 handle=0xb74d5970
  | state=S schedstat=( 8136352 7087604 71 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb73d2000-0xb73d4000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1678" prio=5 tid=203 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13ec13f8 self=0xcc1fc600
  | sysTid=26797 nice=0 cgrp=default sched=0/0 handle=0xb73cf970
  | state=S schedstat=( 8304216 6378804 113 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb72cc000-0xb72ce000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1679" prio=5 tid=204 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13ec15a0 self=0xcc1fcc00
  | sysTid=26798 nice=0 cgrp=default sched=0/0 handle=0xb72c9970
  | state=S schedstat=( 7721042 3659581 53 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb71c6000-0xb71c8000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1680" prio=5 tid=205 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13ec1748 self=0xcc1fd200
  | sysTid=26799 nice=0 cgrp=default sched=0/0 handle=0xb71c3970
  | state=S schedstat=( 667258827 13563119 90 ) utm=62 stm=3 core=6 HZ=100
  | stack=0xb70c0000-0xb70c2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:225)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1681" prio=5 tid=206 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13e411a0 self=0xcc1fd800
  | sysTid=26800 nice=0 cgrp=default sched=0/0 handle=0xb70bd970
  | state=S schedstat=( 7876824 7190778 106 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb6fba000-0xb6fbc000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1682" prio=5 tid=207 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13e41428 self=0xcc1fde00
  | sysTid=26801 nice=0 cgrp=default sched=0/0 handle=0xb6fb7970
  | state=S schedstat=( 6788072 11562498 61 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb6eb4000-0xb6eb6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1683" prio=5 tid=208 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13e415d0 self=0xcc1fe400
  | sysTid=26802 nice=0 cgrp=default sched=0/0 handle=0xb6eb1970
  | state=S schedstat=( 5049062 3968335 42 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb6dae000-0xb6db0000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1684" prio=5 tid=209 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13dc1118 self=0xcfb65a00
  | sysTid=26803 nice=0 cgrp=default sched=0/0 handle=0xb6dab970
  | state=S schedstat=( 4960785 4806666 47 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb6ca8000-0xb6caa000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1685" prio=5 tid=210 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13dc13a0 self=0xc6269a00
  | sysTid=26804 nice=0 cgrp=default sched=0/0 handle=0xb6ca5970
  | state=S schedstat=( 5796669 5739687 46 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb6ba2000-0xb6ba4000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1686" prio=5 tid=211 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13dc1548 self=0xc6138e00
  | sysTid=26805 nice=0 cgrp=default sched=0/0 handle=0xb6b9f970
  | state=S schedstat=( 6654108 11385000 52 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb6a9c000-0xb6a9e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1687" prio=5 tid=212 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13dc1738 self=0xc51f1e00
  | sysTid=26806 nice=0 cgrp=default sched=0/0 handle=0xb6a99970
  | state=S schedstat=( 4518857 2461355 29 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb6996000-0xb6998000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1688" prio=5 tid=213 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13bc0f78 self=0xcb8c4200
  | sysTid=26807 nice=0 cgrp=default sched=0/0 handle=0xb6993970
  | state=S schedstat=( 4967495 3603855 45 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb6890000-0xb6892000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1689" prio=5 tid=214 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13bc1120 self=0xc8667600
  | sysTid=26808 nice=0 cgrp=default sched=0/0 handle=0xb688d970
  | state=S schedstat=( 4393180 8902914 33 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb678a000-0xb678c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1690" prio=5 tid=215 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13bc12c8 self=0xc858de00
  | sysTid=26809 nice=0 cgrp=default sched=0/0 handle=0xb6787970
  | state=S schedstat=( 8185778 2578648 52 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb6684000-0xb6686000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1691" prio=5 tid=216 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13b81478 self=0xc858e400
  | sysTid=26810 nice=0 cgrp=default sched=0/0 handle=0xb6681970
  | state=S schedstat=( 6021456 2700001 53 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb657e000-0xb6580000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1692" prio=5 tid=217 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13b81658 self=0xc858ea00
  | sysTid=26811 nice=0 cgrp=default sched=0/0 handle=0xb657b970
  | state=S schedstat=( 6297656 3238335 46 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb6478000-0xb647a000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1693" prio=5 tid=218 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13b81838 self=0xc858f000
  | sysTid=26812 nice=0 cgrp=default sched=0/0 handle=0xb6475970
  | state=S schedstat=( 6064428 7318433 44 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb6372000-0xb6374000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.c.f(SourceFile:75)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.j(SourceFile:16)
  at org.hapjs.render.css.p$c.a(SourceFile:220)
  at org.hapjs.render.css.a.b(SourceFile:303)
  at org.hapjs.render.css.a.a(SourceFile:263)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1694" prio=5 tid=219 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13b01010 self=0xc858f600
  | sysTid=26813 nice=0 cgrp=default sched=0/0 handle=0xb636f970
  | state=S schedstat=( 644452894 4933484 63 ) utm=63 stm=0 core=7 HZ=100
  | stack=0xb626c000-0xb626e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1695" prio=5 tid=220 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13b011b8 self=0xc8591a00
  | sysTid=26814 nice=0 cgrp=default sched=0/0 handle=0xb6269970
  | state=S schedstat=( 5958961 6115367 30 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb6166000-0xb6168000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1696" prio=5 tid=221 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13b01360 self=0xc8594000
  | sysTid=26815 nice=0 cgrp=default sched=0/0 handle=0xb6163970
  | state=S schedstat=( 3642708 496561 23 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb6060000-0xb6062000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1697" prio=5 tid=222 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a80a70 self=0xc8594600
  | sysTid=26816 nice=0 cgrp=default sched=0/0 handle=0xb605d970
  | state=S schedstat=( 4739064 2488124 22 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb5f5a000-0xb5f5c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:308)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1698" prio=5 tid=223 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a80c18 self=0xc8594c00
  | sysTid=26817 nice=0 cgrp=default sched=0/0 handle=0xb5f57970
  | state=S schedstat=( 4885777 3577293 37 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb5e54000-0xb5e56000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1699" prio=5 tid=224 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a80df8 self=0xc8595200
  | sysTid=26818 nice=0 cgrp=default sched=0/0 handle=0xb5e51970
  | state=S schedstat=( 5133956 660627 31 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb5d4e000-0xb5d50000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1700" prio=5 tid=225 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a80fa0 self=0xc8599400
  | sysTid=26819 nice=0 cgrp=default sched=0/0 handle=0xb5d4b970
  | state=S schedstat=( 2367501 2191820 27 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb5c48000-0xb5c4a000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1701" prio=5 tid=226 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a81148 self=0xc8599a00
  | sysTid=26820 nice=0 cgrp=default sched=0/0 handle=0xb5c45970
  | state=S schedstat=( 3381045 2864946 25 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb5b42000-0xb5b44000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1702" prio=5 tid=227 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a40d88 self=0xc7203000
  | sysTid=26821 nice=0 cgrp=default sched=0/0 handle=0xb5abf970
  | state=S schedstat=( 2431249 1179375 33 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb59bc000-0xb59be000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1703" prio=5 tid=228 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a40f78 self=0xc7203600
  | sysTid=26822 nice=0 cgrp=default sched=0/0 handle=0xb59b9970
  | state=S schedstat=( 1795157 1368696 23 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb58b6000-0xb58b8000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1704" prio=5 tid=229 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a41120 self=0xc7203c00
  | sysTid=26823 nice=0 cgrp=default sched=0/0 handle=0xb58b3970
  | state=S schedstat=( 1941248 1043284 22 ) utm=0 stm=0 core=3 HZ=100
  | stack=0xb57b0000-0xb57b2000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1705" prio=5 tid=230 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a412c8 self=0xc7204200
  | sysTid=26824 nice=0 cgrp=default sched=0/0 handle=0xb57ad970
  | state=S schedstat=( 2292812 991042 28 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb56aa000-0xb56ac000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1706" prio=5 tid=231 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a414a8 self=0xc7204800
  | sysTid=26825 nice=0 cgrp=default sched=0/0 handle=0xb56a7970
  | state=S schedstat=( 1895830 916981 17 ) utm=0 stm=0 core=0 HZ=100
  | stack=0xb55a4000-0xb55a6000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1707" prio=5 tid=232 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a41650 self=0xc7204e00
  | sysTid=26826 nice=0 cgrp=default sched=0/0 handle=0xb55a1970
  | state=S schedstat=( 2230259 306145 16 ) utm=0 stm=0 core=4 HZ=100
  | stack=0xb549e000-0xb54a0000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1708" prio=5 tid=233 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13a417f8 self=0xc7205400
  | sysTid=26827 nice=0 cgrp=default sched=0/0 handle=0xb549b970
  | state=S schedstat=( 4398853 5704116 23 ) utm=0 stm=0 core=1 HZ=100
  | stack=0xb5398000-0xb539a000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1709" prio=5 tid=234 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13980cc0 self=0xc7205a00
  | sysTid=26828 nice=0 cgrp=default sched=0/0 handle=0xb5395970
  | state=S schedstat=( 3108596 2070782 11 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xb5292000-0xb5294000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1710" prio=5 tid=235 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13980e68 self=0xcec3b000
  | sysTid=26829 nice=0 cgrp=default sched=0/0 handle=0xb528f970
  | state=S schedstat=( 2362084 894426 16 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xb518c000-0xb518e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1711" prio=5 tid=236 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x139810f0 self=0xcec3b600
  | sysTid=26830 nice=0 cgrp=default sched=0/0 handle=0xb5189970
  | state=S schedstat=( 4274896 296406 16 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xb5086000-0xb5088000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1712" prio=5 tid=237 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13980000 self=0xcec3bc00
  | sysTid=26831 nice=0 cgrp=default sched=0/0 handle=0xb5083970
  | state=S schedstat=( 4255473 1094060 18 ) utm=0 stm=0 core=5 HZ=100
  | stack=0xb4f80000-0xb4f82000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1713" prio=5 tid=238 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13900d80 self=0xcec3c200
  | sysTid=26832 nice=0 cgrp=default sched=0/0 handle=0xb4f7d970
  | state=S schedstat=( 4238336 840155 22 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xb4e7a000-0xb4e7c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:217)
  at org.hapjs.render.css.n.q(SourceFile:91)
  at org.hapjs.render.a.d.a(SourceFile:224)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1714" prio=5 tid=239 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13900f28 self=0xcec3c800
  | sysTid=26833 nice=0 cgrp=default sched=0/0 handle=0xb4e77970
  | state=S schedstat=( 3105781 607865 14 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xb4d74000-0xb4d76000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1715" prio=5 tid=240 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13900190 self=0xcb7f3a00
  | sysTid=26834 nice=0 cgrp=default sched=0/0 handle=0xb4d71970
  | state=S schedstat=( 6324994 2496566 29 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xb4c6e000-0xb4c70000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1716" prio=5 tid=241 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x138c0a28 self=0xc8c55000
  | sysTid=26835 nice=0 cgrp=default sched=0/0 handle=0xb4c6b970
  | state=S schedstat=( 4531666 714323 12 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xb4b68000-0xb4b6a000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.d.a(SourceFile:243)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:293)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1717" prio=5 tid=242 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x138c0bd0 self=0xc8c55600
  | sysTid=26836 nice=0 cgrp=default sched=0/0 handle=0xb4b65970
  | state=S schedstat=( 3691252 674843 10 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xb4a62000-0xb4a64000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1718" prio=5 tid=243 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13880b70 self=0xc8c55c00
  | sysTid=26837 nice=0 cgrp=default sched=0/0 handle=0xb4a5f970
  | state=S schedstat=( 3306407 513540 12 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xb495c000-0xb495e000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.c(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.h(SourceFile:99)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1719" prio=5 tid=244 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13880df8 self=0xc8c56200
  | sysTid=26838 nice=0 cgrp=default sched=0/0 handle=0xb4959970
  | state=S schedstat=( 2442761 0 7 ) utm=0 stm=0 core=5 HZ=100
  | stack=0xb4856000-0xb4858000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1720" prio=5 tid=245 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x13441698 self=0xc21a9200
  | sysTid=26840 nice=0 cgrp=default sched=0/0 handle=0xb4853970
  | state=S schedstat=( 4616301 415105 13 ) utm=0 stm=0 core=5 HZ=100
  | stack=0xb4750000-0xb4752000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.d(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.c.a(SourceFile:118)
  at org.hapjs.render.a.c.h(SourceFile:98)
  at org.hapjs.render.css.a.a(SourceFile:72)
  at org.hapjs.render.css.a.a(SourceFile:67)
  at org.hapjs.render.css.n.a(SourceFile:87)
  at org.hapjs.render.a.d.a(SourceFile:229)
  at org.hapjs.render.a.d.a(SourceFile:218)
  at org.hapjs.render.a.d.b(SourceFile:321)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1721" prio=5 tid=246 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x134418e8 self=0xc7206000
  | sysTid=26841 nice=0 cgrp=default sched=0/0 handle=0xb474d970
  | state=S schedstat=( 4215470 903438 14 ) utm=0 stm=0 core=4 HZ=100
  | stack=0xb464a000-0xb464c000 stackSize=1042KB
  | held mutexes=
  at org.hapjs.render.a.a.a(SourceFile:-1)
  - waiting to lock <0x007aea59> (a org.hapjs.render.a.a) held by thread 98
  at org.hapjs.render.a.d.b(SourceFile:291)
  at org.hapjs.render.a.d.a(SourceFile:332)
  at org.hapjs.render.a.d.a(SourceFile:282)
  at org.hapjs.render.a.d.a(SourceFile:44)
  at org.hapjs.render.a.b.a(SourceFile:86)
  at org.hapjs.render.a.b.a(SourceFile:38)
  at org.hapjs.render.a.b$a.a(SourceFile:263)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"[io]-1722" prio=5 tid=247 WaitingForGcToComplete
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x134407e0 self=0xc7206600
  | sysTid=26842 nice=0 cgrp=default sched=0/0 handle=0xb4647970
  | state=S schedstat=( 735573 5014062 2 ) utm=0 stm=0 core=6 HZ=100
  | stack=0xb4544000-0xb4546000 stackSize=1042KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26842/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 000a6ba3  /system/lib/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+78)
  native: #02 pc 0017ba95  /system/lib/libart.so (art::gc::Heap::WaitForGcToCompleteLocked(art::gc::GcCause, art::Thread*)+244)
  native: #03 pc 00184b53  /system/lib/libart.so (art::gc::Heap::WaitForGcToComplete(art::gc::GcCause, art::Thread*)+186)
  native: #04 pc 0018272f  /system/lib/libart.so (art::gc::Heap::AllocateInternalWithGc(art::Thread*, art::gc::AllocatorType, bool, unsigned int, unsigned int*, unsigned int*, unsigned int*, art::ObjPtr<art::mirror::Class>*)+74)
  native: #05 pc 003ccb8b  /system/lib/libart.so (artAllocObjectFromCodeInitializedRegionTLAB+246)
  native: #06 pc 00412d47  /system/lib/libart.so (art_quick_alloc_object_initialized_region_tlab+70)
  native: #07 pc 00031d4f  [anon:dalvik-jit-code-cache] (Java_org_hapjs_render_a_b_00024a_a__+78)
  at org.hapjs.render.a.b$a.a(SourceFile:255)
  at org.hapjs.render.a.b$a.run(SourceFile:242)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
  at java.lang.Thread.run(Thread.java:764)

"V8 DefaultWorke" prio=5 (not attached)
  | sysTid=23773 nice=0 cgrp=default
  | state=S schedstat=( 1368895257 885919056 1989 ) utm=128 stm=8 core=4 HZ=100
  kernel: (couldn't read /proc/self/task/23773/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 0001d34f  /system/lib/libc.so (__futex_wait_ex(void volatile*, bool, int, bool, timespec const*)+90)
  native: #02 pc 00071a6f  /system/lib/libc.so (pthread_cond_wait+32)
  native: #03 pc 001a9d87  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #04 pc 001a9a33  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #05 pc 001a7925  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #06 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #07 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)

"V8 DefaultWorke" prio=5 (not attached)
  | sysTid=23774 nice=0 cgrp=default
  | state=S schedstat=( 1298047640 930136931 1697 ) utm=124 stm=4 core=4 HZ=100
  kernel: (couldn't read /proc/self/task/23774/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 0001d34f  /system/lib/libc.so (__futex_wait_ex(void volatile*, bool, int, bool, timespec const*)+90)
  native: #02 pc 00071a6f  /system/lib/libc.so (pthread_cond_wait+32)
  native: #03 pc 001a9d87  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #04 pc 001a9a33  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #05 pc 001a7925  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #06 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #07 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)

"V8 DefaultWorke" prio=5 (not attached)
  | sysTid=23775 nice=0 cgrp=default
  | state=S schedstat=( 1388959650 1023751828 2106 ) utm=134 stm=4 core=6 HZ=100
  kernel: (couldn't read /proc/self/task/23775/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 0001d34f  /system/lib/libc.so (__futex_wait_ex(void volatile*, bool, int, bool, timespec const*)+90)
  native: #02 pc 00071a6f  /system/lib/libc.so (pthread_cond_wait+32)
  native: #03 pc 001a9d87  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #04 pc 001a9a33  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #05 pc 001a7925  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #06 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #07 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)

"V8 DefaultWorke" prio=5 (not attached)
  | sysTid=23776 nice=0 cgrp=default
  | state=S schedstat=( 1281935546 824778085 2787 ) utm=121 stm=6 core=5 HZ=100
  kernel: (couldn't read /proc/self/task/23776/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 0001d34f  /system/lib/libc.so (__futex_wait_ex(void volatile*, bool, int, bool, timespec const*)+90)
  native: #02 pc 00071a6f  /system/lib/libc.so (pthread_cond_wait+32)
  native: #03 pc 001a9d87  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #04 pc 001a9a33  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #05 pc 001a7925  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #06 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #07 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)

"V8 DefaultWorke" prio=5 (not attached)
  | sysTid=23777 nice=0 cgrp=default
  | state=S schedstat=( 1252741899 861943552 2396 ) utm=117 stm=7 core=4 HZ=100
  kernel: (couldn't read /proc/self/task/23777/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 0001d34f  /system/lib/libc.so (__futex_wait_ex(void volatile*, bool, int, bool, timespec const*)+90)
  native: #02 pc 00071a6f  /system/lib/libc.so (pthread_cond_wait+32)
  native: #03 pc 001a9d87  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #04 pc 001a9a33  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #05 pc 001a7925  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #06 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #07 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)

"V8 DefaultWorke" prio=5 (not attached)
  | sysTid=23778 nice=0 cgrp=default
  | state=S schedstat=( 1309997418 836562964 1958 ) utm=128 stm=2 core=6 HZ=100
  kernel: (couldn't read /proc/self/task/23778/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 0001d34f  /system/lib/libc.so (__futex_wait_ex(void volatile*, bool, int, bool, timespec const*)+90)
  native: #02 pc 00071a6f  /system/lib/libc.so (pthread_cond_wait+32)
  native: #03 pc 001a9d87  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #04 pc 001a9a33  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #05 pc 001a7925  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #06 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #07 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)

"V8 DefaultWorke" prio=5 (not attached)
  | sysTid=23779 nice=0 cgrp=default
  | state=S schedstat=( 1180479063 828995254 2292 ) utm=109 stm=8 core=4 HZ=100
  kernel: (couldn't read /proc/self/task/23779/stack)
  native: #00 pc 00019eb0  /system/lib/libc.so (syscall+28)
  native: #01 pc 0001d34f  /system/lib/libc.so (__futex_wait_ex(void volatile*, bool, int, bool, timespec const*)+90)
  native: #02 pc 00071a6f  /system/lib/libc.so (pthread_cond_wait+32)
  native: #03 pc 001a9d87  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #04 pc 001a9a33  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #05 pc 001a7925  /data/app/com.meizu.flyme.directservice-fsR0hHe9Y-1XWkjiE3Fmbg==/lib/arm/libjsenv.so (???)
  native: #06 pc 00072301  /system/lib/libc.so (__pthread_start(void*)+22)
  native: #07 pc 0001dfd9  /system/lib/libc.so (__start_thread+24)

----- end 23755 -----

```

-->