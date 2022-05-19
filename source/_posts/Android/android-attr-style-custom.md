---
title: Android -- 自定义Attribute和Style
categories: Android
comments: true
tags: [Attribute和Style]
description: 自定义Attribute和Style
date: 2016-7-2 10:00:00
---

## 自定义属性

### 实践

 1. 编写values/custom_attrs.xml，在其中编写styleable和item等标签元素
 2. 自定义一个CustomView(extends View)类
 3. 在布局文件中CustomView使用自定义的属性（注意namespace）
 4. 在CustomView的构造方法中通过TypedArray获取

custom_attrs.xml

```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="CustomAttrs">
        <attr name="custom_src" format="string" />
        <attr name="custom_text" format="string" />
        <attr name="custom_text_size" format="dimension" />
        <attr name="custom_text_color" format="color" />
        <attr name="custom_bg" format="reference|color" />
    </declare-styleable>
</resources>
```

`<declare-styleable name="CustomAttrs">` 这里的自定义属性的名称 name 如果不设置成和自定义 View的名字CustomView 一样的话，我们在xml布局文件里面将不会有这个属性的自动提示。所以这里最好是自定义属性的名称 和 自定义View一致。这该自定义View包含它的子类，在使用自定义属性时会有自动提示。
在布局文件中给 CustomView 配置自定义属性：

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:custom="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center">
    <com.example.heqiang.testsomething.graphic.CustomView
        android:layout_width="100dp"
        android:layout_height="100dp"
        android:background="#ffff0000"
        custom:custom_src="src"
        custom:custom_text="text"
        custom:custom_text_size="16sp"
        custom:custom_text_color="#0000ff"
        custom:custom_bg="@mipmap/ic_launcher"/>
</LinearLayout>
```

注意要在根布局中加入命名空间xmlns:custom，此处的 custom 名字可以随意指定。 

在 CustomView 中获取自定义属性：

```
public class CustomView extends View {
    private static final String TAG = "CustomView";

    public CustomView(Context context) {
        this(context, null);
    }

    public CustomView(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public CustomView(Context context, AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }


    public CustomView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
        init(attrs);
    }

    private void init(AttributeSet attrs) {
        // 获取所有设置的属性值中 R.styleable.CustomAttrs 相关的属性。
        // 如果在 R.styleable.CustomAttrs 中声明了但是在 attrs 中没有设置，返回默认值或null
        TypedArray ta = getContext().obtainStyledAttributes(attrs, R.styleable.CustomAttrs);

        String src = ta.getString(R.styleable.CustomAttrs_custom_src);
        String text = ta.getString(R.styleable.CustomAttrs_custom_text);
        float textSize = ta.getDimension(R.styleable.CustomAttrs_custom_text_size, 10);
        Drawable drawable = ta.getDrawable(R.styleable.CustomAttrs_custom_bg);

        Log.e(TAG, "src = " + src + " , text = " + text + " , textSize = "+textSize);
        Log.e(TAG, "drawable = " + drawable);

        ta.recycle();
}
```

测试打印结果：

```
src = src , text = text , textSize = 48.0
color = -16776961
drawable = android.graphics.drawable.BitmapDrawable@9bd65c6
```

### format选项

 - boolean:布尔值
 - color:颜色值
 - dimension:尺寸值
 - enum:枚举值
 - flag:位或运算
 - float:浮点值
 - fraction:百分数
 - integer:整型值
 - reference:参考某一资源ID
 - string:字符串

flag与enum的差别：flag表示这几个值可以做或运算，你可以叠加使用，如用bold|italic表示既加粗也变成斜体，而enum只能让你选择其中一个值。
format即使用错，只要你自定义的View中获取对应类型值也是可以的，只是在布局中写代码时，IDE就不会根据你定义的format给出相应的提示了，所以最好在自定义View时还是仔细斟酌下类型。

### AttributeSet与TypedArray

AttributeSet中保存的是该View声明的所有的属性，并且外面的确可以通过它去获取（自定义的）属性。但它和通过TypedArray获取到的有什么区别呢？ 
我们再添加下面的代码：

```
        int count = attrs.getAttributeCount();
        for (int i = 0; i < count; i++) {
            String attrName = attrs.getAttributeName(i);
            String attrVal = attrs.getAttributeValue(i);
            Log.e(TAG, "attrName = " + attrName + " , attrVal = " + attrVal);
        }
```

测试打印结果：

```
attrName = background , attrVal = #ffff0000
attrName = layout_width , attrVal = 100.0dip
attrName = layout_height , attrVal = 100.0dip
attrName = custom_bg , attrVal = @2131492864
attrName = custom_src , attrVal = src
attrName = custom_text , attrVal = text
attrName = custom_text_color , attrVal = #ff0000ff
attrName = custom_text_size , attrVal = 16.0sp
```

看到custom_bg这个属性的值   相信大家就明白了，如果某个属性我们用的是引用，那么通过AttributeSet获取的值只是这个资源的ID，而TypedArray获得的是解析过的ID的值。因此使用TypedArray是比较方便的。

## 在自定义Style中加入自定义attr

先在 values/styles.xml 中添加自定义 style：

```
    <style name="CustomStyle">
        <item name="android:background">#ffff0000</item>
        <item name="custom_src">Hello</item>
        <item name="custom_text">@string/hello_world</item>
        <item name="custom_bg">@mipmap/ic_launcher</item>
        <item name="custom_text_size">10sp</item>
        <item name="custom_text_color">#ffffff00</item>
    </style>
```

在布局文件中使用：

```
    <com.example.heqiang.testsomething.graphic.CustomView
        android:layout_width="100dp"
        android:layout_height="100dp"
        style="@style/CustomStyle"/>
```

属性的获取和上面的是一样的。

```
        TypedArray ta = getContext().obtainStyledAttributes(attrs, R.styleable.CustomAttrs);

        String src = ta.getString(R.styleable.CustomAttrs_custom_src);
        String text = ta.getString(R.styleable.CustomAttrs_custom_text);
        float textSize = ta.getDimension(R.styleable.CustomAttrs_custom_text_size, 10);
        int color = ta.getColor(R.styleable.CustomAttrs_custom_text_color, 0x000000);
        Drawable drawable = ta.getDrawable(R.styleable.CustomAttrs_custom_bg);

        Log.e(TAG, "src = " + src + " , text = " + text + " , textSize = "+textSize);
        Log.e(TAG, "color = " + color);
        Log.e(TAG, "drawable = " + drawable);

        ta.recycle();
        //
        int count = attrs.getAttributeCount();
        for (int i = 0; i < count; i++) {
            String attrName = attrs.getAttributeName(i);
            String attrVal = attrs.getAttributeValue(i);
            Log.e(TAG, "attrName = " + attrName + " , attrVal = " + attrVal);
        }
```

结果：

```
src = Hello , text = Hello world! , textSize = 30.0
color = -256
drawable = android.graphics.drawable.BitmapDrawable@1d679e

attrName = layout_width , attrVal = 100.0dip
attrName = layout_height , attrVal = 100.0dip
attrName = style , attrVal = @2131624100
```

## 在自定义Attr中加入自定义Style

我们在 custom_attrs.xml 新加下面的属性：

```
<?xml version="1.0" encoding="utf-8"?>
<resources>
    ......

    <declare-styleable name="CustomTheme">
        <attr name="customTheme" format="reference" />
    </declare-styleable>
</resources>
```

在布局文件中使用：

```
    <com.example.heqiang.testsomething.graphic.CustomView
        android:layout_width="100dp"
        android:layout_height="100dp"
        custom:customTheme="@style/CustomStyle"/>
```

获取属性值：

```
    private void init(AttributeSet attrs,int defStyleAttr, int defStyleRes) {
        TypedArray ta = getContext().obtainStyledAttributes(attrs,
                R.styleable.CustomTheme, defStyleAttr, defStyleRes);
        int customTheme = ta.getResourceId(R.styleable.CustomTheme_customTheme, 0);
        ta.recycle();
        Log.e(TAG, "customTheme=" + customTheme);

        //获取自定义的5个属性
        TypedArray a = getContext().obtainStyledAttributes(customTheme, R.styleable.CustomAttrs);

        String src = null;
        String text = null;
        float textSize = 0f;
        Drawable drawable = null;
        int textColor = 0;

        int N = a.getIndexCount();
        Log.e(TAG, "N="+N);
        for (int i = 0; i < N; i++) {
            int attr = a.getIndex(i);
            if(attr == R.styleable.CustomAttrs_custom_src){
                src = a.getString(R.styleable.CustomAttrs_custom_src);
            } else if(attr == R.styleable.CustomAttrs_custom_text){
                text = a.getString(R.styleable.CustomAttrs_custom_text);
            } else if(attr == R.styleable.CustomAttrs_custom_text_size){
                textSize = a.getDimension(R.styleable.CustomAttrs_custom_text_size, 8);
            } else if(attr == R.styleable.CustomAttrs_custom_bg){
                drawable = a.getDrawable(R.styleable.CustomAttrs_custom_bg);
            } else if(attr == R.styleable.CustomAttrs_custom_text_color){
                textColor = a.getColor(R.styleable.CustomAttrs_custom_text_color,0xffffffff);
            }
        }

        Log.e(TAG, "src = " + src + " , text = " + text + " , textSize = "+textSize);
        Log.e(TAG, "drawable = " + drawable+", textColor = "+textColor);
        a.recycle();

        //获取<item name="android:background">#ffff0000</item>
        TypedArray a1 = getContext().obtainStyledAttributes(customTheme, styleable.View);
        int N1 = a.getIndexCount();
        for (int i = 0; i < N1; i++) {
            int attr = a.getIndex(i);
            if(attr == styleable.View_background){
                Log.e(TAG,"View_background attr "+attr);
                Drawable backgroundDr = a.getDrawable(attr);
                setBackground(backgroundDr);
            }
        }
        a1.recycle();
    }
    
    public static class styleable {
        private static final Class<?> sClassStyleable = getStyleableClass();
        public static int[] View;
        public static int View_background;
        static {
            try {
                View = (int[]) sClassStyleable.getField("View").get(null);

            } catch (Exception e) {
                Log.w(TAG, "", e);
            }

            try {
                View_background = sClassStyleable.getField("View_background").getInt(null);
            } catch (Exception e) {
                Log.w(TAG, "", e);
            }
        }

        private static Class<?> getStyleableClass() {
            try {
                Class<?> clz = Class.forName("com.android.internal.R$styleable");
                return clz;
            } catch (Exception e) {
                Log.e(TAG, "", e);
            }
            return null;
        }
    }
```

结果：

```
customTheme=2131624100
N=5
src = Hello , text = Hello world! , textSize = 30.0
drawable = android.graphics.drawable.BitmapDrawable@9bd65c6, textColor = -256
View_background attr 13
```

## 获取Android原生属性值

参考上面代码中针对android:background属性的获取。

## 相关文章

[Attr、Style和Theme详解](https://www.jianshu.com/p/dd79220b47dd)