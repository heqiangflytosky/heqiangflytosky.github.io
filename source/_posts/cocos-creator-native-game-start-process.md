---
title: Cocos Creator -- JSB 游戏启动运行流程
categories: cocos
comments: true
tags: [cocos]
description:  游戏启动运行流程
date: 2020-3-5 10:00:00
---

## 概述

本文简单介绍一下游戏的运行流程。本文只简单介绍整个游戏加载、初始化、运行和渲染流程。具体的细节会分类专门介绍。

## 加载启动流程

main.js 开始加载，标志着游戏开始运行了。

```
├── window.boot // js main.js
    ├── cc.AssetLibrary.init  // js main.js
    ├── cc.game.run // js main.js
        ├── _initConfig  // engine CCGame.js
        ├── prepare  // engine CCGame.js
            ├── _loadPreviewScript  // engine CCGame.js
                ├── _prepareFinished  // engine CCGame.js
                    ├── _initEngine()   // engine CCGame.js
                        ├── _initRenderer()  // engine CCGame.js
                            ├── initWebGL // engine renderer/index.js
                    ├── _setAnimFrame()
                    ├── cc.AssetLibrary._loadBuiltins
                        ├── _runMainLoop() // engine CCGame.js 
                            ├── window.requestAnimFrame // engine CCGame.js 运行游戏，游戏按帧率刷新
                                ├── CCDirector.mainLoop // engine CCDirector.js 开始渲染游戏，具体后面章节介绍
                        ├── onStart // main.js, 在 main.js 中设置的回调函数，下面分析
``` 



### main.js 

首先从 main.js 开发分析，这是游戏的执行入口。
main.js 位于 build/jsb-link 目录下，在 `AppDelegate::applicationDidFinishLaunching()` 被加载。

```
window.boot = function () {
    var settings = window._CCSettings;
    window._CCSettings = undefined;

    if ( !settings.debug ) {
        var uuids = settings.uuids;

        var rawAssets = settings.rawAssets;
        var assetTypes = settings.assetTypes;
        var realRawAssets = settings.rawAssets = {};
        for (var mount in rawAssets) {
            var entries = rawAssets[mount];
            var realEntries = realRawAssets[mount] = {};
            for (var id in entries) {
                var entry = entries[id];
                var type = entry[1];
                // retrieve minified raw asset
                if (typeof type === 'number') {
                    entry[1] = assetTypes[type];
                }
                // retrieve uuid
                realEntries[uuids[id] || id] = entry;
            }
        }

        var scenes = settings.scenes;
        for (var i = 0; i < scenes.length; ++i) {
            var scene = scenes[i];
            if (typeof scene.uuid === 'number') {
                scene.uuid = uuids[scene.uuid];
            }
        }

        var packedAssets = settings.packedAssets;
        for (var packId in packedAssets) {
            var packedIds = packedAssets[packId];
            for (var j = 0; j < packedIds.length; ++j) {
                if (typeof packedIds[j] === 'number') {
                    packedIds[j] = uuids[packedIds[j]];
                }
            }
        }

        var subpackages = settings.subpackages;
        for (var subId in subpackages) {
            var uuidArray = subpackages[subId].uuids;
            if (uuidArray) {
                for (var k = 0, l = uuidArray.length; k < l; k++) {
                    if (typeof uuidArray[k] === 'number') {
                        uuidArray[k] = uuids[uuidArray[k]];
                    }
                }
            }
        }
    }

    function setLoadingDisplay () {
        // loading 页面实现
    }

    // 设置游戏初始化之后的回调函数
    var onStart = function () {
        cc.loader.downloader._subpackages = settings.subpackages;

        cc.view.enableRetina(true);
        cc.view.resizeWithBrowserSize(true);

        if (cc.sys.isBrowser) {
            // 浏览器上显示 loading页面
            setLoadingDisplay();
        }

        if (cc.sys.isMobile) {
            if (settings.orientation === 'landscape') {
                cc.view.setOrientation(cc.macro.ORIENTATION_LANDSCAPE);
            }
            else if (settings.orientation === 'portrait') {
                cc.view.setOrientation(cc.macro.ORIENTATION_PORTRAIT);
            }
            cc.view.enableAutoFullScreen([
                cc.sys.BROWSER_TYPE_BAIDU,
                cc.sys.BROWSER_TYPE_WECHAT,
                cc.sys.BROWSER_TYPE_MOBILE_QQ,
                cc.sys.BROWSER_TYPE_MIUI,
            ].indexOf(cc.sys.browserType) < 0);
        }
        if (cc.sys.isBrowser && cc.sys.os === cc.sys.OS_ANDROID) {
            cc.macro.DOWNLOAD_MAX_CONCURRENT = 2;
        }

        function loadScene(launchScene) {
            cc.director.loadScene(launchScene,
                function (err) {
                    // 错误处理回调
                    ......
                }
            );

        }
        // 根配置文件设置 launchScene
        var launchScene = settings.launchScene;

        // 加载场景
        loadScene(launchScene);
    };

    // jsList
    var jsList = settings.jsList;

    var bundledScript = settings.debug ? 'src/project.dev.js' : 'src/project.js';
    if (jsList) {
        jsList = jsList.map(function (x) {
            return 'src/' + x;
        });
        jsList.push(bundledScript);
    }
    else {
        jsList = [bundledScript];
    }

    var option = {
        id: 'GameCanvas',
        scenes: settings.scenes,
        debugMode: settings.debug ? cc.debug.DebugMode.INFO : cc.debug.DebugMode.ERROR,
        showFPS: settings.debug,
        frameRate: 60,
        jsList: jsList,
        groupList: settings.groupList,
        collisionMatrix: settings.collisionMatrix,
    };

    // 初始化资源文件，将会加载一些资源文件
    cc.AssetLibrary.init({
        libraryPath: 'res/import',
        rawAssetsBase: 'res/raw-',
        rawAssets: settings.rawAssets,
        packedAssets: settings.packedAssets,
        md5AssetsMap: settings.md5AssetsMap,
        subpackages: settings.subpackages
    });

    cc.game.run(option, onStart);
};

if (window.jsb) {
    var isRuntime = (typeof loadRuntime === 'function');
    // 加载一些 js 文件
    if (isRuntime) {
        require('src/settings.js');
        require('src/cocos2d-runtime.js');
        if (CC_PHYSICS_BUILTIN || CC_PHYSICS_CANNON) {
            require('src/physics.js');
        }
        require('jsb-adapter/engine/index.js');
    }
    else {
        require('src/settings.js');
        require('src/cocos2d-jsb.js');
        if (CC_PHYSICS_BUILTIN || CC_PHYSICS_CANNON) {
            require('src/physics.js');
        }
        require('jsb-adapter/jsb-engine.js');
    }

    cc.macro.CLEANUP_IMAGE_CACHE = true;
    // 调用上面的 boot 方法
    window.boot();
}
```

## 刷新流程

前面介绍过，初始化流程到 `_runMainLoop` 就结束了，剩下的就是游戏按帧率进行刷新然后运行游戏了。
本节介绍一下 cocos 的刷新机制。
先来看一下 `_runMainLoop` 方法，这里面最重要的就是注册了一个刷新回调方法：

```
// engine CCGame.js 

    _runMainLoop: function () {
        ......
        // 设置是否在界面显示帧率等调试信息
        debug.setDisplayStats(config.showFPS);
        // 设置刷新回调
        callback = function (now) {
            if (!self._paused) {
                self._intervalId = window.requestAnimFrame(callback);
                if (!CC_JSB && !CC_RUNTIME && frameRate === 30) {
                    if (skip = !skip) {
                        return;
                    }
                }
                director.mainLoop(now);
            }
        };
        // 调用 requestAnimFrame 执行刷新频率，这个方法的实现在 jsb-adapter
        self._intervalId = window.requestAnimFrame(callback);
        self._paused = false;
    }
```

```
// jsb-adapter index.js
window.requestAnimationFrame = function(cb) {
    let id = ++_requestAnimationFrameID;
    // 放到回到方法集合中
    _requestAnimationFrameCallbacks[id] = cb;
    return id;
};
```

然后在 tick 方法中调用：

```
function tick(nowMilliSeconds) {
    if (_firstTick) {
        _firstTick = false;
        if (window.onload) {
            var event = new Event('load');
            event._target = window;
            window.onload(event);
        }
    }
    fireTimeout(nowMilliSeconds);

    for (let id in _requestAnimationFrameCallbacks) {
        _oldRequestFrameCallback = _requestAnimationFrameCallbacks[id];
        if (_oldRequestFrameCallback) {
            delete _requestAnimationFrameCallbacks[id];
            _oldRequestFrameCallback(nowMilliSeconds);
        }
    }
    flushCommands();
}

window.gameTick = tick;
```

gameTick 由运行时调用。
流程如下：

```
├── Cocos2dxRenderer.onDrawFrame  // app Cocos2dxRenderer.java
    ├── Cocos2dxRenderer.nativeRender() // app Cocos2dxRenderer.java
        ├── EventDispatcher::dispatchTickEvent(dt)  // cocos2d-x-lite JniImp.cpp
            ├── gameTick  // 调用 js 的 gameTick 函数
                ├── tick // jsb-adapter index.js
```

GLSurfaceView.Renderer 的 onDrawFrame 方法在 Android 绘制每一帧时都会回调这个方法，这个方法会调用到 js 侧的 tick 方法来完成游戏的刷新。
然后再调用 director.mainLoop 方法。