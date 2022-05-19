---
title: Cocos Creator -- HelloWorld 工程
categories: cocos
comments: true
tags: [cocos]
description: Cocos Creator HelloWorld 工程简介
date: 2020-2-13 10:00:00
---

## 概述

前面的一篇文章介绍了如何使用 cocos creator 创建 HelloWorld 工程，本文就来简单分析一下这个工程，以便对 cocos 引擎的工作原理有个初步的了解。

## 项目结构

https://docs.cocos.com/creator/2.2/manual/zh/getting-started/project-structure.html

## Android 工程文件

Android 工程目录在 `HelloWorld/build/jsb-link/frameworks/runtime-src/proj.android-studio` 目录下面，我们先从 settings.gralde 中看看它包含哪些模块：

```
include ':libcocos2dx',':game', ':instantapp'
project(':libcocos2dx').projectDir = new File('/Users/heqiang/workspace/android-studio-workspace/cocos2d/cocos2d-x-lite/cocos/platform/android/libcocos2dx')
include ':hello_world'
project(':hello_world').projectDir = new File(settingsDir, 'app')
```

game 和 instantapp 模块可以先不用去管，下面我们主要分析的就是 hello_world 和 libcocos2dx 模块了，hello_world 是我们的 app 模块，libcocos2dx 是引擎中关于 Android 平台的相关类。
另外，和 proj.android-studio 同级有个 Classes 目录，里面的 AppDelegate.cpp、AppDelegate.h、jsb_module_register.cpp 文件也是我们工程需要的文件，后面会介绍。

## APK 目录

```
├── AndroidManifest.xml
├── assets
    ├── jsb-adapter
    │   ├── jsb-builtin.js
    │   └── jsb-engine.js
    ├── main.js
    ├── project.json
    ├── res
    └── src
├── classes.dex
├── lib
├── META-INF
├── org.cocos2dx.
├── res
└── resources.arsc
```

assets 目录中是游戏相关的文件，这些文件是从来的呢？我们先来看一下 Android 工程中的 build.gradle 文件中相关的一段代码：

```
android.applicationVariants.all { variant ->
    // delete previous files first
    delete "${buildDir}/intermediates/merged_assets/${variant.dirName}"

    variant.mergeAssets.doLast {
        def sourceDir = "${buildDir}/../../../../.."

        copy {
            from "${sourceDir}/res"
            into "${outputDir}/res"
        }

        copy {
            from "${sourceDir}/subpackages"
            into "${outputDir}/subpackages"
        }

        copy {
            from "${sourceDir}/src"
            into "${outputDir}/src"
        }

        copy {
            from "${sourceDir}/jsb-adapter"
            into "${outputDir}/jsb-adapter"
        }

        copy {
            from "${sourceDir}/main.js"
            from "${sourceDir}/project.json"
            into outputDir
        }
    }
}
```

编译时，从cocos游戏工程的 build/jsb-link 目录中把相关的文件打包到 assets 目录。
这个 outputDir 目录就是`build/jsb-link/frameworks/runtime-src/proj.android-studio/app/build/intermediates/merged_assets/debug/mergeDebugAssets/`
至于这些文件何时加载，后面将会详细的介绍。

## jni 部分

来看一下工程根目录下面的 jni 目录，里面的文件列表如下：

```
├── CocosAndroid.mk
├── CocosApplication.mk
├── hellojavascript
    └── main.cpp
```


```
//CocosAndroid.mk

LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := cocos2djs_shared
LOCAL_MODULE_FILENAME := libcocos2djs
ifeq ($(USE_ARM_MODE),1)
LOCAL_ARM_MODE := arm
endif
LOCAL_SRC_FILES := hellojavascript/main.cpp \
				   ../../Classes/AppDelegate.cpp \
				   ../../Classes/jsb_module_register.cpp \
LOCAL_C_INCLUDES := $(LOCAL_PATH)/../../Classes
LOCAL_STATIC_LIBRARIES := cocos2dx_static
include $(BUILD_SHARED_LIBRARY)
$(call import-module, cocos)
```

把上面的 `hellojavascript/main.cpp` 文件和 `runtime-src/Classes` 目录下面的 AppDelegate.cpp 和 jsb_module_register.cpp 三个文件编译生成 libcocos2djs.so。
AppDelegate.cpp 中定义了 AppDelegate 类，该类继承自 cocos/platfrom/android/CCApplication-android.cpp 中的 Application 类。
AppDelegate 类是游戏程序的入口，主要用于游戏程序的逻辑初始化，并创建运行程序的入口界面（即第一个游戏界面场景）。这部分的源码分析后面再详细介绍。
AppDelegate 类在 main.cpp 中的 cocos_android_app_init 方法中初始化：

```
// main.cpp
#include "AppDelegate.h"
#include "cocos2d.h"
#include "platform/android/jni/JniHelper.h"
#include <jni.h>
#include <android/log.h>

#define LOG_TAG "main"
#define LOGD(...) __android_log_print(ANDROID_LOG_DEBUG, LOG_TAG, __VA_ARGS__)
using namespace cocos2d;

//called when JNI_OnLoad()
void cocos_jni_env_init(JNIEnv *env)
{
    LOGD("cocos_jni_env_init");
}

//called when onSurfaceCreated()
Application *cocos_android_app_init(JNIEnv *env, int width, int height)
{
    LOGD("cocos_android_app_init");
    auto app = new AppDelegate(width, height);
    return app;
}

extern "C"
{
    /*JNI_CALL_FUNCTION*/
}
```

那么 cocos_android_app_init 方法在哪里被调用呢？在cocos引擎的 cocos/platfrom/android/jni/JniImp.cpp 中的 `nativeInit` 方法中被初始化。这部分等后面的渲染流程中详细介绍。

## Java 部分

工程中 AppActivity 继承自 Cocos2dxActivity，除了在 `onCreateView` 中创建了 Cocos2dxGLSurfaceView 实例，其他并没有做什么工作，那么，我们就来看看 Cocos2dxActivity 类。
Cocos2dxActivity 主要就是创建了一个 Cocos2dxGLSurfaceView，并把它加入到 contentView 中去，由 Cocos2dxGLSurfaceView 对游戏进行渲染。
下面先对 onCreate 方法进行一些解释：

```
    @Override
    protected void onCreate(final Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        Utils.setActivity(this);

        // Workaround in https://stackoverflow.com/questions/16283079/re-launch-of-activity-on-home-button-but-only-the-first-time/16447508
        if (!isTaskRoot()) {
            // Android launched another instance of the root activity into an existing task
            //  so just quietly finish and go away, dropping the user back into the activity
            //  at the top of the stack (ie: the last state of this task)
            finish();
            Log.w(TAG, "[Workaround] Ignore the activity started from icon!");
            return;
        }
        // 隐藏一些底栏的虚拟按钮，全屏显示
        Utils.hideVirtualButton();
        // 监听电量
        Cocos2dxHelper.registerBatteryLevelReceiver(this);
        // 把工程中libs下面的so文件load进来，定义在AndroidManifest, meta-data标签下
        onLoadNativeLibraries();

        sContext = this;
        this.mHandler = new Cocos2dxHandler(this);
        
        Cocos2dxHelper.init(this);
        CanvasRenderingContext2DImpl.init(this);
        // 获取OpenGLEs的相关属性
        this.mGLContextAttrs = getGLContextAttrs();
        // 初始化方法，添加布局，下面介绍
        this.init();

        if (mVideoHelper == null) {
            mVideoHelper = new Cocos2dxVideoHelper(this, mFrameLayout);
        }

        if(mWebViewHelper == null){
            mWebViewHelper = new Cocos2dxWebViewHelper(mFrameLayout);
        }

        Window window = this.getWindow();
        window.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE);
        this.setVolumeControlStream(AudioManager.STREAM_MUSIC);
    }
```

```
    public void init() {
        ViewGroup.LayoutParams frameLayoutParams =
                new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                                           ViewGroup.LayoutParams.MATCH_PARENT);
        mFrameLayout = new FrameLayout(this);
        mFrameLayout.setLayoutParams(frameLayoutParams);

        Cocos2dxRenderer renderer = this.addSurfaceView();
        this.addDebugInfo(renderer);

        mEditBox = new Cocos2dxEditBox(this, mFrameLayout);

        // 将 mFrameLayout 设置为 content view
        setContentView(mFrameLayout);
    }
    
    private Cocos2dxRenderer addSurfaceView() {
        // 创建 Cocos2dxGLSurfaceView 实例
        this.mGLSurfaceView = this.onCreateView();
        this.mGLSurfaceView.setPreserveEGLContextOnPause(true);
        mGLSurfaceView.setBackgroundColor(Color.TRANSPARENT);
        // Switch to supported OpenGL (ARGB888) mode on emulator
        if (isAndroidEmulator())
            this.mGLSurfaceView.setEGLConfigChooser(8, 8, 8, 8, 16, 0);
        // 设置渲染器 Cocos2dxRenderer
        Cocos2dxRenderer renderer = new Cocos2dxRenderer();
        this.mGLSurfaceView.setCocos2dxRenderer(renderer);
        // 将 glSurfaceView 添加到 布局上
        mFrameLayout.addView(this.mGLSurfaceView);

        return renderer;
    }
```

### Cocos2dxRenderer

下面来看看渲染器 Cocos2dxRenderer 类：

```
    @Override
    public void onSurfaceCreated(final GL10 GL10, final EGLConfig EGLConfig) {
        mNativeInitCompleted = false;
        Cocos2dxRenderer.nativeInit(this.mScreenWidth, this.mScreenHeight, mDefaultResourcePath);
        mOldNanoTime = System.nanoTime();
        this.mLastTickInNanoSeconds = System.nanoTime();
        mNativeInitCompleted = true;
        if (mGameEngineInitializedListener != null) {
            Cocos2dxHelper.getActivity().runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    mGameEngineInitializedListener.onGameEngineInitialized();
                }
            });
        }
    }
```

onSurfaceCreated 方法里面主要调用了 `Cocos2dxRenderer.nativeInit` 本地方法对做一些引擎的初始化工作。

```
    @Override
    public void onDrawFrame(final GL10 gl) {
        if (mNeedToPause)
            return;

        if (mNeedShowFPS) {
            /////////////////////////////////////////////////////////////////////
            //IDEA: show FPS in Android Text control rather than outputing log.
            ++mFrameCount;
            long nowFpsTime = System.nanoTime();
            long fpsTimeInterval = nowFpsTime - mOldNanoTime;
            if (fpsTimeInterval > 1000000000L) {
                double frameRate = 1000000000.0 * mFrameCount / fpsTimeInterval;
                Cocos2dxHelper.OnGameInfoUpdatedListener listener = Cocos2dxHelper.getOnGameInfoUpdatedListener();
                if (listener != null) {
                    listener.onFPSUpdated((float) frameRate);
                }
                mFrameCount = 0;
                mOldNanoTime = System.nanoTime();
            }
            /////////////////////////////////////////////////////////////////////
        }
        /*
         * No need to use algorithm in default(60 FPS) situation,
         * since onDrawFrame() was called by system 60 times per second by default.
         */
        if (sAnimationInterval <= INTERVAL_60_FPS) {
            Cocos2dxRenderer.nativeRender();
        } else {
            final long now = System.nanoTime();
            final long interval = now - this.mLastTickInNanoSeconds;

            if (interval < Cocos2dxRenderer.sAnimationInterval) {
                try {
                    Thread.sleep((Cocos2dxRenderer.sAnimationInterval - interval) / Cocos2dxRenderer.NANOSECONDSPERMICROSECOND);
                } catch (final Exception e) {
                }
            }
            /*
             * Render time MUST be counted in, or the FPS will slower than appointed.
            */
            this.mLastTickInNanoSeconds = System.nanoTime();
            Cocos2dxRenderer.nativeRender();
        }
    }
```

onDrawFrame 每一帧都会调用一次来进行帧面面的渲染，渲染的方法主要是调用本地方法 `Cocos2dxRenderer.nativeRender()` 。
我们从 Cocos2dxRenderer 类中可以看到有很多的native 方法，这些方法会调用的c++实现的引擎中去，是我们研究cocos引擎游戏渲染的关键，后面我们会详细介绍。

## 总结

本文我们主要对 HelloWorld 工程进行了概要的介绍。接下来将会深入到 cocos 引擎，介绍一些它的实现原理。