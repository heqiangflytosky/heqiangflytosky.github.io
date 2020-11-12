---
title: Cocos Creator -- JSB 事件分发机制源码分析
categories: cocos
comments: true
tags: [cocos]
description:  JSB 事件分发机制源码分析
date: 2020-3-26 10:00:00
---

## 概述

Cocos2d-x-lite 通过事件分发机制响应用户事件，已内置支持常见的事件如触摸事件，键盘事件等。同时提供了创建自定义事件的方法，满足我们在游戏的开发过程中，特殊的事件响应需求。
本分分析源码是基于 cocos creator 2.3.1 版本源码进行的

## 监听事件

在游戏中注册一个监听有下面的几种方法：

第一种：

在脚本中声明函数：

```
    btnClick1: function (event, customEventData) {
        //这里 event 是一个 Touch Event 对象，你可以通过 event.target 取到事件的发送节点
        var node = event.target;
        var button = node.getComponent(cc.Button);
        //这里的 customEventData 参数就等于你之前设置的 "click1 user data"
        cc.log("node=", node.name, " event=", event.type, " data=", customEventData);
    }
```

然后在编辑器中添加事件。

方法二：

```
        var clickEventHandler = new cc.Component.EventHandler();
        clickEventHandler.target = this.node; //这个 node 节点是你的事件处理代码组件所属的节点，这里就是Button2
        clickEventHandler.component = "Button";//这个是脚本文件名
        clickEventHandler.handler = "testEvtent"; //回调函名称
        clickEventHandler.customEventData = "click2 user data"; //用户数据

        var button = this.node.getComponent(cc.Button); //获取cc.Button组件
        button.clickEvents.push(clickEventHandler); //增加处理
```

```
    testEvtent(event, customEventData) {
        log("TestEvent function " +event+", "+ customEventData);
    },
```

方法三：

```
    onLoad() {
        this.node.on(cc.Node.EventType.TOUCH_START, function (event) {
            cc.log("TOUCH_START event=", event.type);
        });

        this.node.on(cc.Node.EventType.TOUCH_MOVE, function (event) {
            cc.log("TOUCH_MOVE event=", event.type);
        });

        this.node.on(cc.Node.EventType.TOUCH_END, function (event) {
            cc.log("TOUCH_END event=", event.type);
        });
    }
```

这种方式不能传递自定义数据，不推荐使用。

我们使用第二种方式创建一个事件监听。
那么，组件中是如何把我们注册的事件监听方法和事件联系起来的呢？
下来简单看一下 CCButton 的代码。

```
// engine CCButton.js

    _registerNodeEvent () {
        this.node.on(cc.Node.EventType.TOUCH_START, this._onTouchBegan, this);
        this.node.on(cc.Node.EventType.TOUCH_MOVE, this._onTouchMove, this);
        this.node.on(cc.Node.EventType.TOUCH_END, this._onTouchEnded, this);
        this.node.on(cc.Node.EventType.TOUCH_CANCEL, this._onTouchCancel, this);

        this.node.on(cc.Node.EventType.MOUSE_ENTER, this._onMouseMoveIn, this);
        this.node.on(cc.Node.EventType.MOUSE_LEAVE, this._onMouseMoveOut, this);
    },
```

```
// engine CCNode.js

    on (type, callback, target, useCapture) {
        let forDispatch = this._checknSetupSysEvent(type);
        if (forDispatch) {
            return this._onDispatch(type, callback, target, useCapture);
        }
        else {
            switch (type) {
                case EventType.POSITION_CHANGED:
                this._eventMask |= POSITION_ON;
                break;
                ......
                case EventType.COLOR_CHANGED:
                this._eventMask |= COLOR_ON;
                break;
            }
            if (!this._bubblingListeners) {
                this._bubblingListeners = new EventTarget();
            }
            // 这里监听事件
            return this._bubblingListeners.on(type, callback, target);
        }
    },
```

好了，这里就只需要静等click事件来就行了。

## 事件注册

接下来我们看一下事件的注册流程，了解一下事件如何与我们的回调函数联系起来的：

```
├── prepare    // engine CCGame.js
    ├── _prepareFinished   // engine CCGame.js
        ├── _initEngine()  // engine CCGame.js
            ├── _initEvents()   // engine CCGame.js
                ├── registerSystemEvent // engine CCInputManager.js
                    ├── registerTouchEvent  // engine CCInputManager.js
                        ├── addEventListener // jsb-adapter EventTarget.js
```

下面通过源码来看一下几个关键的方法。

```
    registerSystemEvent (element) {
        ......
        //register touch event
        // 建立事件和回调方法的联系
        if (supportTouches) {
            // 事件名和回调方法集合
            let _touchEventsMap = {
                "touchstart": function (touchesToHandle) {
                    selfPointer.handleTouchesBegin(touchesToHandle);
                    element.focus();
                },
                "touchmove": function (touchesToHandle) {
                    selfPointer.handleTouchesMove(touchesToHandle);
                },
                "touchend": function (touchesToHandle) {
                    selfPointer.handleTouchesEnd(touchesToHandle);
                },
                "touchcancel": function (touchesToHandle) {
                    selfPointer.handleTouchesCancel(touchesToHandle);
                }
            };

            let registerTouchEvent = function (eventName) {
                console.log("TestEvent engine registerTouchEvent")
                // 获取上面的回调方法
                let handler = _touchEventsMap[eventName];
                // 为事件添加监听，建立事件和回调方法的联系
                element.addEventListener(eventName, (function(event) {
                    if (!event.changedTouches) return;
                    let body = document.body;

                    canvasBoundingRect.adjustedLeft = canvasBoundingRect.left - (body.scrollLeft || 0);
                    canvasBoundingRect.adjustedTop = canvasBoundingRect.top - (body.scrollTop || 0);
                    // 执行该事件的回调方法
                    handler(selfPointer.getTouchesByEvent(event, canvasBoundingRect));
                    event.stopPropagation();
                    event.preventDefault();
                }), false);
            };
            // 为每个事件注册回调方法
            for (let eventName in _touchEventsMap) {
                registerTouchEvent(eventName);
            }
        }

        this._registerKeyboardEvent();

        this._isRegisterEvent = true;
    },
```

再来看看 jsb-adapter 是如何绑定这些事件和回调方法的。

```
// jsb-adapter EventTarget.js

    addEventListener(eventName, listener, options) {
        if (!listener) {
            return false
        }
        if (typeof listener !== "function" && !isObject(listener)) {
            throw new TypeError("'listener' should be a function or an object.")
        }

        const listeners = this._listeners
        const optionsIsObj = isObject(options)
        const capture = optionsIsObj ? Boolean(options.capture) : Boolean(options)
        const listenerType = (capture ? CAPTURE : BUBBLE)
        const newNode = {
            listener,
            listenerType,
            passive: optionsIsObj && Boolean(options.passive),
            once: optionsIsObj && Boolean(options.once),
            next: null,
        }

        // Set it as the first node if the first node is null.
        let node = listeners.get(eventName)
        if (node === undefined) {
            // 添加到 listeners
            listeners.set(eventName, newNode)
            this._associateSystemEventListener(eventName);
            return true
        }

        // Traverse to the tail while checking duplication..
        let prev = null
        while (node) {
            if (node.listener === listener && node.listenerType === listenerType) {
                // Should ignore duplication.
                return false
            }
            prev = node
            node = node.next
        }

        // Add it.
        prev.next = newNode
        this._associateSystemEventListener(eventName);
        return true
    }
    
    _associateSystemEventListener(eventName) {
        var handleEventNames;
        for (var key in __handleEventNames) {
            handleEventNames = __handleEventNames[key];
            if (handleEventNames.indexOf(eventName) > -1) {
                if (__enableCallbackMap[key] && __listenerCountMap[key] === 0) {
                    __enableCallbackMap[key]();
                }

                // 把事件名称和事件回调保存在 __listenerMap 中
                // 后面等事件传递过来时，会从这个map中获取元素进行事件的分发
                if (this._listenerCount[key] === 0)
                    __listenerMap[key][this._targetID] = this;
                ++this._listenerCount[key];
                ++__listenerCountMap[key];
                break;
            }
        }
    }
```

## 事件分发

建立好事件和回调方法的联系后，下面我们就来看一下事件分发机制。

```

├── Cocos2dxGLSurfaceView.onTouchEvent
    ├── Cocos2dxRenderer.handleActionDown
        ├── Cocos2dxRenderer.nativeTouchesBegin
            ├── nativeTouchesBegin   cocos2d-x-lite JniImp.cpp
                ├── dispatchTouchEventWithOnePoint  cocos2d-x-lite JniImp.cpp
                    ├── cocos2d::EventDispatcher::dispatchTouchEvent  EventDispatcher.cpp
                        ├── touchEventHandlerFactory  jsb-adapter EventTarget.js
                            ├── EventTarget.dispatchEvent
                                
```

在 EventTarget.js 建立 c++ 和 js 的绑定。

```
jsb.onTouchStart = touchEventHandlerFactory('touchstart');
```

收到 c++ 传递过来的事件后开始分发事件。

```
function touchEventHandlerFactory(type) {
    var _this = this
    return (touches) => {
        //console.log("TestEvent jsb-adapter touchEventHandlerFactory")
        console.log("TestEvent jsb-adapter touchEventHandlerFactory "+type)
        const touchEvent = new TouchEvent(type)

        touchEvent.touches = touches;
        touchEvent.targetTouches = Array.prototype.slice.call(touchEvent.touches)
        touchEvent.changedTouches = touches;//event.changedTouches
        // touchEvent.timeStamp = event.timeStamp

        var i = 0, touchCount = touches.length;
        var target;
        var touchListenerMap = __listenerMap.touch;
        for (let key in touchListenerMap) {
            target = touchListenerMap[key];
            for (i = 0; i < touchCount; ++i) {
                touches[i].target = target;
            }
            target.dispatchEvent(touchEvent);
        }
    }
}

    dispatchEvent(event) {
        console.log("TestEvent jsb-adapter dispatchEvent "+event.type)
        if (!event || typeof event.type !== "string") {
            throw new TypeError("\"event.type\" should be a string.")
        }

        const eventName = event.type
        var onFunc = this['on' + eventName];
        if (onFunc && typeof onFunc === 'function') {
            event._target = event._currentTarget = this;
            onFunc.call(this, event);
            event._target = event._currentTarget = null
            event._eventPhase = 0
            event._passiveListener = null

            if (event.defaultPrevented)
                return false;
        }

        // If listeners aren't registered, terminate.
        const listeners = this._listeners
        
        let node = listeners.get(eventName)
        if (!node) {
            return true
        }

        event._target = event._currentTarget = this;

        // This doesn't process capturing phase and bubbling phase.
        // This isn't participating in a tree.
        let prev = null
        while (node) {
            // Remove this listener if it's once
            if (node.once) {
                if (prev) {
                    prev.next = node.next
                }
                else if (node.next) {
                    listeners.set(eventName, node.next)
                }
                else {
                    listeners.delete(eventName)
                }
            }
            else {
                prev = node
            }

            // Call this listener
            event._passiveListener = node.passive ? node.listener : null
            if (typeof node.listener === "function") {
                // 执行前面添加的改事件对应的回调方法
                node.listener.call(this, event)
            }

            // Break if `event.stopImmediatePropagation` was called.
            if (event._stopped) {
                break
            }

            node = node.next
        }
        event._target = event._currentTarget = null
        event._eventPhase = 0
        event._passiveListener = null

        return !event.defaultPrevented
    }
```

这里的执行的回调方法就是前面 engine CCInputManager.js 中 registerSystemEvent 为改事件添加的回调方法。
我们接下来再看下面的执行流程。

```
├── handleTouchesBegin // engine CCInputManager.js
    ├── dispatchEvent //  engine CCEventManager.js
        ├── _dispatchTouchEvent  //  engine CCEventManager.js
            ├── _onTouchEventCallback //  engine CCEventManager.js
                ├── onTouchBegan // engine CCNode.js
                    ├── _touchStartHandler // engine CCNode.js
                        ├── dispatchEvent   // engine CCNode.js
                            ├── _doDispatchEvent // engine CCNode.js
```

```
function _doDispatchEvent (owner, event) {
                ...
                // fire event
                // 向前面在 CCButton 里面注册的监听发送事件
                target._bubblingListeners.emit(event.type, event);
                // check if propagation stopped
                if (event._propagationStopped) {
                    _cachedArray.length = 0;
                    return;
                }
            }
        }
    }
    _cachedArray.length = 0;
}
```

下面就回到了 CCButton.js ，这里我们不再追踪 onTouchBegan ，因为这个事件的方法体里面没有干什么事情，只修改了一些状态位，click 事件是在 onTouchEnded 中触发的。
我们来看 onTouchEnded。

```
    _onTouchEnded (event) {
        if (!this.interactable || !this.enabledInHierarchy) return;

        if (this._pressed) {
            // 发送事件到我们前面在应用里面写的事件回调里面去。
            cc.Component.EventHandler.emitEvents(this.clickEvents, event);
            this.node.emit('click', this);
        }
        this._pressed = false;
        this._updateState();
        event.stopPropagation();
    },
```

## 推荐阅读

https://www.cnblogs.com/chevin/p/7893963.html