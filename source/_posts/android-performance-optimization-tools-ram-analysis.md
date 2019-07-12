---
title: Android 性能优化工具篇 -- 内存查看和分析工具
categories: Android性能优化
comments: true
tags: [内存优化]
description: 内存查看和分析工具汇总，并简单介绍一下procrank和dumpsys meminfo的用法
date: 2018-10-10 10:00:00
---


## 概述

在内存优化前我们有必要借助一些工具来查看我们的 APP 目前使用内存的状况，为我们内存优化指明方向。
Android 中关于的工具很多，参考下表。所以我们要灵活地选用我们需要的工具。在定位内存问题的过程中，推荐使用涵盖一定初步定位和定位能力的工具，可以让我们一步到位地剖析问题、提升效率。

| 工具 | 分析问题 | 能力 | 备注 |
|:-------------:|:-------------:|:-------------:|:-------------:|
| top/procrank | 内存占用过大，内存泄漏 | 发现 |  |
| meminfo |  |  |  |
| StrickMode | Activity 内存泄漏 | 自动发现+初步定位 |  |
| LeakCanary | Activity 内存泄漏 | 自动发现+定位 |  |
| Systrace | GC导致的卡顿 | 发现 |  |
| Memory Monitor/Memory Profiler | 申请内存大小和时间 | 发现 | Android Studio 3.0 采用全新的 Android Profiler 窗口取代 Android Monitor 工具。 |
| HeapViewer | 查看APP内存分配大小和空闲内存大小 | 发现 |  |
| Allocation Tracker | 申请内存次数过多和多大，辅助定位GC log发现的问题 | 发现+定位 |  |
| Android Lint | 内存优化建议 | 发现+定位 |  |
| MAT | 内存泄漏 | 发现+定位 |  |



## procrank

需要root权限才能运行

```
bogon:TestSomething heqiang$ adb shell procrank
  PID       Vss      Rss      Pss      Uss  cmdline
18283  1730504K  375064K  263512K  247516K  com.android.browser
 3030  2947812K  412340K  166613K  110032K  system_server
 3224  2918272K  245324K  114479K   98196K  com.android.systemui
 3880  2676696K  232392K  113036K  105568K  com.meizu.safe
 5198  1976632K  201196K   85256K   44832K  com.tencent.mm
 7982  1916864K  182152K   79986K   75428K  com.android.mms
 9154  2282932K  185652K   70449K   64024K  com.meizu.flyme.launcher
 6049  1909944K  175776K   65801K   26212K  com.tencent.mm:push
 5034  1311876K  122188K   40598K   34560K  com.meizu.mstore
 3211  1305292K  122488K   38756K   32440K  com.meizu.flyme.input
 4051  1920588K  137356K   38402K   34620K  com.meizu.alphame
10578  2240584K  146452K   36268K   30252K  com.android.hq.ganktoutiao
 3373  2386504K  135396K   35837K   26664K  com.android.systemui:recents
 3777  1830536K  137488K   35311K   31452K  com.flyme.systemuitools
 7901  1276760K  110056K   35073K   29628K  com.meizu.media.music
 3454  1842652K  132772K   33837K   28928K  com.android.phone
 8179  1788376K  106376K   31661K   26564K  com.meizu.netcontactservice
18024  2229044K  132416K   30948K   26108K  com.meizu.connectivitysettings
12256  2351712K  133980K   28352K   20208K  com.example.heqiang.testsomething
 8302    65608K   29980K   28222K   28196K  /data/local/tmp/perfd/perfd
23169  1817152K  120388K   26505K   23308K  com.meizu.location
 3766  1769968K  100584K   26504K   20976K  com.aricent.ims.service
 2674  2220164K  135512K   26358K   13476K  zygote64
 2675  1661988K  120100K   24547K    6932K  zygote
13214  1805296K  122076K   24393K   20576K  com.meizu.flyme.weather
18473    89192K   27524K   23947K   23188K  /data/app/com.android.browser-1/lib/arm/libweexjsb.so
 8091  1815400K  117116K   23922K   20872K  com.android.settings
 7973    88168K   26464K   23115K   22580K  /data/app/com.meizu.media.music-2/lib/arm/libweexjsb.so
 4359  1819236K  116936K   22713K   19668K  com.meizu.dataservice
 5666  1804476K  111588K   22225K   19512K  com.meizu.testdev.woody
 5964  1824344K  113516K   21408K   18608K  com.android.providers.downloads
 4901  1257932K   96244K   21112K   16716K  com.meizu.account
 4751  1904100K  114672K   20689K   17728K  com.meizu.battery
 4012  1883384K  107076K   20310K   17796K  com.zq.powerdoctor
 4826  1237936K   97156K   19583K   14632K  com.meizu.flyme.service.find
 3815  1816684K  111760K   19560K   16656K  com.meizu.pps
 8031  1778064K   82436K   19298K   15800K  com.meizu.media.gallery
23110  1798528K  111224K   18338K   15428K  com.meizu.mzsyncservice
 2530   602168K   32796K   18308K   13616K  /system/bin/surfaceflinger
 2677   136676K   30624K   16063K   10740K  /system/bin/cameraserver
29320  1855236K  108244K   15492K   12644K  com.meizu.logreport
 5918  1796792K  107920K   14689K   11768K  com.meizu.wifiadmin
 5945  1236844K   92000K   14636K    9932K  com.meizu.net.pedometer
 4070  1241304K   90588K   14061K    8376K  com.meizu.cloud
29878  1791328K  101224K   13785K   11224K  com.meizu.flyme.update
 4995  1226176K   85184K   12021K    6608K  com.meizu.cloud:mzservice_v1
 4033  1775632K   84580K    9008K    6828K  com.android.incallui
 6013  1786868K   83752K    8263K    6248K  com.meizu.mzbasestationsafe
 2676    45476K   12336K    7147K    6760K  /system/bin/audioserver
 3786  1768168K   83260K    6587K    4424K  android.ext.services
 4092  1771200K   84608K    6359K    4188K  com.meizu.experiencedatasync
 5089  1211960K   69776K    6176K    2664K  com.activate.activatephone
 7885  1769652K   82556K    5920K    3864K  android.process.media
 8737  1767884K   79344K    5332K    3352K  com.android.carrierconfig
 3270  1772648K   80116K    5106K    3056K  com.meizu.facerecognition
 2687    64804K   14556K    5059K    3484K  /system/bin/mediaserver
 2468    28244K    7024K    4932K    4864K  /system/bin/logd
 7932  1778632K   79032K    4702K    2724K  com.samsung.slsi.telephony.usbmodeswitch
 4002  1768104K   76208K    4480K    2548K  com.goodix.fingerprint
 4878  1772096K   79028K    4464K    2452K  com.sec.jniril
 6041  1768856K   76504K    4355K    2384K  com.samsung.slsi.CtcSelfRegister
 5832  1769012K   77868K    4346K    2352K  com.wolfsonmicro.ez2control:ez2control_service
 4081  1767784K   77220K    4193K    2160K  com.wolfsonmicro.ez2control
 2684    37532K   10632K    4116K    3512K  media.codec
 2686    44948K   11684K    3522K    2688K  media.extractor
 2690    49204K    8156K    3366K    3132K  /system/bin/rild_exynos
 2672    26204K    7204K    3250K    3132K  /system/bin/glgps
 2688    45504K    6540K    3132K    3032K  /system/bin/netd
 3394    13472K    7056K    2681K    2540K  /system/bin/wpa_supplicant
 2685    18204K    9196K    2370K    1512K  /system/bin/mediadrmserver
 2471    53924K    6068K    1974K    1796K  /system/bin/vold
 2669    22000K    7024K    1887K    1500K  /system/bin/ez2controld
 2668    19120K    5056K    1648K    1568K  /system/bin/fingerprintd
 2681    15512K    6624K    1566K    1284K  /system/bin/drmserver
 8140    20524K    1664K    1518K    1500K  /sbin/adbd
    1    10484K    2196K    1382K    1020K  /init
 2683    12744K    5408K    1304K    1180K  /system/bin/keystore
 2689    13324K    3712K    1281K    1236K  /sbin/rfsd
 2671    14416K    4828K    1252K    1148K  /system/bin/lhd
 2467    12260K    2952K    1050K    1020K  /system/vendor/bin/teei_daemon
 8795     9500K    3000K    1049K    1012K  procrank
 2696    12432K    4516K     912K     828K  /system/bin/gatekeeperd
 2682     9728K    3712K     878K     796K  /system/bin/installd
 1678     5972K    1580K     794K     436K  /sbin/ueventd
 2692    19068K    3476K     771K     720K  /system/bin/vcd
 2525     6724K     968K     734K     732K  /sbin/healthd
 2836    10724K    3592K     702K     652K  /system/bin/ifaad
 2528     9044K    3012K     702K     668K  /system/bin/lmkd
 2837    10676K    3504K     667K     620K  /system/bin/fidod
 2695    10676K    3616K     666K     616K  /system/vendor/bin/facerecod
 2678    10884K    3488K     656K     608K  /sbin/cbd
 2529     9100K    2916K     648K     608K  /system/bin/servicemanager
 2697     8384K    2976K     635K     584K  /system/xbin/perfprofd
 2526     7700K    2500K     629K     464K  /system/bin/sh
 2670     9568K    3416K     616K     556K  /system/bin/autobingend
 8300     7700K    2528K     598K     436K  /system/bin/sh
 2694     8144K    2784K     558K     508K  /system/bin/mdevice_manager
 8277     7988K    2700K     549K     508K  logcat
 2693     8020K    2736K     525K     488K  /system/bin/mpower_manager
 2680    17300K    2668K     508K     472K  /system/bin/dmd
 2691     7628K    2332K     420K     388K  /system/bin/sced
 2431     5640K     952K     409K     144K  /sbin/watchdogd
 2470    10252K    3044K     367K      52K  /system/bin/debuggerd64
 2469     5500K    2600K     349K      64K  /system/bin/debuggerd
 2482     9868K    1168K     335K      52K  debuggerd64:signaller
 2483     5116K     948K     272K      52K  debuggerd:signaller
                           ------   ------  ------
                          2174242K  1731360K  TOTAL

 RAM: 5825640K total, 384368K free, 101036K buffers, 2805404K cached, 16472K shmem, 541972K slab
```

输出结果是按照 PSS 值从大到小进行排列的。

 - VSS：Virtual Set Size 虚拟耗用内存（包含共享库占用的内存）是单个进程全部可访问的地址空间
 - RSS：Resident Set Size 实际使用物理内存（包含共享库占用的内存）是单个进程实际占用的内存大小，对于单个共享库， 尽管无论多少个进程使用，实际该共享库只会被装入内存一次。
 - PSS：Proportional Set Size 实际使用的物理内存（比例分配共享库占用的内存）
 - USS：Unique Set Size 进程独自占用的物理内存（不包含共享库占用的内存）USS 是一个非常非常有用的数字， 因为它揭示了运行一个特定进程的真实的内存增量大小。如果进程被终止， USS 就是实际被返还给系统的内存大小。

USS 是针对某个进程开始有可疑内存泄露的情况，进行检测的最佳数字。怀疑某个程序有内存泄露可以查看这个值是否一直有增加

一般来说内存占用大小有如下规律：VSS >= RSS >= PSS >= USS

## dumpsys meminfo

我们可以使用下面的命令来查看某个进程使用内存的情况：

```
    adb shell dumpsys meminfo -a <进程名称>
```

输出结果：

```
Applications Memory Usage (in Kilobytes):
Uptime: 638409159 Realtime: 1205461779

** MEMINFO in pid 8118 [com.android.hq.ganktoutiao] **
                   Pss      Pss   Shared  Private   Shared  Private  SwapPss     Heap     Heap     Heap
                 Total    Clean    Dirty    Dirty    Clean    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------   ------   ------   ------
  Native Heap    18071        0      640    18004        0        0       18    29696    22848     6847
  Dalvik Heap     3992        0      196     3972        0        0       23     6443     3222     3221
 Dalvik Other     1600        0       16     1600        0        0        0                           
        Stack      120        0        0      120        0        0        0                           
       Ashmem        2        0        4        0        8        0        0                           
      Gfx dev    14500        0        0    11136        0     3364        0                           
    Other dev       19        0      108        0        0       16        0                           
     .so mmap     9349     6092     1072      420    14984     6092       31                           
    .jar mmap        8        0        0        8        0        0        0                           
    .apk mmap      227      100        0        0     1388      100        0                           
    .ttf mmap      258      192        0        0      188      192        0                           
    .dex mmap    11583     9716        0        4     8008     9716        0                           
    .oat mmap      379       80        0        0     2568       80        0                           
    .art mmap     5942      248      324     5344     5232      248        4                           
   Other mmap     2603        0       16        4     3076     1708        0                           
   EGL mtrack    27744        0        0    27744        0        0        0                           
    GL mtrack    28956        0        0    28956        0        0        0                           
      Unknown     1469        0      280     1420        0        0        4                           
        TOTAL   126902    16428     2656    98732    35452    21516       80    36139    26070    10068
 
 Dalvik Details
        .Heap     3528        0        0     3528        0        0        0                           
         .LOS      156        0        8      156        0        0       20                           
      .Zygote      236        0      188      216        0        0        3                           
   .NonMoving       72        0        0       72        0        0        0                           
 .LinearAlloc      320        0        0      320        0        0        0                           
          .GC       84        0       12       84        0        0        0                           
    .JITCache     1088        0        0     1088        0        0        0                           
 .IndirectRef      108        0        4      108        0        0        0                           
   .Boot vdex     8099     6236        0        0     8008     6236        0                           
     .App dex     1036     1032        0        4        0     1032        0                           
    .App vdex     2448     2448        0        0        0     2448        0                           
     .App art     1092        8        0     1084        0        8        0                           
    .Boot art     4850      240      324     4260     5232      240        4                           
 
 App Summary
                       Pss(KB)
                        ------
           Java Heap:     9564
         Native Heap:    18004
                Code:    16612
               Stack:      120
            Graphics:    71200
       Private Other:     4748
              System:     6654
 
               TOTAL:   126902       TOTAL SWAP PSS:       80
 
 Objects
               Views:      593         ViewRootImpl:        2
         AppContexts:        2           Activities:        1
              Assets:        4        AssetManagers:        3
       Local Binders:       17        Proxy Binders:       25
       Parcel memory:        5         Parcel count:       23
    Death Recipients:        3      OpenSSL Sockets:        1
            WebViews:        0
 
 Dalvik
         isLargeHeap:    false
 
 SQL
         MEMORY_USED:      165
  PAGECACHE_OVERFLOW:       24          MALLOC_SIZE:      117
 
 DATABASES
      pgsz     dbsz   Lookaside(b)          cache  Dbname
         4       24             59         2/17/3  /data/user/0/com.android.hq.ganktoutiao/databases/gank.db
 
 Asset Allocations
    zip:/data/app/com.android.hq.ganktoutiao-jidQh1vu54x_dgItu-668g==/base.apk:/assets/fonts/material-design-iconic-font-v2.2.0.ttf: 97K
```

现在对上面输出内容的各项指标来做个解释：

横轴指标：

 - Native Heap：Native 分配的内存，就是非 Java 代码分配的内存。
 - Dalvik Heap：Java对象分配的占据内存。
 - Dalvik Other：类数据结构和索引占据内存
 - Stack：栈内存
 - Ashmem：匿名共享内存
 - Other dev：内部driver占用的内存
 - .so mmap：so库占用的内存
 - .apk mmap：apk占用的内存
 - .ttf mmap：ttf文件占用过的内存
 - .dex mmap：
 - .oat mmap：
 - .art mmap：
 - Other mmap
 - App Summary：
  - Java Heap：从 Java 或 Kotlin 代码分配的对象内存。
  - Native Heap：从 C 或 C++ 代码分配的对象内存。
  - Code：您的应用用于处理代码和资源（如 dex 字节码、已优化或已编译的 dex 码、.so 库和字体）的内存。
  - Stack：您的应用中的原生堆栈和 Java 堆栈使用的内存。 这通常与您的应用运行多少线程有关。
  - Graphics：图形缓冲区队列向屏幕显示像素（包括 GL 表面、GL 纹理等等）所使用的内存。 （请注意，这是与 CPU 共享的内存，不是 GPU 专用内存。）
  - Private Other
  - System
 
对于应用的内存，一般我们主要关注 App Summary 就行了。

## Memory Profiler

[Android 官方文档：使用 Memory Profiler 查看 Java 堆和内存分配](https://developer.android.com/studio/profile/memory-profiler)

## Mat

[Android 性能优化工具篇 -- MAT分析内存泄漏 ](http://www.heqiangfly.com/2016/02/20/android-performance-optimization-use-mat-to-analyse-leak/)


## 参考

《Android 移动性能实战》