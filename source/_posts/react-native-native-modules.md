---
title: React Native创建原生模块
categories: React Native
comments: true
tags: [React Native]
description:
date: 2017-01-14 12:00:00
---
在React Native开发过程中，有时候我们可能需要访问平台的API，但React Native还没有相应的实现，或者是React Native还不支持一些原生的属性，我们需要调用原生代码来实现，或者是我们需要复用一些原来的Java代码，这个时候我们就需要创建一个原生模块来自己实现对我们需要功能的封装。
<!-- more -->
可以参考[官方文档](https://facebook.github.io/react-native/docs/native-modules-android.html)或[中文文档](http://reactnative.cn/docs/0.41/native-modules-android.html#content)。
## 开发模块
### 实现模块
下面我们就通过实现一个自定义模块，来熟悉编写原生模块需要用的一些知识。该模块主要实现调用一些Android原生的功能，比如弹`Toast`，启动`Activity`等。
我们首先来创建一个原生模块。一个原生模块是一个继承了 `ReactContextBaseJavaModule` 的Java类，它有一个必须实现的方法`getName()`，它返回一个字符串名字，在JS中我们就使用这个名字调用这个模块；还有构造函数`NativeModule`。
然后在这个类里面实现我们需要实现的方法：
```java

public class MyNativeModule extends ReactContextBaseJavaModule {
    private final static String MODULE_NAME = "MyNativeModule";
    private static final  String TestEvent = "TestEvent";
    private ReactApplicationContext mContext;
    public MyNativeModule(ReactApplicationContext reactContext) {
        super(reactContext);
        mContext = reactContext;
    }

    @Override
    public String getName() {
        return MODULE_NAME;
    }

    @Nullable
    @Override
    public Map<String, Object> getConstants() {
        final Map<String, Object> constants = new HashMap<>();
        constants.put("SHORT", Toast.LENGTH_SHORT);
        constants.put("LONG", Toast.LENGTH_LONG);
        constants.put("NATIVE_MODULE_NAME", MODULE_NAME);
        constants.put(TestEvent, TestEvent);
        return constants;
    }

    @ReactMethod
    public void startActivity(){
        Intent intent = new Intent(mContext,SecondActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        mContext.startActivity(intent);
    }

    @ReactMethod
    public void showToast(String msg, int duration){
        Toast.makeText(mContext, msg, duration).show();
    }
}
```
React Native调用的方法需要使用@ReactMethod注解。
方法的返回类型必须为void。
#### 参数类型
原生Java数据类型和JS数据类型的映射关系：
```
Boolean -> Bool
Integer -> Number
Double -> Number
Float -> Number
String -> String
Callback -> function
ReadableMap -> Object
ReadableArray -> Array
```
详情参考：[ReadableMap](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/bridge/ReadableMap.java)和[ReadableArray](https://github.com/facebook/react-native/blob/master/ReactAndroid/src/main/java/com/facebook/react/bridge/ReadableArray.java)
#### 导出常量
可以实现`getContants`方法导出需要给JavaScript使用的常量。
```java
    @Nullable
    @Override
    public Map<String, Object> getConstants() {
        final Map<String, Object> constants = new HashMap<>();
        constants.put("SHORT", Toast.LENGTH_SHORT);
        constants.put("LONG", Toast.LENGTH_LONG);
        constants.put("NATIVE_MODULE_NAME", MODULE_NAME);
        constants.put(TestEvent, TestEvent);
        return constants;
    }
```
### 注册模块
然后我还要注册这个模块，通过实现`ReactPackage`接口来实现：
```java
public class MyReactPackage implements ReactPackage {
    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        List<NativeModule> modules = new ArrayList<>();
        modules.add(new MyNativeModule(reactContext));
        return modules;
    }

    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }
}
```
### 添加模块
在`Application`的`getPackages()`方法中添加模块：
```java
        @Override
        protected List<ReactPackage> getPackages() {
            return Arrays.<ReactPackage>asList(
                    new MainReactPackage(), new MyReactPackage()
            );
        }
```
或者是在`Activity`的`onCreate`中：
```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mReactRootView = new ReactRootView(this);
        mReactInstanceManager = ReactInstanceManager.builder()
                .setApplication(getApplication())
                .setBundleAssetName("index.android.bundle")
                .setJSMainModuleName("index.android")
                .addPackage(new MainReactPackage())
                .addPackage(new MyReactPackage())
                .setUseDeveloperSupport(BuildConfig.DEBUG)
                .setInitialLifecycleState(LifecycleState.RESUMED)
                .build();
        mReactRootView.startReactApplication(mReactInstanceManager, "HelloWorld", null);
        setContentView(mReactRootView);
    }
```
### 封装模块
为了使JavaScript端访问起来更为方便，通常我们都会把原生模块封装成一个JavaScript模块。在`index.android.js`文件的同一目录下面创建一个MyNativeModule.js。
```javascript
'use strict';

import { NativeModules } from 'react-native'; 

// 这里的MyNativeModule必须对应
// public String getName()中返回的字符串

export default NativeModules.MyNativeModule;
```
### 调用模块
现在，在别处的JavaScript代码中可以这样调用你的方法：
```javascript
import MyNativeModule from './MyNativeModule'; 
class HelloWorld extends React.Component {
  startActivity(){
    console.log("MODULE NAME: ",MyNativeModule.NATIVE_MODULE_NAME);
    MyNativeModule.startActivity();
  }
  showToast(){
    console.log("MODULE NAME: ",MyNativeModule.NATIVE_MODULE_NAME);
    MyNativeModule.showToast("From JS", MyNativeModule.LONG);
  }
  render() {
    return (
      <View style={styles.container}>
        <TouchableOpacity onPress={this.startActivity}>  
          <Text style={styles.hello}>start Activity</Text>  
        </TouchableOpacity>
      </View>
    )
  }
}
```
## 其他特性
React Native的跨语言访问是异步进行的，所以想要给JavaScript返回一个值的唯一办法是使用回调函数或者发送事件。
### 回调函数
原生模块还支持一种特殊的参数——回调函数。它提供了一个函数来把返回值传回给JS。
```java
    @ReactMethod
    public void testCallback(int para1, int para2, Callback resultCallback){
        int result = para1 + para2;
        resultCallback.invoke(result);
    }
```
可以在JS中调用：
```javascript
  testCallback(){
    MyNativeModule.testCallback(100,100,(result) => {
    console.log("result: ",result); //'result: ', 200
    });
  }
```
原生模块通常只应调用回调函数一次。但是，它可以保存callback并在将来调用。
callback并非在对应的原生函数返回后立即被执行——注意跨语言通讯是异步的，这个执行过程会通过消息循环来进行。
### 发送事件到JavaScript
原生模块可以在没有被调用的情况下往JavaScript发送事件通知。最简单的办法就是通过`RCTDeviceEventEmitter`，这可以通过`ReactContext`来获得对应的引用：
```java
    public void sendEvent(){
        WritableMap params = Arguments.createMap();
        params.putString("module", "MyNativeModule");
        mContext
                .getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter.class)
                .emit(TestEvent, params);
    }
```
在JS中这样调用：
```javascript
import { DeviceEventEmitter } from 'react-native';
......
  componentWillMount() {
    console.log("componentWillMount");
    //接收事件
    DeviceEventEmitter.addListener(MyNativeModule.TestEvent, info => {
      console.log(info);
    });
  }
```
### Promise
如果对ES6的`Promise`对象不太熟悉的话，可以点[这里](http://es6.ruanyifeng.com/#docs/promise)进行了解。
原生模块还可以使用`promise`来简化代码，搭配ES2016(ES7)标准的`async/await`语法则效果更佳。如果桥接原生方法的最后一个参数是一个`Promise`，则对应的JS方法就会返回一个`Promise`对象。
`Promise`是异步编程的一种解决方案，比传统的解决方案（回调函数和事件）更合理和更强大。
```java
    @ReactMethod
    public void testPromise(Boolean isResolve, Promise promise) {
        if(isResolve) {
            promise.resolve(isResolve.toString());
        }
        else {
            promise.reject(isResolve.toString());
        }
    }
```
在JS中调用：
```javascript
  testPromise(){
    MyNativeModule.testPromise(true)
    .then(result => {
      console.log("result1 is ", result);
    })
    .catch(result => {
      console.log("result2 is ", result);
    });
  }
```
这里可以用`then`方法分别指定`Resolved`状态和`Reject`状态的回调函数。第一个回调函数是`Promise`对象的状态变为`Resolved`时调用，第二个回调函数是`Promise`对象的状态变为`Reject`时调用。其中，第二个函数是可选的，不一定要提供。
`catch`方法用于指定发生错误时的回调函数。
结果：`'result1 is ', 'true'`

### 从startActivityForResult中获取结果
参考[官方文档](http://reactnative.cn/docs/0.41/native-modules-android.html#content)
### 监听生命周期
监听activity的生命周期事件（比如`onResume`, `onPause`等等），模块必须实现`LifecycleEventListener`，然后需要在构造函数中注册一个监听函数：
```java
    public MyNativeModule(ReactApplicationContext reactContext) {
        super(reactContext);
        mContext = reactContext;
        //添加监听
        reactContext.addLifecycleEventListener(this);
    }
```
实现`LifecycleEventListener`的几个接口：
```java
    @Override
    public void onHostResume() {
        Log.e(MODULE_NAME, "onHostResume");
    }

    @Override
    public void onHostPause() {
        Log.e(MODULE_NAME, "onHostPause");
    }

    @Override
    public void onHostDestroy() {
        Log.e(MODULE_NAME, "onHostDestroy");
    }
```
然后就可以监听ReactNative应用的生命周期了。

