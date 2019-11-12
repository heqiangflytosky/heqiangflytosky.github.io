---
title: Android 架构 -- MVC、MVP 和 MVVP 架构浅析
categories: Android 架构
comments: true
tags: [Android 架构]
description: 简单介绍 MVC、MVP 和 MVVP 架构三种架构设计
date: 2017-9-2 10:00:00
---

## 关于架构设计

架构设计的目的就是通过设计使程序模块化，做到模块内部的高聚合和模块之间的低耦合。只要做的好处是使得程序员在开发过程中提高开发效率，并且更容易进行后续的测试以及定位问题。但是，设计不能违背目的，对于不同量级的工程，具体架构的实现方式也是不同的，切记不能为了设计而设计，为了架构而架构。
架构设计没有严格意义上的好与坏，合适的才是最好的。

## MVC、MVP 和 MVVP 概述

架构模式有很多种，MVC（Model-View-Controller） 是其中一种，MVP （Model-View-Presenter）和 MVVM（Model-View-ViewModel）可以说是在 MVC 基础上发展而来的，它们是一种进化的关系：MVC --> MVP --> MVVM，这三种架构模式是我们在 Android 开发过程以及面试中经常涉及到的，本文对这三种架构模式做一些介绍。
MVP 隔离了MVC中的 M 与 V 的直接联系，靠 P 来中转，所以使用 MVP 时 P 是直接调用 V 的接口实现对视图的操作的，这些 View 接口比如：showLoading，updateView 等。M 与 V 已经隔离了，方便测试了，但是代码还是不够优雅简洁，所以出现了 MVVM 来弥补这些缺陷，MVVM 是 MVP 的进一步发展与规范，出现了 Data Binding 这个概念，意思就是 View 接口的showLoading，updateView不用再写了，直接数据绑定自动更新视图。
在不同的开发环境下，这三种模式的使用频率也是不一样的，在 Android 环境下，常用的是 MVC和MVP架构模式，但随着一些开源框架比如 Jetpack 的 databinding、live等的兴起，MVVM 在Android开发中也经常遇到。在iOS和HTML5的开发环境中，会经常使用 MVC和MVVM 架构模式，Java/PHP开发环境下常用到MVC架构模式等。
对于不同的开发环境相同的模式的使用也是略有差距的，本文仅这种介绍一下 Android 环境下的使用。


## MVC 架构

MVC分别是Model（模型）、View（视图）、Controller（控制器）三个模块。

 - View：主要完成界面的展示，UI呈现
 - Controller：对数据的接收处理和对用户触发事件的响应和传递
 - Model：对数据的储存和提供，传递给视图层响应或者展示。

下面我们通过一个图来展示 MVC 架构的设计：

<img src="/images/android-architecture-mvc-mvp-mvvm/mvc.png" width="495" height="318"/>

上面的箭头描述数据的流向。比如和下面介绍的 MVP 和 MVVP 差异较大的 Model --> View  的流向，我们可以设置 Model 层提供的方法的回调函数来监听数据的变化，直接更新 UI。
在 Android 开发中，View 层对应于 xml 布局文件或者java的动态 view 部分。
Controller 层主要是由 Activity 来承担的，因为在 Android 中，xml 的视图功能比较弱，因此，Activity 既要负责逻辑的控制，充当 Controller 的角色，又要负责视图的显示，承担的功能过多。
Model 层主要负责网络请求，数据库处理，I/O操作等。
由于 Activity 扮演了 View 和 Controller 的角色，所以，V 层和 C 层的解耦做得不够好。
这样就会导致一些问题，没有办法单独验证应用逻辑的正确性，当出了问题之后，因为各个模块耦合的原因，不能快速判断是哪个模块出现的问题。而且，也给单元测试造成了很大的困难。

现在我们来总结一下 MVC 架构的特点：

 - 具有一定的分层设计，Model 层解耦比较彻底，但 Controller 和 View 并没有解耦。
 - 数据处理放在 Model 层，能够更好地复用和修改数据处理。

## MVP 架构

MVP（Model-View-Presenter）可以说是 MVC 架构的一种进化，可以分为下面三个模块：

 - View：主要完成界面的展示，UI呈现。它持有 Presenter 的引用，当需要数据相关操作时，通知 Presenter 去处理，以及根据 Presenter 提供的内容来决定如何显示他们。
 - Presenter：负责表现层逻辑处理，告诉 View 层该显示什么，以及响应界面事件，再通知 Model 层修改或者存储数据。Model 层提供好数据后，通知 View 层展示。
 - Model：业务逻辑处理，对数据的储存和提供。

在 MVP 模式中，Model 和 View 不能直接通信，Presenter 充当桥梁的作用，实现 Model 和 View 的通信。因此，View 层和 Presenter 层需要互相持有对方的引用，Presenter 需要持有 Model 的引用。

MVP 架构的结构如下图设计：

<img src="/images/android-architecture-mvc-mvp-mvvm/mvp.png" width="489" height="320"/>

MVP模式的主要优点是：

 - 分离了 Model 层和 View 层，分离了视图操作和业务逻辑，降低了耦合。方便功能扩展和代码变更和重构。
 - Presenter 层和 View 层分离，几乎处理了所有的表现层逻辑，提高了可测试性，因为对视图控件的显示等等进行单元测试太难了，所以 View 是基本上没法进行单元测试的，但是 Presenter 是完全可以进行单元测试的。

缺点是：

 - View 层和 Presenter 层需要互相持有对方的引用，Presenter 需要持有 Model 的引用。
 - 还有就是可能需要写更多的类，不过这相对于它带给我们的优点和便利，是非常值得我们去做的。

## MVVM 架构

MVVM（Model-View-ViewModel）可以说是 MVP 模式的一种进化，它也分为三个模块：

 - View：主要完成界面的展示，UI呈现。它持有ViewModel层的引用，当需要进行业务逻辑处理时通知ViewModel层。
 - ViewModel：负责表现层逻辑处理。ViewModel层不涉及任何的视图操作。View 层和 ViewModel 层中的数据可以实现绑定，ViewModel 层中数据的变化可以自动通知 View 层进行更新，因此 ViewModel 层不需要持有 View 层的引用。ViewModel 层可以看作是 View 层的数据模型和 Presenter 层的结合。
 - Model：业务逻辑处理，对数据的储存和提供。

可以看出，MVVM 像对于 MVP 的核心在于数据绑定。
它们的最大的区别在于：ViewModel 层不持有 View 层的引用。这样进一步降低了耦合，View 层代码的改变不会影响到 ViewModel 层。

MVP 架构的结构如下图设计：

<img src="/images/android-architecture-mvc-mvp-mvvm/mvvm.png" width="501" height="314"/>

总结一下 MVVM 像对于 MVP 的优点：

 - 进一步降低了耦合，ViewModel 层不持有 View 层的引用，当 View 层代码发生改变时，只要 View 层绑定的数据不变，那么 ViewModel 层代码就不需要改变。而在 MVP 模式下，当 View 层发生改变时，操作视图的接口就要进行相应的改变，那么 Presenter 层就需要修改了。
 - 不用再编写很多样板代码。通过 Data Binding 库，UI和数据之间可以实现绑定，不用再编写大量的findViewById()和操作视图的代码了。总之，Activity/Fragment 的代码可以做到相当简洁。
 

## 总结

如果你对上面对于 MVC、MVP和MVVM的区别还是比较模糊的话，下面通过一个简单的例子来介绍一下它们的区别。
比如：有个下拉刷新功能，用户下拉页面后，拉取数据，更新页面。

 1.View 获取下拉事件，通知 Controller 、Presenter 或者 ViewModel。
 2.Controller 、Presenter 或者 ViewModel 向Model 发起数据请求，请求内容为下拉刷新。
 3.Model 层向网络请求刷新内容，服务器返回新的内容。
 4.后面的处理可能就不一样了：
  - MVC：拿到UI节点，渲染展示新的数据。
  - MVP：Presenter 拿到数据后，可能会做数据加工处理，然后通过 View 提供的接口更新数据，后面 View 渲染展示新的数据。
  - MVVM：无需任何操作，只要 ViewModel 的数据发生变化，通过双向绑定，View 自动更新。View 上面有了更新，ViewModel 这边的数据也会自动更新。

## 相关文章

[关于MVC/MVP/MVVM的一些错误认识](https://mp.weixin.qq.com/s/RVKaXY349jSBLtAiQsmwCg)