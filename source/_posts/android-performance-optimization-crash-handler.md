---
title: Android 性能优化实战篇 -- 使用 CrashHandler 处理崩溃问题
categories: Android性能优化
comments: true
tags: [稳定性]
description: 介绍使用 CrashHandler 处理崩溃问题
date: 2017-2-21 10:00:00
---

[GitHub源码地址](https://github.com/heqiangflytosky/AndroidUtils)

## 概述

Android应用开发中不可避免地会发生崩溃，特别是在用户使用过程中，一些特定场景的偶然概率的crash会通常让开发者抓狂。幸运的是Android提供了处理这类问题的方法，可以设置一个统一的接口来处理我们代码中没有处理的异常。
通过这个接口，我们可以做一些事情，比如：

 - 忽略一些不影响应用运行的异常
 - 可以记录一些崩溃日志并上传服务器，以便开发者迅速定位问题原因。 
 - 可以添加一些友好的交互提示

下面分析一下它的一些应用场景。

## CrashHandler类

实现这个功能我们需要实现 `Thread.UncaughtExceptionHandler` 这个接口。里面只有一个方法 `uncaughtException`。当我们注册一个 `UncaughtExceptionHandler` 之后，当我们的程序 crash 时就会回调 `uncaughtException` 方法，我们就可以再这个方法中添加我们的处理。

```
public class CrashHandler implements Thread.UncaughtExceptionHandler {
    private static final Thread.UncaughtExceptionHandler sDefaultHandler = Thread.getDefaultUncaughtExceptionHandler();

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        // TODO
        // 添加对异常的处理
        
        // 抛给系统去处理
        if (sDefaultHandler != null) {
            sDefaultHandler.uncaughtException(t, e);
        }
    }
    
    public static void registerExceptionHandler() {
        CrashHandler handler = new CrashHandler();
        Thread.setDefaultUncaughtExceptionHandler(handler);
    }
    
}
```

## 解决 FinalizerWatchdogDaemon 线程的 TimeoutException 问题

关于这个问题的介绍，情况这篇文章[滴滴出行安卓端 finalize time out 的解决方案 ](https://mp.weixin.qq.com/s/uFcFYO2GtWWiblotem2bGg)。
代码：

```
    @Override
    public void uncaughtException(Thread t, Throwable e) {

        if (t != null && "FinalizerWatchdogDaemon".equals(t.getName()) && e instanceof TimeoutException) {
            Log.e("CrashHandler", "CrashHandler", e);
            return;
        }

        if (sDefaultHandler != null) {
            sDefaultHandler.uncaughtException(t, e);
        }
    }
```

处理方案就是忽略掉这个异常，减少 App 的 Crash。

## 获取应用的 crash 信息

当应用发生 Crash 时，在` uncaughtException` 方法中就可以获取到异常信息，我们可以选择把异常信息存储到SD卡中，然后在合适的时机上传到服务器。或者我们还可以在crash发生时，弹出一个对话框告诉用户程序crash了，然后再退出，这样会比闪退温和一些。
下面我们实现了一个捕获OOM的UncaughtExceptionHandler 的类，发生OOM时可以抓取一些日志，然后可以通过MAT分析生成的hprof文件来定位问题。下面来看一下这个类的用法：

```
public class CrashHandler implements Thread.UncaughtExceptionHandler {
    final String TAG = "CrashHandler";
    private Thread.UncaughtExceptionHandler mDefaultUncaughtExceptionHandler = Thread.getDefaultUncaughtExceptionHandler();
    private String mPackageName;
    private int mPid;
    private StringBuilder mFilename = new StringBuilder(120);
    private final String OOM_TARGET_FOLDER = "/crash_log/";
    private static final long ONE_DAY_TIME_IN_MILLISECONDS = 24*60*60*1000;
    private boolean mDumpingHprof;

    CrashHandler(String pkgName, int pid) {
        this.mPackageName = pkgName;
        this.mPid = pid;
        this.mDumpingHprof = false;
    }


    public static void registerExceptionHandler(Context context) {
        CrashHandler handler = new CrashHandler(context.getPackageName(), android.os.Process.myPid());
        Thread.setDefaultUncaughtExceptionHandler(handler);
    }

    public void uncaughtException(Thread thread, Throwable throwable) {
        // 处理 OOM 日志问题
        if(throwable instanceof OutOfMemoryError && !mDumpingHprof) {
            Log.w(TAG, "CrashHandler capture a oom exception !!!");
            if(!dumpHprofData()) {
                Log.e(TAG, "Aborting ...");
            }else{
                //uploadExceptionToServer(); // TODO
            }
        }

        if(mDefaultUncaughtExceptionHandler != null){
            mDefaultUncaughtExceptionHandler.uncaughtException(thread, throwable);
        } else {
            Process.killProcess(Process.myPid());
            //System.exit(0);
        }
    }

    //获取目录大小，单位 M
    private double getDirSize(File file) {
        if(!file.exists()) {
            Log.w(TAG, file.toString() + " may not exists !");
            return 0.0D;
        } else if(!file.isDirectory()) {
            double size = (double)file.length() / 1024.0D / 1024.0D;
            return size;
        } else {
            File[] files = file.listFiles();
            double size = 0.0D;
            int length = files.length;

            for(int i = 0; i < length; ++i) {
                File f = files[i];
                size += this.getDirSize(f);
            }

            return size;
        }
    }

    //当目录大于1G时且创建时间大于1天时删除旧文件
    private void deleteOldFiles(File file) {
        if(getDirSize(file) >= 1024.0D) {
            Log.w(TAG, "begin to delete old files !");
            long currentTimeMillis = System.currentTimeMillis();
            File[] listFiles = file.listFiles();

            for(int i = 0; i < listFiles.length; ++i) {
                if(listFiles[i].isFile()) {
                    if(currentTimeMillis - listFiles[i].lastModified() > ONE_DAY_TIME_IN_MILLISECONDS) {
                        listFiles[i].delete();
                    }
                } else if(listFiles[i].isDirectory()) {
                    deleteOldFiles(listFiles[i]);
                }
            }

        }
    }

    private boolean getDumpDestinationOfHeap() {
        mFilename.delete(0, mFilename.length());
        mFilename.append(Environment.getExternalStorageDirectory());
        mFilename.append(OOM_TARGET_FOLDER);

        File target = new File(mFilename.toString());
        if (!target.exists()) {
            if (!target.mkdirs()) {
                Log.e(TAG, "Creating target hprof directory: \"" + mFilename.toString() + "\" was failed!");
                return false;
            }
        }
        deleteOldFiles(target);
        mFilename.append(mPackageName);
        mFilename.append("_PID:" + mPid);
        String time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date(System.currentTimeMillis()));
        mFilename.append("_TIME:"+time);
        mFilename.append(".hprof");
        target = new File(mFilename.toString().trim());
        try {
            target.createNewFile();
        } catch (IOException e) {
            Log.e(TAG, "Creating target hprof file: \"" + mFilename.toString() + "\" was failed! Reason:" + e);

            return false;
        }

        return true;
    }

    private boolean dumpHprofData() {
        if (!getDumpDestinationOfHeap()) {
            return false;
        }
        Log.w(TAG, "Begin to dump hprof to " + mFilename.toString());
        long beginDumpTime = SystemClock.uptimeMillis();
        try {
            mDumpingHprof = true;
            Debug.dumpHprofData(mFilename.toString());
        } catch (IOException e) {
            Log.e(TAG, "Dump hprof to " + mFilename.toString() + " failed ! " + e);
            return false;
        }
        long endDumpTime = (SystemClock.uptimeMillis() - beginDumpTime) / 1000;
        Log.w(TAG, "Dump succeed!, Took " + endDumpTime + "s");
        return true;
    }

}

```

模拟测试场景：

```
    private void createOOM(){
        OomCrashHandler.registerExceptionHandler(this);
        ArrayList<Bitmap> arrayList = new ArrayList<>();
        for(int i = 0; i<100000; i++){
            arrayList.add(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher));
        }
    }
```