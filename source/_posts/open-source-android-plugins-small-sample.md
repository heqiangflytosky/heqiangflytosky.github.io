---
title: Android 插件化框架Small解析--开始了解Small
categories: Android 插件化
comments: true
tags: [Android开源项目, Android 插件化, Small]
description: 根据官方提供的例子初步了解Small的用法
date: 2017-5-8 10:00:00
---

## 概述
Small是一个非常简洁的插件化框架，它的口号是做最轻巧的跨平台插件化框架。
[Small 官方使用文档](http://code.wequick.net/Small/cn/home)
[Small Github源码地址](https://github.com/wequick/Small)
那么首先我们从官方给出的 Sample 中初步了解一下Small框架。

## 运行DevSample
从上面给出的Small Github源码地址中现在Small源码。
在 Small/Android/ 下面有两个目录：

 - Sample/：使用者模式
 - DevSample/：开发者模式

他们有什么区别呢？可以看他们编译后的结果：
Sample：

```
| type |      name      |  PP  | sdk |  aapt  | support | file(armeabi)  |   size   |
|------|----------------|------|-----|--------|---------|----------------|----------|
| host | app            |      | 25  | 25.0.2 | 25.1.0  |                |          |
| stub | app+stub       |      | 25  | 25.0.2 | 25.1.0  |                |          |
| app  | app.main       | 0x77 | 25  | 25.0.2 | 25.1.0  | *_main.so      | 11.7 KB  |
| app  | app.mine       | 0x16 | 25  | 25.0.2 |         | *_mine.so      | 47.6 KB  |
| app  | app.ok-if-stub | 0x6a | 25  | 25.0.2 |         | *_stub.so      | 19.7 KB  |
| app  | app.detail     | 0x67 | 25  | 25.0.2 | 25.1.0  | *_detail.so    | 7.4 KB   |
| app  | app.home       | 0x70 | 25  | 25.0.2 |         | *_home.so      | 11.3 KB  |
| lib  | lib.analytics  | 0x76 | 25  | 25.0.2 |         | *_analytics.so | 126.6 KB |
| lib  | lib.utils      | 0x73 | 25  | 25.0.2 | 25.1.0  | *_utils.so     | 6.8 KB   |
| lib  | lib.style      | 0x79 | 25  | 25.0.2 | 25.1.0  | *_style.so     | 5.5 KB   |
| web  | web.about      |      | 25  | 25.0.2 | 25.1.0  | *_about.so     | 24.3 KB  |

```

DevSample：

```
| type |      name      |  PP  | sdk |  aapt  | support |        file         |   size   |
|------|----------------|------|-----|--------|---------|---------------------|----------|
| host | app            |      | 25  | 25.0.2 | 25.1.0  |                     |          |
| stub | app+stub       |      | 25  | 25.0.2 | 25.1.0  |                     |          |
| app  | app.main       | 0x77 | 25  | 25.0.2 | 25.1.0  | *.main.apk          | 11.7 KB  |
| app  | app.mine       | 0x16 | 25  | 25.0.2 |         | *.mine.apk          | 47.6 KB  |
| app  | app.ok-if-stub | 0x6a | 25  | 25.0.2 |         | *.appok_if_stub.apk | 19.7 KB  |
| app  | app.detail     | 0x67 | 25  | 25.0.2 | 25.1.0  | *.detail.apk        | 7.4 KB   |
| app  | app.home       | 0x70 | 25  | 25.0.2 |         | *.home.apk          | 11.3 KB  |
| lib  | lib.afterutils | 0x45 | 25  | 25.0.2 | 25.1.0  | *.afterutils.apk    | 3.7 KB   |
| lib  | lib.analytics  | 0x76 | 25  | 25.0.2 |         | *.analytics.apk     | 126.6 KB |
| lib  | lib.utils      | 0x73 | 25  | 25.0.2 | 25.1.0  | *.utils.apk         | 6.8 KB   |
| lib  | lib.style      | 0x79 | 25  | 25.0.2 | 25.1.0  | *.style.apk         | 5.5 KB   |
| web  | web.about      |      | 25  | 25.0.2 | 25.1.0  | *.about.apk         | 24.3 KB  |

```

编译后的结果 Sample 是 *.so，DevSample 是 *.apk。
造成这样的区别原因是在开发者模式的 build.gradle 里面有下面的设置：

```
small {
    buildToAssets = true
    ...
}
```

`buildToAssets` 决定是否将插件作为 apk 文件打包到宿主 apk 的 assets 目录下。默认为 false，即作为 so 文件打包到宿主 apk 的 lib 目录下。
配套的，还需要在宿主的 `Application` 里指定读取插件的位置：

```java
@Override
public void onCreate() {
    super.onCreate();
    ...
    Small.setLoadFromAssets(BuildConfig.LOAD_FROM_ASSETS);
}
```

```java
                if (Small.isLoadFromAssets()) {
                    mBuiltinAssetName = pkg + ".apk";
                    mBuiltinFile = new File(FileUtils.getInternalBundlePath(), mBuiltinAssetName);
                    mPatchFile = new File(FileUtils.getDownloadBundlePath(), mBuiltinAssetName);
                    // Extract from assets to files
                    try {
                        extractBundle(mBuiltinAssetName, mBuiltinFile);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                } else {
                    String soName = "lib" + pkg.replaceAll("\\.", "_") + ".so";
                    mBuiltinFile = new File(sUserBundlesPath, soName);
                    mPatchFile = new File(FileUtils.getDownloadBundlePath(), soName);
                }
```

可以看到，`BuildConfig.LOAD_FROM_ASSETS` 为 true 时，从 /data/data/<packageName>/small_base 下面读取 apk 格式插件，为 false 时从 `nativeLibraryDir` 中读取 so 格式插件。

那么现在我们就导入 DevSample 工程。

### 导入DevSample工程

打开Android Studio，File->Open->Open File or Project... 选择DevSample文件夹然后打开。
如图：

![效果图](/images/open-source-android-plugins-small-sample/small-dev-sample-import-project.png)

DevSample 示例工程中有下面的模块：

 - app 宿主工程
 - app.* 应用插件，包含Activity/Fragment的插件
 - lib.* 公共库插件
 - web.* 本地网页插件
 - app+* [宿主分身模块](http://code.wequick.net/Small/cn/stub-module)
 - gradle-small Small中的一个gradle自定义插件，用于打包组件

出于业务需求考虑，Small定义了两类插件：公共库插件与应用插件。
*应用插件*相对简单，就是用来把大应用拆分成一个个小的业务单元。而*公共库插件*则是为这些业务单元提供公共的代码与资源，比如可以将在多个应用插件间可以复用的一些主题、界面边距资源提取出来作为一个公共库插件。

### 编译插件

在运行前必须要编译生成插件。

 1. 编译公共库插件：

指的是编译lib.*的插件。

```
./gradlew buildLib -q (-q是安静模式，可以让输出更好看，也可以不加)
```

输出：

```
  Small building library 1 of 5 - app (0x7f)
  Small building library 2 of 5 - lib.utils (0x73)
        [lib.utils] split library res files...                          [  OK  ]
        [lib.utils] slice asset package and reset package id...         [  OK  ]
        [lib.utils] split library R.java files...                       [  OK  ]
        [lib.utils] add flags: 10000...                                 [  OK  ]
        [lib.utils] split R.class...                                    [  OK  ]
        -> smallLibs/net.wequick.example.small.lib.utils.apk (7004 bytes = 6.8 KB)
  Small building library 3 of 5 - lib.afterutils (0x45)
        [lib.afterutils] remove resources dir...                        [  OK  ]
        [lib.afterutils] remove resources.arsc...                       [  OK  ]
        [lib.afterutils] remove R.java...                               [  OK  ]
        [lib.afterutils] add flags: 1...                                [  OK  ]
        -> smallLibs/net.wequick.example.small.lib.afterutils.apk (3744 bytes = 3.7 KB)
  Small building library 4 of 5 - lib.analytics (0x76)
        [lib.analytics] remove resources dir...                         [  OK  ]
        [lib.analytics] remove resources.arsc...                        [  OK  ]
        [lib.analytics] remove R.java...                                [  OK  ]
        [lib.analytics] add flags: 1...                                 [  OK  ]
        -> smallLibs/net.wequick.example.lib.analytics.apk (129638 bytes = 126.6 KB)
  Small building library 5 of 5 - lib.style (0x79)
        [lib.style] split library res files...                          [  OK  ]
        [lib.style] slice asset package and reset package id...         [  OK  ]
        [lib.style] split library R.java files...                       [  OK  ]
        [lib.style] split R.class...                                    [  OK  ]
        -> smallLibs/com.example.mysmall.lib.style.apk (5603 bytes = 5.5 KB)
```

 2. 编译应用插件

指的是编译app.*的插件。

```
./gradlew buildBundle -q (-q是安静模式，可以让输出更好看，也可以不加)
```

输出：

```
  Small building bundle 1 of 6 - app.detail (0x67)
        [app.detail] split library res files...                         [  OK  ]
        [app.detail] slice asset package and reset package id...        [  OK  ]
        [app.detail] split library R.java files...                      [  OK  ]
        [app.detail] split R.class...                                   [  OK  ]
        -> smallLibs/net.wequick.example.small.app.detail.apk (7616 bytes = 7.4 KB)
  Small building bundle 2 of 6 - app.home (0x70)
        [app.home] split library res files...                           [  OK  ]
        [app.home] slice asset package and reset package id...          [  OK  ]
        [app.home] split library R.java files...                        [  OK  ]
        [app.home] split R.class...                                     [  OK  ]
        -> smallLibs/net.wequick.example.small.app.home.apk (11582 bytes = 11.3 KB)
  Small building bundle 3 of 6 - app.main (0x77)
        [app.main] split library res files...                           [  OK  ]
        [app.main] slice asset package and reset package id...          [  OK  ]
        [app.main] split library R.java files...                        [  OK  ]
        [app.main] add flags: 10000...                                  [  OK  ]
        [app.main] split R.class...                                     [  OK  ]
        -> smallLibs/net.wequick.example.small.app.main.apk (12017 bytes = 11.7 KB)
  Small building bundle 4 of 6 - app.mine (0x16)
        [app.mine] split library res files...                           [  OK  ]
        [app.mine] slice asset package and reset package id...          [  OK  ]
        [app.mine] split library R.java files...                        [  OK  ]
        [app.mine] add flags: 11111110...                               [  OK  ]
        [app.mine] split R.class...                                     [  OK  ]
        -> smallLibs/net.wequick.example.small.app.mine.apk (48747 bytes = 47.6 KB)
  Small building bundle 5 of 6 - app.ok-if-stub (0x6a)
        [app.ok-if-stub] split library res files...                     [  OK  ]
        [app.ok-if-stub] slice asset package and reset package id...    [  OK  ]
        [app.ok-if-stub] split library R.java files...                  [  OK  ]
        [app.ok-if-stub] split R.class...                               [  OK  ]
        -> smallLibs/net.wequick.example.small.appok_if_stub.apk (20197 bytes = 19.7 KB)

```

单独编译一个组件可以使用：

```
./gradlew -p app.main assembleRelease
```

或者

```
./gradlew :app.main:assembleRelease
```

编译后生成的文件在 Android/Sample/app/smallLibs 目录中。

 3. 检查编译情况

命令：

```
./gradlew small
```

输出：

```
:small

### Compile-time


  gradle-small plugin : 1.2.0-alpha6 (project)
            small aar : 1.2.0-alpha6 (project)
          gradle core : 3.3
       android plugin : 2.3.0
                   OS : Linux 3.8.0-35-generic (amd64)


### Bundles

| type |      name      |  PP  | sdk |  aapt  | support |        file         |   size   |
|------|----------------|------|-----|--------|---------|---------------------|----------|
| host | app            |      | 25  | 25.0.2 | 25.1.0  |                     |          |
| stub | app+stub       |      | 25  | 25.0.2 | 25.1.0  |                     |          |
| app  | app.main       | 0x77 | 25  | 25.0.2 | 25.1.0  | *.main.apk          | 11.7 KB  |
| app  | app.mine       | 0x16 | 25  | 25.0.2 |         | *.mine.apk          | 47.6 KB  |
| app  | app.ok-if-stub | 0x6a | 25  | 25.0.2 |         | *.appok_if_stub.apk | 19.7 KB  |
| app  | app.detail     | 0x67 | 25  | 25.0.2 | 25.1.0  | *.detail.apk        | 7.4 KB   |
| app  | app.home       | 0x70 | 25  | 25.0.2 |         | *.home.apk          | 11.3 KB  |
| lib  | lib.afterutils | 0x45 | 25  | 25.0.2 | 25.1.0  | *.afterutils.apk    | 3.7 KB   |
| lib  | lib.analytics  | 0x76 | 25  | 25.0.2 |         | *.analytics.apk     | 126.6 KB |
| lib  | lib.utils      | 0x73 | 25  | 25.0.2 | 25.1.0  | *.utils.apk         | 6.8 KB   |
| lib  | lib.style      | 0x79 | 25  | 25.0.2 | 25.1.0  | *.style.apk         | 5.5 KB   |
| web  | web.about      |      | 25  | 25.0.2 | 25.1.0  | *.about.apk         | 24.3 KB  |


BUILD SUCCESSFUL

Total time: 4.21 secs

```

### 运行

点击
![效果图](/images/open-source-android-plugins-small-sample/small-dev-sample-run-module.png)
运行需要的组件。
app.home 无法单独运行是因为它只包含一个 `Fragment`，没有 Launcher Activity。
如果需要修改某个插件，修改完后重新编译该插件，在重新编译宿主工程就可以了。

### 清除插件 

清除基础库：

```
./gradlew cleanLib -q 
```

清除所有插件：

```
./gradlew cleanBundle -q
```

## Small工程解析

### 插件路由

在宿主工程 app 的 assets 目录下面有个文件 bundle.json，它就想一个插件路由表一样，Small 中用它来对插件进行管理：

```
{
  "version": "1.0.0",
  "bundles": [
    {
      "uri": "lib.utils",
      "pkg": "net.wequick.example.small.lib.utils"
    },
    ...
    {
      "uri": "mine",
      "pkg": "net.wequick.example.small.app.mine"
    },
    {
      "uri": "detail",
      "pkg": "net.wequick.example.small.app.detail",
      "rules": {
        "sub": "Sub"
        "page": MyPage"
      }
    },
    {
      "uri": "stub",
      "type": "app",
      "pkg": "net.wequick.example.small.appok_if_stub"
    },
    ...
  ]
}
```
bundle.json 中的每一个元素都是对一个插件的声明。

 - uri：跳转插件的界面都是通过uri来指定的，也就是一个uri唯一对应一个插件
 - pkg：插件的包名
 - type：插件类型：app应用插件；lib公共库插件
 - rules：插件的子路由表

在加载插件时会根据type类型来寻找合适的 `BundleLauncher`，如果这里没有指定type，那么就要根据包名来判断插件类型，比如包名里面要包含 app 、lib 或者 web 等。

#### 主路由

在代码中通过：

```java
Small.openUri("detail", context);
```

来通过包名来查找对应插件，调用该插件下面的 Launcher Activity。即 `net.wequick.example.small.app.detail`下面的`MainActivity`。

#### 子路由

如果是：

```java
Small.openUri("detail/Sub", context);
```

就会调用 `net.wequick.example.small.app.detail` 下面的 `SubActivity`。

#### 传递参数

在调用插件的时候还可以传递参数：

```java
Small.openUri("detail?from=app.home", context);
```

在调起的插件`app.detail/MainActivity`中通过下面方法来查询参数：

```java
Uri uri = Small.getUri(this);
if (uri != null) {
    String from = uri.getQueryParameter("from");
    // Do stuff by `from'
}
```


