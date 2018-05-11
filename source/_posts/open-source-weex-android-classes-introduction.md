---
title: Weex 源码研究 -- 相关类概述
categories: Weex
comments: true
tags: [Weex]
description: 介绍 Weex 里面比较重要的几个类
date: 2018-3-10 10:00:00
---

这里先来介绍几个 Component 相关的几个类：

## WXSDKManager

这个是 weex 的核心所在，直接或者间接的管理着 weex 上下文相关的所有类。以组合的形式带上 `WXBridgeManager`、`WXRendermanager` 和 `WXDomManager`。三个管理类如名称一样，负责各自的功能区域。后面会分别再介绍这三个类

## WXBridgeManager

主要负责 和 JS 引擎交互 ，发送 native 端 java 的请求到 jni 层，并接收js引擎处理后从jni上返回的消息。

## WXRendermanager

将对应的Dom的节点渲染成组件。
`WXRenderManager` 中的 `WXRenderHandler` 根据代码：

```
  public WXRenderHandler() {
    super(Looper.getMainLooper());
  }
```

可以看到是运行在 UI 线程中的，因此，渲染都是在 UI 线程中执行的。

## RenderActionContextImpl

是 `RenderActionContext` 的实现类。是渲染动作的执行类。`mRegistry` 保存了 `WXSDKInstance` 对应的所有的组件。

## RenderAction

## WXDomManager

负责构建客户端的dom结构，在 `WXBridgeManager` 接收到消息后，会交给 `WxDomManager` 处理，`WxDomManager` 根据消息创建自己的 Dom 结构，添加、删除、修改元素。
WXDomManager.batch() 批处理dom操作。
`WXDomManager` 中的 `mDomHandler` 我们根据代码：

```
mDomThread = new WXThread("WeeXDomThread", new WXDomHandler(this));
```

可以知道它是一个名称为 WeeXDomThread 的后台线程。

## DOMActionContextImpl

## DOMAction

 - AddEventAction
 - AnimationAction
 - RemoveElementAction
 - RemoveEventAction
 - ScrollToElementAction
 - UpdateAttributeAction
 - UpdateFinishAction
 - UpdateStyleAction

## WXDomObject
 
 - DomObject 包括了 <template> 在 Dom 树中的所有信息，如 style、attr、event、ref（结点的唯一标识符）、parent、children

## ImmutableDomObject

 - 表示一个 Dom 节点

## WXComponent

 - Component 负责承载 Native View，可以通过泛型指定承载 View的类型。
 - 所有组件相关的类都要继承这个类。
 - Component 会保留 DomObject 的强引用，两者实例是一一对应的。
 - 通过调用 initComponentHostView 创建 Component 需要承载的 View，所有 的Component 必须重写 initComponentHostView 方法，返回需要承载的 View 的最外层容器。如下：

 ```
 @Component(lazyload = false)
 public class WXText extends WXComponent<WXTextView> implements FlatComponent<TextWidget> {
  protected WXTextView initComponentHostView(@NonNull Context context) {
    WXTextView textView =new WXTextView(context);
    textView.holdComponent(this);
    return textView;
  }
 }
 ```

## WXModule
 
 - 通过 Module 可以将 Native Api 暴露给Js。
 - 所有的模块都要继承这个类。

## WXSDKInstance

这个类前面的 demo 中已经展示了它的用法。

 - Weex 的渲染单位，是 weex 渲染的实例对象。
 - 它可以是一个纯 weex View，和可以和 Native 混合。
 - 它提供了一系列跟页面渲染相关的接口：render()、renderByUrl()等。
 - 提供了几个比较重要的回调接口，比如生命周期回调和渲染结果回调。方便开发者根据不同的业务场景去处理他们的逻辑。
 - 这里明确两个容器的概念： 
  - Instance RootView：Weex 最外层容器， Native 接入方可以设置 Instance RootView 大小, 最终通过 `onViewCreated()` 返回给用户的View也就是 Instance RootView。用 `RenderContainer` 类来表示。
  - JS Root: JS 可以描述的最外层容器， 为 RootView 的唯一子节点， 受 JS 样式的控制。用 `WXComponent` 来表示。
 - Instance 宽高设置遵循以下几个原则：
  - Instance RootView 的宽高优先遵循 instance。 设置的宽高，如果开发者没有设置，则与 JS Root 节点的宽高保持一致。
  - JS Root 节点的宽高优先遵循 CSS 样式设置的宽高，如果没有设置，则使用 instance 上设置的宽高，如果 instance 也没有设置，则使用 layout 出来的宽高
  - 特殊情况，当 `scroller` 和 `list` 作为 `JS Root` 时，如果不设置高度, 会给 `scroller` 和 `list` 设置 `flex:1`。
  - 综上所述，Instance RootView 和 JS Root 的宽高可以不一致，应该根据需求正确的设置 Instance 的宽高，也可以在运行时动态的改变 Instance 的宽高。

## WXDomModule

派发渲染指令的枢纽，提供了 `callDomMethod()` 和 `postAction()`方法来派发渲染指令。

 - JSFramework 根据 Virtual Dom 计算出来的 dif，将渲染指令（Json）通过 Js Engine 发送给 Native Render 进行渲染。而 `WXDomModule` 会接收到所有渲染指令，然后将指令post 给 `DomHandler`，最后由 `DomHandler` 来派发渲染任务。
 - `DomStatement` 在 Dom 线程中创建 `DomObject` 和 `Component`，`RenderStatement` 负责在 UI 线程中渲染 View；每个 `WXSDKInstance` 会持有一个 `DomStatement` 和 `RenderStatement` 实例。
 - `RenderStatement` 会从 `DomStatementclone` 一份 `DomObject`，是为了避免两个线程同时操作 Dom 造成的同步问题。
 - 主要有如下指令：
  - createBody：`DomStatement` 首先在 Dom 线程中创建 JS Root 对应的 Component，然后会将 JS Root 添加到 `WXSDKInstance` 作为 GodCom 的子节点，从而生成 Component 树的最顶端。生成 Component 树后，将 createBody 任务 post 到 UI 线程，由` RenderStatement` 创建 `WXSDKInstance` 的 Rootview，并通过 onViewCreated 回调给 WXSDKInstance 的上下文。
  - addElement：首先，`DomStatement` 在 Dom 线程中创建 DomObject 和对应的 Component 实例，加入 Dom 树和 Component 树；然后将 addElement 任务 post 到 UI 线程，`RenderStatement` 会触发 Component 完成以下任务： createView（初始化 Component 承载的 View）、applyLayoutAndEvent（触发 setLayout 和 setPadding、绑定 Event）、bindData（给 View 设置 style、attr）、addChild（将 View 加入 View 树）
  - removeElement：是 addElement 的逆向操作，将 View、Component、DomObject 分别从各自的树中删除，并销毁数据回收资源。
  - moveElement：将 View、Component、DomObject 在树中移动位置，move 操作最终被拆分成一次 remove 操作和一次 add 操作。
  - addEvent：绑定事件。
  - removeEvent：撤销事件绑定。
  - updateAttrs：当结点 attr 被改变时，会触发 updateAttrs，最终会触发 `WXComponent` 中的 updateProperties 刷新 UI。
  - updateStyle：与 updateAttrs 类似。
  - createFinish：JsFramework 将所有渲染指令都发出后，会触发 createFinish，最后会触发 `onRenderSuccess` 回调。
  - updateFinish：JsFramework 将所有 update 指令发出后，会触发 updateFinish，最后会触发 `onUpdateFinish` 回调。
 

