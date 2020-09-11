---
title: Vue -- 启动篇
categories: JavaScript
comments: true
tags: [Vue]
description: 介绍一下Vue的学习资料
date: 2019-1-8 10:00:00
---

## 简介

Vue  是一套用于构建用户界面的渐进式JavaScript框架？渐进式就是指我们可以由浅入深的、由简单到复杂的方式去使用Vue.js。
[Vue 官方文档](https://cn.vuejs.org/index.html)
Vue.js 优点：

 - 体积小，压缩后33K。
 - 更高的运行效率：基于虚拟Dom。
 - 双向数据绑定：让开发者不用再去操作dom对象，把更多的精力放到业务逻辑上。
 - 生态丰富，学习成本低：有大量的成熟稳定的基于Vue的ui框架、常用组件，可以拿来直接用。入门容易，学习资料多。

## Vue 生态圈

 - [Vue Router](https://router.vuejs.org/zh/)：是 vue 官方的路由。它能快速的帮助你构建一个单页面或者多页面的项目。
 - [Vuex](https://vuex.vuejs.org/zh/)：是一个专为 Vue.js 应用程序开发的状态管理模式。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。它能解决你很多全局状态或者组件之间通信的问题。
 - Vue CLI：是官方提供的一个 vue 项目脚手架，本项目也是基于它进行构建的。它帮你封装了大量的 webpack、babel 等其它配置，让你能花更少的精力在搭建环境上，从而能更专注于页面代码的编写。不过所有的脚手架都是针对大部分情况的，所以一些特殊的需求还是需要自己进行配置。建议先阅读一遍它的文档，对一些配置有一些基本的了解。
 - Vue Loader：    是为 vue 文件定制的一个 webpack 的 loader，它允许你以一种名为单文件组件 (SFCs)的格式撰写 Vue 组件。它能在开发过程中使用热重载来保持状态，为每个组件模拟出 scoped CSS 等等功能。不过大部分情况下你不需要对它直接进行配置，脚手架都帮你封装好了。
 - Vue Test Utils：是官方提供的一个单元测试工具。它能让你更方便的写单元测试。
 - Vue Dev-Tools：Vue 在浏览器下的调试工具。写 vue 必备的一个浏览器插件，能大大的提高你调试的效率。
 - Vetur：    是 VS Code 的插件. 如果你使用 VS Code 来写 vue 的话，这个插件是必不可少的。
 - mockjs：可以模拟后台返回数据，这样就可以脱离后台进行一些前端的开发。可以结合 axios-mock-adapter 使用。

## Vue ui 框架

 - [桌面端后台UI组件库element-ui](https://element.eleme.cn/#/zh-CN/)：饿了么团队维护
 - [移动端UI组件库Vant](https://youzan.github.io/vant/#/zh-CN/home)：有赞团队出品
 - [移动端UI组件库NutUI](http://nutui.jd.com/#/index)：京东团队推出

## 相关开源项目

### Vue

 - [vue-element-admin：一款基于 ElementUI 二次开发的后台开源项目](https://panjiachen.github.io/vue-element-admin-site/zh/)
 - [vue-manage-system：基于 Vue + Element UI 的后台管理系统解决方案](https://github.com/lin-xin/vue-manage-system)
 - [vue2-element-touzi-admin：基于 Vue2.0 + vuex + ElementUI 后台管理系统](https://github.com/wdlhao/vue2-element-touzi-admin)
 - [element3：慕课网讲师蜗牛老师个人维护的一个 ElementUI + Vue3.0 版本，当然现在可能就是 beta 版本的 Vue3.0。自己平时做项目拿来把玩可以，但是用于公司生产环境需要三思]()

### Vant

 - [newbee-mall-vue-app：新蜂商城 Vue 版本](https://github.com/newbee-ltd/newbee-mall-vue-app)
 - [Vant 官方示例合集](https://github.com/youzan/vant-demo)
 - [Vant Weapp 是移动端 Vue 组件库 Vant 的小程序版本](https://github.com/youzan/vant-weapp)

### NutUI

 - [nutui-demo：基于 Vue CLI 搭建 NutUI 的相关示例项目](https://github.com/jdf2e/nutui-demo)

### 其他

 - Material-Dashboard
 - iview-admin
 - layui
 - AdminLTE
 - tabler
 - gentelella
 - ng2-admin
 - ant-design-pro
 - blur-admin

## 工具安装

## 创建项目

```
vue init webpack my-project
```

安装依赖，运行项目：

```
npm install

```

开发时安装单个依赖：

```
npm install --save vue-resource
//或者
npm i vue-resource --save
```

运行开发环境：

```
npm run dev
```

根据 webPack 配置在浏览器中打开。

运行生产环境：

```
nom run build
```

如果想查看生产环境运行效果，打开dist目录，然后运行：

```
php -S 127.0.0.1:9999
```

然后在浏览器中打开 127.0.0.1:9999。

## 学习资料

[Font Awesome图标库](http://www.fontawesome.com.cn/)
https://www.runoob.com/vue2/vue-tutorial.html
https://github.com/opendigg/awesome-github-vue
https://github.com/vuejs/awesome-vue
https://github.com/bradtraversy/vanillawebprojects
https://github.com/eveningwater/my-web-projects
[ElementUI 不维护？供我们选择 Vue 组件库有很多](https://www.toutiao.com/i6864495017101099531/?timestamp=1598275821&app=news_article&group_id=6864495017101099531&use_new_style=1&req_id=202008242130210101310750941F0DDACA)
[vue.js实例项目有木有？](https://www.zhihu.com/question/37984203)