---
title: 搭建 React Native 开发环境
categories: React Native
comments: true
tags: [React Native]
description: 介绍在Ubuntu环境下搭建 React Native 开发环境
date: 2017-01-10 12:00:00
---
本文是对Ubuntu环境下开发环境的搭建
<!-- more -->
## Android Studio
下载Android Studio2.2，React Native目前需要Android Studio2.0或更高版本: [下载地址](http://www.androiddevtools.cn/)
下载android-sdk；
这部分相信Android开发者都懂，不做详细介绍。

## 安装nodeJS
下载node-v5.0.0-linux-x64解压即可； 
[下载地址](https://nodejs.org/en/download/)
创建软链接
```bash
sudo ln -s /XXXXXX/tools/node-v5.0.0-linux-x64/bin/node /usr/bin/node
sudo ln -s /XXXXXX/tools/node-v5.0.0-linux-x64/bin/npm /usr/bin/npm
sudo npm install -g yarn react-native-cli
sudo ln -s /home/heqiang/install/tools/node-v5.0.0-linux-x64/bin/react-native /usr/bin/react-native
sudo ln -s /home/heqiang/install/tools/node-v5.0.0-linux-x64/bin/yarn /usr/bin/yarn
sudo ln -s /home/heqiang/install/tools/node-v5.0.0-linux-x64/bin/yarnpkg /usr/bin/yarnpkg
```
 - `react-native-cli`是React Native的一个命令行工具，用于执行创建、初始化、更新项目、运行打包服务（packager）等任务。

## 开始创建第一个工程
```bash
react-native init AwesomeProject
cd AwesomeProject/
```
连接好手机
```bash
react-native run-android
```
发现下面的错误：
```
heqiang@EF-heqiang:~/react-native-workspace/AwesomeProject$ react-native run-android
Starting JS server...
Running adb -s M960ADPBB7C7M reverse tcp:8081 tcp:8081
Building and installing the app on the device (cd android && ./gradlew installDebug...
Failed to notify ProjectEvaluationListener.afterEvaluate(), but primary configuration failure takes precedence.
java.lang.RuntimeException: SDK location not found. Define location with sdk.dir in the local.properties file or with an ANDROID_HOME environment variable.
	at com.android.build.gradle.internal.SdkHandler.getAndCheckSdkFolder(SdkHandler.java:102)
	at com.android.build.gradle.internal.SdkHandler.getSdkLoader(SdkHandler.java:112)
	at com.android.build.gradle.internal.SdkHandler.initTarget(SdkHandler.java:86)
	at com.android.build.gradle.BasePlugin.ensureTargetSetup(BasePlugin.groovy:507)
	at com.android.build.gradle.BasePlugin.createAndroidTasks(BasePlugin.groovy:455)
	at com.android.build.gradle.BasePlugin$_createTasks_closure13_closure17.doCall(BasePlugin.groovy:415)
	at com.android.build.gradle.BasePlugin$_createTasks_closure13_closure17.doCall(BasePlugin.groovy)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
```
按他说的去配置local.properties的adk.dir
添加
```bash
sdk.dir=/home/heqiang/install/android-studio/android-sdk-linux
```
再次启动，OK了。
这个时候你发现应用没有启动起来，需要进入应用管理，把悬浮窗权限打开。
终于见到启动界面了，只不过是这种。
![react demo](/images/react-native-build-development-environment/image1.png)
配置一下网络环境：
(Android 5.0及以上)使用adb reverse命令
> 注意，这个选项只能在5.0以上版本(API 21+)的安卓设备上使用。

 1. 首先把你的设备通过USB数据线连接到电脑上，并开启USB调试。
 2. 运行adb reverse tcp:8081 tcp:8081。（每次连接usb都要运行一下）
 3.  不需要更多配置，你就可以使用Reload JS和其它的开发选项了。


(Android 5.0以下)通过Wi-Fi连接你的本地开发服务器

 1. 首先确保你的电脑和手机设备在同一个Wi-Fi环境下。
 2. 在设备上运行你的React Native应用。和打开其它App一样操作。
 3. 你应该会看到一个“红屏”错误提示。这是正常的，下面的步骤会解决这个报错。
 4. 摇晃设备，或者运行adb shell input keyevent 82，可以打开开发者菜单。
 5. 点击进入Dev Settings。
 6. 点击Debug server host for device。
 7. 输入你电脑的IP地址和端口号（譬如10.0.1.1:8081）。在Mac上，你可以在系统设置/网络里找查询你的IP地址。在Windows上，打开命令提示符并输入ipconfig来查询你的IP地址。在Linux上你可以在终端中输入ifconfig来查询你的IP地址。
 8. 回到开发者菜单然后选择Reload JS。 

由于我的环境是Android 5.0以上，所以选择第一种方式。
还是不行，在电脑pc上输入一下网址检查一下：
http://172.17.137.68:8081/index.android.bundle?platform=android&dev=true&hot=false&minify=false
果然是打不开。
哦，应该是server没有启动起来，启动server了，电脑上启动 react-native start。
```
Scanning 716 folders for symlinks in /home/heqiang/react-native-workspace/AwesomeProject/node_modules (8ms)
 ┌────────────────────────────────────────────────────────────────────────────┐ 
 │  Running packager on port 8081.                                            │ 
 │                                                                            │ 
 │  Keep this packager running while developing on any JS projects. Feel      │ 
 │  free to close this tab and run your own packager instance if you          │ 
 │  prefer.                                                                   │ 
 │                                                                            │ 
 │  https://github.com/facebook/react-native                                  │ 
 │                                                                            │ 
 └────────────────────────────────────────────────────────────────────────────┘ 
Looking for JS files in
   /home/heqiang/react-native-workspace/AwesomeProject 

[Hot Module Replacement] Server listening on /hot

React packager ready.

[2016-12-07 09:45:21] <START> Initializing Packager
[2016-12-07 09:45:21] <START> Building in-memory fs for JavaScript
[2016-12-07 09:45:21] <END>   Building in-memory fs for JavaScript (71ms)
[2016-12-07 09:45:21] <START> Building Haste Map
[2016-12-07 09:45:21] <END>   Building Haste Map (226ms)
[2016-12-07 09:45:21] <END>   Initializing Packager (431ms)
[2016-12-07 09:45:40] <START> Requesting bundle bundle_url: /index.Android.bundle?platform=android&dev=true&hot=false&minify=false
[2016-12-07 09:45:40] <START> Transforming files
```
点击一下RELOAD，有东西显示了，可以对React Native Say Hello了！
![react demo](/images/react-native-build-development-environment/image2.png)
这个时候可以打开JS，index.android.js，随便写点什么，然后摇一摇，点击RELOAD就可以显示出来了。
**总结**：运行一个工程需要的步骤：
 1. `react-native start`启动React packager
 2. `react-native run-android`
 3. `adb reverse tcp:8081 tcp:8081`（Android5.0以上）
 4. 打开悬浮窗权限

## 开发工具
### Sublime Text 3
#### 安装
下载安装包：https://www.sublimetext.com/
下载完成后点击，会打开Ubuntu软件中心进行安装，然后终端打开。
```
subl
```
![subl](/images/react-native-build-development-environment/image3.png)
版本3126
但是会显示显示unregistered字样。
破解方法：参考https://my.oschina.net/u/574928/blog/505481
复制文中的注册码，subl中打开Help->enter license粘贴即可。
#### 导入项目
点击菜单栏的“Project”-->"Add Folder to Project" ，选择项目的目录，就将项目导入进来了。
#### 安装插件

 1. 点击菜单栏的“Preferences”-->"Package Control",或者可以使用快捷键CTRL+SHIFT+P打开。如果找不到Package Control，那么打开Tools-->Install Package Control就可以了。
 2. 在打开的终端窗口，输入“install”,下方就会提示“Package Control:install package”,用鼠标点击。这时候等待几秒，就会弹出一个终端，在终端输入你想要安装的插件，进行查找，点击下方列表中插件，就会自动会为你安装了。

React Native开发推荐的一些插件：

 1. ReactJS : 支持React开发，代码提示，高亮显示。[介绍](https://github.com/facebookarchive/sublime-react "介绍")
 2. Emmet 前端开发必备。

> **功能：** jsx 文件中快速通过 emmet 编写自定义组件。
> **安装：** PC上ctrl+shift+p（MacCmd+shift+p）打开面板输入emmet安装
> **配置：** 打开 preferences -> Key bindings - Users，先用[]扩展为数组，然后把下面代码复制到[]内部。
```json
{
      "keys": [
        "super+e"
      ],
      "args": {
        "action": "expand_abbreviation"
      },
      "command": "run_emmet_action",
      "context": [{
        "key": "emmet_action_enabled.expand_abbreviation"
      }]
    },
    {
      "keys": ["tab"],
      "command": "expand_abbreviation_by_tab",
      "context": [{
        "operand": "source.js",
        "operator": "equal",
        "match_all": true,
        "key": "selector"
      }, {
        "key": "preceding_text",
        "operator": "regex_contains",
        "operand": "(\\b(a\\b|div|span|p\\b|button)(\\.\\w*|>\\w*)?([^}]*?}$)?)",
        "match_all": true
      }, {
        "key": "selection_empty",
        "operator": "equal",
        "operand": true,
        "match_all": true
      }]
    }
```
使用super+e触发 emmet，自动补齐；
 3. Terminal : 在sublime中打开终端并定位到当前目录，神器。快捷键：command+shift+T
 4. react-native-snippets：react native 的代码片。[介绍](https://github.com/Shrugs/react-native-snippets "介绍")
 5. Babel
 6. ESLint
 
> **功能：** 支持ES6和React JSX语法定义，我一般用它替代Sublime自带的js语法定义。
> **安装：** PC上ctrl+shift+p（MacCmd+shift+p）打开面板输入Babel安装.
> **配置：** 打开.js, .jsx 后缀的文件;打开菜单view-> Syntax -> Open all with current extension as… -> Babel -> JavaScript (Babel)，选择babel为默认 javascript 打开syntax,即可看到对react的语法高亮。

Sublime Text 3的使用可以参考下面的文章：
如何优雅地使用Sublime Text3： http://www.jianshu.com/p/3cb5c6f2421c
http://www.jianshu.com/p/2ddfff095e90

### VisualStudio Code
相当强大的IDE工具，推荐使用。
#### 下载安装
地址：https://code.visualstudio.com/
#### 插件

 1. ESLint js,jsx,es6代码语法检测
 2. React Native Tools 微软官方出的ReactNative插件
 3. File Navigator Ctrl + l 导航文件
 4. Auto Close Tag 写完开始标签，会自动补全结束标签
 5. Auto Rename Tag 要修改标签名称的时候自动修改结束标签
 6. vscode-icons 不同的问题有不同的图标标识
 7. vscode-fileheader 支持头部注释,自动更新文件更新时间

#### 调试
http://mp.weixin.qq.com/s?__biz=MjM5NDkxMTgyNw==&mid=2653058224&idx=1&sn=553b4ce24b22680d8f46cba082eb8661&scene=21#wechat_redirect
#### 配置 
http://www.jianshu.com/p/1d3f9cdd15dc

## 参考文章
http://reactnative.cn/docs/0.39/getting-started.html
http://reactnative.cn/docs/0.39/running-on-device-android.html#content

