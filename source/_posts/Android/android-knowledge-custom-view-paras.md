---
title: Android -- View 构造参数解析
categories: Android
comments: true
tags: [Attribute和Style]
description: View 构造参数解析
date: 2016-7-10 10:00:00
---

## 概述

还记得我们自定义 View 的场景吗？写完 `extends View` 后，快捷键一挥，自动生成了几个构造函数：

```
    public TestTextView(Context context) {
        super(context);
    }

    public TestTextView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    public TestTextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    public TestTextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
    }
```

对于重载的这几个构造方法，他们的参数都是不一样的，不知道大家有没有思考过这些参数的作用呢？本文就这些参数做一个简单介绍。

## AttributeSet

`attrs` 参数中保存的是我们为该View设置的所有的属性，比如我们在xml布局文件中配置如下：

```
    <com.example.heqiang.testsomething.attr.TestTextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="测试Attr"/>
```

那么我们在构造方法中打印 attrs 的信息：

```
    public TestTextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        int count = attrs.getAttributeCount();
        for (int i = 0; i < count; i++) {
            String attrName = attrs.getAttributeName(i);
            String attrVal = attrs.getAttributeValue(i);
            Log.e("Test", "attrName = " + attrName + " , attrVal = " + attrVal);
        }
    }
```

得到的结果为：

```
attrName = layout_width , attrVal = -2
attrName = layout_height , attrVal = -2
attrName = text , attrVal = 测试Attr
```

也就是我们在布局文件中配置的三个属性。

## defStyleAttr

先来看看源码文档中对 defStyleAttr 的解释：

```
     * @param defStyleAttr An attribute in the current theme that contains a
     *        reference to a style resource that supplies default values for
     *        the view. Can be 0 to not look for defaults.
```

defStyleAttr 是定义在 theme 中的一个引用，这个引用指向一个style资源，而这个style资源包含了一些属性的默认值，如果我们在布局中没有为 View 设置的一些属性（attrs 中没有的属性），就会在 defStyleAttr 查找默认值。如果设置为 0 就不会去查找默认属性。

我们先在 attrs.xml 中声明一个自定义的属性：

```
    <declare-styleable name="myViewStyle">
        <attr name="myCutsomViewStyle" format="reference" />
    </declare-styleable>
```

然后在 styles.xml 中声明一个 View 的默认样式，然后把这个样式在 Activity 的主题中为 `myCutsomViewStyle` 赋值：

```
    <style name="customText2">
        <item name="android:textSize">40sp</item>
        <item name="android:background">#ff0000ff</item>
    </style>
    
    <style name="TestActivityTheme" parent="Theme.AppCompat">
        <item name="android:textColor">#FFFF0000</item>
        <item name="android:textSize">10sp</item>
        <item name="myCutsomViewStyle">@style/customText2</item>
    </style>
```

然后在构造方法中为 defStyleAttr 设置：

```
    public TestTextView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, R.attr.myCutsomViewStyle);
    }
```

## defStyleRes

先来看看源码文档中对 defStyleRes 的解释：

```
     * @param defStyleRes A resource identifier of a style resource that
     *        supplies default values for the view, used only if
     *        defStyleAttr is 0 or can not be found in the theme. Can be 0
     *        to not look for defaults.
```

defStyleRes 也是为 View 配置的一些默认的属性值，但是仅仅在 defStyleAttr 为 0 或者在 主题中找不到设置的 defStyleAttr 时，defStyleRes 才会生效。如果设置为 0，那么就不会在查找 defStyleRes 配置的默认值。

首先在 styles.xml 中设置一个样式：

```
    <style name="customText">
        <item name="android:textSize">20sp</item>
        <item name="android:textColor">#FF00FF00</item>
        <item name="android:background">#ffff0000</item>
    </style>
```

然后在构造方法中设置：

```
    public TestTextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, R.style.customText);
    }
```

## 优先级

在 [Android -- Attr、Style和Theme](http://www.heqiangfly.com/2015/05/10/android-knowledge-attr-style-theme/) 一文中介绍了属性使用的优先级，那么现在我们再加上一个：布局文件中设置的值 > View 构造方法中的 defStyleAttr（不设置 defStyleAttr 时是defStyleRes） > View style > Activity theme > Application theme。