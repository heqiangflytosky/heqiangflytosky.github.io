---
title: Android  应该了解的知识系列 - 关于 Style 命名和继承方式的那些事儿
categories: Android
comments: true
tags: [Android]
description: 介绍关于 Style 命名和继承方式的知识
date: 2015-5-6 10:00:00
---

## Style 的继承方式

在面向对象编码中我们对继承是司空见惯的，在 Android 主题风格的使用上我们也是可以使用继承的，否则，每种主题都要我们重新配置属性的话效率会非常低。
对于 style 的下面两种书写方式我们很常见：


```
    <style name="CustomTheme" parent="Theme.AppCompat.Light.DarkActionBar">

    </style>
```

```
    <style name="Theme.AppCompat.Light.DarkActionBar.CustomTheme">

    </style>
```

这其实就是 style 的两种继承方式：

 - 通过 . 来继承
 - 通过 parent 属性来继承

命名的 sytle 中点有 . 的 style，这种绝不是命名上的偏好，而是通过 . 的方式来继承父 sytle 的属性。
因此，我们要确保每个 . 之前的 style 是确实存在的，否则编译器会用红色的字体来警告你，而且这时也是编译不过的。
会报下面错误：

```
    <style name="MyTheme.Custom.Test">

    </style>
    
    
Error:(1497) Error retrieving parent for item: No resource found that matches the given name 'MyTheme.Custom'.
```

### 属性的自定义

下面的代码是我们继承了系统提供的 NoActionBar 主题。

```
    <style name="MyTheme" parent="Theme.AppCompat.Light.NoActionBar">

    </style>
```

如果我们想在这个基础上做一些自定义，比如现在我们想把 ActionBar 给显示出来，可以这样做：

```
    <style name="MyTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <!-- Customize your theme here. -->

        <item name="windowActionBar">true</item>
        <item name="windowNoTitle">false</item>
    </style>
```

### 继承的优先级

上面的属性自定义中可以得出的结论是：**自定义的属性会覆盖父 style 中的属性**。
那么如果我们同时用 . 继承和 parent 继承方式呢？
下面我们做个测试：

```
    <style name="Theme.AppCompat.Light.NoActionBar.MyTheme" parent="Theme.AppCompat.Light.DarkActionBar">

    </style>
```

最后显示的 DarkActionBar 的主题。
最后再添加下面的代码，又会把 ActionBar 去掉：

```
    <style name="Theme.AppCompat.Light.NoActionBar.MyTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <item name="windowActionBar">false</item>
        <item name="windowNoTitle">true</item>
    </style>
```

因此我们可以得到下面的结论：

**parent 继承会覆盖 . 继承的主题，而自定义的属性又会覆盖前面两者定义的属性。**
自定义 > parent 继承 > . 继承

<!-- 
https://blog.csdn.net/chenak74u/article/details/41213761
https://blog.csdn.net/mingli198611/article/details/7254275
https://blog.csdn.net/lskycity/article/details/41774651
https://blog.csdn.net/daweibalang717/article/details/40738073
-->
