---
title: Android项目中集成React Native
categories: React Native
comments: true
tags: [React Native]
description:
date: 2017-01-14 10:00:00
---
React Native是非常强大的，但有的时候我们可能并不需要从0开始去开发一个React Native应用，而是需要把它集成到我们现有的Android工程中去，去添加单个的React Native View。本章将主要介绍在原生Android中集成React Native。
<!-- more -->
可以参考[官方文档](http://facebook.github.io/react-native/releases/next/docs/integration-with-existing-apps.html)或[中文文档](http://reactnative.cn/docs/0.41/integration-with-existing-apps.html#content)。
## 创建Android工程
新建一个ReactNativeDemo的Android工程作为已有的工程。这部分步骤略过……

## 集成 React Native
### 添加JS到App中
进入工程根目录执行以下命令：
```
npm init
npm install --save react react-native
curl -o .flowconfig https://raw.githubusercontent.com/facebook/react-native/master/.flowconfig

```
`npm init`命令是根据提示生成 package.json 文件的。
```
This utility will walk you through creating a package.json file.
It only covers the most common items, and tries to guess sensible defaults.

See `npm help json` for definitive documentation on these fields
and exactly what they do.

Use `npm install <pkg> --save` afterwards to install a package and
save it as a dependency in the package.json file.

Press ^C at any time to quit.
name: (ReactNativeDemo) reactnativedemo
version: (1.0.0) 1.0.0
description: react native app demo
entry point: (index.js) index.android.js
test command: 
git repository: 
keywords: 
author: hq
license: (ISC) 
About to write to /home/heqiang/react-native-workspace/ReactNativeDemo/package.json:

{
  "name": "reactnativedemo",
  "version": "1.0.0",
  "description": "react native app demo",
  "main": "index.android.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "hq",
  "license": "ISC"
}


Is this ok? (yes)
```
`npm install --save react react-native`命令是安装React Native依赖的模块，在根目录下生成`node_modules`。这个下载过程比较慢，如果其他工程下面有，复制过来也是可以的。
`curl`用来下载.flowconfig文件

在`package.json`文件中添加下面语句：
```
"start": "node node_modules/react-native/local-cli/cli.js start"
```
现在的`package.json`是这样的：
```
{
  "name": "reactnativedemo",
  "version": "1.0.0",
  "description": "react native app demo",
  "main": "index.android.js",
  "scripts": {
    "start": "node node_modules/react-native/local-cli/cli.js start",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "hq",
  "license": "ISC",
  "dependencies": {
    "react": "^15.4.2",
    "react-native": "^0.41.2"
  }
}
```

创建`index.android.js`文件：
```
'use strict';
import React from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View
} from 'react-native';
class HelloWorld extends React.Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.hello}>Hello, World</Text>
      </View>
    )
  }
}
var styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
  },
  hello: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
});
AppRegistry.registerComponent('HelloWorld', () => HelloWorld);
```

### 工程配置
在App的`build.gradle`中加入以下依赖：
```
dependencies {
 ...
 compile "com.facebook.react:react-native:+" // From node_modules.
}
```
在工程的`build.gradle`中加入以下配置：
```
allprojects {
 repositories {
     ...
     maven {
         // All of React Native (JS, Android binaries) is installed from npm
         url "$rootDir/node_modules/react-native/android"
     }
 }
 ...
}
```
如果sync过程中出现下面的错误：
```
Error:Execution failed for task ':app:prepareDebugAndroidTestDependencies'.
```
参考[这里](http://stackoverflow.com/questions/30558737/execution-failed-for-task-apppreparedebugandroidtestdependencies)，在App的`build.gradle`中加入：
```
configurations.all {
    resolutionStrategy.force 'com.google.code.findbugs:jsr305:3.0.1'
}
```
在`AndroidManifest.xml`中加入网络权限：
```
<uses-permission android:name="android.permission.INTERNET" />
```
### 添加Native Code
下面将修改原生代码，实现嵌入React Native功能。
#### 方法一
官方文档的做法，首先就是Activity实现DefaultHardwareBackBtnHandler接口：
```
public class MainActivity extends AppCompatActivity implements DefaultHardwareBackBtnHandler {

    private ReactRootView mReactRootView;
    private ReactInstanceManager mReactInstanceManager;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mReactRootView = new ReactRootView(this);
        mReactInstanceManager = ReactInstanceManager.builder()
                .setApplication(getApplication())
                .setBundleAssetName("index.android.bundle")
                .setJSMainModuleName("index.android")
                .addPackage(new MainReactPackage())
                .setUseDeveloperSupport(BuildConfig.DEBUG)
                .setInitialLifecycleState(LifecycleState.RESUMED)
                .build();
        mReactRootView.startReactApplication(mReactInstanceManager, "HelloWorld", null);
        setContentView(mReactRootView);
    }

    @Override
    public void invokeDefaultOnBackPressed() {
        super.onBackPressed();
    }
}
```
#### 方法二
新版本中可以用下面的方法，更简单：
```
public class MainActivity extends ReactActivity{

    /**
     * Returns the name of the main component registered from JavaScript.
     * This is used to schedule rendering of the component.
     */
    @Nullable
    @Override
    protected String getMainComponentName() {
        return "HelloWorld"; //这个是在AppRegistry.registerComponent里面注册的。
    }
}
```
但这个方法需要我们自定义一个Application否则 运行时会报错：
```
java.lang.RuntimeException: Unable to start activity ComponentInfo{com.android.hq.reactnativedemo/com.android.hq.reactnativedemo.MainActivity}: java.lang.ClassCastException: android.app.Application cannot be cast to com.facebook.react.ReactApplication
```
Application的代码：
```
public class Application extends android.app.Application implements ReactApplication  {

    private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {

        /**
         * 是否支持开发模式，如果支持就返回true，此时需要打开悬浮窗权限。
         * 不支持的话就返回false
         * @return
         */
        @Override
        public boolean getUseDeveloperSupport() {
            return BuildConfig.DEBUG;
        }

        @Override
        protected List<ReactPackage> getPackages() {
            return Arrays.<ReactPackage>asList(
                    new MainReactPackage()
            );
        }
    };

    @Override
    public ReactNativeHost getReactNativeHost() {
        return mReactNativeHost;
    }
}
```
还要记得修改AndroidManifest.xml。

### 运行
根目录运行下面命令：
```
adb reverse tcp:8081 tcp:8081
nmp start
```
启动server。
然后点击Android Studio的运行按钮：
如果运行时报下面的错误：
```
02-22 19:18:59.050 24413 24413 E AndroidRuntime: java.lang.UnsatisfiedLinkError: dlopen failed: "/data/data/com.android.hq.reactnativedemo/lib-main/libgnustl_shared.so" is 32-bit instead of 64-bit
```
可以参考[这里](http://blog.csdn.net/ssksuke/article/details/52611689)或者[这里](https://corbt.com/posts/2015/09/18/mixing-32-and-64bit-dependencies-in-android.html)的解决方案，在App的`build.gradle`里面添加：
```
    defaultConfig {
        ……
        ndk {
            abiFilters "armeabi-v7a", "x86"
        }
    }
```
再次运行，`Hello World`界面运行起来了。
另外记得去应用管理界面打开应用的悬浮窗权限，方便调试。

### 生成Realease版本
在前面的App运行前，我们需要执行两条命令来启动development server，这在开发环境中可以这么做，但Realese版本这么做肯定是不行的。
下面就来生成Realease版本。
#### 方法一
在`app/src/main/`中新建`assets`，根目录下执行下面命令：
```
react-native bundle --platform android --dev false --entry-file index.android.js --bundle-output app/src/main/assets/index.android.bundle --assets-dest app/src/main/res/
```
会在`assets`目录中生成`index.android.bundle`和`index.android.bundle.meta`文件。`index.android.bundle`文件是所有的React Native js文件打包生成的一个js文件，`index.android.bundle.meta`中存储的是bundle的sha1值，每次打包都会生成一个meta唯一标识bundle
再次编译运行App，不用启动启动服务App就可以正常运行了。
#### 方法二
其实也可以通过复制React Native Server里面bundle文件的方法来实现。
在根目录执行以下命令：
```
curl "http://localhost:8081/index.android.bundle?platform=android" -o "app/src/main/assets/index.android.bundle"
```
也是可以的。

