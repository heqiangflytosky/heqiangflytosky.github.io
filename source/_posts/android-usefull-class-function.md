---
title: Android -- 那些让你相见恨晚的方法、类或接口
categories:  Android
comments: true
tags: [Android]
description: 介绍 Android 一些有用的方法和类
date: 2016-6-12 10:00:00
---


## Android

 1. getParent().requestDisallowInterceptTouchEvent(true);剥夺父view 对touch 事件的处理权，谁用谁知道。
 2. ArgbEvaluator.evaluate(float fraction, Object startValue, Object endValue); 用于根据一个起始颜色值和一个结束颜色值以及一个偏移量生成一个新的颜色，分分钟实现类似于微信底部栏滑动颜色渐变。
 3. Canvas中clipRect、clipPath和clipRegion 剪切区域的API。
 4. Bitmap.extractAlpha ();返回一个新的Bitmap，capture原始图片的alpha 值。有的时候我们需要动态的修改一个元素的背景图片又不希望使用多张图片的时候，通过这个方法，结合Canvas 和Paint 可以动态的修改一个纯色Bitmap的颜色。
 5. HandlerThread，代替不停new Thread 开子线程的重复体力写法。
 6. IntentService,一个可以干完活后自己去死且不需要我们去管理子线程的Service。
 7. Palette，5.0加入的可以提取一个Bitmap 中突出颜色的类，结合上面的Bitmap.extractAlpha，你懂的。
 8. Executors. newSingleThreadExecutor();这个是java 的，之前不知道它，自己花很大功夫去研究了单线程顺序执行的任务队列。
 9. android:animateLayoutChanges=”true”，LinearLayout中添加View 的动画的办法，支持通过setLayoutTransition()自定义动画。
 10. ViewDragHelper，自定义一个子View可拖拽的ViewGroup 时，处理各种事件很累吧
 11. GradientDrawable，阴影效果。
 12. AsyncQueryHandler，如果做系统工具类的开发，比如联系人短信辅助工具等，肯定免不了和ContentProvider打交道，如果数据量不是很大的情况下，随便搞，如果数据量大的情况下，了解下这个类是很有必要的，需要注意的是，这玩意儿吃异常
 13. ViewFlipper，实现多个view的切换(循环)，可自定义动画效果，且可针对单个切换指定动画。
 14. 有朋友提到了在自定义View时有些方法在开启硬件加速的时候没有效果的问题，在API16之后确实有很多方法不支持硬件加速，通常我们关闭硬件加速都是在清单文件中通过，其实android也提供了针对特定View关闭硬件加速的方法,调用View.setLayerType(View.LAYER_TYPE_SOFTWARE, null);即可。
 15. android util包中的Pair类，可以方便的用来存储一”组”数据。注意不是key value。
 16. PointF，graphics包中的一个类，我们经常见到在处理Touch事件的时候分别定义一个downX，一个downY用来存储一个坐标，如果坐标少还好，如果要记录的坐标过多那代码就不好看了。用PointF(float x, float y);来描述一个坐标点会清楚很多。
 17. StateListDrawable，定义Selector通常的办法都是xml文件，但是有的时候我们的图片资源可能是从服务器动态获取的，比如很多app所谓的皮肤，这种时候就只能通StateListDrawable来完成了，各种addState即可。
 18. android:descendantFocusability，ListView的item中CheckBox等元素抢焦点导致item点击事件无法响应时，除了给对应的元素设置 focusable,更简单的是在item根布局加上android:descendantFocusability=”blocksDescendants”
 19. android:duplicateParentState=”true”，让子View跟随其Parent的状态，如pressed等。常见的使用场景是某些时候一个按钮很小，我们想要扩大其点击区域的时候通常会再给其包裹一层布局，将点击事件写到Parent上，这时候如果希望被包裹按钮的点击效果对应的Selector继续生效的话，这时候duplicateParentState就派上用场了。
 20. includeFontPadding=”false”，TextView默认上下是有一定的padding的，有时候我们可能不需要上下这部分留白，加上它即可。
 21. Messenger，面试的时候通常都会被问到进程间通信，一般情况下大家都是开始背书，AIDL巴拉巴拉。。有一天在鸿神的博客看到这个，嗯，如他所说，又可以装一下了。
 22. TextView.setError();用于验证用户输入。
 23. ViewConfiguration.getScaledTouchSlop();触发移动事件的最小距离，自定义View处理touch事件的时候，有的时候需要判断用户是否真的存在movie，系统提供了这样的方法。
 24. ValueAnimator.reverse(); 顺畅的取消动画效果。
 25. ViewStub，有的时候一块区域需要根据情况显示不同的布局，通常我们都会通过setVisibility的方法来显示和隐藏不同的布局，但是这样默认是全部加载的，用ViewStub可以更好的提升性能。
 26. onTrimMemory，在Activity中重写此方法，会在内存紧张的时候回调（支持多个级别），便于我们主动的进行资源释放，避免OOM。
 27. EditTxt.setImeOptions， 使用EditText弹出软键盘时，修改回车键的显示内容(一直很讨厌用回车键来交互，所以之前一直不知道这玩意儿)
 28. TextView.setCompoundDrawablePadding，代码设置TextView的drawable padding。
 29. ViewSwitcher，ImageSwitcher，可以用来做图片切换的一个类，类似于幻灯片。
 31. TextInputLayout 是一个能够把EditText包裹在当中的一个布局，当输入文字时，它可以把Hint文字飘到EditText的上方。可以提供一些错误提示功能。
 32. Space是Android 4.0中新增的一个控件，它可以用来分隔不同的控件，其中形成一个空白的区域。
 33. FloatingActionButton 浮动按钮
 34. CoordinatorLayout 可以很好的帮助我们协调子View之间的布局
 35. CollapsingToolbarLayout 用于实现折叠（其实就是看起来像伸缩）的App Bar效果
 36. 有序广播：context.sendOrderedBroadcastAsUser或者context.sendOrderedBroadcast，接收者按照优先级处理广播，并且前面处理广播的接受者可以中止广播的传递。允许发送者自己注册一个广播接收器，而且这个接收器是最后一个接收到广播的。
 37. 粘性广播：context. sendStickyBroadcast或context.sendStickyOrderedBroadcast。该广播可以发送给以后注册的接受者，意思是系统会将前面的粘性广播保存在AMS中，一旦注册了与以保存的粘性广播符合的广播，在注册结束后会立即收到广播。
 38. Android StatFs类：StatFs用于获取存储空间
 39. 进程间通信获取调用方的包名：`int uid = Binder.getCallingUid();getPackageManager().getNameForUid(uid);`。或者是反射调用 `ActivityManagerNative` 的 `getLaunchedFromPackage` 方法。
 40. 设置ListView的某个Item不可点击：重写Adapter的`isEnabled(int position)`方法。
 41. `@ViewDebug.ExportedProperty(category = "launcher")` 如果你想在查看布局列表时查看view某个属性，可以把上面的注解加上，`category = "launcher"` 可以把他们归结都某个组中。


## Java

 1. FileFilter 和 FilenameFilter 文件搜索，过滤文件
 2.  WeakHashMap，直接使用HashMap有时候会带来内存溢出的风险，使用WaekHashMap实例化Map。当使用者不再有对象引用的时候，WeakHashMap将自动被移除对应Key值的对象。

## 推荐阅读

https://www.zhihu.com/question/33636939/answer/57239990?group_id=612750833369153536