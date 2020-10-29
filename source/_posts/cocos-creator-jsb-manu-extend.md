---
title: Cocos Creator -- JSB 手动绑定扩展
categories: cocos
comments: true
tags: [cocos]
description: JSB 手动绑定扩展
date: 2020-2-16 10:00:00
---

## 概述

本文主要介绍一下在 Android 应用层扩展 JSB 的一些方法。
JSB 就是在 JS 代码里面可以调用 c++ 实现的变量、方法和类。大部分的jsb都是由官方绑定好的，很方便的使用 cc.Xxx 就可以调用，官方绑定好的代码在 `cocos/scripting/js-bindings/manual/`（手动绑定） 和 `cocos/scripting/js-bindings/auto/`（自动绑定）。自动绑定的代码是不允许手动修改的，所有修改必须经由修改配置文件，然后重新生成自动绑定代码完成。但有时我们想要加入自己客制化的需求，就需要对 JSB 进行扩展。

## 扩展全局属性

我们开始为 js 全局属性 `myDefineStr` 绑定一个值，下面采用C++的代码来完成。如果想采用 Java 来进行这个值的设置，可以在 C++ 层通过 JNI 调用 Java 来完成，对此 cocos 已经封装好一个 JniHelper 工具类（"platform/android/jni/JniHelper.h"）。
在Android工程对应的 Classes 目录下新建一个 CustomJSB.h 文件和 CustomJSB.cpp 文件：

```
// CustomJSB.h 

#ifndef CustomJSB_H
#define CustomJSB_H

void customBind();

#endif
```

```
//CustomJSB.cpp

#include "CustomJSB.h"

#include "scripting/js-bindings/jswrapper/SeApi.h"

void customBind() {
    se::Object* globalObj = se::ScriptEngine::getInstance()->getGlobalObject();
    globalObj->setProperty("myDefineStr", se::Value("Hello"));
}
```

然后在同样在 Classes 目录下 AppDelegate.cpp 的 `applicationDidFinishLaunching` 方法中调用该方法：

```
bool AppDelegate::applicationDidFinishLaunching()
{
    ......
    
    jsb_register_all_modules();
    
    se->start();

    se::AutoHandleScope hs;
    // 注意，一定要放在 se::AutoHandleScope hs; 的后面
    customBind();

    jsb_run_script("jsb-adapter/jsb-builtin.js");
    jsb_run_script("main.js");
    
    ......
}
```

然后在 Android 工程的 jni目录下面的 CocosAndroid.mk 中将源文件添加进去：

```
LOCAL_SRC_FILES := hellojavascript/main.cpp \
				   ../../Classes/AppDelegate.cpp \
				   ../../Classes/jsb_module_register.cpp \
				   ../../Classes/CustomJSB.cpp \
```

最后，我们在 JS 端调用一下：

```
        if(cc.sys.isNative && cc.sys.os == cc.sys.OS_ANDROID){
            this.label.string = myDefineStr;
        }else{
            this.label.string = this.text;
        }
```

因为 Android 原生平台能使用，做个平台区分。
重新构建和运行工程，在Web和Android平台均正常运行。

## 扩展静态方法

关于扩展静态方法，实现 JavaScript 和 Java 如何互相调用，请参考官方文档 [如何在 Android 平台上使用 JavaScript 直接调用 Java 方法](https://docs.cocos.com/creator/2.2/manual/zh/advanced-topics/java-reflection.html)。
通过 `callStaticMethod` 方法我们可以通过反射机制直接在 JavaScript 中调用 Java 的静态方法。而通过 `evalString` 方式，则可以执行 JS 代码，这样便可以进行双端通信。
下面的例子来介绍一下如何实现 JavaScript 和 Java 的同步调用和异步调用。
在 Android 客户端工程创建 Test.java 类。

```
// Test.java
package org.cocos2dx.javascript;

import android.util.Log;

import org.cocos2dx.lib.Cocos2dxJavascriptJavaBridge;

import java.util.HashMap;

public class Test {
    public static String testSync(String text) {
        Log.e("Test","called testSync "+text);
        return "Test";
    }

    public static void testAsync() {
        Log.e("Test","called testAsync ");
        AppActivity.sApp.runOnGLThread(new Runnable() {
            @Override
            public void run() {
                HashMap map = new HashMap();
                map.put("para1","ret1");
                map.put("para2","ret2");
                // 注意这里单引号和双引号的使用，如果使用错误会导致 evalString 失败
                String jsEvalString = "window.testCallback('" + map.toString() + "')";;
                Cocos2dxJavascriptJavaBridge.evalString(jsEvalString);
            }
        });
    }
}
```

JS 端调用：

```
//js
    test:function() {
        log("test function onclick");
        // 测试同步调用方法
        this.testSync()
        // 测试异步调用，设置一个回调函数
        this.testAsync(function(para){
            log( "callback = " + para)
        })
    },
    
    testSync() {
        // 调用同步方法
        var ret = jsb.reflection.callStaticMethod("org/cocos2dx/javascript/Test", "testSync", "(Ljava/lang/String;)Ljava/lang/String;", "str");
        log("testSync()")
        log("Test testSync = "+ret)
    },

    testAsync(callback){
        // 先设置一个异步回调
        window.testCallback = callback
        // 调用方法
        jsb.reflection.callStaticMethod("org/cocos2dx/javascript/Test", "testAsync", "()V");
    },
```


## 相关文章

https://blog.csdn.net/u013654125/article/details/78772239
https://www.codercto.com/a/86501.html
https://blog.csdn.net/Yishionwang/article/details/89497522
https://docs.cocos.com/creator/2.2/manual/zh/advanced-topics/jsb-manual-binding.html
