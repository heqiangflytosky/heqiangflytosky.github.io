---
title: Cocos Creator -- 启动篇
categories: cocos
comments: true
tags: [cocos]
description: Cocos2d-x 介绍以及一些学习资料
date: 2020-2-10 10:00:00
---

## 概述

项目中正在使用 Cocos Creator（cocos2d-x-lite + jsb-adapter + engine） 来打造小游戏运行平台，来运行包括 cocos、laya、egret 引擎编译生成的 web 版游戏，web 版游戏是 js 标准接口。
从本文开始，将会通过一系列文章来介绍 Cocos Creator 的使用以及源码分析。
源码是基于 cocos creator 2.3.1 版本进行的，并根据版本发展持续跟新中。

## Cocos2d-x 特点

 - 跨平台。Cocos 支持使用同一套代码构建生成 Web、iOS、Android 等几个端，最新的版本还支持发布到微信小游戏、Facebook Instant Games 和 QQ 玩一玩；
 - 性能。在 Android 平台上，Cocos 的原理是在 Activity 中绘制一个 OpenGL 的 SurfaceView ，并由其完成页面的渲染的。与基于 WebView 渲染的 Hybrid 应用相比，Cocos 的渲染速度更快，性能更好。
 - 效率。借助可视化的 Cocos Creator 工具，界面的开发和资源的管理非常便捷，设计团队也可以参与进来设计界面和动效，提升开发效率。
 - 表现力。Cocos 本身是为游戏开发设计的，可以带来很多诸如游戏、画图、音乐等带游戏和娱乐性质的场景。

cocos2d-x-lite 在 Android 平台上采用v8 作为js的运行环境，提供js运行基础类库。通过Android OpenGL 接口模拟实现 WebGL 接口。
js-framework 模拟浏览器标准接口、生命周期规范，js层简化实现逻辑，减小开发成本。

同样是运行js开发的游戏，那么和H5游戏方案相比，它有什么优势呢？

 - 更高的性能：支持原生游戏品质，高性能渲染器，而H5游戏的性能受限于浏览器。
 - 更强扩展能力：
 - 生态可控度：运行环境安全可控，内容可管控，支付通道原生支持，不会被非法切支付。H5在这方面的可管控度较低，可能会被切支付和广告。
 - 游戏移植成本低：支持 Canvas 2d api 和 WebGL api，adapter的引入可以提供大部分H5游戏所依赖的浏览器接口。

## Cocos Creator

### Cocos Creator 和 Cocos2d-x 的区别

Cocos2d-x 是游戏引擎，Cocos Creator 是一个完整的游戏开发解决方案，包含了游戏引擎+图形界面工具(编辑器)。
Cocos Creator 是以内容创作为核心的游戏开发工具，在 Cocos2d-x 基础上实现了彻底脚本化、组件化和数据驱动等特点。Cocos Creator 编辑器提供面向设计和开发的两种工作流，提供简单顺畅的分工合作方式。除了开发人员，也可以让策划和美术人员也加入到对引擎的使用中，可以大大加速整个游戏的开发过程。
Cocos2d-x一直在追求「性能快」，而Cocos Creator则追求在「性能快」和「开发快」之间取一个最佳平衡点，并且附加上AnySDK打包工具能带来的「发布快」优势。
Cocos Creator 目前支持发布游戏到 Web、iOS、Android、各类"小游戏"、PC 客户端等平台，真正实现一次开发，全平台运行。

### Cocos Creator 引擎简介

Cocos Creator 的引擎部分包括 JavaScript、Cocos2d-x-lite 和 adapter 三个部分。全部都在 GitHub 上开源。

[JavaScript 引擎](https://github.com/cocos-creator/engine)：cocos js 引擎，游戏开发者集成到游戏包中，封装一些游戏开发的组件和API供开发者调用。这部分是集成在IDE中打包时根据所选的平台转换成运行时环境匹配的接口（我们只适配运行web-mobile平台）。这部分代码编译成 cocos2d-js.js 或者 cocos2d-js-min.js（debug模式）集成在游戏应用包中。

[jsb-adapter](https://github.com/cocos-creator-packages/jsb-adapter)：js 适配层，提供js到cocos运行时引擎的桥接。
Cocos Creator 为了实现跨平台，在 JavaScript 层需要对不同平台做一些适配工作。 这些工作包括：

 - 为不同平台适配 BOM 和 DOM 等运行环境
 - 一些引擎层面的适配 

目前适配层主要包括两个部分：

 - jsb-adapter 适配原生平台
 - mini-game-adapters 适配各类小游戏

在 jsb-adapter 目录下，主要包括以下两个目录结构：

 - builtin：适配原生平台的 runtime。
 - engine：适配和实现 cocos 引擎层面的一些 api，如果游戏平台是适配 web 标准 api 的话就不用关注这个目录。

builtin 部分除了适配 BOM 和 DOM 运行环境，还包括了一些相关的 jsb 接口，如 openGL, audioEngine 等。
cocos 把各种引擎编写的js代码转成标准js接口，jsb-adapter 完成这些标准接口到 cocos 运行时的调用。
这部分其实就是 JS binding 的 js 部分。这部分编译成的 jsb-builtin.js 文件集成 Android 客户端中，供 Cocos2d-x-lite 在初始化时读取。
它架设在 Cocos2d-x-lite 引擎之上，底层通过 JSB 绑定调用 Cocos2d-x-lite 引擎。

[Cocos2d-x-lite](https://github.com/cocos-creator/cocos2d-x-lite)：cocos 渲染引擎，我们可以称之为 cocos 运行时引擎，集成了Android、ios和windows等原生平台特性。它是 Cocos2d-x 的轻量级版本，删除了 3D 以及其他一些特性。
综上所述，cocos 的原理就是通过 cocos IDE 把开发者通过调用 cocos JavaScript 引擎 API 形成的游戏代码转换成标准的 js 代码，然后通过 v8 解析js代码，回传给 jsb，通过 jsb 桥接调用 cocos2d-x-lite 。

## 环境准备

### 下载 cocos-creator

下载地址：https://www.cocos.com/creator

### 运行 HelloWorld

打开 cocos creator 后新建工程，选择 HelloWorld 工程。

### 定制引擎

如果需要研究定制引擎或者有定制引擎的需求，请参考下面定制引擎章节。

### 构建 Android 工程

在构建的对话框中 发布平台 选项中选择 Android，构建完成后就在 HelloWorld 工程的 `HelloWorld/build/jsb-link/frameworks/runtime-src/proj.android-studio` 目录下面生成了 Android 工程，下面的一篇文章会对 HelloWorld 工程进行介绍。
如果 发布平台 选项中选择 Web Mobile 平台，就对应的是 HelloWorld/build/web-mobile 目录，这是在 web 上运行的游戏。生成标准的 html 标准接口。

## 定制引擎

### 定制 JavaScript 引擎

我们可以使用 cocos creator 内置的 JavaScript 引擎和 Cocossd-x 引擎，当然也可以使用自定义的引擎，因为他们都是开源的，方便我们加入个性化的需求。
[引擎定制工作流程](https://docs.cocos.com/creator/manual/zh/advanced-topics/engine-customization.html)
JavaScript 引擎的版本最好要和 CocosCreater 的版本保持一致。

```
# 安装 gulp 构建工具
npm install -g gulp
# 安装依赖的模块
npm install
gulp build  
gulp build --max-old-space-size=8192 // 出现 JavaScript heap out of memory 时
```

编译完成后通过 项目 -> 项目设置 面板的 自定义引擎 选项卡，设置本地定制后的 JavaScript 引擎路径。

### 定制 jsb

在 CocosCreator 的安装目录 `/Applications/CocosCreator.app/Contents/Resources/builtin` 下面重新 clone 一份 jsb-adapter，然后可以基于这份代码上面修改。
Creator v2.0.7 之后的版本打开编辑器时会自动编译，注意修改后一定要重新通过 Cocos Creater 编辑器打开项目代码，这样才会自动重新编译 jsb-adapter。编译后会在 `/Applications/CocosCreator.app/Contents/Resources/builtin/jsb-adapter/bin` 下面生成对应文件 jsb-engine.js 和 jsb-builtin.js。
如果要单独手动编译 jsb-adapter，编译按照 [引擎定制工作流程](https://docs.cocos.com/creator/manual/zh/advanced-topics/engine-customization.html) 提供的 `Creator v2.0.5 和 v2.0.6 的定制流程` 步骤即可：
定制前需要先安装相关依赖，请在命令行中执行：

```
# 在命令行进入 jsb-adapter 路径
cd jsb-adapter/
# 安装 gulp 构建工具
npm install -g gulp
# 安装依赖的模块
npm install
```

接下来就可以对 jsb-adapter 的代码进行定制修改了，修改完成之后请在命令行中继续执行：

```
# jsb-adapter 目录下
gulp
```

gulp 命令执行会将 bultin 部分的代码打包到 jsb-builtin.js 文件，并且将 engine 部分的代码从 ES6 转为 ES5。所有这些编译生成的文件会输出到 dist 目录下。

#### 可能遇到的问题

1.
编译过程中，如果遇到 ` No gulpfile found` 的问题，可以添加下面的 gulpfile.babel.js 文件，该文件在 2018.11.27 的变更中被删除。

```
const gulp = require('gulp');
const sourcemaps = require("gulp-sourcemaps");
const babelify = require("babelify");
const browserify = require('browserify');
const source = require("vinyl-source-stream");
const uglify = require('gulp-uglify');
const buffer = require('vinyl-buffer');
const babel = require('gulp-babel');

gulp.task("default", ['build-jsb-builtin', 'build-engine-es5']);

gulp.task('build-jsb-builtin', () => {
  return browserify('./builtin/index.js')
         .transform(babelify, {presets: ['env']} )
         .bundle()
         .pipe(source('jsb-builtin.js'))
         .pipe(buffer())
         // .pipe(sourcemaps.init({ loadMaps: true }))
         // .pipe(uglify()) // Use any gulp plugins you want now
         // .pipe(sourcemaps.write('./'))
         .pipe(gulp.dest('./dist'));
});

gulp.task('build-engine-es5', () => {
  return gulp.src('./engine/**/*.js')
         .pipe(babel( {presets: ['@babel/preset-env']} ))
         .pipe(gulp.dest('./dist/engine'));
});
```

2.
运行 gulp 后还报找不到一些 module 的错误，在 package.json 加入下面的 devDependencies，或者缺少什么添加什么，然后再运行 npm install和 gulp：

```
{
  "name": "jsb-adapter",
  "version": "1.0.0",
  "reload": {
    "ignore": [
      "**/*"
    ]
  },
  "devDependencies": {
    "@babel/core": "^7.1.2",
    "@babel/preset-env": "^7.1.0",
    "babel-cli": "^6.26.0",
    "babel-core": "^6.26.0",
    "babel-preset-env": "^1.6.1",
    "babelify": "^8.0.0",
    "browserify": "^16.2.2",
    "gulp": "^3.9.1",
    "gulp-babel": "^8.0.0",
    "gulp-sourcemaps": "^2.6.4",
    "gulp-uglify": "^3.0.0",
    "vinyl-buffer": "^1.0.1",
    "vinyl-source-stream": "^2.0.0"
  }
}
```

4.
如果遇到：

```
npm ERR! Unexpected end of JSON input while parsing near '...m":"^1.0.0","shell-qu'

npm ERR! A complete log of this run can be found in:
npm ERR!     /Users/heqiang/.npm/_logs/2020-03-03T08_25_58_261Z-debug.log
```

执行 `npm cache clean --force`，然后再执行 `npm install`。

5.
如果遇到：

```
bogon:cocos-js-adapter heqiang$ gulp
[17:29:42] Requiring external module babel-register
fs.js:35
} = primordials;
    ^

ReferenceError: primordials is not defined
    at fs.js:35:5
    at req_ (/Users/heqiang/workspace/project/cocos-js-adapter/node_modules/natives/index.js:143:24)
    at Object.req [as require] (/Users/heqiang/workspace/project/cocos-js-adapter/node_modules/natives/index.js:55:10)
    at Object.<anonymous> (/Users/heqiang/workspace/project/cocos-js-adapter/node_modules/vinyl-fs/node_modules/graceful-fs/fs.js:1:37)
    at Module._compile (internal/modules/cjs/loader.js:1158:30)
    at Module._extensions..js (internal/modules/cjs/loader.js:1178:10)
    at Object.require.extensions.<computed> [as .js] (/Users/heqiang/workspace/project/cocos-js-adapter/node_modules/babel-register/lib/node.js:152:7)
    at Module.load (internal/modules/cjs/loader.js:1002:32)
    at Function.Module._load (internal/modules/cjs/loader.js:901:14)
    at Module.require (internal/modules/cjs/loader.js:1044:19)
```

参考 https://blog.csdn.net/zxxzxx23/article/details/103000393 可能是你的node版本过高，通过下面命令修改node版本：

```
sudo n v11.15.0
```

然后重新`npm cache clean --force`，然后再执行 `npm install`。

6.
如果遇到下面问题：

```
bogon:cocos-js-adapter heqiang$ gulp
[17:18:37] Requiring external module babel-register
assert.js:385
    throw err;
    ^

AssertionError [ERR_ASSERTION] [ERR_ASSERTION]: Task function must be specified
    at Gulp.set [as _setTask] (/Users/heqiang/workspace/project/cocos-js-adapter/node_modules/undertaker/lib/set-task.js:10:3)
    at Gulp.task (/Users/heqiang/workspace/project/cocos-js-adapter/node_modules/undertaker/lib/task.js:13:8)
    at Object.<anonymous> (/Users/heqiang/workspace/project/cocos-js-adapter/gulpfile.babel.js:34:6)
    at Module._compile (internal/modules/cjs/loader.js:1158:30)
    at loader (/Users/heqiang/workspace/project/cocos-js-adapter/node_modules/babel-register/lib/node.js:144:5)
    at Object.require.extensions.<computed> [as .js] (/Users/heqiang/workspace/project/cocos-js-adapter/node_modules/babel-register/lib/node.js:154:7)
    at Module.load (internal/modules/cjs/loader.js:1002:32)
    at Function.Module._load (internal/modules/cjs/loader.js:901:14)
    at Module.require (internal/modules/cjs/loader.js:1044:19)
    at require (internal/modules/cjs/helpers.js:77:18) {
  generatedMessage: false,
  code: 'ERR_ASSERTION',
  actual: false,
  expected: true,
  operator: '=='
}
```

看看 package.json 里面的gulp版本是不是大于4.0的，参考 https://www.jianshu.com/p/c30ff8592421 这篇文章，改为 3.9.1，然后重新`npm cache clean --force`，然后再执行 `npm install`。

### 定制Runtime引擎

在 HelloWorld Android 工程中直接修改编译即可。
编译按照 [引擎定制工作流程](https://docs.cocos.com/creator/manual/zh/advanced-topics/engine-customization.html) 提供的步骤即可。

## 相关资料

[cocos 官网](https://www.cocos.com/)
[Cocos2d-x github 源码](https://github.com/cocos2d/cocos2d-x)
[Cocos2d-x 用户手册](https://docs.cocos.com/cocos2d-x/manual/zh/)

[Cocos Creator 用户手册](https://docs.cocos.com/creator/manual/zh/)
[Cocos Creator API 参考](https://docs.cocos.com/creator/api/zh/)

## 文章推荐

[Cocos 引擎创始人王哲：选择和努力一样重要](https://xw.qq.com/amphtml/20171204015864/GAM2017120401586400)
[微信「跳一跳」带火小游戏，开发者如何快速上手？](http://www.gamelook.com.cn/2018/01/317302)