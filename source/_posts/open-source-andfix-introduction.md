---
title: Android热修复之AndFix使用简介
categories: Android开源项目
comments: true
tags: [Android, 热修复, AndFix]
date: 2016-12-15 10:00:00
---
## AndFix简介

AndFix，全称是 Android hot-fix。是阿里开源的一个Android热补丁框架，允许APP在不重新发布版本的情况下修复线上的bug，支持Android 2.3 到 6.0。
[GitHub源码](https://github.com/alibaba/AndFix)

## AndFix使用

### 使用方法

在build.gradle中添加依赖：

```
compile 'com.alipay.euler:andfix:0.5.0@aar'
```

添加读取SDcard权限

```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
```

Application：

```java
public class MainApplication extends Application {
    private static final String TAG = "MainApplication";
    private static final String APATCH_PATH = "/out.apatch";
    private PatchManager mPatchManager;
    @Override
    public void onCreate() {
        super.onCreate();

        // initialize
        mPatchManager = new PatchManager(this);
        mPatchManager.init("1.0");
        Log.e(TAG, "inited.");

        // load patch
        mPatchManager.loadPatch();
        Log.e(TAG, "apatch loaded.");

        // add patch at runtime
        try {
            // .apatch file path
            String patchFileString = Environment.getExternalStorageDirectory()
                    .getAbsolutePath() + APATCH_PATH;
            Log.e(TAG, "path:" + patchFileString );
            mPatchManager.addPatch(patchFileString);
            Log.e(TAG, "apatch:" + patchFileString + " added.");
        } catch (IOException e) {
            Log.e(TAG, "", e);
        }
    }
}
```

这里通过一个测试类来验证，修复之前：

```java
public class Test {
    public String getString(){
        return "There is a bug!";
    }
}
```

修复之后：

```java
public class Test {
    public String getString(){
        return "Bug has been fixed!";
    }
}
```

把获得的字符串现实在一个`TextView`上面。

### 生成Patch文件

使用工具apkpatch-1.0.3来生成patch文件，这个工具可以在这里[下载](https://github.com/alibaba/AndFix/raw/master/tools/apkpatch-1.0.3.zip)。
用法：

```
usage: apkpatch -f <new> -t <old> -o <output> -k <keystore> -p <***> -a <alias> -e <***>
 -a,--alias <alias>     keystore entry alias.
 -e,--epassword <***>   keystore entry password.
 -f,--from <loc>        new Apk file path.
 -k,--keystore <loc>    keystore path.
 -n,--name <name>       patch name.
 -o,--out <dir>         output dir.
 -p,--kpassword <***>   keystore password.
 -t,--to <loc>          old Apk file path.
```
原理：根据两个apk包，一个是线上已经发布的包，另外一个是新的修复bug的包，生成一个差异文件，就是所谓的补丁文件即patch文件。
根据下面命令生成一个.patch文件
```
./apkpatch.sh -f new.apk -t old.apk -o out.apatch -k debug.keystore -p android -a androiddebugkey -e android
add modified Method:Ljava/lang/String;  getString()  in Class:Lcom/android/hq/andfixtest/Test;
```
会在当前目录下面生成out.apatch目录，目录中.apatch后缀的文件就是我们需要的patch文件。

### 验证

进入目录将.apatch文件重命名后copy到手机中。先安装old.apk，运行，显示There is a bug！然后再将patch文件copy到sdcard，重启应用显示Bug has been fixed！测试成功！
之后应用会把patch放在/data/data/*****/files/apatch，sdcard目录下面的删除掉就可以了。

## 遇到的错误

### File too short to be a zip file

```
01-19 17:16:16.138  9508  9508 D AndFix  : vm is: art , apilevel is: 23
01-19 17:16:16.143  9508  9508 E PatchManager: addPatch
01-19 17:16:16.143  9508  9508 E PatchManager: java.util.zip.ZipException: File too short to be a zip file: 0
01-19 17:16:16.143  9508  9508 E PatchManager: 	at java.util.zip.ZipFile.readCentralDir(ZipFile.java:388)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at java.util.zip.ZipFile.<init>(ZipFile.java:175)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at java.util.jar.JarFile.<init>(JarFile.java:199)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at java.util.jar.JarFile.<init>(JarFile.java:182)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at java.util.jar.JarFile.<init>(JarFile.java:168)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at com.alipay.euler.andfix.patch.Patch.init(Patch.java:75)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at com.alipay.euler.andfix.patch.Patch.<init>(Patch.java:67)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at com.alipay.euler.andfix.patch.PatchManager.addPatch(PatchManager.java:126)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at com.alipay.euler.andfix.patch.PatchManager.initPatchs(PatchManager.java:112)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at com.alipay.euler.andfix.patch.PatchManager.init(PatchManager.java:105)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at com.android.hq.andfixtest.MainApplication.onCreate(MainApplication.java:25)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at com.android.tools.fd.runtime.BootstrapApplication.onCreate(BootstrapApplication.java:370)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at android.app.Instrumentation.callApplicationOnCreate(Instrumentation.java:1020)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at android.app.ActivityThread.handleBindApplication(ActivityThread.java:4952)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at android.app.ActivityThread.-wrap1(ActivityThread.java)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1531)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at android.os.Handler.dispatchMessage(Handler.java:102)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at android.os.Looper.loop(Looper.java:148)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at android.app.ActivityThread.main(ActivityThread.java:5660)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at java.lang.reflect.Method.invoke(Native Method)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:775)
01-19 17:16:16.143  9508  9508 E PatchManager: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:665)
```

加上读取SDCard权限，就不报这个错误了

```xml
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
```

### open failed: EISDIR

```
01-20 09:54:50.351 25113 25113 D MainApplication: inited.
01-20 09:54:50.352 25113 25113 D MainApplication: apatch loaded.
01-20 09:54:50.354 25113 25113 E MainApplication: 
01-20 09:54:50.354 25113 25113 E MainApplication: java.io.FileNotFoundException: /storage/emulated/0/out.apatch: open failed: EISDIR (Is a directory)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at libcore.io.IoBridge.open(IoBridge.java:468)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at java.io.FileInputStream.<init>(FileInputStream.java:76)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at com.alipay.euler.andfix.util.FileUtil.copyFile(FileUtil.java:51)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at com.alipay.euler.andfix.patch.PatchManager.addPatch(PatchManager.java:162)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at com.android.hq.andfixtest.MainApplication.onCreate(MainApplication.java:37)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at com.android.tools.fd.runtime.BootstrapApplication.onCreate(BootstrapApplication.java:370)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at android.app.Instrumentation.callApplicationOnCreate(Instrumentation.java:1020)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at android.app.ActivityThread.handleBindApplication(ActivityThread.java:4952)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at android.app.ActivityThread.-wrap1(ActivityThread.java)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1531)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at android.os.Handler.dispatchMessage(Handler.java:102)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at android.os.Looper.loop(Looper.java:148)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at android.app.ActivityThread.main(ActivityThread.java:5660)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at java.lang.reflect.Method.invoke(Native Method)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:775)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:665)
01-20 09:54:50.354 25113 25113 E MainApplication: Caused by: android.system.ErrnoException: open failed: EISDIR (Is a directory)
01-20 09:54:50.354 25113 25113 E MainApplication: 	at libcore.io.IoBridge.open(IoBridge.java:453)
01-20 09:54:50.354 25113 25113 E MainApplication: 	... 15 more
```
遇到这个错误是你要把out.apatch目录下面的patch文件copy到sdcard，而不是out.apatch整个目录。

