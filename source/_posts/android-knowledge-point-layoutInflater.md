---
title: Android：使用LayoutInflater应该注意的问题
categories: Android
comments: true
tags: [Android, LayoutInflater]
description: 通过解决在使用ViewGroup.addView()添加LayoutInflater生成的布局文件时遇到的布局文件的layout参数被忽略的问题来介绍如何正确使用LayoutInflater
date: 2016-2-15 10:00:00
---

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

