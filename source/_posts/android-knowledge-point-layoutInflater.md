---
title: Android -- 使用 LayoutInflater
categories: Android
comments: true
tags: [Android, LayoutInflater]
description: 使用 LayoutInflater 的一些知识点
date: 2016-2-18 10:00:00
---

## LayoutInflater root 参数的使用

我们通常使用`addView`这个方法时，会先通过`LayoutInflater`的`inflate`生成一个`View`视图，然后添加到当前`ViewGroup`中，如果使用不恰当，就会出现这样的问题：

```java
        setContentView(R.layout.layout_inflate_test);
        LinearLayout viewGroup = (LinearLayout) findViewById(R.id.root);

        //1.inflate_test根布局layout参数被忽略
//        View v = LayoutInflater.from(this).inflate(R.layout.inflate_test, null);
//        viewGroup.addView(v);

        //2.不会忽略
//        View v = LayoutInflater.from(this).inflate(R.layout.inflate_test, viewGroup, false);
//        viewGroup.addView(v);

        //3.不会忽略
//        LayoutInflater.from(this).inflate(R.layout.inflate_test, viewGroup);

        //4.不会忽略
//        LayoutInflater.from(this).inflate(R.layout.inflate_test, viewGroup, true);
```

上面的代码中，第一种用法根布局 layout 参数会被忽略，后面都不会。我们从 `LayoutInflater` 源码中可以看出来原因，在`public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot)`方法中：

```java
                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }
```

如果`root`不为空，且`attachToRoot`为`false`，会把布局参数`params`加上。

```java
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }
```

如果`root`不为空，且`attachToRoot`为`true`，会通过`addView(temp, params)`方法加上布局参数。
因此，我们不能因为暂时不需要绑定到`root`上面就忽视掉root的作用，没有的话设置的布局参数就不起作用了哦！
比如我们在使用`ListView`的时候就经常碰到，`ListView` 添加`HeaderView`之后尺寸布局被忽略的情况：
通常添加头部的方法是

```java
LayoutInflater lif = (LayoutInflater) getSystemService(Context.LAYOUT_INFLATER_SERVICE);
View headerView = lif.inflate(R.layout.header, null);
mListView.addHeaderView(headerView);
```

原因就是`lif.inflate(R.layout.header, null)`丢失了 XML 布局中根 `View` 的 `LayoutParam`，其实使用下面的方法就可以了：

```java
lif.inflate(R.layout.header, mListView, false);
```

## 设置 LayoutInflater.Factory

在开发中我们可能有遇到这样的需求，需要全局的设置或者修改一些 View 的属性。这种情况下可以考虑使用 `LayoutInflaterCompat` 的 `setFactory` 方法来实现。它可以拦截 `LayoutInflater` 创建 `View` 的过程，我们可以实现一些针对 `View` 的一些客制化操作。
`LayoutInflater` 提供了两个方法：

 - setFactory(Factory factory)
 - setFactory2(Factory2 factory)

Factory2 继承自 Factory，多提供了一个 onCreateView 方法。
Android 提供了一个 `LayoutInflaterCompat` 来完成不同 Android 版本的兼容，比如可以通过下面的方法来修改整个应用的 TextView 的字体类型：

```
        LayoutInflaterCompat.setFactory2(getLayoutInflater(), new LayoutInflater.Factory2() {
            @Override
            public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
                Log.e("Test","LayoutInflaterCompat 1" +name);
                if ("TextView".equals(name)){
                    TextView textView = new TextView(context, attrs);
                    textView.setTypeface(Typeface.create("SFDIN-medium",Typeface.NORMAL));
                    return textView;
                }
                return null;
            }

            @Override
            public View onCreateView(String name, Context context, AttributeSet attrs) {
                Log.e("Test","LayoutInflaterCompat 2" +name);
                return null;
            }
        });
```

如果我们不想自定义View，只是想修改View的一些属性的话可以采用下面的方法：

```
            @Override
            public View onCreateView(View parent, String name, Context context, AttributeSet attrs) {
                Log.e("Test","LayoutInflaterCompat 1" +name);
                if ("TextView".equals(name)){
                    //TextView textView = new TextView(context, attrs);
                    View view = getDelegate().createView(parent,name,context,attrs);
                    if (view instanceof TextView) {
                        ((TextView)view).setTypeface(Typeface.create("SFDIN-medium",Typeface.NORMAL));
                        return view;
                    }
                }
                return null;
            }
```

`AppCompatDelegate.createView` 会通过一系列方法调用链，创建系统默认的 View。
如果我们需要修改View，那么就返回我们自定义的 View，如果返回 null 的话就会用系统默认的构造方式来构造View。
下面是 LayoutInflater 中用默认Factory来构造View的代码：

```
            if (view == null && mPrivateFactory != null) {
                view = mPrivateFactory.onCreateView(parent, name, context, attrs);
            }
```

`LayoutInflaterCompat.setFactory2` 一定要设置 `super.onCreate(savedInstanceState);` 的前面，否则会报错。