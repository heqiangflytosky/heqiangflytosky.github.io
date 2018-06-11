---
title: Weex 源码研究 -- 事件响应流程
categories: Weex
comments: true
tags: [Weex]
description: 介绍 Weex 事件响应流程
date: 2018-4-16 10:00:00
---

## 调用流程
```
├── WXComponent.addClickListener
    ├── onClick()
        ├── WXComponent.OnClickListener.onHostViewClick()
            ├── WXComponent.fireEvent()
                ├── WXSDKInstance.fireEvent()
                    ├── WXBridgeManager.fireEventOnNode()
                        ├── WXBridgeManager.addJSEventTask()
                        ├── WXBridgeManager.sendMessage(WXJSBridgeMsgType.CALL_JS_BATCH)
                        ----------------------------------------------------------------
                        ├── WXBridgeManager.handleMessage(WXJSBridgeMsgType.CALL_JS_BATCH)
                            ├── WXBridgeManager.invokeCallJSBatch()
                                ├── WXJsonUtils.wsonWXJSObject(tasks)
                                ├── WXBridgeManager.invokeExecJS()
                                    ├── WXBridge.execJS()
```

## 代码解析

```
    public void onHostViewClick() {
      Map<String, Object> param= WXDataStructureUtil.newHashMapWithExpectedSize(1);
      Map<String, Object> position = WXDataStructureUtil.newHashMapWithExpectedSize(4);
      int[] location = new int[2];
      mHost.getLocationOnScreen(location);
      position.put("x", WXViewUtils.getWebPxByWidth(location[0],mInstance.getInstanceViewPortWidth()));
      position.put("y", WXViewUtils.getWebPxByWidth(location[1],mInstance.getInstanceViewPortWidth()));
      position.put("width", WXViewUtils.getWebPxByWidth(mDomObj.getLayoutWidth(),mInstance.getInstanceViewPortWidth()));
      position.put("height", WXViewUtils.getWebPxByWidth(mDomObj.getLayoutHeight(),mInstance.getInstanceViewPortWidth()));
      param.put(Constants.Name.POSITION, position);
      fireEvent(Constants.Event.CLICK,param);
    }
```

在 `WXComponent.onHostViewClick()` 方法中会把当前 `View` 的 location 和宽高作为参数传递给 `fireEvent` 方法。

来看一下 `WXBridgeManager.fireEventOnNode()` 方法：

```
  public void fireEventOnNode(final String instanceId, final String ref,
                              final String type, final Map<String, Object> data,
                              final Map<String, Object> domChanges, List<Object> params,  EventResult callback) {
    if (TextUtils.isEmpty(instanceId) || TextUtils.isEmpty(ref)
        || TextUtils.isEmpty(type) || mJSHandler == null) {
      return;
    }
    if (!checkMainThread()) {
      throw new WXRuntimeException(
          "fireEvent must be called by main thread");
    }
    // 这里会根据是否注册回调函数分两种不同情况来执行
    if(callback == null) {
      // 没有注册回调，添加fireEvent事件到待执行的JSEventTask列表，并向 mJSHandler 发送 CALL_JS_BATCH 消息
      addJSEventTask(METHOD_FIRE_EVENT, instanceId, params, ref, type, data, domChanges);
      sendMessage(instanceId, WXJSBridgeMsgType.CALL_JS_BATCH);
    }else{
      // 注册了回调函数，则在 JS 线程立即执行，并执行回调函数。
      asyncCallJSEventWithResult(callback, METHD_FIRE_EVENT_SYNC, instanceId, params, ref, type, data, domChanges);
    }
  }
```

这里 `onClick` 事件并没有注册回调函数，因此会向 `mJSHandler` 发送 `CALL_JS_BATCH` 消息。
后面执行的 `invokeCallJSBatch` 和前面博客 [Module 注册、调用和回调函数的执行流程](http://www.heqiangfly.com/2018/04/10/open-source-weex-android-module-register-callback/) 执行流程是一样的，这里不在介绍。
这里 `WXBridge.execJS` 执行的参数为：

```
[{"args":["11","click",{"position":{"height":19.791666,"width":106.25,"x":321.875,"y":1315.625}},null],"method":"fireEvent"}]
```

把参数传递给前端后会根据 instanceId 和当前元素的 Id 执行对应的方法。

