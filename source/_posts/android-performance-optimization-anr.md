---
title: Android 性能优化理论篇 -- ANR
categories: Android性能优化
comments: true
tags: [ANR]
description: 介绍一些关于处理 ANR 问题的理论知识
date: 2018-01-08 10:00:00
---


## 概述

无论我们是在 Android 设备的使用过程中还是 Android 开发过程中，基本都遇到过这样的情况，系统卡死，弹框提示应用无响应。也就是我们常说的 ANR -- Application Not Responding。

## 理论知识

ANR 是 Android 系统设计的一种机制，我们常见的有下面三种 ANR：

 - KeyDispatchTimeout：按键或触摸事件在特定时间内无响应 
 - BroadcastTimeout：BroadcastReceiver 在特定时间内无法处理完成
 - ServiceTimeout：Service 在特定的时间内无法处理完成 
 - ContentProvider：ContentProvider 执行超时，这种比较少见

系统认定产生 ANR 的超时时间设置如下：

ActiveServices.java

```
    // How long we wait for a service to finish executing.
    // Service 超时时间
    static final int SERVICE_TIMEOUT = 20*1000;

    // How long we wait for a service to finish executing.
    // 后台 Service 超时时间
    static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;
```

ActivityManagerService.java

```
    // How long we allow a receiver to run before giving up on it.
    static final int BROADCAST_FG_TIMEOUT = 10*1000;
    static final int BROADCAST_BG_TIMEOUT = 60*1000;

    // How long we wait until we timeout on key dispatching.
    static final int KEY_DISPATCHING_TIMEOUT = 5*1000;
```

而产生 ANR 的原因一般有：

 - 主线程耗时操作
 - 主线程被其它线程 Block
 - 被 Binder 对端 Block
 - Binder 被占满导致主线程无法和 SystemServer 通信
 - 得不到系统资源（CPU/RAM/IO）

从进程的角度看：

 - 问题出在当前进程：
  - 主线程本身耗时, 或则主线程的消息队列存在耗时操作;
  - 主线程被本进程的其他子线程所blocked;
 - 问题出在远端进程(一般是binder call或socket等通信方式)

上面的每种场景我们尽量都在后面的性能优化实战系列中挑选有代表性的来介绍。

## 日志获取

ANR 问题发生了，我们首先要获取日志，才能接下来进一步分析。

 - 使用 Log Report App 抓取。
 - 使用 adb bugreprot <file>
 - 使用 adb pull /data/anr/traces.txt <file>

## 小技巧

下面分析看一下各种日志对我们分析 ANR 的作用。

| Log 名称 | 作用 | 关键词 |
|:-------------:|:-------------:|:-------------:|
|trace.txt | 记录了在发生ANR时刻系统各个线程的执行状态 | 搜索发生 anr 的进程名称 |
|system.log | 包含ANR发生时间点信息、ANR发生前的CPU信息,还包含大量系统服务输出的信息 | "ANR in" |
| main.log | 包含ANR发生前应用自身输出的信息,可供分析应用是否有异常;此外还包含输出的GC信息，可供分析内存回收的速度,判断系统是否处于低内存或内存碎片化状态 |  |
| event.log | 包含AMS与WMS输出的应用程序声明周期信息,可供分析窗口创建速度以及焦点转换情况 | "am_anr" |
| kernel.log | 包含kernel打出的信息,LowMemoryKiller杀进程、内存碎片化或内存不足,mmc驱动异常都可以在这里找到。 |  |

拿到日志后，我们首先要确定 ANR 发生的时间点，可以通过 trace.txt 里面搜索进程的名字来查看，或者通过在 是system log里搜索 Anr in 来搜索，或者通过在 event log 里面搜索 am_anr 关键字来确定发生的时间。

## system 分析

```
15:57:26.938  1380  1420 E ActivityManager: ANR in com.android.systemui:recents
E ActivityManager: PID: 2443
E ActivityManager: Reason: Broadcast of Intent { act=android.intent.action.SCREEN_OFF flg=0x50200010 }
E ActivityManager: Load: 5.01 / 6.61 / 5.59
E ActivityManager: CPU usage from 0ms to 6626ms later (2018-01-23 15:57:20.275 to 2018-01-23 15:57:26.900):
E ActivityManager:   49% 685/surfaceflinger: 33% user + 15% kernel / faults: 1437 minor 2 major
E ActivityManager:   41% 830/media.codec: 19% user + 21% kernel / faults: 7619 minor 5 major
E ActivityManager:   26% 820/mediaserver: 16% user + 10% kernel / faults: 514 minor
E ActivityManager:   24% 7710/com.flyme.systemuiex: 20% user + 3.7% kernel / faults: 1775 minor
E ActivityManager:   23% 1380/system_server: 11% user + 11% kernel / faults: 3762 minor 4 major
E ActivityManager:   22% 682/audioserver: 3.7% user + 19% kernel / faults: 583 minor 2 major
E ActivityManager:   13% 512/logd: 5.5% user + 7.8% kernel
E ActivityManager:   12% 5114/com.tencent.weread: 8.7% user + 3.7% kernel / faults: 2587 minor 5 major
E ActivityManager:   11% 643/android.hardware.graphics.composer@2.2-service: 6.4% user + 4.6% kernel / faults: 490 minor 6 major
E ActivityManager:   8.9% 2044/com.android.systemui: 6.6% user + 2.2% kernel / faults: 2476 minor 2 major
E ActivityManager:   5.8% 437/crtc_commit:110: 0% user + 5.8% kernel
E ActivityManager:   5.4% 246/kgsl_worker_thr: 0% user + 5.4% kernel
E ActivityManager:   4% 2207/com.android.phone: 2.7% user + 1.3% kernel / faults: 1896 minor 8 major
E ActivityManager:   3.6% 8497/wtlogcat: 2.2% user + 1.3% kernel
E ActivityManager:   3.4% 8676/main-log: 1.9% user + 1.5% kernel
E ActivityManager:   3.1% 3016/com.sohu.inputmethod.sogou.meizu: 2.7% user + 0.4% kernel / faults: 2317 minor
E ActivityManager:   3% 2809/com.meizu.alphame: 1.6% user + 1.3% kernel / faults: 1164 minor 7 major
E ActivityManager:   2.8% 2025/com.meizu.facerecognition: 1.3% user + 1.5% kernel / faults: 874 minor 7 major
E ActivityManager:   2.7% 480/kworker/u16:10: 0% user + 2.7% kernel
E ActivityManager:   2.7% 2443/com.android.systemui:recents: 2.4% user + 0.3% kernel / faults: 2163 minor 10 major
E ActivityManager:   2.5% 2645/kworker/u16:7: 0% user + 2.5% kernel
E ActivityManager:   0.7% 18304/kworker/u16:11: 0% user + 0.7% kernel
E ActivityManager:   1.9% 1//init: 1% user + 0.9% kernel / faults: 11 minor 1 major
E ActivityManager:   0.1% 2790/com.android.incallui: 0% user + 0% kernel / faults: 1606 minor
E ActivityManager:   0.6% 20057/kworker/u16:13: 0% user + 0.6% kernel
E ActivityManager:   1.8% 649/android.hardware.sensors@1.0-service: 0.9% user + 0.9% kernel / faults: 228 minor 3 major
E ActivityManager:   1.6% 25046/kworker/u16:9: 0% user + 1.6% kernel
E ActivityManager:   0% 2863/com.qualcomm.qti.services.secureui:sui_service: 0% user + 0% kernel / faults: 810 minor 2 major
E ActivityManager:   1.2% 2160/.dataservices: 0.7% user + 0.4% kernel / faults: 796 minor
E ActivityManager:   0% 2193/com.qualcomm.qti.telephonyservice: 0% user + 0% kernel / faults: 719 minor
E ActivityManager:   0.3% 2828/com.meizu.dataservice: 0.3% user + 0% kernel / faults: 797 minor
E ActivityManager:   1.2% 6850/com.tencent.qqlive.player.meizu: 1% user + 0.1% kernel / faults: 2 minor
E ActivityManager:   0.3% 815/media.extractor: 0.1% user + 0.1% kernel / faults: 1387 minor 2 major
E ActivityManager:   0.3% 2803/com.android.se: 0.2% user + 0% kernel / faults: 719 minor
E ActivityManager:   1% 20602/kworker/u16:3: 0% user + 1% kernel
E ActivityManager:   0.9% 328/mmc-cmdqd/0: 0% user + 0.9% kernel
E ActivityManager:   0.9% 2000/kworker/u16:2: 0% user + 0.9% kernel
E ActivityManager:   0.7% 289/irq/136-hxcommo: 0% user + 0.7% kernel
E ActivityManager:   0.7% 438/crtc_event:110: 0% user + 0.7% kernel
E ActivityManager:   0% 2845/com.wingtech.logcat: 0% user + 0% kernel / faults: 712 minor
E ActivityManager:   0.7% 5834/com.flyme.systemuitools: 0.4% user + 0.3% kernel / faults: 1 minor
E ActivityManager:   0.7% 8070/com.android.mms: 0.3% user + 0.4% kernel / faults: 850 minor 71 major
E ActivityManager:   0.6% 2766/irq/66-90b6300.: 0% user + 0.6% kernel
E ActivityManager:   0.4% 9/rcu_preempt: 0% user + 0.4% kernel
E ActivityManager:   0.4% 632/android.hardware.audio@2.0-service: 0% user + 0.4% kernel / faults: 85 minor 8 major
E ActivityManager:   0.4% 661/vendor.qti.hardware.perf@1.0-service: 0.1% user + 0.3% kernel / faults: 18 minor
E ActivityManager:   0.4% 869/tombstoned: 0% user + 0.4% kernel / faults: 1 minor
E ActivityManager:   0.4% 8325/com.ct.client: 0.3% user + 0.1% kernel
E ActivityManager:   0% 10/rcu_sched: 0% user + 0% kernel
E ActivityManager:   0.3% 12/rcuop/0: 0% user + 0.3% kernel
E ActivityManager:   0.3% 50/migration/5: 0% user + 0.3% kernel
E ActivityManager:   0% 865/dpmd: 0% user + 0% kernel / faults: 168 minor
E ActivityManager:   0.3% 8369/com.ct.client:pushcore: 0% user + 0.3% kernel
E ActivityManager:   0.1% 19/ksoftirqd/1: 0% user + 0.1% kernel
E ActivityManager:   0% 23/rcuos/1: 0% user + 0% kernel
E ActivityManager:   0.1% 30/rcuop/2: 0% user + 0.1% kernel
E ActivityManager:   0% 35/ksoftirqd/3: 0% user + 0% kernel
E ActivityManager:   0.1% 38/rcuop/3: 0% user + 0.1% kernel
E ActivityManager:   0% 39/rcuos/3: 0% user + 0%
I ActivityManager: addErrorToDropBox com.android.systemui:recents anr

```

`ActivityManager: Load: 5.01 / 6.61 / 5.59` 这一行表示当时的 CPU 负载，另外我们还可以通过 `adb shell uptime` 命令来来获取设备的 CPU 负载：

```
 20:08:58 up  5:52,  0 users,  load average: 1.27, 0.77, 0.76
```

Load后面的三个数字的意思分别是1分钟、5分钟、15分钟内系统的平均负荷。当CPU完全空闲的时候，平均负荷为0；当CPU工作量饱和的时候，平均负荷为1，通过Load可以判断系统负荷是否过重。
经验法则是这样的：
当系统负荷持续大于0.7，你必须开始调查了，问题出在哪里，防止情况恶化。
当系统负荷持续大于1.0，你必须动手寻找解决办法，把这个值降下来。
当系统负荷达到5.0，就表明你的系统有很严重的问题。

更准确的CPU使用信息我们还可以通过 Log Report 目录中 dumpsys/dumpsys_cpuinfo 来查看。

```
  49% 685/surfaceflinger: 33% user + 15% kernel / faults: 1437 minor 2 major
  41% 830/media.codec: 19% user + 21% kernel / faults: 7619 minor 5 major
  26% 820/mediaserver: 16% user + 10% kernel / faults: 514 minor
  ......
44% TOTAL: 21% user + 21% kernel + 0.3% iowait + 1.7% irq + 0.4% softirq
```

上面这一段日志可以得到ANR发生的时候，Top进程的Cpu占用情况，user代表是用户空间，kernel是内核空间。对于 CPU 的占用率，在多核中每个核最大占用率都是100%，如果机器是8核的，那么每个进程的CPU最大占用率就是800%，这也就是为什么我们会经常看到某些进程 CPU 占用率会大于100%，其实百分之一百多的CPU占用率不算很高的。
分析这一部分，一般的有如下的规律。

 - kswapd0 cpu占用率偏高，系统整体运行会缓慢，从而引起各种ANR。把问题转给"内存优化"，请他们进行优化。
 - logd　CPU占用率偏高，也会引起系统卡顿和ANR，因为各个进程输出LOG的操作被阻塞从而执行的极为缓慢。
 - Vold占用CPU过高，会引起系统卡顿和ANR，请负责存储的同学先调查
 - qcom.sensor CPU占用率过高，会引起卡顿，请系统同学调查
 - 应用自身CPU占用率较高，高概率应用自身问题
 - 系统CPU占用率不高，但主线程在等待一个锁，高概率应用自身问题
 - 应用处于D状态，发生ANR，如果最后的操作是refriger，那么是应用被冻结了，正常情况下是功耗优化引起的


## trace 分析

要查看应用被阻塞的位置，还是需要分析 trace 文件。
下面通过一个人为制造的 ANR 事件来讲解一下 traces 的分析技巧。
traces 文件在 /data/anr/traces.txt 可以找到，一般 ANR 发生时，在弹出无响应对话框之前，AMS 会把 ANR 相关的信息保存在这个文件中，因此，这个文件是分析 ANR 的利器。

```
Cmd line: com.example.heqiang.testsomething
Build fingerprint: 'alps/m1891/m1891:7.0/NRD90M/eng.heqian.20180417.201920:userdebug/test-keys'
ABI: 'arm64'
Build type: optimized
Zygote loaded classes=4383 post zygote classes=162
Intern table: 44871 strong; 182 weak
JNI: CheckJNI is on; globals=506 (plus 226 weak)
Libraries: /data/app/com.example.heqiang.testsomething-1/lib/arm64/libJniDemo.so /system/lib64/libandroid.so /system/lib64/libcompiler_rt.so /system/lib64/libfilterUtils.so /system/lib64/libjavacrypto.so /system/lib64/libjnigraphics.so /system/lib64/libmedia_jni.so /system/lib64/libwebviewchromium_loader.so libjavacore.so libopenjdk.so (10)
Heap: 18% free, 11MB/13MB; 33582 objects
Dumping cumulative Gc timings
Start Dumping histograms for 1 iterations for sticky concurrent mark sweep
ScanGrayImageSpaceObjects:	Sum: 1.959ms 99% C.I. 0.264us-1375us Avg: 54.416us Max: 1557us
MarkConcurrentRoots:	Sum: 1.573ms 99% C.I. 3us-1570us Avg: 786.500us Max: 1570us
FreeList:	Sum: 1.281ms 99% C.I. 4us-147.750us Avg: 71.166us Max: 149us
ProcessMarkStack:	Sum: 1.076ms 99% C.I. 4us-933us Avg: 269us Max: 943us
SweepArray:	Sum: 528us 99% C.I. 528us-528us Avg: 528us Max: 528us
ScanGrayAllocSpaceObjects:	Sum: 451us 99% C.I. 1us-271us Avg: 112.750us Max: 271us
MarkRootsCheckpoint:	Sum: 444us 99% C.I. 203us-241us Avg: 222us Max: 241us
ScanGrayZygoteSpaceObjects:	Sum: 388us 99% C.I. 9us-379us Avg: 194us Max: 379us
SweepSystemWeaks:	Sum: 271us 99% C.I. 271us-271us Avg: 271us Max: 271us
MarkingPhase:	Sum: 126us 99% C.I. 126us-126us Avg: 126us Max: 126us
ReMarkRoots:	Sum: 95us 99% C.I. 95us-95us Avg: 95us Max: 95us
AllocSpaceClearCards:	Sum: 79us 99% C.I. 2us-38us Avg: 19.750us Max: 38us
ImageModUnionClearCards:	Sum: 63us 99% C.I. 0.250us-21us Avg: 1.750us Max: 21us
MarkNonThreadRoots:	Sum: 62us 99% C.I. 22us-40us Avg: 31us Max: 40us
BindBitmaps:	Sum: 55us 99% C.I. 55us-55us Avg: 55us Max: 55us
ProcessCards:	Sum: 27us 99% C.I. 13us-14us Avg: 13.500us Max: 14us
ZygoteModUnionClearCards:	Sum: 26us 99% C.I. 8us-18us Avg: 13us Max: 18us
ResetStack:	Sum: 24us 99% C.I. 24us-24us Avg: 24us Max: 24us
FinishPhase:	Sum: 23us 99% C.I. 23us-23us Avg: 23us Max: 23us
EnqueueFinalizerReferences:	Sum: 22us 99% C.I. 22us-22us Avg: 22us Max: 22us
InitializePhase:	Sum: 18us 99% C.I. 18us-18us Avg: 18us Max: 18us
(Paused)PausePhase:	Sum: 17us 99% C.I. 17us-17us Avg: 17us Max: 17us
PreCleanCards:	Sum: 15us 99% C.I. 15us-15us Avg: 15us Max: 15us
ForwardSoftReferences:	Sum: 13us 99% C.I. 13us-13us Avg: 13us Max: 13us
(Paused)ScanGrayImageSpaceObjects:	Sum: 9us 99% C.I. 250ns-5000ns Avg: 500ns Max: 5000ns
ReclaimPhase:	Sum: 7us 99% C.I. 7us-7us Avg: 7us Max: 7us
RevokeAllThreadLocalAllocationStacks:	Sum: 6us 99% C.I. 6us-6us Avg: 6us Max: 6us
MarkRoots:	Sum: 2us 99% C.I. 2us-2us Avg: 2us Max: 2us
(Paused)ProcessMarkStack:	Sum: 1us 99% C.I. 1us-1us Avg: 1us Max: 1us
FindDefaultSpaceBitmap:	Sum: 0 99% C.I. 0ns-0ns Avg: 0ns Max: 0ns
Done Dumping histograms
sticky concurrent mark sweep paused:	Sum: 183us 99% C.I. 183us-183us Avg: 183us Max: 183us
sticky concurrent mark sweep total time: 8.697ms mean time: 8.697ms
sticky concurrent mark sweep freed: 16488 objects with total size 1933KB
sticky concurrent mark sweep throughput: 2.061e+06/s / 236MB/s
Total time spent in GC: 8.697ms
Mean GC size throughput: 109MB/s
Mean GC object throughput: 1.89456e+06 objects/s
Total number of allocations 50059
Total bytes allocated 12MB
Total bytes freed 977KB
Free memory 2MB
Free memory until GC 2MB
Free memory until OOME 244MB
Total memory 13MB
Max memory 256MB
Zygote space size 1196KB
Total mutator paused time: 183us
Total time waiting for GC to complete: 5.617us
Total GC count: 1
Total GC time: 8.697ms
Total blocking GC count: 0
Total blocking GC time: 0
Histogram of GC count per 10000 ms: 0:22
Histogram of blocking GC count per 10000 ms: 0:22
Histogram of native allocation 0:749,128:245,256:108,384:61,512:21,640:1,896:16,1152:2 bucket size 128
Histogram of native free 0:13,64:72,192:223,256:37,448:10,512:10 bucket size 64
/data/app/com.example.heqiang.testsomething-1/oat/arm64/base.odex: interpret-only
Current JIT code cache size: 9KB
Current JIT data cache size: 8KB
Current JIT capacity: 64KB
Current number of JIT code cache entries: 11
Total number of JIT compilations: 11
Total number of JIT compilations for on stack replacement: 0
Total number of deoptimizations: 0
Total number of JIT code cache collections: 0
Memory used for stack maps: Avg: 451B Max: 1520B Min: 24B
Memory used for compiled code: Avg: 815B Max: 2024B Min: 112B
Memory used for profiling info: Avg: 300B Max: 848B Min: 32B
Start Dumping histograms for 11 iterations for JIT timings
Compiling:	Sum: 28.336ms 99% C.I. 0.305ms-6.501ms Avg: 2.576ms Max: 6.535ms
TrimMaps:	Sum: 480us 99% C.I. 11us-103us Avg: 43.636us Max: 103us
Done Dumping histograms
Memory used for compilation: Avg: 321KB Max: 869KB Min: 48KB
ProfileSaver total_bytes_written=308
ProfileSaver total_number_of_writes=1
ProfileSaver total_number_of_code_cache_queries=1
ProfileSaver total_number_of_skipped_writes=0
ProfileSaver total_number_of_failed_writes=0
ProfileSaver total_ms_of_sleep=212475
ProfileSaver total_ms_of_work=2
ProfileSaver total_number_of_foreign_dex_marks=0
ProfileSaver max_number_profile_entries_cached=89
ProfileSaver total_number_of_hot_spikes=0
ProfileSaver total_number_of_wake_ups=1

suspend all histogram:	Sum: 157us 99% C.I. 1us-31us Avg: 9.812us Max: 31us
DALVIK THREADS (15):
"Signal Catcher" daemon prio=5 tid=3 Runnable
  | group="system" sCount=0 dsCount=0 obj=0x12c67e50 self=0x750eff9000
  | sysTid=8147 nice=0 cgrp=default sched=0/0 handle=0x7518808450
  | state=R schedstat=( 17661038 27346 3 ) utm=0 stm=0 core=1 HZ=100
  | stack=0x751870e000-0x7518710000 stackSize=1005KB
  | held mutexes= "mutator lock"(shared held)
  native: #00 pc 0000000000476d74  /system/lib64/libart.so (_ZN3art15DumpNativeStackERNSt3__113basic_ostreamIcNS0_11char_traitsIcEEEEiP12BacktraceMapPKcPNS_9ArtMethodEPv+220)
  native: #01 pc 0000000000476d70  /system/lib64/libart.so (_ZN3art15DumpNativeStackERNSt3__113basic_ostreamIcNS0_11char_traitsIcEEEEiP12BacktraceMapPKcPNS_9ArtMethodEPv+216)
  native: #02 pc 000000000044b5bc  /system/lib64/libart.so (_ZNK3art6Thread9DumpStackERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEEbP12BacktraceMap+472)
  native: #03 pc 0000000000462cdc  /system/lib64/libart.so (_ZN3art14DumpCheckpoint3RunEPNS_6ThreadE+820)
  native: #04 pc 000000000045afbc  /system/lib64/libart.so (_ZN3art10ThreadList13RunCheckpointEPNS_7ClosureE+456)
  native: #05 pc 000000000045abcc  /system/lib64/libart.so (_ZN3art10ThreadList4DumpERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEEb+288)
  native: #06 pc 000000000045aa68  /system/lib64/libart.so (_ZN3art10ThreadList14DumpForSigQuitERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEE+804)
  native: #07 pc 00000000004369ec  /system/lib64/libart.so (_ZN3art7Runtime14DumpForSigQuitERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEE+344)
  native: #08 pc 000000000043d278  /system/lib64/libart.so (_ZN3art13SignalCatcher13HandleSigQuitEv+2240)
  native: #09 pc 000000000043bd38  /system/lib64/libart.so (_ZN3art13SignalCatcher3RunEPv+612)
  native: #10 pc 00000000000680ec  /system/lib64/libc.so (_ZL15__pthread_startPv+196)
  native: #11 pc 000000000001d940  /system/lib64/libc.so (__start_thread+16)
  (no managed stack frames)

"main" prio=5 tid=1 Sleeping
  | group="main" sCount=1 dsCount=0 obj=0x75c5c348 self=0x7519295a00
  | sysTid=8141 nice=0 cgrp=default sched=0/0 handle=0x751d25fa98
  | state=S schedstat=( 173407732 13795308 237 ) utm=14 stm=2 core=5 HZ=100
  | stack=0x7ff4d74000-0x7ff4d76000 stackSize=8MB
  | held mutexes=
  at java.lang.Thread.sleep!(Native method)
  - sleeping on <0x0c2a799d> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:371)
  - locked <0x0c2a799d> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:313)
  at com.example.heqiang.testsomething.aidl.AIDLClientActivity.hasShortcutInstalled(AIDLClientActivity.java:105)
  at java.lang.reflect.Method.invoke!(Native method)
  at android.view.View$DeclaredOnClickListener.onClick(View.java:4729)
  at android.view.View.performClick(View.java:5660)
  at android.view.View$PerformClick.run(View.java:22436)
  at android.os.Handler.handleCallback(Handler.java:751)
  at android.os.Handler.dispatchMessage(Handler.java:95)
  at android.os.Looper.loop(Looper.java:154)
  at android.app.ActivityThread.main(ActivityThread.java:6351)
  at java.lang.reflect.Method.invoke!(Native method)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:983)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:873)
```

**第一行**： `"main" prio=5 tid=1 Sleeping`
这一行的信息主要包括：

 - 线程名称（main）
 - 线程的优先级（prio=5）
 - 线程id（tid=1）
 - 线程状态（Sleeping）。

线程状态在 art/runtime/thread_state.h 文件中，主要有下面的状态：

```
enum ThreadState {
  //                                   Thread.State   JDWP state
  kTerminated = 66,                 // TERMINATED     TS_ZOMBIE    Thread.run has returned, but Thread* still around
  kRunnable,                        // RUNNABLE       TS_RUNNING   runnable
  kTimedWaiting,                    // TIMED_WAITING  TS_WAIT      in Object.wait() with a timeout
  kSleeping,                        // TIMED_WAITING  TS_SLEEPING  in Thread.sleep()
  kBlocked,                         // BLOCKED        TS_MONITOR   blocked on a monitor
  kWaiting,                         // WAITING        TS_WAIT      in Object.wait()
  kWaitingForGcToComplete,          // WAITING        TS_WAIT      blocked waiting for GC
  kWaitingForCheckPointsToRun,      // WAITING        TS_WAIT      GC waiting for checkpoints to run
  kWaitingPerformingGc,             // WAITING        TS_WAIT      performing GC
  kWaitingForDebuggerSend,          // WAITING        TS_WAIT      blocked waiting for events to be sent
  kWaitingForDebuggerToAttach,      // WAITING        TS_WAIT      blocked waiting for debugger to attach
  kWaitingInMainDebuggerLoop,       // WAITING        TS_WAIT      blocking/reading/processing debugger events
  kWaitingForDebuggerSuspension,    // WAITING        TS_WAIT      waiting for debugger suspend all
  kWaitingForJniOnLoad,             // WAITING        TS_WAIT      waiting for execution of dlopen and JNI on load code
  kWaitingForSignalCatcherOutput,   // WAITING        TS_WAIT      waiting for signal catcher IO to complete
  kWaitingInMainSignalCatcherLoop,  // WAITING        TS_WAIT      blocking/reading/processing signals
  kWaitingForDeoptimization,        // WAITING        TS_WAIT      waiting for deoptimization suspend all
  kWaitingForMethodTracingStart,    // WAITING        TS_WAIT      waiting for method tracing to start
  kWaitingForVisitObjects,          // WAITING        TS_WAIT      waiting for visiting objects
  kWaitingForGetObjectsAllocated,   // WAITING        TS_WAIT      waiting for getting the number of allocated objects
  kWaitingWeakGcRootRead,           // WAITING        TS_WAIT      waiting on the GC to read a weak root
  kWaitingForGcThreadFlip,          // WAITING        TS_WAIT      waiting on the GC thread flip (CC collector) to finish
  kStarting,                        // NEW            TS_WAIT      native thread started, not yet ready to run managed code
  kNative,                          // RUNNABLE       TS_RUNNING   running in a JNI native method
  kSuspended,                       // RUNNABLE       TS_RUNNING   suspended by GC or debugger
};
```

**第二行**：`  | group="main" sCount=1 dsCount=0 obj=0x75c5c348 self=0x7519295a00`
主要信息包括：

 - 线程所处的线程组（group="main"）
 - 线程被正常挂起的次处（sCount=1）
 - 线程因调试而挂起次数（dsCount=0），当一个进程被调试后，sCount会重置为0，调试完毕后sCount会根据是否被正常挂起增长，但是dsCount不会被重置为0，所以dsCount也可以用来判断这个线程是否被调试过
 - 当前线程所关联的java线程对象（obj=0x75c5c348）
 - 该线程本身的地址（self=0x7519295a00）。

**第三行**：`  | sysTid=8141 nice=0 cgrp=default sched=0/0 handle=0x751d25fa98`
主要信息包括线程调度信息，分别是：

 - 线程在linux系统下的内核线程id （sysTid=8141）
 - 线程的调度有优先级（nice=0），nice值越小，优先级越高
 - 调度策略（sched=0/0）
 - 优先组属（cgrp=default）
 - 处理函数地址（handle=0x751d25fa98）。

**第四行**：`  | state=S schedstat=( 173407732 13795308 237 ) utm=14 stm=2 core=5 HZ=100`
主要信息包括当前线程的上下文信息：

 - 分别是线程调度状态（state=S）
 - schedstat=( 173407732 13795308 237 ) 从 /proc/[pid]/task/[tid]/schedstat读出，三个值分别表示线程在cpu上执行的时间、线程的等待时间和线程执行的时间片长度，有的android内核版本不支持这项信息，得到的三个值都是0
 - 线程用户态下使用的时间值(单位是jiffies）（utm=14） 
 - 内核态下得调度时间值（stm=2）
 - 最后运行该线程的cup核的序号（core=5）；

**第五行**：`  | stack=0x7ff4d74000-0x7ff4d76000 stackSize=8MB`
主要信息包括：

 - 线程栈的地址（stack=0x7ff4d74000-0x7ff4d76000）
 - 栈大小（stackSize=8MB）；

另外，

```
Total number of allocations 50059
Total bytes allocated 12MB
Total bytes freed 977KB
Free memory 2MB
Free memory until GC 2MB
Free memory until OOME 244MB
Total memory 13MB
Max memory 256MB
```

这里介绍了进程的内存使用情况，有时可以辅助我们分析 ANR。

再后面就是线程的调用栈信息，也是我们分析ANR的关键信息。
由于上面的log是在主线程中调用 Thread.sleep 导致的，直接通过调用栈信息就可以定位到问题发生的原因，这里不再介绍。