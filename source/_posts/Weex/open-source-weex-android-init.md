---
title: Weex 源码研究 -- 初始化过程
categories: Weex
comments: true
tags: [Weex]
description: 介绍 Weex 的初始化过程
date: 2018-3-15 10:00:00
---


## 概述

从本篇博客开始，会陆续进行一些 Weex 源码分析的文章，源码的 Weex SDK Version 是基于0.17.0，JS Framework 是基于 0.23.9， 平台是基于 Android 来进行分析的。

## 初始化过程

### WXSDKEngine 初始化

```
public class WXApplication extends Application {

  @Override
  public void onCreate() {
    super.onCreate();
    InitConfig config=new InitConfig.Builder().setImgAdapter(new ImageAdapter()).build();
    WXSDKEngine.initialize(this,config);
  }
}
```

调用了 `WXSDKEngine.initialize()` 方法，这里有 `InitConfig` 作为参数来初始化。
先来看一下 `InitConfig` 类。
`InitConfig` 使用 Builder 模式来构造一个 Weex 初始化配置的一个类。不熟悉 Builder 模式的同学可以先了解一下这个设计模式。
`InitConfig` 主要是配置一些 Adapter：

 - IWXHttpAdapter
 - IWXImgLoaderAdapter
 - IDrawableLoader
 - IWXUserTrackAdapter
 - IWXStorageAdapter
 - IWXSoLoaderAdapter
 - URIAdapter
 - IWXJSExceptionAdapter
 - framework
 - IWebSocketAdapterFactory

配置了这些 Adapter，Weex 才能更好地实现一些功能。比如前面示例代码中，如果想显示一个图片的话就必须配置 `IWXImgLoaderAdapter`，自定义完成图片的加载工作。

```
├── WXSDKEngine.initialize()
    └── WXSDKEngine.doInitInternal()
        ├── WXBridgeManager.getInstance().post()
            ├── WXSDKManager.onSDKEngineInitialize()
            ├── WXSDKManager.setInitConfig()
            ├── WXSoInstallMgrSdk.init()
            ├── WXSoInstallMgrSdk.initSo()
            └── WXSDKManager.initScriptsFramework()
                └── WXBridgeManager.initScriptsFramework()
                    └── WXBridgeManager.invokeInitFramework()
                        └── WXBridgeManager.initFramework()
                            ├── WXBridge.initFrameworkEnv()
                                ├── initFrameworkMultiProcess()
                                └── initFramework()
                            └── WXBridgeManager.registerDomModule()
                                └── WXBridgeManager.registerModules()
                                    └── WXBridgeManager.invokeRegisterModules()
                                        └── WXBridge.execJS()
        └── WXSDKEngine.register()
            ├── WXSDKEngine.registerComponent()
            ├── WXSDKEngine.registerModule()
            └── WXSDKEngine.registerDomObject()
```

``` java
  private static void doInitInternal(final Application application,final InitConfig config){
    WXEnvironment.sApplication = application;
	if(application == null){
	  WXLogUtils.e(TAG, " doInitInternal application is null");
	  WXExceptionUtils.commitCriticalExceptionRT(null,
			  WXErrorCode.WX_KEY_EXCEPTION_SDK_INIT.getErrorCode(),
			  "doInitInternal",
			  WXErrorCode.WX_KEY_EXCEPTION_SDK_INIT.getErrorMsg() + "WXEnvironment sApplication is null",
			  null);
	}
    WXEnvironment.JsFrameworkInit = false;

    WXBridgeManager.getInstance().post(new Runnable() {
      @Override
      public void run() {
        long start = System.currentTimeMillis();
        WXSDKManager sm = WXSDKManager.getInstance();
        sm.onSDKEngineInitialize();
        if(config != null ) {
          //把初始化配置设置给 WXSDKManager
          sm.setInitConfig(config);
        }
        // 加载 libweexjsc.so V8 引擎库。
        WXSoInstallMgrSdk.init(application,
                              sm.getIWXSoLoaderAdapter(),
                              sm.getWXStatisticsListener());
        boolean isSoInitSuccess = WXSoInstallMgrSdk.initSo(V8_SO_NAME, 1, config!=null?config.getUtAdapter():null);
        if (!isSoInitSuccess) {
		  WXExceptionUtils.commitCriticalExceptionRT(null,
				  WXErrorCode.WX_KEY_EXCEPTION_SDK_INIT.getErrorCode(),
				  "doInitInternal",
				  WXErrorCode.WX_KEY_EXCEPTION_SDK_INIT.getErrorMsg() + "isSoInit false",
				  null);

          return;
        }
        // 初始化 js framework
        sm.initScriptsFramework(config!=null?config.getFramework():null);

        WXEnvironment.sSDKInitExecuteTime = System.currentTimeMillis() - start;
        WXLogUtils.renderPerformanceLog("SDKInitExecuteTime", WXEnvironment.sSDKInitExecuteTime);
      }
    });
    // 注册 Component 、 Module 以及 DomObject
    register();
  }
```

这里大部分的初始化工作是在 `WXBridgeManager.getInstance().post()` 方法中进行的，这个方法会把这些操作放到 WeexJSBridgeThread 线程中去执行。
有关 Weex 的线程模型后面文章再介绍。

#### 初始化 Js Framework

```java
  private void initFramework(String framework) {
    if (!isJSFrameworkInit()) {
      if (TextUtils.isEmpty(framework)) {
        // 这里如果在 InitConfig 没有配置 framework js 的话就默认加载 sdk 里面的 main.js
        // if (WXEnvironment.isApkDebugable()) {
        WXLogUtils.d("weex JS framework from assets");
        // }
        framework = WXFileUtils.loadAsset("main.js", WXEnvironment.getApplication());
      }
      // 如果加载 js framework 不成功，报错返回。
      if (TextUtils.isEmpty(framework)) {
        setJSFrameworkInit(false);
		WXExceptionUtils.commitCriticalExceptionRT(null, WXErrorCode.WX_ERR_JS_FRAMEWORK.getErrorCode(),
				"initFramework", "framework is empty!! ", null);
        return;
      }
      try {
        if (WXSDKManager.getInstance().getWXStatisticsListener() != null) {
          WXSDKManager.getInstance().getWXStatisticsListener().onJsFrameworkStart();
        }

        long start = System.currentTimeMillis();
        String crashFile = "";
        try {
          crashFile = WXEnvironment.getApplication().getApplicationContext().getCacheDir().getPath();
        } catch (Exception e) {
          e.printStackTrace();
        }
        boolean pieSupport = true;
        try {
          if (Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN) {
            pieSupport = false;
          }
        } catch (Exception e) {
          e.printStackTrace();
        }
        WXLogUtils.d("[WXBridgeManager] initFrameworkEnv crashFile:" + crashFile + " pieSupport:" + pieSupport);
        // 调用 WXBridge.initFrameworkEnv() 方法调用 native 方法初始化 js framework
        if (mWXBridge.initFrameworkEnv(framework, assembleDefaultOptions(), crashFile, pieSupport) == INIT_FRAMEWORK_OK) {
          WXEnvironment.sJSLibInitTime = System.currentTimeMillis() - start;
          WXLogUtils.renderPerformanceLog("initFramework", WXEnvironment.sJSLibInitTime);
          WXEnvironment.sSDKInitTime = System.currentTimeMillis() - WXEnvironment.sSDKInitStart;
          WXLogUtils.renderPerformanceLog("SDKInitTime", WXEnvironment.sSDKInitTime);
          setJSFrameworkInit(true);

          if (WXSDKManager.getInstance().getWXStatisticsListener() != null) {
            WXSDKManager.getInstance().getWXStatisticsListener().onJsFrameworkReady();
          }

          execRegisterFailTask();
          WXEnvironment.JsFrameworkInit = true;
          // 注册 DomModule，后面会统一介绍注册 Module 的步骤
          registerDomModule();
          String reinitInfo = "";
          if (reInitCount > 1) {
            reinitInfo = "reinit Framework:";
          }
        } else {
          ......
        }
      } catch (Throwable e) {
        ......
      }
    }
  }
```













