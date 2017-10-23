---
title: 开始了解React Native
categories: React Native
comments: true
tags: [React Native]
description: 初步介绍一下 React Native 及其在移动开发中的优势
date: 2017-01-10 10:00:00
---
## React是什么？
React是一个比较热门的前端框架，可以描述为一个用来构建用户界面的javascript库，（[GitHub地址](https://github.com/facebook/react)）。它起源于Facebook的内部项目，因为 FB 对市场上所有 JavaScript MVC 框架，都不满意，就决定自己写一套，用来架设 Instagram 的网站。做出来以后，发现这套东西很好用，就在2013年5月开源了。
由于 React 的设计思想极其独特，属于革命性创新，性能出众，代码逻辑却非常简单。所以，越来越多的人开始关注和使用，认为它可能是将来 Web 开发的主流工具。
和Backbone、Angular 等MV*框架不一样，它只处理View逻辑 。所以如果你喜欢它，你可以很容易的将它接入到现有工程中，然后用React重写HTML部分即可，不用修改逻辑。
近几年web领域的技术革新非常迅速，React 也是一项新技术，其实React出来也多年了，其实并不算什么新技术了，只是在国内还没有普及开。国内在热更新方面更热衷于纯Native的插件化技术。可以说，Android的未来必将是React-Native和插件化的天下……
<!-- more -->

## React Native
React Native是由React衍生出来的项目（[GitHub地址](https://github.com/facebook/react-native)），基于React来写的支持编写原生App的框架，目标是希望用写Web App的方式去写Native App。React Native看起来很像React，只不过其基础组件是原生组件而非web组件。

## React Native能给移动开发带来什么？
像比较于Hybird app，Native app 以及Webapp，React Native有以下优势：
 1. 极速的渲染性能：不用Webview，彻底摆脱了Webview让人不爽的交互和性能问题。
 2. 组件互相独立，关系隔离，可复用，有较强的扩展性：Native端提供的是基本控件，JS可以自由组合使用。
 3. 跨平台： Write once,run anywhere. React 能够用一套代码同时运行在浏览器和 node 里，而且能够以原生 App 的姿势运行在 iOS 和 Android 系统中，即拥有了 web 迭代迅速的特性，又拥有原生 App 的体验。
 4. JS与Native原生优势互补：JS可以直接使用原生一些控件或者是JS里面比较难以实现的UI效果。
 5. 可以通过更新远端JS，直接更新App，不过现在热修复也基本已成为Native app的标配了。

精通了React开发，感觉自己是不是有『全栈工程师』的特点了呢？

## 其他类似的一些开源框架

 - Weex：基于vue.js开发，使用V8作为JS引擎。

## 一些学习资料
React Native的相关文档https://facebook.github.io/react/   https://facebook.github.io/react-native/
[React 中文网](http://react-china.org/)
[React Native中文网](http://reactnative.cn)
[ECMAScript 6入门](http://es6.ruanyifeng.com/)
[React/React Native 的ES5 ES6写法对照表](http://bbs.reactnative.cn/topic/15/react-react-native-%E7%9A%84es5-es6%E5%86%99%E6%B3%95%E5%AF%B9%E7%85%A7%E8%A1%A8)
[分析reactnative优劣](http://bbs.reactnative.cn/topic/320/2016%E5%B9%B41%E6%9C%889%E6%97%A5-react-native%E5%AE%9E%E6%88%98%E7%BB%8F%E9%AA%8C%E5%88%86%E4%BA%AB%E4%BC%9A-%E5%8C%97%E4%BA%AC-%E5%88%86%E4%BA%AB%E8%AE%B0%E5%BD%95)
[ReactNativeAndroid源码分析-Js如何调用Native的代码](https://zhuanlan.zhihu.com/p/20464825?refer=program-life)
[ReactNative For Android 项目实战总结 QQ空间](https://zhuanlan.zhihu.com/p/20587485?refer=magilu)
[React Native For Android 架构初探 QQ空间](https://zhuanlan.zhihu.com/p/20259704)
[React native for Android 初步实践(淘宝)](https://yq.aliyun.com/articles/2757?spm=5176.100240.searchblog.21.VfnaXO)
[ReactNative开发指导](https://github.com/cnsnake11/blog/tree/master/ReactNative%E5%BC%80%E5%8F%91%E6%8C%87%E5%AF%BC)


随后继续补充

## 参考资料：
http://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=401107957&idx=1&sn=200418877771f656c1a0ab33ad407516&scene=1&srcid=1119XfFA8t5QQprIjzp76fcr&key=ff7411024a07f3ebf6601418be94ccd6219ed18e580029547278b6eadd5def524defc8dbfdfcf673a7daa87723cfa4bb&ascene=0&uin=NTYzMDc5MTc1&devicetype=iMac+MacBookPro11%2C1+OSX+OSX+10.11.1+build%2815B42%29&version=11020201&pass_ticket=a82zcv0P%2B6ztN4xgcdnD%2FWtFbQjxhMOiiUJGZVbk6FUhTeozLqrMlGuES%2FvVmaI0

