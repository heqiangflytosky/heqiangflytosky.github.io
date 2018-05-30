---
title: Weex 源码研究 -- Module 注册和调用流程
categories: Weex
comments: true
tags: [Weex]
description: 介绍 Weex Module 注册和调用流程
date: 2018-3-20 10:00:00
---

## 注册 Module

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

WXSDKEngine.registerModule(String moduleName, Class<T> moduleClass,boolean global)

```
  public static <T extends WXModule> boolean registerModule(String moduleName, Class<T> moduleClass,boolean global) throws WXException {
    return moduleClass != null && registerModule(moduleName, new TypeModuleFactory<>(moduleClass), global);
  }
```

这时候会以 `WXModalUIModule.class` 为参数生成一个 `TypeModuleFactory` 实例。它有三个作用分别对应三个方法：

 - buildInstance()：生成一个 Module 对应 class 的实例
 - getMethods()：获取 class 所有的 JSMethod 注解的方法 
 - getMethodInvoker(String name)：获取注册 Module 提供的API的执行体
 - generateMethodMap()：解析该模块中的所有方法，并为它生成一个方法的执行体 `MethodInvoker`

先来看一下 `TypeModuleFactory.generateMethodMap()` 方法：

```
  private void generateMethodMap() {
    if(WXEnvironment.isApkDebugable()) {
      WXLogUtils.d(TAG, "extractMethodNames:" + mClazz.getSimpleName());
    }
    HashMap<String, Invoker> methodMap = new HashMap<>();
    try {
      for (Method method : mClazz.getMethods()) {
        // 获取该方法的所有注解
        for (Annotation anno : method.getDeclaredAnnotations()) {
          if (anno != null) {
            if(anno instanceof JSMethod) {
              JSMethod methodAnnotation = (JSMethod) anno;
              String name = JSMethod.NOT_SET.equals(methodAnnotation.alias())? method.getName():methodAnnotation.alias();
              // 生成MethodInvoker实例，methodAnnotation.uiThread() 是@JSMethod(uiThread = true)注解的uiThread参数，
              // 方法执行时会根据不同的uiThread参数在不同的线程执行
              methodMap.put(name, new MethodInvoker(method, methodAnnotation.uiThread()));
              break;
            }else if(anno instanceof WXModuleAnno) {
              WXModuleAnno methodAnnotation = (WXModuleAnno)anno;
              methodMap.put(method.getName(), new MethodInvoker(method,methodAnnotation.runOnUIThread()));
              break;
            }
          }
        }
      }
    } catch (Throwable e) {
      WXLogUtils.e("[WXModuleManager] extractMethodNames:", e);
    }
    // 生成所有方法的 Map
    mMethodMap = methodMap;
  }
```

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

## Module 方法调用流程

方法执行流程图：

```
├── WXBridge.callNativeModule
    ├── WXModuleManager.callModuleMethod
        ├── WXModuleManager.findModule
        ├── TypeModuleFactory.getMethodInvoker
        ├── WXModuleManager.dispatchCallModuleMethod
            ├── NativeInvokeHelper.invoke
                ├── NativeInvokeHelper.prepareArguments
                ├── MethodInvoker.isRunOnUIThread()
```    

```
  public Object invoke(final Object target,final Invoker invoker,JSONArray args) throws Exception {
    // 解析执行方法的参数
    final Object[] params = prepareArguments(
        invoker.getParameterTypes(),
        args);
    // 如果设置@JSMethod(uiThread = true)，那么在UI线程执行，返回值为null。
    if (invoker.isRunOnUIThread()) {
      WXSDKManager.getInstance().postOnUiThread(new Runnable() {
        @Override
        public void run() {
          try {
            // 在UI线程执行方法
            invoker.invoke(target, params);
          } catch (Exception e) {
            throw new RuntimeException(target + "Invoker " + invoker.toString() ,e);
          }
        }
      }, 0);
    } else {
      //设置@JSMethod(uiThread = false)的话走到这里
      return invoker.invoke(target, params);
    }
    return null;
  }
```

从这个方法我们可以看出，如果模块的自定义方法需要返回值，那么就只能设置 `@JSMethod(uiThread = false)`。相当于同步方法。

解析参数：
`paramClazzs` 是模块中定义的方法的参数类型数组，`args` 是JS传递过来的参数数组。

```
  private Object[] prepareArguments(Type[] paramClazzs, JSONArray args) throws Exception {
    Object[] params = new Object[paramClazzs.length];
    Object value;
    Type paramClazz;
    // 遍历参数
    for (int i = 0; i < paramClazzs.length; i++) {
      paramClazz = paramClazzs[i];
      // 确保args中的参数在模块方法定义的范围之内。
      if(i>=args.size()){
        if(!paramClazz.getClass().isPrimitive()) {
          params[i] = null;
          continue;
        }else {
          throw new Exception("[prepareArguments] method argument list not match.");
        }
      }
      //  从数组中获取单个参数，并放入params数组
      value = args.get(i);
      if (paramClazz == JSONObject.class) {
        // 如果是 JSONObject 类型
        if(value instanceof  JSONObject || value == null) {
          params[i] = value;
        }else if (value instanceof String){
          params[i] = JSON.parseObject(value.toString());
        }
      } else if(JSCallback.class == paramClazz){
        // 如果是 callback类型参数，这里的 value 其实是JS定义的callbackID。
        if(value instanceof String){
          // 根据sdkInstanceId和 callbackId 创建一个 SimpleJSCallback 实例对象
          params[i] = new SimpleJSCallback(mInstanceId,(String)value);
        }else{
          throw new Exception("Parameter type not match.");
        }
      } else {
        params[i] = WXReflectionUtils.parseArgument(paramClazz,value);
      }
    }
    return params;
  }
```

## callback执行流程

为了测试 callback 的执行流程，我们自定义了下面模块：

```
public class MyModule extends WXModule {
  @JSMethod(uiThread = true)
  public void testCallback(JSONObject options, JSCallback callback1, JSCallback callback2) {
    Log.e("Test","testCallback = "+options.toString());
    if(callback1 != null) {
      callback1.invoke(new Data("success","paraSuccess1","paraSuccess2"));
    }
    if(callback2 !=null){
      callback2.invoke("Test2");
    }
  }

  public static class Data{
    public String method;
    public Object paras;
    public Data(String method, Object... objArr){
      this.method = method;
      this.paras = objArr;
    }
  }
}
```

调用该模块：

```
        weex.requireModule('myModule').testCallback({
          type:"test", 
          data:2
          },
          function (e) {
            console.log("callback1");
          },
          function (e) {
            console.log("callback2");
          }
        );
```

下面来看一下 callback1 的调用流程：

```
├── SimpleJSCallback.invoke()
    ├── WXBridgeManager.callbackJavascript
        ├── WXBridgeManager.addJSTask()
            ├── WXBridgeManager.addJSTask()
                ├── WXBridgeManager.addJSEventTask()
        ├── sendMessage(WXJSBridgeMsgType.CALL_JS_BATCH)
        -------------------------------------------------
        ├── WXBridgeManager.handleMessage(WXJSBridgeMsgType.CALL_JS_BATCH)
            ├── WXBridgeManager.invokeCallJSBatch
                ├── WXJsonUtils.wsonWXJSObject(tasks)
                ├── WXBridgeManager.invokeExecJS
                    ├── WXBridge.execJS
```

在 `callbackJavascript()` 方法中，先通过 `addJSTask()` 方法在JS线程把执行回调函数的任务放到 `mNextTickTasks` 中，等待下次执行 `invokeCallJSBatch()` 时执行该任务。

```
  void callbackJavascript(final String instanceId, final String callback,
                          final Object data, boolean keepAlive) {
    if (TextUtils.isEmpty(instanceId) || TextUtils.isEmpty(callback)
        || mJSHandler == null) {
      return;
    }
    // 在JS线程把callback加入到执行队列中
    addJSTask(METHOD_CALLBACK, instanceId, callback, data, keepAlive);
    // 发送 CALL_JS_BATCH，执行一次批处理任务
    sendMessage(instanceId, WXJSBridgeMsgType.CALL_JS_BATCH);
  }
```

```
  private void addJSEventTask(final String method, final String instanceId, final List<Object> params, final Object... args) {
    post(new Runnable() {
      @Override
      public void run() {
        if (args == null || args.length == 0) {
          return;
        }
        // 把参数放到参数列表中，包括回调函数传递的 callbackid，回调参数（data 参数），keepAlive参数
        ArrayList<Object> argsList = new ArrayList<>();
        for (Object arg : args) {
          argsList.add(arg);
        }
        if (params != null) {
          ArrayMap map = new ArrayMap(4);
          map.put(KEY_PARAMS, params);
          argsList.add(map);
        }
        // 把要执行的方法类型和参数列表放到 map 中。
        WXHashMap<String, Object> task = new WXHashMap<>();
        task.put(KEY_METHOD, method);
        task.put(KEY_ARGS, argsList);

        // 把该callback放到和instanceId对应的列表中，等待下次批处理执行。
        if (mNextTickTasks.get(instanceId) == null) {
          ArrayList<WXHashMap<String, Object>> list = new ArrayList<>();
          list.add(task);
          mNextTickTasks.put(instanceId, list);
        } else {
          mNextTickTasks.get(instanceId).add(task);
        }
      }
    });
  }
```

```
  private void invokeCallJSBatch(Message message) {
    // 如果待执行队列中没有任务或者是未初始化framework则退出
    if (mNextTickTasks.isEmpty() || !isJSFrameworkInit()) {
      if (!isJSFrameworkInit()) {
        WXLogUtils.e("[WXBridgeManager] invokeCallJSBatch: framework.js uninitialized!!  message:" + message.toString());
      }
      return;
    }

    try {
      Object instanceId = message.obj;
      // 轮询队列中的所有instance，看对应是否有待执行任务
      ArrayList<WXHashMap<String, Object>> task = null;
      Stack<String> instanceStack = mNextTickTasks.getInstanceStack();
      int size = instanceStack.size();
      for (int i = size - 1; i >= 0; i--) {
        instanceId = instanceStack.get(i);
        task = mNextTickTasks.remove(instanceId);
        // 发现待执行任务则退出轮询流程
        if (task != null && !((ArrayList) task).isEmpty()) {
          break;
        }
      }
      Object[] tasks = ((ArrayList) task).toArray();
      // 解析参数：
      WXJSObject[] args = {
          new WXJSObject(WXJSObject.String, instanceId),
          WXJsonUtils.wsonWXJSObject(tasks)};
      // 执行方法
      invokeExecJS(String.valueOf(instanceId), null, METHOD_CALL_JS, args);
      task.clear();
      for(int i=0; i<tasks.length; i++){
        args[i] = null;
      }
      args = null;
    } catch (Throwable e) {
      WXLogUtils.e("WXBridgeManager", e);
      String err = "invokeCallJSBatch#" + WXLogUtils.getStackTrace(e);
	  WXExceptionUtils.commitCriticalExceptionRT(null, WXErrorCode.WX_ERR_JS_FRAMEWORK.getErrorCode(),
			  "invokeCallJSBatch", err, null);
    }

    // 如果还有待执行任务，则继续发CALL_JS_BATCH消息执行
    if (!mNextTickTasks.isEmpty()) {
      mJSHandler.sendEmptyMessage(WXJSBridgeMsgType.CALL_JS_BATCH);
    }
  }
```

解析参数：
WXJsonUtils.wsonWXJSObject(tasks)：

```
  public static final WXJSObject wsonWXJSObject(Object tasks){
    //CompatibleUtils.checkDiff(tasks);
    if(USE_WSON) {
      return new WXJSObject(WXJSObject.WSON, Wson.toWson(tasks));
    }else{
      return new WXJSObject(WXJSObject.JSON, WXJsonUtils.fromObjectToJSONString(tasks));
    }
  }
```

最后调用 `JSON.toJSONString(obj)` 方法把对象解析为 json 数据。
解析成 `WXJSObject` 类型的参数：
```
[{"args":["1",{"method":"success","paras":["paraSuccess1","paraSuccess2"]},false],"method":"callback"}]
```
<!-- 
http://weex.apache.org/cn/wiki/module-introduction.html
-->

