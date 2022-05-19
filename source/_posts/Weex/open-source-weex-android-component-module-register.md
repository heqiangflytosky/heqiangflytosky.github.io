---
title: Weex 源码研究 -- Component 和 DomObject 注册过程
categories: Weex
comments: true
tags: [Weex]
description: 介绍 Weex Component 和 DomObject注册过程
date: 2018-3-20 10:00:00
---


在 WXSDKEngine 的 `register()` 方法中进行。这个方法里面是注册 Weex 原生自带的一些组件、模块以及 DomObject，如果你想扩展组件或者模块，也是需要走注册流程的，流程和这里的是一样的。
[扩展 Android 的功能 ](http://weex.apache.org/cn/guide/extend-android.html)

## 注册 Component

```
├── WXSDKEngine.registerComponent()
    └── WXComponentRegistry.registerComponent()
        ├── WXComponentRegistry.registerNativeComponent()
        └── WXComponentRegistry.registerJSComponent()
            └── WXSDKManager.registerComponents()
                └── WXBridgeManager.registerComponents()
                    └── WXBridgeManager.invokeRegisterComponents()
```

注册 `Component` 是在 `WXSDKEngine.registerComponent()` 方法中进行的，这个方法有几个重载方法，但最终都会执行到这个方法中来：

```java
  public static boolean registerComponent(IFComponentHolder holder, boolean appendTree, String ... names) throws WXException {
    boolean result =  true;
    for(String name:names) {
      Map<String, Object> componentInfo = new HashMap<>();
      if (appendTree) {
        componentInfo.put("append", "tree");
      }
      result  = result && WXComponentRegistry.registerComponent(name, holder, componentInfo);
    }
    return result;
  }
```

先来介绍一下 `IFComponentHolder` 这个参数。我们就以 `WXText` 的注册过来介绍一下。

```
      registerComponent(
        new SimpleComponentHolder(
          WXText.class,
          new WXText.Creator()
        ),
        false,
        WXBasicComponentType.TEXT
      );
```

注册是实例化了一个 `SimpleComponentHolder` 类。这个类有几个方法，对应它的几个主要作用：

 - createInstance()：生成 Component 实现类的一个实例，比如 `WXText`
 - getMethods(Class clz)：获取实现类中 `WXComponentProp` 和 `JSMethod` 注解标记的属性和方法。
 - getMethods()：获取所有 `JSMethod` 注解标记方法。
 - getMethodInvoker(String name)：获取某个方法对应的实现类的实体方法。
 - getPropertyInvoker(String name)：获取某个属性对应的实现类的实体方法。

![效果图](http://www.plantuml.com/plantuml/svg/TP1DIWD148NtVOeYgyaYEO6BG40SKH4YA8YBihiI6xkhXkxAGFmvYQTmBNWRyHfECX8qCxFzUFNUHysoOj9r3ERAQo0OBNoi0iqbLiB4UYB1KVf-__Xw-nmPurafBT4IbCS76NWs0BLu1q7GbSiBuJDysXJZ1fTSosCJMP5U9gaewUON5GjDdbV066biNlyCxEldYL2bxR--sMEmMqubPqMsLFo_FiKQiqs-qjqGtWU2NKExTtktTJadVH2N3nLRF21e0-OClLyofkDyz3ATTbzbUko6eXtI12UJ0O4PiLl7y0C0)

```
  public SimpleComponentHolder(Class<? extends WXComponent> clz,ComponentCreator customCreator) {
    this.mClz = clz;
    this.mCreator = customCreator;
  }
```

这里传入的两个参数分别是 `Class` 和 `ComponentCreator`。
我们来看一下 `WXText.Creator()` 类。

```
  public static class Creator implements ComponentCreator {
    public WXComponent createInstance(WXSDKInstance instance, WXDomObject node, WXVContainer parent) throws IllegalAccessException, InvocationTargetException, InstantiationException {
      return new WXText(instance, node, parent);
    }
  }
```

`WXText.Creator()` 类实现了 `ComponentCreator`，在 `createInstance()` 方法中实例化了 `WXText` 类，这样就建立了和 Native Java 类的连接。
我们知道，`IFComponentHolder` 是继承自 `ComponentCreator` 的，那么这里在实例化 `SimpleComponentHolder` 又传入一个 `ComponentCreator` 实例是用来做什么的呢？
我们再来看一下 `SimpleComponentHolder` 实现的 `createInstance()` 方法：

```
  @Override
  public synchronized WXComponent createInstance(WXSDKInstance instance, WXDomObject node, WXVContainer parent) throws IllegalAccessException, InvocationTargetException, InstantiationException {
    WXComponent component = mCreator.createInstance(instance,node,parent);
    component.bindHolder(this);
    return component;
  }
```

这里其实就是调用的 `WXText.Creator()` 实例的 `createInstance()` 方法，并把 `SimpleComponentHolder` 绑定到生成的 `WXComponent` 上。这个方法什么时候使用我们后面再说。

再来看一下 `WXComponentRegistry.registerComponent()` 方法：
```
  public static boolean registerComponent(final String type, final IFComponentHolder holder, final Map<String, Object> componentInfo) throws WXException {
    ...
    // 在js线程中执行注册操作
    WXBridgeManager.getInstance()
        .post(new Runnable() {
      @Override
      public void run() {
        try {
          Map<String, Object> registerInfo = componentInfo;
          if (registerInfo == null){
            registerInfo = new HashMap<>();
          }
          // 将 component 名字作为类型
          registerInfo.put("type",type);
          // 获取 `JSMethod` 注解标记的方法
          // 这里或许有个疑问，自定义的属性去哪里了？
          registerInfo.put("methods",holder.getMethods());
          registerNativeComponent(type, holder);
          registerJSComponent(registerInfo);
          sComponentInfos.add(registerInfo);
        } catch (WXException e) {
          WXLogUtils.e("register component error:", e);
        }
      }
    });
    return true;
  }
```

```
  private void invokeRegisterComponents(List<Map<String, Object>> components, List<Map<String, Object>> failReceiver) {
    ...
    // 生成参数：[{"methods":["recoverImageList","releaseImageList"],"type":"container"}]
    WXJSObject[] args = {new WXJSObject(WXJSObject.JSON,
        WXJsonUtils.fromObjectToJSONString(components))};
    try {
    // 通过jni运行js方法，注册到js
      mWXBridge.execJS("", null, METHOD_REGISTER_COMPONENTS, args);
    } catch (Throwable e) {
    }
  }
```

这里再来介绍一下 `appendTree` 这个参数。
Weex 注册的 `Component` 有两种类型，一类是有 `{@"append":@"tree"}` 属性的标签，另一类是没有 `{@"append":@"tree"}` 属性的标签。


## 注册 DomObject

前面博客中也说过，DomObject 包括了 `<template>` 在 Dom 树中的所有信息，如 style、attr、event、ref（结点的唯一标识符）、parent、children 等。
在源码中注册过 Component 和 Moudle 后，紧接着是注册 DomObject。

```
      registerDomObject(WXBasicComponentType.TEXT, WXTextDomObject.class);
```

注册流程：

```
├── WXSDKEngine.registerDomObject()
    ├── WXDomRegistry.registerDomObject()
```

WXDomRegistry.registerDomObject():

```
  public static boolean registerDomObject(String type, Class<? extends WXDomObject> clazz) throws WXException {
    if (clazz == null || TextUtils.isEmpty(type)) {
      return false;
    }

    if (sDom.containsKey(type)) {
      if (WXEnvironment.isApkDebugable()) {
        throw new WXException("WXDomRegistry had duplicate Dom:" + type);
      } else {
        WXLogUtils.e("WXDomRegistry had duplicate Dom: " + type);
        return false;
      }
    }
    sDom.put(type, clazz);
    return true;
  }
```

可以看到，这部分是比较简单的。仅仅是把注册的 `WXDomObject` 放到一个 `Map` 中去，在生成 Native Dom 的时候再生成 `WXDomObject` 对象。

```
├── AbstractAddElementAction.addDomInternal()
    ├── WXDomObject.parse()
        ├── WXDomObjectFactory.newInstance(type)
            ├── WXDomRegistry.getDomObjectClass()
```

