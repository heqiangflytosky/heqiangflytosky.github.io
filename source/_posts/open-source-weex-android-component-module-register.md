---
title: Weex 源码研究 -- Component 和 Module注册过程
categories: Weex
comments: true
tags: [Weex]
description: 介绍 Weex Component 和 Module注册过程
date: 2018-3-20 10:00:00
---

## 注册 Component 和 Module

在 WXSDKEngine 的 `register()` 方法中进行。这个方法里面是注册 Weex 原生自带的一些组件和模块，如果你想扩展组件或者模块，也是需要走注册流程的，流程和这里的是一样的。
http://weex.apache.org/cn/guide/extend-android.html

### 注册 Component

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

### 注册 Module

```
├── WXSDKEngine.registerModule()
    └── WXModuleManager.registerModule()
        ├── WXModuleManager.registerNativeModule()
        └── WXModuleManager.WXSDKManager.getInstance().registerModules()
            └── WXBridgeManager.registerModules()
                 └── WXBridgeManager.invokeRegisterModules()
                     └── WXBridge.execJS()
                 
```

注册 Module 的流程和注册 Component 流程比较类似。
`registerModule()` 也有多个重载方法，但最终调用的是 `WXModuleManager.registerModule()` 方法。
在介绍 `WXModuleManager.registerModule()` 方法之前，我们先看一下它的参数之一 `ModuleFactory`。这个类和注册 Component 时的 ComponentHolder 类比较类似。
一般情况下，我们会调用 `registerModule(String moduleName, Class<T> moduleClass,boolean global)` 方法来注册 Module，比如：

```
registerModule("modal", WXModalUIModule.class, false);
```

这时候会以 `WXModalUIModule.class` 为参数生成一个 `TypeModuleFactory` 实例。它有三个作用分别对应三个方法：

 - buildInstance()：生成一个 Module 对应 class 的实例
 - getMethods()：获取 class 所有的 JSMethod 注解的方法 
 - getMethodInvoker(String name)：获取注册 Module 提供的API的执行体

WXModuleManager.registerModule() 方法：

```
  public static boolean registerModule(final String moduleName, final ModuleFactory factory, final boolean global) throws WXException {
    ...
    // 在JS线程中执行注册任务
    WXBridgeManager.getInstance()
        .post(new Runnable() {
      @Override
      public void run() {
        ...
        // 如果这个Module是全局的，则直接生成该Module对象，然后存储在 sGlobalModuleMap 中，那么就可以通过 WXModuleManager.findModule() 方法直接调用 Module 的方法。
        if (global) {
          try {
            WXModule wxModule = factory.buildInstance();
            wxModule.setModuleName(moduleName);
            sGlobalModuleMap.put(moduleName, wxModule);
          } catch (Exception e) {
            WXLogUtils.e(moduleName + " class must have a default constructor without params. ", e);
          }
        }

        // 在 Java 层注册 Module
        try {
          registerNativeModule(moduleName, factory);
        } catch (WXException e) {
          WXLogUtils.e("", e);
        }
        // 在 JS 层注册 Module
        registerJSModule(moduleName, factory);
      }
    });
    return true;

  }
```

Native 层的注册过程很简单，在 `sModuleFactoryMap` 中保存了 Module 的 Name 及对应的 `ModuleFactory`，提供给 `callModuleMethod()` 方法调用。

```
static boolean registerNativeModule(String moduleName, ModuleFactory factory) throws WXException {
    if (factory == null) {
      return false;
    }

    try {
      sModuleFactoryMap.put(moduleName, factory);
    }catch (ArrayStoreException e){
      e.printStackTrace();
    }
    return true;
  }
```

JS 层的注册也比较简单，调用 `WXBridge.execJS()` 注册。
WXBridgeManager.invokeRegisterModules()

```
  private void invokeRegisterModules(Map<String, Object> modules, List<Map<String, Object>> failReceiver) {
    ...
    // 生成参数：{"modal":["confirm","removeAllEventListeners","toast","prompt","alert","addEventListener"]}
    WXJSObject[] args = {new WXJSObject(WXJSObject.JSON,
        WXJsonUtils.fromObjectToJSONString(modules))};
    try {
    // 通过jni运行js方法
      mWXBridge.execJS("", null, METHOD_REGISTER_MODULES, args);
    } catch (Throwable e) {
  }
```
