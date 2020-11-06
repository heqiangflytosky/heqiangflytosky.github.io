---
title: Cocos Creator -- JSB 引擎初始化流程
categories: cocos
comments: true
tags: [cocos]
description:  引擎初始化流程
date: 2020-2-25 10:00:00
---



## 概述

本文开始对 cocos 引擎的初始化过程做一些简单的介绍。

## 渲染初始化

我们在上一篇 [Cocos Creator -- HelloWorld 工程 ](http://www.heqiangfly.com/2020/02/13/cocos-creator-hello-world-project/) 文章中介绍 `Cocos2dxRenderer` 时看到在 `onSurfaceCreated` 中调用 `Cocos2dxRenderer.nativeInit` 来对做一些引擎的初始化工作。
现在我们具体看看它的实现。

```
// Cocos2dxRenderer.nativeInit
// JniImp.cpp
    JNIEXPORT void JNICALL JNI_RENDER(nativeInit)(JNIEnv*  env, jobject thiz, jint w, jint h, jstring jDefaultResourcePath)
    {
        // 设置全局变量，游戏的宽高
        g_width = w;
        g_height = h;
        // 初始化 cocos2d::Application 对象
        // 调用 jni/hellojavascript/main.cpp 中的 cocos_android_app_init 方法
        g_app = cocos_android_app_init(env, w, h);

        // 游戏开始
        g_isGameFinished = false;
        ccInvalidateStateCache();
        std::string defaultResourcePath = JniHelper::jstring2string(jDefaultResourcePath);
        LOGD("nativeInit: %d, %d, %s", w, h, defaultResourcePath.c_str());
        
        if (!defaultResourcePath.empty())
            FileUtils::getInstance()->setDefaultResourceRootPath(defaultResourcePath);

        se::ScriptEngine* se = se::ScriptEngine::getInstance();
        se->addRegisterCallback(setCanvasCallback);
        // 初始化 EventDispatcher
        EventDispatcher::init();
        // 调用 cocos2d-x-lite/cocos/platform/android/CCApplication-android.cpp
        // Application 的 start 方法。
        g_app->start();
        // 初始化完成
        g_isStarted = true;
    }

```

### 初始化 Application

在 `nativeInit` 方法中，调用 `cocos_android_app_init` 生成 cocos2d::Application 对象。

```
//jni/hellojavascript/main.cpp
//called when onSurfaceCreated()
Application *cocos_android_app_init(JNIEnv *env, int width, int height)
{
    LOGD("cocos_android_app_init");
    auto app = new AppDelegate(width, height);
    return app;
}
```

这个方法就是实例化了一个 AppDelegate 对象。这个类继承自 Application，重写了 applicationDidFinishLaunching 方法。

```
// runtime-src/Classes/AppDelegate.cpp
AppDelegate::AppDelegate(int width, int height) : Application("Cocos Game", width, height)
{
}

AppDelegate::~AppDelegate()
{
}

bool AppDelegate::applicationDidFinishLaunching()
{
    se::ScriptEngine *se = se::ScriptEngine::getInstance();
    
    jsb_set_xxtea_key("2d120e27-0679-44");
    jsb_init_file_operation_delegate();
    
#if defined(COCOS2D_DEBUG) && (COCOS2D_DEBUG > 0)
    // Enable debugger here
    jsb_enable_debugger("0.0.0.0", 6086, false);
#endif
    //设置 JS 层异常回调
    se->setExceptionCallback([](const char *location, const char *message, const char *stack) {
        // Send exception information to server like Tencent Bugly.
    });
    // 注册各个模块
    jsb_register_all_modules();
    // 启动 ScriptEngine
    se->start();
    
    se::AutoHandleScope hs;
    // 运行 apk 中的引擎的 jsb-builtin.js 文件
    jsb_run_script("jsb-adapter/jsb-builtin.js");
    // 运行 apk 中的游戏的 main.js 文件
    jsb_run_script("main.js");
    
    se->addAfterCleanupHook([]() {
        JSBClassType::destroy();
    });
    
    return true;
}

// This function will be called when the app is inactive. When comes a phone call,it's be invoked too
void AppDelegate::onPause()
{
    EventDispatcher::dispatchOnPauseEvent();
}

// this function will be called when the app is active again
void AppDelegate::onResume()
{
    EventDispatcher::dispatchOnResumeEvent();
}
```

`nativeInit` 方法中调用 `g_app->start()` ，可见 app 的初始化工作主要在 `applicationDidFinishLaunching` 中完成的。

```
// cocos2d-x-lite/cocos/platform/android/CCApplication-android.cpp
// Application
void Application::start()
{
    // 调用 上面的 AppDelegate 的 applicationDidFinishLaunching 方法
    if(!applicationDidFinishLaunching())
        return;
}
```

### 注册模块

注册模块主要是完成 JSB 的绑定，这些注册模块的实现都在 `cocos/scripting/js-bindings/manual/`（手动绑定） 和 `cocos/scripting/js-bindings/auto/`（自动绑定） 里面定义。定义了一些可以在JS端调用的类、方法和属性。

```
// cocos/scripting/js-bindings/manual/jsb_module_register.cpp

bool jsb_register_all_modules()
{
    se::ScriptEngine* se = se::ScriptEngine::getInstance();

    se->addBeforeInitHook([](){
        JSBClassType::init();
    });

    se->addBeforeCleanupHook([se](){
        se->garbageCollect();
        PoolManager::getInstance()->getCurrentPool()->clear();
        se->garbageCollect();
        PoolManager::getInstance()->getCurrentPool()->clear();
    });
    // 注册js全局变量，比如 jsb 和 __gl 等
    se->addRegisterCallback(jsb_register_global_variables);
    se->addRegisterCallback(JSB_register_opengl);
    se->addRegisterCallback(register_all_engine);
    se->addRegisterCallback(register_all_cocos2dx_manual);
    se->addRegisterCallback(register_platform_bindings);
    // 注册网络下载相关 api
    se->addRegisterCallback(register_all_network);
    se->addRegisterCallback(register_all_cocos2dx_network_manual);
    se->addRegisterCallback(register_all_xmlhttprequest);
    // extension depend on network
    se->addRegisterCallback(register_all_extension);

#if USE_GFX_RENDERER
    se->addRegisterCallback(register_all_gfx);
    se->addRegisterCallback(jsb_register_gfx_manual);
    // 初始化渲染相关接口，native 渲染相关
    se->addRegisterCallback(register_all_renderer);
    se->addRegisterCallback(jsb_register_renderer_manual);
#endif

#if (CC_TARGET_PLATFORM == CC_PLATFORM_IOS || CC_TARGET_PLATFORM == CC_PLATFORM_MAC)
    se->addRegisterCallback(register_javascript_objc_bridge);
#endif
// 如果是 Android 平台，注册 JavascriptJavaBridge
#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
    se->addRegisterCallback(register_javascript_java_bridge);
#endif

#if USE_AUDIO
    se->addRegisterCallback(register_all_audioengine);
#endif
    
#if USE_SOCKET
    // 注册websocket相关接口
    se->addRegisterCallback(register_all_websocket);
    // 注册网络请求相关接口
    se->addRegisterCallback(register_all_socketio);
#if USE_WEBSOCKET_SERVER
    se->addRegisterCallback(register_all_websocket_server);
#endif
#endif

#if USE_GFX_RENDERER && USE_MIDDLEWARE
    se->addRegisterCallback(register_all_cocos2dx_editor_support);

#if USE_SPINE
    se->addRegisterCallback(register_all_cocos2dx_spine);
    se->addRegisterCallback(register_all_spine_manual);
#endif

#if USE_DRAGONBONES
    se->addRegisterCallback(register_all_cocos2dx_dragonbones);
    se->addRegisterCallback(register_all_dragonbones_manual);
#endif

#if USE_PARTICLE
    se->addRegisterCallback(register_all_cocos2dx_particle);
#endif

#endif // USE_MIDDLEWARE

#if (CC_TARGET_PLATFORM == CC_PLATFORM_IOS || CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)

#if USE_VIDEO
    se->addRegisterCallback(register_all_video);
#endif

#if USE_WEB_VIEW
    se->addRegisterCallback(register_all_webview);
#endif

#endif // (CC_TARGET_PLATFORM == CC_PLATFORM_IOS || CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)

    se->addAfterCleanupHook([](){
        PoolManager::getInstance()->getCurrentPool()->clear();
        JSBClassType::destroy();
    });
    return true;
}
```

`jsb_register_global_variables` 回调方法会注册一些全局的变量和方法。

```
bool jsb_register_global_variables(se::Object* global)
{
    __threadPool = ThreadPool::newFixedThreadPool(3);

    global->defineFunction("require", _SE(require));
    global->defineFunction("requireModule", _SE(moduleRequire));
    // 为 JS 端生成一个全局的 jsb 对象
    getOrCreatePlainObject_r("jsb", global, &__jsbObj);

    auto glContextCls = se::Class::create("WebGLRenderingContext", global, nullptr, nullptr);
    glContextCls->install();

    SAFE_DEC_REF(__glObj);
    __glObj = se::Object::createObjectWithClass(glContextCls);
    global->setProperty("__gl", se::Value(__glObj));

    __jsbObj->defineFunction("garbageCollect", _SE(jsc_garbageCollect));
    __jsbObj->defineFunction("dumpNativePtrToSeObjectMap", _SE(jsc_dumpNativePtrToSeObjectMap));

    __jsbObj->defineFunction("loadImage", _SE(js_loadImage));
    __jsbObj->defineFunction("saveImageData", _SE(js_saveImageData));
    __jsbObj->defineFunction("setDebugViewText", _SE(js_setDebugViewText));
    __jsbObj->defineFunction("openDebugView", _SE(js_openDebugView));
    __jsbObj->defineFunction("disableBatchGLCommandsToNative", _SE(js_disableBatchGLCommandsToNative));
    __jsbObj->defineFunction("openURL", _SE(JSB_openURL));
    __jsbObj->defineFunction("copyTextToClipboard", _SE(JSB_copyTextToClipboard));

    __jsbObj->defineFunction("setPreferredFramesPerSecond", _SE(JSB_setPreferredFramesPerSecond));
    __jsbObj->defineFunction("showInputBox", _SE(JSB_showInputBox));
    __jsbObj->defineFunction("hideInputBox", _SE(JSB_hideInputBox));

    global->defineFunction("__getPlatform", _SE(JSBCore_platform));
    global->defineFunction("__getOS", _SE(JSBCore_os));
    global->defineFunction("__getOSVersion", _SE(JSB_getOSVersion));
    global->defineFunction("__getCurrentLanguage", _SE(JSBCore_getCurrentLanguage));
    global->defineFunction("__getCurrentLanguageCode", _SE(JSBCore_getCurrentLanguageCode));
    global->defineFunction("__getVersion", _SE(JSBCore_version));
    global->defineFunction("__restartVM", _SE(JSB_core_restartVM));
    global->defineFunction("__cleanScript", _SE(JSB_cleanScript));
    global->defineFunction("__isObjectValid", _SE(JSB_isObjectValid));
    global->defineFunction("close", _SE(JSB_closeWindow));

    se::HandleObject performanceObj(se::Object::createPlainObject());
    performanceObj->defineFunction("now", _SE(js_performance_now));
    global->setProperty("performance", se::Value(performanceObj));

    se::ScriptEngine::getInstance()->clearException();

    se::ScriptEngine::getInstance()->addBeforeCleanupHook([](){
        delete __threadPool;
        __threadPool = nullptr;

        PoolManager::getInstance()->getCurrentPool()->clear();
    });

    se::ScriptEngine::getInstance()->addAfterCleanupHook([](){

        PoolManager::getInstance()->getCurrentPool()->clear();

        __moduleCache.clear();

        SAFE_DEC_REF(__jsbObj);
        SAFE_DEC_REF(__glObj);
    });

    return true;
}
```

```
bool register_all_renderer(se::Object* obj)
{
    // Get the ns
    se::Value nsVal;
    if (!obj->getProperty("renderer", &nsVal))
    {
        se::HandleObject jsobj(se::Object::createPlainObject());
        nsVal.setObject(jsobj);
        obj->setProperty("renderer", nsVal);
    }
    se::Object* ns = nsVal.toObject();

    js_register_renderer_ProgramLib(ns);
    js_register_renderer_EffectBase(ns);
    js_register_renderer_Camera(ns);
    js_register_renderer_AssemblerBase(ns);
    js_register_renderer_Assembler(ns);
    js_register_renderer_AssemblerSprite(ns);
    js_register_renderer_SimpleSprite3D(ns);
    js_register_renderer_MemPool(ns);
    js_register_renderer_NodeProxy(ns);
    js_register_renderer_SimpleSprite2D(ns);
    js_register_renderer_SlicedSprite3D(ns);
    js_register_renderer_Effect(ns);
    js_register_renderer_CustomAssembler(ns);
    js_register_renderer_MeshAssembler(ns);
    js_register_renderer_MaskAssembler(ns);
    js_register_renderer_Light(ns);
    js_register_renderer_NodeMemPool(ns);
    js_register_renderer_TiledMapAssembler(ns);
    js_register_renderer_Particle3DAssembler(ns);
    js_register_renderer_BaseRenderer(ns);
    js_register_renderer_ForwardRenderer(ns);
    js_register_renderer_View(ns);
    // native RenderFlow
    js_register_renderer_RenderFlow(ns);
    js_register_renderer_SlicedSprite2D(ns);
    js_register_renderer_EffectVariant(ns);
    js_register_renderer_Scene(ns);
    js_register_renderer_RenderDataList(ns);
    return true;
}
```

```
bool register_all_cocos2dx_manual(se::Object* obj)
{
    register_plist_parser(obj);
    register_sys_localStorage(obj);
    register_device(obj);
    // 注册 Canvas 相关接口和属性
    register_canvas_context2d(obj);
    return true;
}
```

## 启动 ScriptEngine

```
// cocos/scripting/js-bindings/jswrapper/v8/ScriptEngine.cpp
    bool ScriptEngine::start()
    {
        // 初始化
        if (!init())
            return false;

        se::AutoHandleScope hs;

        // debugger
        if (isDebuggerEnabled())
        {
#if SE_ENABLE_INSPECTOR
            // V8 inspector stuff, most code are taken from NodeJS.
            _isolateData = node::CreateIsolateData(_isolate, uv_default_loop());
            _env = node::CreateEnvironment(_isolateData, _context.Get(_isolate), 0, nullptr, 0, nullptr);

            node::DebugOptions options;
            options.set_wait_for_connect(_isWaitForConnect);// the program will be hung up until debug attach if _isWaitForConnect = true
            options.set_inspector_enabled(true);
            options.set_port((int)_debuggerServerPort);
            options.set_host_name(_debuggerServerAddr.c_str());
            bool ok = _env->inspector_agent()->Start(_platform, "", options);
            assert(ok);
#endif
        }
        //
        bool ok = false;
        _startTime = std::chrono::steady_clock::now();

        // 执行前面注册的各个模块的回调方法，全局对象 _globalObj 作为参数
        for (auto cb : _registerCallbackArray)
        {
            ok = cb(_globalObj);
            assert(ok);
            if (!ok)
                break;
        }

        // ScriptEngine 启动之后，_registerCallbackArray 就不在需要了，清空回调方法
        _registerCallbackArray.clear();

        return ok;
    }
```

## 渲染

初始化工作完成后就开始进行游戏渲染工作了，这里只简单介绍一下基本的渲染机制。
还是从 `Cocos2dxRenderer.onDrawFrame` 开始，前面也介绍过，`onDrawFrame` 每一帧都会调用一次来进行帧面面的渲染，渲染的方法主要是调用本地方法 `Cocos2dxRenderer.nativeRender()` 。

```
// JniImp.cpp

	JNIEXPORT void JNICALL JNI_RENDER(nativeRender)(JNIEnv* env)
	{
	    // 游戏结束时
        if (g_isGameFinished)
        {
            // with Application destructor called, native resource will be released
            delete g_app;
            g_app = nullptr;

            JniHelper::callStaticVoidMethod(JCLS_HELPER, "endApplication");
            return;
        }


        if (!g_isStarted)
        {
            auto scheduler = Application::getInstance()->getScheduler();
            scheduler->removeAllFunctionsToBePerformedInCocosThread();
            scheduler->unscheduleAll();

            se::ScriptEngine::getInstance()->cleanup();
            cocos2d::PoolManager::getInstance()->getCurrentPool()->clear();

            //REFINE: Wait HttpClient, WebSocket, Audio thread to exit

            ccInvalidateStateCache();
          
            se::ScriptEngine* se = se::ScriptEngine::getInstance();
            se->addRegisterCallback(setCanvasCallback);

            EventDispatcher::init();

            if(!g_app->applicationDidFinishLaunching())
            {
                g_isGameFinished = true;
                return;
            }

            g_isStarted = true;
        }

        static std::chrono::steady_clock::time_point prevTime;
        static std::chrono::steady_clock::time_point now;
        static float dt = 0.f;
        static float dtSum = 0.f;
        static uint32_t jsbInvocationTotalCount = 0;
        static uint32_t jsbInvocationTotalFrames = 0;
        bool downsampleEnabled = g_app->isDownsampleEnabled();
        
        if (downsampleEnabled)
            g_app->getRenderTexture()->prepare();

        g_app->getScheduler()->update(dt);
        EventDispatcher::dispatchTickEvent(dt);
       
        if (downsampleEnabled)
            g_app->getRenderTexture()->draw();

        PoolManager::getInstance()->getCurrentPool()->clear();

        now = std::chrono::steady_clock::now();
        dt = std::chrono::duration_cast<std::chrono::microseconds>(now - prevTime).count() / 1000000.f;

        prevTime = std::chrono::steady_clock::now();

        if (__isOpenDebugView)
        {
            dtSum += dt;
            ++jsbInvocationTotalFrames;
            jsbInvocationTotalCount += __jsbInvocationCount;

            if (dtSum > 1.0f)
            {
                dtSum = 0.0f;
                setJSBInvocationCountJNI(jsbInvocationTotalCount / jsbInvocationTotalFrames);
                jsbInvocationTotalCount = 0;
                jsbInvocationTotalFrames = 0;
            }
        }
        __jsbInvocationCount = 0;
    }
```
