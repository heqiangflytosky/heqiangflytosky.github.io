---
title: Android PackageManager相关源码分析之PMS初始化
categories: Android源码分析
comments: true
tags: [Android源码分析, PackageManager]
description: 基于Android6.0来分析Android的应用管理系统，本文简单介绍PackageManagerService相关类以及相关adb shell命令，重点讲解PackageManagerService的初始化。
date: 2016-5-6 10:00:00
---
	
## 概述

`PackageManagerService` 作为Android系统中最常用的服务之一，是我们经常要与之打交道的。应用的安装、卸载、优化以及系统已安装应用信息的扫描和查询等应用管理工作都是 PMS 来完成的。
与其它系统服务的实现类似，应用管理也采用了经由 `Binder` 调用的远程服务机制。`PackageManager`为暴露给用户的接口，`PackageManagerService`为接口的底层实现。
PMS（下文会以此来代替`PackageManagerService`）在启动时会扫描所有的 APK 文件和 Jar 包，然后把他们的信息读取出来，保存在内存中，这样系统运行时就能迅速找到各种应用和组建的信息。扫描中如果遇到没有优化过的文件还要进行优化工作（dex格式转换成oat格式（Android5.0以前是odex）），优化后的文件放在 /data/dalvik-cache/ 下面，可以看一下[这篇文章](http://blog.csdn.net/cnzx219/article/details/48714121)。
PMS要扫描的应用分为系统应用和普通应用通常都在下面的三个目录中：/system/app，/system/priv-app，/data/app。

 - 系统应用：安装在/system/app，/system/priv-app，/vendor/app或者/oem/app中。/system/app存放的是一些系统级的应用，比如电话和联系人等，/system/priv-app存放的是系统底层的应用，比如SystemUI，Setting和Laucher等。通常情况下这些应用是不能卸载的，可以升级的，升级的安装包放在/data/app下面
 - 普通应用：一般指用户安装的第三方应用，位于/data/app中。

`/data/dalvik-cache`目录下面保存的就是被优化过的APK文件和Jar文件。
`/data/data/<包名>/`保存的是应用的一些数据，包括数据库和一些设置文件。

本文的代码是基于Android6.0来进行介绍。
**Android应用管理系统**相关代码位置：

```
frameworks/base/core/java/android/content/pm/
frameworks/base/services/core/java/com/android/server/pm/
frameworks/base/services/core/java/android/content/pm/PackageParser.java
frameworks/base/services/core/java/com/android/server/SystemConfig.java
frameworks/base/packages/DefaultContainerService/src/com/android/defcontainer/DefaultContainerService.java
```

## PackageManagerService及相关类介绍

下图列出了 PMS 及客户端的类关系：

![效果图](/images/android-source-code-analysis-package-manager-init/packagemanagerservice_relations.png)

### ApplicationPackageManager

为什么首先介绍`ApplicationPackageManager`呢？因为我们客户端和 PMS 打交道全靠它了。
不管我们是想安装还是卸载应用，都要首先调用的是`Context`的`getPackageManager()`来获取`PackageManager`，这便是我们的主角`ApplicationPackageManager`。
当我们通过如下代码来获取系统安装应用的信息：

```java
        final PackageManager packageManager = getPackageManager();
        final Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);
        mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);

        List<ResolveInfo> apps = packageManager.queryIntentActivities(mainIntent, 0);
```

便会调用到`ApplicationPackageManager.queryIntentActivities(Intent intent,Intent intent,)`方法。
`ApplicationPackageManager` 继承自 `PackageManager`，它的初始化是在`ContextImpl`中进行的：

```java
    @Override
    public PackageManager getPackageManager() {
        if (mPackageManager != null) {
            return mPackageManager;
        }

        IPackageManager pm = ActivityThread.getPackageManager();
        if (pm != null) {
            // Doesn't matter if we make more than one instance.
            return (mPackageManager = new ApplicationPackageManager(this, pm));
        }

        return null;
    }
```

`PackageManager`类定义了应用可以操作PMS的接口。
`ApplicationPackageManager`可以称为PMS的代理对象，它通过它的成员变量`mPM`来与`PackageManagerService`进行进程间的通信，`mPM`指向一个`IPackageManager.Stub.Proxy`类型的对象。我们调用`ApplicationPackageManager`的方法，方法内都是通过调用`mPM`的相应方法来实现的。

### IPackageManager

`IPackageManager.java`是通过`IPackageManager.aidl`来生成的，源文件在`out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/src/core/java/android/content/pm/IPackageManager.java`，如果没有编译过代码的话，也可使用aidl工具单独处理`IPackageManager.aidl`。
`IPackageManager`接口类中定义了服务端和客户端通信的业务函数，还定义了内部类`Stub`，该类继承自`Binder`并实现了`IPackageManager`接口。
`Stub`类中定义了一个内部类`Proxy`，该类有一个`IBinder`类型（实际类型为`BinderProxy`）的成员变量`mRemote`，它用于和服务端`PackageManagerService`通信。

### PackageManagerService

PMS 继承自`IPackageManager.Stub`，`Stub`类从`Binder`派生，因此 PMS 将作为服务端参与`Binder`通信。
先来看几个重要的成员变量：

 - mInstallerService：`PackageInstallerService`的实例，一个应用的安装时间比较长，Android就用`PackageInstallerService`来管理应用的安装过程。在构造函数的最后创建。
 - mInstaller：被`@GuardedBy`注解标记，它是`Installer`的实例，用于和`Deamon`进程`installd`交互。实际上系统中进行APK格式转换、建立数据目录等工作都是由`installd`进程来完成的，后面会有介绍。
 - mSettings：`Settings`类的实例，保存一些PMS动态设置信息，后面会详细介绍；
 - mPackages：是被`@GuardedBy`注解标记的，代表系统中已经安装的package；
 - mExpectingBetter：被升级过的应用列表
 - mOnlyCore：用于判断是否只扫描系统库

PMS的创建是在`SystemServer`中的`SystemServer().run()`->`startBootstrapServices()`方法中进行的，它通过调用`PackageManagerService.main`来实现：

```java
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        //判断是不是第一次开机启动
        mFirstBoot = mPackageManagerService.isFirstBoot();
```

在`startCoreServices()`和`startOtherServices()`中调用：

```java
        mPackageManagerService.getUsageStatsIfNoPackageUsageInfo();

        ……

        //做dex优化，dex是Android上针对Java字节码的一种优化技术，可提高运行效率
        mPackageManagerService.performBootDexOpt();

        ……

        //通知系统进入就绪状态
        mPackageManagerService.systemReady();
```

### DefaultContainerService
`DefaultContainerService`服务，这个服务执行一些针对的文件复制和删除等相关工作

## 相关adb命令

### adb shell pm

```
adb shell pm

usage: pm list packages [-f] [-d] [-e] [-s] [-3] [-i] [-u] [--user USER_ID] [FILTER]
       pm list permission-groups
       pm list permissions [-g] [-f] [-d] [-u] [GROUP]
       pm list instrumentation [-f] [TARGET-PACKAGE]
       pm list features
       pm list libraries
       pm list users
       pm path PACKAGE
       pm dump PACKAGE
       pm install [-lrtsfd] [-i PACKAGE] [--user USER_ID] [PATH]
       pm install-create [-lrtsfdp] [-i PACKAGE] [-S BYTES]
               [--install-location 0/1/2]
               [--force-uuid internal|UUID]
       pm install-write [-S BYTES] SESSION_ID SPLIT_NAME [PATH]
       pm install-commit SESSION_ID
       pm install-abandon SESSION_ID
       pm uninstall [-k] [--user USER_ID] PACKAGE
       pm set-installer PACKAGE INSTALLER
       pm move-package PACKAGE [internal|UUID]
       pm move-primary-storage [internal|UUID]
       pm clear [--user USER_ID] PACKAGE
       pm enable [--user USER_ID] PACKAGE_OR_COMPONENT
       pm disable [--user USER_ID] PACKAGE_OR_COMPONENT
       pm disable-user [--user USER_ID] PACKAGE_OR_COMPONENT
       pm disable-until-used [--user USER_ID] PACKAGE_OR_COMPONENT
       pm hide [--user USER_ID] PACKAGE_OR_COMPONENT
       pm unhide [--user USER_ID] PACKAGE_OR_COMPONENT
       pm grant [--user USER_ID] PACKAGE PERMISSION
       pm revoke [--user USER_ID] PACKAGE PERMISSION
       pm reset-permissions
       pm set-app-link [--user USER_ID] PACKAGE {always|ask|never|undefined}
       pm get-app-link [--user USER_ID] PACKAGE
       pm set-install-location [0/auto] [1/internal] [2/external]
       pm get-install-location
       pm set-permission-enforced PERMISSION [true|false]
       pm trim-caches DESIRED_FREE_SPACE [internal|UUID]
       pm create-user [--profileOf USER_ID] [--managed] USER_NAME
       pm remove-user USER_ID
       pm get-max-users
```

### adb shell dumpsys package

```
$ adb shell dumpsys package com.android.hq.ganktoutiao

Activity Resolver Table:
  Non-Data Actions:
      android.intent.action.MAIN:
        c51a490 com.android.hq.ganktoutiao/.ui.activity.MainActivity

Registered ContentProviders:
  com.android.hq.ganktoutiao/.provider.GankContentProvider:
    Provider{6472189 com.android.hq.ganktoutiao/.provider.GankContentProvider}

ContentProvider Authorities:
  [gank]:
    Provider{6472189 com.android.hq.ganktoutiao/.provider.GankContentProvider}
      applicationInfo=ApplicationInfo{177358e com.android.hq.ganktoutiao}

Key Set Manager:
  [com.android.hq.ganktoutiao]
      Signing KeySets: 11

Packages:
  Package [com.android.hq.ganktoutiao] (be741af):
    userId=10082
    pkg=Package{3d479bc com.android.hq.ganktoutiao}
    codePath=/data/app/com.android.hq.ganktoutiao-1
    resourcePath=/data/app/com.android.hq.ganktoutiao-1
    legacyNativeLibraryDir=/data/app/com.android.hq.ganktoutiao-1/lib
    primaryCpuAbi=null
    secondaryCpuAbi=null
    versionCode=1 targetSdk=23
    versionName=1.0.0
    splits=[base]
    applicationInfo=ApplicationInfo{177358e com.android.hq.ganktoutiao}
    flags=[ DEBUGGABLE HAS_CODE ALLOW_CLEAR_USER_DATA ALLOW_BACKUP ]
    dataDir=/data/user/0/com.android.hq.ganktoutiao
    supportsScreens=[small, medium, large, xlarge, resizeable, anyDensity]
    timeStamp=2017-06-01 17:22:26
    firstInstallTime=2017-06-01 17:22:28
    lastUpdateTime=2017-06-01 17:22:28
    signatures=PackageSignatures{1f74d45 [8351c9a]}
    installPermissionsFixed=true installStatus=1
    pkgFlags=[ DEBUGGABLE HAS_CODE ALLOW_CLEAR_USER_DATA ALLOW_BACKUP ]
    requested permissions:
      android.permission.INTERNET
      android.permission.WRITE_EXTERNAL_STORAGE
      android.permission.READ_EXTERNAL_STORAGE
      android.permission.ACCESS_NETWORK_STATE
    install permissions:
      android.permission.INTERNET: granted=true
      android.permission.ACCESS_NETWORK_STATE: granted=true
    User 0:  installed=true hidden=false stopped=false notLaunched=false enabled=0
      gids=[3003, 1028, 1015]
      runtime permissions:
        android.permission.READ_EXTERNAL_STORAGE: granted=true
        android.permission.WRITE_EXTERNAL_STORAGE: granted=true
```

## PackageManagerService 初始化

### PMS 方法名LI、LP的含义

PMS中很多的方法都带有LI、LP、Lpr、LPw这样的后缀，它们代表着什么意思呢？首先进行一下这个释疑，以助于理解这些方法。
先看一下代码注释的解释：

```java
    // Lock for state used when installing and doing other long running
    // operations.  Methods that must be called with this lock held have
    // the suffix "LI".
    final Object mInstallLock = new Object();

    // Keys are String (package name), values are Package.  This also serves
    // as the lock for the global state.  Methods that must be called with
    // this lock held have the prefix "LP".
    @GuardedBy("mPackages")
    final ArrayMap<String, PackageParser.Package> mPackages =
            new ArrayMap<String, PackageParser.Package>();
```

这下就比较容易理解了：

 - LI：带LI后缀的方法名表示该函数被调用时需要持有`mInstallLock`锁
 - LP：带LP后缀的方法名表示该函数被调用时需要持有`mPackages`锁
 - LPr：表示读
 - LPw：表示写

### main函数和构造函数分析

```java
    public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        //调用构造函数
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        //向ServiceManager注册package服务
        ServiceManager.addService("package", m);
        return m;
    }
```
我们可以看到`main`函数其实很简单，就是调用了一下PMS的构造函数以及注册了 PMS，但是 PMS 的构造函数却做了很多工作，代码有将近500行之多，下面就详细分析一下 PMS 的构造函数。

构造函数的主要工作：包括读取配置文件、优化APK文件和Jar包、扫描系统中所有安装的应用以及提取这些应用的信息并保存起来。
下面就分析一些PMS的构造函数，主要做了以下的工作：
 - 变量的初始化工作，包括mSettings，mInstaller，mPackageDexOptimizer等等
 - 读取配制文件
 - 扫描系统Package，包含Dex优化，
 - 保存扫描信息
 - 扫描非系统应用
 - 更新数据

#### 一些初始化工作

```java
    public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        ……
        mContext = context;
        //是否是工厂测试模式
        mFactoryTest = factoryTest;
        //是否需要扫描系统库，默认为false，需要扫描
        mOnlyCore = onlyCore;
        //是否要做dex优化，eng或者标记了lazydexopt则不优化
        mLazyDexOpt = "eng".equals(SystemProperties.get("ro.build.type")) ||
                SystemProperties.getBoolean("persist.sys.perf.lazydexopt", false);
        //初始化mMetrics，存储与显示屏相关的一些属性，例如屏幕的宽/高尺寸，分辨率等信息
        mMetrics = new DisplayMetrics();
        //创建mSettings对象
        mSettings = new Settings(mPackages);
        //设置UID，添加SharedUserSetting对象到Settings中，UID相同的包可以运行在同一个进程中，或者可以相互读取资源。这里添加了6中系统UID：system、radio、log、nfc、bluetooth和shell
        mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        ……
        //创建一个Installer对象
        mInstaller = installer;
        ……
```

#### 读取配制文件

```java
        ……
        mPackageDexOptimizer = new PackageDexOptimizer(this);
        mMoveCallbacks = new MoveCallbacks(FgThread.get().getLooper());

        mOnPermissionChangeListeners = new OnPermissionChangeListeners(
                FgThread.get().getLooper());
        //获取设备屏幕信息
        getDefaultDisplayMetrics(context, mMetrics);

        //SystemConfig用于获取系统的全局配置信息，初始化mGlobalGids、mSystemPermissions和mAvailableFeatures
        SystemConfig systemConfig = SystemConfig.getInstance();
        mGlobalGids = systemConfig.getGlobalGids();
        mSystemPermissions = systemConfig.getSystemPermissions();
        mAvailableFeatures = systemConfig.getAvailableFeatures();

        synchronized (mInstallLock) {
        // writer
        synchronized (mPackages) {
            // 创建用来处理消息的线程，并加入到Watchdog的监控中
            mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
            mHandlerThread.start();
            mHandler = new PackageHandler(mHandlerThread.getLooper());
            Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);

            //在/data目录下面创建子目录
            File dataDir = Environment.getDataDirectory();
            mAppDataDir = new File(dataDir, "data");      // /data/data存放应用数据的目录
            mAppInstallDir = new File(dataDir, "app");      // /data/app存放安装的应用
            mAppLib32InstallDir = new File(dataDir, "app-lib");      // /data/app-lib 存放应用自动的native库
            mAsecInternalPath = new File(dataDir, "app-asec").getPath();
            mUserAppDataDir = new File(dataDir, "user");      // /data/user 存放用户的数据文件
            mDrmAppPrivateInstallDir = new File(dataDir, "app-private");      // /data/app-private 存放drm保护的应用

            //实例化sUserManager，管理多用户
            sUserManager = new UserManagerService(context, this,
                    mInstallLock, mPackages);

            // Propagate permission configuration in to package manager.
            //获取系统中定义的permissions，这些permissions从/etc/permissions目录下面读取的
            ArrayMap<String, SystemConfig.PermissionEntry> permConfig
                    = systemConfig.getPermissions();
            ……
            //保存到mSettings.mPermissions中
            mSettings.mPermissions.put(perm.name, bp);
            ……

            // 通过SystemConfig得到系统中的共享库列表
            ArrayMap<String, String> libConfig = systemConfig.getSharedLibraries();
            for (int i=0; i<libConfig.size(); i++) {
                mSharedLibraries.put(libConfig.keyAt(i),
                        new SharedLibraryEntry(libConfig.valueAt(i), null));
            }

            // 打开SELinux的policy文件/security/current/mac_permissions.xml 或者 etc/security/mac_permissions.xml
            mFoundPolicyFile = SELinuxMMAC.readInstallPolicy();

            // 读取packages.xml的内容，保存到Settings的几个成员变量中去
            mRestoredSettings = mSettings.readLPw(this, sUserManager.getUsers(false),
                    mSdkVersion, mOnlyCore);

```

#### 扫描系统Package

这部分工作主要是扫描系统应用以及Jar包

```java
            // 设置模块来代替framework-res.apk中缺省的ResolverActivity
            String customResolverActivity = Resources.getSystem().getString(
                    R.string.config_customResolverActivity);
            if (TextUtils.isEmpty(customResolverActivity)) {
                customResolverActivity = null;
            } else {
                mCustomResolverComponentName = ComponentName.unflattenFromString(
                        customResolverActivity);
            }

            // 记录开始扫描的时间
            long startTime = SystemClock.uptimeMillis();

            //配置扫描的参数
            final int scanFlags = SCAN_NO_PATHS | SCAN_DEFER_DEX | SCAN_BOOTING | SCAN_INITIAL;
            //保存一些已经进行dex优化过的apk，比如“framework-res.apk”、Java启动类库、framework所有核心库，这部分不需要在优化
            final ArraySet<String> alreadyDexOpted = new ArraySet<String>();

            //获取Java启动类库、framework所有核心库，在init.rc文件配置
            final String bootClassPath = System.getenv("BOOTCLASSPATH");
            final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");

            //把它们加入到已经优化集合中去
            if (bootClassPath != null) {
                String[] bootClassPathElements = splitString(bootClassPath, ':');
                for (String element : bootClassPathElements) {
                    alreadyDexOpted.add(element);
                }
            } else {...}

            if (systemServerClassPath != null) {
                String[] systemServerClassPathElements = splitString(systemServerClassPath, ':');
                for (String element : systemServerClassPathElements) {
                    alreadyDexOpted.add(element);
                }
            } else {...}

            final List<String> allInstructionSets = InstructionSets.getAllInstructionSets();
            final String[] dexCodeInstructionSets =
                    getDexCodeInstructionSets(
                            allInstructionSets.toArray(new String[allInstructionSets.size()]));

            // 对比当前系统的指令集，检查mSharedLibraries中记录的jar包是否需要转换成odex格式
            // mSharedLibraries变量中的动态库是通过SystemConfig.getSharedLibraries()从etc/permissions/platform.xml中读取出来的
            if (mSharedLibraries.size() > 0) {
                for (String dexCodeInstructionSet : dexCodeInstructionSets) {
                    for (SharedLibraryEntry libEntry : mSharedLibraries.values()) {
                        final String lib = libEntry.path;
                        if (lib == null) {
                            continue;
                        }

                        try {
                            int dexoptNeeded = DexFile.getDexOptNeeded(lib, null, dexCodeInstructionSet, false);
                            if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
                                alreadyDexOpted.add(lib);
                                //调用install 的dexopt命令，优化后的文件放在 /data/dalvik-cache/ 下面
                                mInstaller.dexopt(lib, Process.SYSTEM_UID, true, dexCodeInstructionSet, dexoptNeeded);
                            }
                        } catch (FileNotFoundException e) {} catch (IOException e) {}
                    }
                }
            }

            File frameworkDir = new File(Environment.getRootDirectory(), "framework");

            //把framework-res.apk加入到已优化列表中
            alreadyDexOpted.add(frameworkDir.getPath() + "/framework-res.apk");

            //把core-libart.jar加入到已优化列表中
            alreadyDexOpted.add(frameworkDir.getPath() + "/core-libart.jar");

            // 对framework目录下面的文件执行dex优化
            String[] frameworkFiles = frameworkDir.list();
            if (frameworkFiles != null) {
                for (String dexCodeInstructionSet : dexCodeInstructionSets) {
                    for (int i=0; i<frameworkFiles.length; i++) {
                        File libPath = new File(frameworkDir, frameworkFiles[i]);
                        String path = libPath.getPath();
                        if (alreadyDexOpted.contains(path)) {
                            continue;
                        }
                        // 只转化.apk和.jar文件
                        if (!path.endsWith(".apk") && !path.endsWith(".jar")) {
                            continue;
                        }
                        try {
                            int dexoptNeeded = DexFile.getDexOptNeeded(path, null, dexCodeInstructionSet, false);
                            if (dexoptNeeded != DexFile.NO_DEXOPT_NEEDED) {
                                //调用install 的dexopt命令，优化后的文件放在 /data/dalvik-cache/ 下面
                                mInstaller.dexopt(path, Process.SYSTEM_UID, true, dexCodeInstructionSet, dexoptNeeded);
                            }
                        } catch (FileNotFoundException e) {...} catch (IOException e) {...}
                    }
                }
            }

            ……

            // 扫描/vendor/overlay目录
            File vendorOverlayDir = new File(VENDOR_OVERLAY_DIR);
            scanDirLI(vendorOverlayDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags | SCAN_TRUSTED_OVERLAY, 0);

            // 扫描/system/framework目录
            scanDirLI(frameworkDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED,
                    scanFlags | SCAN_NO_DEX, 0);

            //扫描/system/priv-app目录
            final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
            scanDirLI(privilegedAppDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR
                    | PackageParser.PARSE_IS_PRIVILEGED, scanFlags, 0);

            //扫描/system/app目录
            final File systemAppDir = new File(Environment.getRootDirectory(), "app");
            scanDirLI(systemAppDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            //扫描/vendor/app目录下面的应用
            File vendorAppDir = new File("/vendor/app");
            try {
                vendorAppDir = vendorAppDir.getCanonicalFile();
            } catch (IOException e) {...}
            scanDirLI(vendorAppDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            // 扫描/oem/app目录下面的应用
            final File oemAppDir = new File(Environment.getOemDirectory(), "app");
            scanDirLI(oemAppDir, PackageParser.PARSE_IS_SYSTEM
                    | PackageParser.PARSE_IS_SYSTEM_DIR, scanFlags, 0);

            //调用installd执行movefiles命令，执行/system/etc/updatecmds下的命令脚本
            mInstaller.moveFiles();
```

#### 保存扫描信息

将上面扫描得到的信息进行整理，并保存到相对应的配置文件中去。

```java
            // 这个列表记录的是可能有升级包的系统应用
            final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<String>();
            if (!mOnlyCore) {
                Iterator<PackageSetting> psit = mSettings.mPackages.values().iterator();
                //遍历mSettings.mPackages中保存的应用
                while (psit.hasNext()) {
                    PackageSetting ps = psit.next();

                    //忽略掉非系统应用
                    if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
                        continue;
                    }

                    // 从mPackages中取到该应用，注意此处的mPackages不是mSettings.mPackages，而是通过scanDirLI得来的。
                    final PackageParser.Package scannedPkg = mPackages.get(ps.name);
                    if (scannedPkg != null) {
                        //如果这个应用是升级过的应用，那么从mPackages中移除出去
                        if (mSettings.isDisabledSystemPackageLPr(ps.name)) {
                            ...
                            removePackageLI(ps, true);
                            // 放入mExpectingBetter列表，后面会进行处理的
                            mExpectingBetter.put(ps.name, ps.codePath);
                        }

                        continue;
                    }

                    //这里表示的是mPackages中不存在的应用，也即系统中不存在
                    if (!mSettings.isDisabledSystemPackageLPr(ps.name)) {
                        // 如果不在升级过的应用列表中，说明这个应用是残留在packages.xml中的，可能还有数据目录，因此要从删掉
                        psit.remove();
                        logCriticalInfo(Log.WARN, "System package " + ps.name
                                + " no longer exists; wiping its data");
                        // 删除数据目录，内部也是通过installd来执行
                        removeDataDirsLI(null, ps.name);
                    } else {
                        // 这个应用不再系统中，但是被标记了<update-package>，则加入到possiblyDeletedUpdatedSystemApps列表
                        final PackageSetting disabledPs = mSettings.getDisabledSystemPkgLPr(ps.name);
                        if (disabledPs.codePath == null || !disabledPs.codePath.exists()) {
                            possiblyDeletedUpdatedSystemApps.add(ps.name);
                        }
                    }
                }
            }

            // 扫描并删除未安装成功的apk包
            ArrayList<PackageSetting> deletePkgsList = mSettings.getListOfIncompleteInstallPackagesLPr();
            for(int i = 0; i < deletePkgsList.size(); i++) {
                cleanupInstallFailedPackage(deletePkgsList.get(i));
            }
            // 删除临时文件
            deleteTempPackageFiles();

            // 删除掉Settings中的没有关联任何应用的SharedUserSetting对象
            mSettings.pruneSharedUsersLPw();
```

#### 扫描非系统应用

```java
            // 开始处理非系统应用
            if (!mOnlyCore) {
                ...
                // 扫描/data/app目录，保存到mPackages中
                scanDirLI(mAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);
                // 扫描/data/app-private目录
                scanDirLI(mDrmAppPrivateInstallDir, PackageParser.PARSE_FORWARD_LOCK,
                        scanFlags | SCAN_REQUIRE_KNOWN, 0);

                // possiblyDeletedUpdatedSystemApps表示packages.xml中被标记了<update-package>，但系统中没有这个文件，一次这里检查用户安装目录下有没有升级包
                for (String deletedAppName : possiblyDeletedUpdatedSystemApps) {
                    PackageParser.Package deletedPkg = mPackages.get(deletedAppName);
                    //从mSettings.mDisabledSysPackages变量中移除出去
                    mSettings.removeDisabledSystemPackageLPw(deletedAppName);

                    String msg;
                    if (deletedPkg == null) {
                        //用户目录中也没有升级包，则肯定是残留的应用信息，则把它的数据目录删除掉
                        msg = "Updated system package " + deletedAppName
                                + " no longer exists; wiping its data";
                        removeDataDirsLI(null, deletedAppName);
                    } else {
                        //如果/data/app下面有升级包，说明系统目录下的文件可能被删除掉了，因此把应用的系统属性去掉，以普通应用来看待
                        msg = "Updated system app + " + deletedAppName
                                + " no longer present; removing system privileges for "
                                + deletedAppName;

                        deletedPkg.applicationInfo.flags &= ~ApplicationInfo.FLAG_SYSTEM;

                        PackageSetting deletedPs = mSettings.mPackages.get(deletedAppName);
                        deletedPs.pkgFlags &= ~ApplicationInfo.FLAG_SYSTEM;
                    }
                    // 报告系统发生了不一致的情况
                    logCriticalInfo(Log.WARN, msg);
                }

                // 现在来处理mExpectingBetter列表，这个列表的应用是带有升级包的系统应用，前面把他们从mPackages去掉了
                for (int i = 0; i < mExpectingBetter.size(); i++) {
                    final String packageName = mExpectingBetter.keyAt(i);
                    if (!mPackages.containsKey(packageName)) {
                        final File scanFile = mExpectingBetter.valueAt(i);

                        logCriticalInfo(Log.WARN, "Expected better " + packageName
                                + " but never showed up; reverting to system");

                        final int reparseFlags;
                        // 确保应用位于下面的4个系统应用目录，如果不再，不需要处理
                        if (FileUtils.contains(privilegedAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR
                                    | PackageParser.PARSE_IS_PRIVILEGED;
                        } else if (FileUtils.contains(systemAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR;
                        } else if (FileUtils.contains(vendorAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR;
                        } else if (FileUtils.contains(oemAppDir, scanFile)) {
                            reparseFlags = PackageParser.PARSE_IS_SYSTEM
                                    | PackageParser.PARSE_IS_SYSTEM_DIR;
                        } else {
                            Slog.e(TAG, "Ignoring unexpected fallback path " + scanFile);
                            continue;
                        }

                        // 现在把这个apk标识为系统应用，从mSettings.mDisabledSysPackages中删除，
                        // 这里大家可能会有点疑惑，为什么删除呢？因为在scanDirLI->scanPackageLI中会执行mSettings.disableSystemPackageLPw，因此，此时该包名的标签是只有<update-package>
                        // 执行这步之后变成<package>标签，在下面的scanPackageLI中又会添加一个<update-package>标签的
                        mSettings.enableSystemPackageLPw(packageName);

                        try {
                            // 重新扫描一下这个文件，会再添加一个<update-package>标签
                            scanPackageLI(scanFile, reparseFlags, scanFlags, 0, null);
                        } catch (PackageManagerException e) {...}
                    }
                }
            }
            // 清空目录
            mExpectingBetter.clear();
```

#### 更新数据

将 `Settings` 的数据写入 packages.xml 中，实例化 `mInstallerService`。

```java
            // 更新所有应用的动态库路径
            updateAllSharedLibrariesLPw();

            for (SharedUserSetting setting : mSettings.getAllSharedUsersLPw()) {
                adjustCpuAbisForSharedUserLPw(setting.packages, null /* scanned package */,
                        false /* force dexopt */, false /* defer dexopt */);
            }

            mPackageUsage.readLP();

            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                    SystemClock.uptimeMillis());

            // 如果平台的SDK版本和上次启动时发生了变化，可能permission的定义也发生了变化，因此需要重新赋予应用权限
            int updateFlags = UPDATE_PERMISSIONS_ALL;
            if (ver.sdkVersion != mSdkVersion) {
                updateFlags |= UPDATE_PERMISSIONS_REPLACE_PKG | UPDATE_PERMISSIONS_REPLACE_ALL;
            }
            updatePermissionsLPw(null, null, updateFlags);
            ver.sdkVersion = mSdkVersion;

            // 如果是第一次启动或者是从AndroidM升级后的第一次启动，需要执行一些初始化工作
            if (!onlyCore && (mPromoteSystemApps || !mRestoredSettings)) {
                for (UserInfo user : sUserManager.getUsers(true)) {
                    mSettings.applyDefaultPreferredAppsLPw(this, user.id);
                    applyFactoryDefaultBrowserLPw(user.id);
                    primeDomainVerificationsLPw(user.id);
                }
            }

            // 如果是执行OTA后的第一次启动，需要清除cache
            if (mIsUpgrade && !onlyCore) {
                ...
                for (int i = 0; i < mSettings.mPackages.size(); i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if (Objects.equals(StorageManager.UUID_PRIVATE_INTERNAL, ps.volumeUuid)) {
                        deleteCodeCacheDirsLI(ps.volumeUuid, ps.name);
                    }
                }
                ver.fingerprint = Build.FINGERPRINT;
            }

            mExistingSystemPackages.clear();
            mPromoteSystemApps = false;

            ver.databaseVersion = Settings.CURRENT_DATABASE_VERSION;

            // 把Settings的内容保存到packages.xml中去
            mSettings.writeLPr();

            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                    SystemClock.uptimeMillis());

            mRequiredVerifierPackage = getRequiredVerifierLPr();
            mRequiredInstallerPackage = getRequiredInstallerLPr();

            // 创建PackageInstallerService
            mInstallerService = new PackageInstallerService(context, this);

            mIntentFilterVerifierComponent = getIntentFilterVerifierComponentNameLPr();
            mIntentFilterVerifier = new IntentVerifierProxy(mContext,
                    mIntentFilterVerifierComponent);

        } // synchronized (mPackages)
        } // synchronized (mInstallLock)

        // 启动一次内存垃圾回收
        Runtime.getRuntime().gc();

        LocalServices.addService(PackageManagerInternal.class, new PackageManagerInternalImpl());
```
至此，构造函数分析结束，构造函数的执行过程就是先读取保存在packages.xml文件的扫描结果，保存在Settings中，然后扫描几个目录下的文件，并对比上次扫描结果更新packages.xml文件。
