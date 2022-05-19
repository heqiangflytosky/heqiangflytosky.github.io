---
title: Cocos Creator --  JavaScript 和 Java 互相调用
categories: cocos
comments: true
tags: [cocos]
description:  JavaScript 和 Java 互相调用
date: 2020-2-20 10:00:00
---

## 调用方法

关于 JavaScript 和 Java 如何互相调用，请参考官方文档 [如何在 Android 平台上使用 JavaScript 直接调用 Java 方法](https://docs.cocos.com/creator/2.2/manual/zh/advanced-topics/java-reflection.html)

## JavaScript 调用 Java 原理分析

首先看一下 `jsb.reflection.callStaticMethod` 方法。
这个方法的定义在 jsb-adapter 工程的 engine/jsb-reflection.js 文件中。

```
// JS to Native bridges
if(window.JavascriptJavaBridge && cc.sys.os == cc.sys.OS_ANDROID){
    jsb.reflection = new JavascriptJavaBridge();
    cc.sys.capabilities["keyboard"] = true;
}
else if(window.JavaScriptObjCBridge && (cc.sys.os == cc.sys.OS_IOS || cc.sys.os == cc.sys.OS_OSX)){
    jsb.reflection = new JavaScriptObjCBridge();
}
```

```
// cocos/scripting/js-bindings/manual/JavaScriptJavaBridge.cpp
bool register_javascript_java_bridge(se::Object* obj)
{
    se::Class* cls = se::Class::create("JavascriptJavaBridge", obj, nullptr, _SE(JavaScriptJavaBridge_constructor));
    cls->defineFinalizeFunction(_SE(JavaScriptJavaBridge_finalize));

    cls->defineFunction("callStaticMethod", _SE(JavaScriptJavaBridge_callStaticMethod));

    cls->install();
    __jsb_JavaScriptJavaBridge_class = cls;

    se::ScriptEngine::getInstance()->clearException();

    return true;
}
```

register_javascript_java_bridge 方法在游戏引擎初始化时注册：

```
bool jsb_register_all_modules()
{
......

#if (CC_TARGET_PLATFORM == CC_PLATFORM_ANDROID)
    se->addRegisterCallback(register_javascript_java_bridge);
#endif

......
}
```

原理就是js通过js引擎调用c++方法，然后通过jni调用java方法。

## Java 调用 JavaScript 原理分析

先来看一下 `Cocos2dxJavascriptJavaBridge.evalString` 方法：

```
public class Cocos2dxJavascriptJavaBridge {
    public static native int evalString(String value);
}
```

```
// cocos/scripting/js-bindings/manual/JavaScriptJavaBridge.cpp
JNIEXPORT jint JNICALL JNI_JSJAVABRIDGE(evalString)
        (JNIEnv *env, jclass cls, jstring value)
{
    if (!se::ScriptEngine::getInstance()->isValid()) {
        CCLOG("ScriptEngine has not been initialized");
        return 0;
    }

    se::AutoHandleScope hs;
    bool strFlag = false;
    std::string strValue = cocos2d::StringUtils::getStringUTFCharsJNI(env, value, &strFlag);
    if (!strFlag)
    {
        CCLOG("JavaScriptJavaBridge_evalString error, invalid string code");
        return 0;
    }
    se::ScriptEngine::getInstance()->evalString(strValue.c_str());
    return 1;
}
```