---
title: Android ContentProvider 使用以及源码分析
categories: Android
comments: true
tags: [ContentProvider]
description: 本文介绍 ContentProvider 的基本使用
date: 2015-1-26 10:00:00
---

[ContentProvider + SQLite 使用Demo GitHub 源码](https://github.com/heqiangflytosky/AndroidDBDemo)

## 概述

ContentProvider 内容提供者，作为 Android 的四大组件之一，我们会经常使用到。它是 Android 中提供的用于数据共享的组件，你可以跨进城、跨应用来访问这些数据。
对于数据的存储，Android 提供了多种方式供我们选择：SQLite 数据库、Shared Preferences、文件存储等。如果在同一个应用中我们可以直接共享使用这些数据，但是如果时不同的应用需要共享这些数据，那么就需要借助于 ContentProvider。我们平时最常见的是 ContentProvider + SQLite 数据库的组合，当然 ContentProvider 也可以和其他的数据存储方式相结合。
当外部应用需要对 ContentProvider 里面的数据进行增删改查操作时就需要用到 ContentResolver，Andriod 的 Context 提供了 getContentResolver() 方法来获取 ContentResolver 对象。
如果我们使用比如 ContentProvider + Shared Preferences 的组合时，你可能需要将数据转换为 Cursor 返回给 ContentProvider 提供给其他应用查询，那么本文也将介绍如果创建一个 Cursor，通常我们使用 MatrixCursor。

## ContentProvider

创建我们自己的 ContentProvider 通常需要继承 ContentProvider 类并实现它的六个抽象方法。
ContentProvider 提供了下面的四个抽象方法进行增删改查操作，

```
public abstract @Nullable Uri insert(@NonNull Uri uri, @Nullable ContentValues values);
public abstract int delete(@NonNull Uri uri, @Nullable String selection,
            @Nullable String[] selectionArgs);
public abstract int update(@NonNull Uri uri, @Nullable ContentValues values,
            @Nullable String selection, @Nullable String[] selectionArgs);
public abstract @Nullable Cursor query(@NonNull Uri uri, @Nullable String[] projection,
            @Nullable String selection, @Nullable String[] selectionArgs,
            @Nullable String sortOrder);
```

还有下面两个抽象方法：

```
public abstract boolean onCreate();
public abstract @Nullable String getType(@NonNull Uri uri);
```

 - insert：添加数据。
 - delete：删除数据。
 - update：更新数据。
 - query：查询数据。
 - onCreate：ContentProvider 初始化时调用，一般做一些初始化操作。
 - getType：返回参数中 uri 对应的 MIME 类型，格式比较固定：单条记录为 vnd.android.cursor.item/ 开头的字符串，多条记录为 vnd.android.cursor.dir/ 开头的字符串。一般情况下可以不用去关注，返回null即可。在使用 Intent 时有用。

增删改查方法参数说明：

| 参数名字 | 描述 | 是否可以为空 |
|:-------------|:-------------|:---------|
| uri | 为数据资源的位置，比如想查询某个表的数据或者想执行某个操作：content://hq.testprovider/table1/ | 不能为空 |
| values | 想要增加或更新的数据实体 | 是 |
| projection | 来表示查询数据时要返回数据的列。比如获取 Student 表中的数据时我们只想要 Student 的名字和地址，那么 projection参数就可以如下表示：`new String []{DBOpenHelper.StudentTAB.NAME,DBOpenHelper.StudentTAB.ADDR}`，那么返回的 Cursor 中的数据就只有名字和地址两列。 | 可以为空，表示查询所有列 |
| selection | 设置的筛选条件，表示接下来的删改查操作只针对符合条件的行进行。`DBOpenHelper.StudentTAB.COUNTRY + "=? AND " + DBOpenHelper.StudentTAB.GENDER + "=?"` 表示指操作指定国家或者性别的行。或者 `DBOpenHelper.StudentTAB.COUNTRY + "='China' AND " + DBOpenHelper.StudentTAB.GENDER + "='male'"` | 可以为空，表示，不进行筛选，查询所有列 |
| selectionArgs | 配合 selection 使用的参数，将会替换掉 selection 参数中的 `?`：`new String []{"China", "male"}` 。当然也可以写在 selection，效果也是一样的。 | 可以为空 |
| sortOrder | 为返回数据的排序方式，比如是升序还是降序 | 可以为空 |

然后需要在 AndroidManifest.xml 中对 ContentProvider 进行注册

```
        <provider
            android:authorities="hq.testprovider"
            android:name=".contentprovider.MyContentProvider"
            android:exported="true"/>
```

 - android:authorities：ContentProvider 的唯一标识符，外部应用可以通过URI中的此属性值来找到想要访问的 ContentProvider，建议用包名来标示，当然你也可以随意指定，但要有唯一性。
 - android:name： ContentProvider 的类名
 - android:exported：是否允许其他应用访问该 ContentProvider，true为允许，false为禁止。
 - android:multiProcess：是否允许在调用者进程启动 ContentProvider。一般情况下，ContentProvider 都是在创建者的进程启动的，如果该进程没有启动，那么会把该进程启动。如果设置了这个属性，就可以在调用者进程启动。但前提是调用者和创建者的 android:sharedUserId 相同才行，否则还是不能在调用者进程启动 ContentProvider。

## ContentResolver

我们使用 ContentResolver 来对 ContentProvider 的数据进行增删改查的操作。
ContentResolver 对象可以通过 `Context.getContentResolver()` 来获取，它提供了一系列针对 ContentProvider 操作的方法，和 ContentProvider 里面的方法一一对应。

## SQLiteOpenHelper

数据库操作帮助类，如果我们的 ContentProvider 时结合 SQLite 数据库来实现，那么就会用到这个类。

## URI

上面介绍的几个 ContentProvider 几个方法中，大多都有一个 URI 的参数，它表示我们想操作的数据资源的位置。比如：`content://hq.testprovider/table1/`。
完整的 URI 一般由下面三部分组成：
`<standard_prefix>://<authority>/<data_path>/`

 - standard_prefix：URI 的前缀，ContentProvider 的前缀为 content://
 - authority：URI 的标识，用于唯一标识这个 ContentProvider。指的是我们在 AndroidManifest.xml 定义的 android:authorities 属性。
 - data_path：路径。一般为要执行的操作的名称或者数据库中的表的名字，也可以自己定义，只要在使用时保持一致就行了。

因此，URI 中的数据包含了需要操作哪个 ContentProvider、以及对 ContentProvider 中哪些数据进行操作的信息。
如果我们是使用 SQLite 来存储数据，那么我们数据库中可能包含多个表，那么 URI 就既要包含操作哪个数据库，也要包含操作数据库中的哪个表。
我们需要执行的一些动作或者调用的方法也可以包含在 URI 中。
那么如何确定一个 UIR 是要执行哪项操作呢？这里需要用到 UriMatcher 来帮忙了。

## MatrixCursor

如果我们数据存储用的不是数据库，我们就需要把数据封装成 Cursor 来提供给 ContentProvider。
当我们需要自己创建 Cursor 时，可以使用 Android 为我们提供的 MatrixCursor 类。
下面的例子演示了如何创建一个 MatrixCursor 实例。

```
        MatrixCursor cursor = new MatrixCursor(new String[]{"NAME", "COUNTRY", "TELEPHONE"});
        cursor.addRow(new Object[] {"ZhangSan","China",13800000000L});
        cursor.addRow(new Object[] {"James","USA",13900000000L});
```

## UriMatcher

UriMatcher 就是用来匹配 URI 的，我们可以在 ContentProvider 中使用 UriMatcher 来注册URI，然后使用 UriMatcher 来匹配 URI 对应的执行动作或者数据表。
UriMatcher 只有两个方法：

 - addURI(String authority, String path, int code)
 - match(Uri uri)

 比如用下面的方法注册 URI：

```
    static final UriMatcher URI_MATCHER = new UriMatcher(UriMatcher.NO_MATCH);
    static {
        final UriMatcher matcher = URI_MATCHER;
        matcher.addURI("hq.testprovider", "students", 0x0001);
        matcher.addURI("hq.testprovider", "teachers", 0x0002);
    }
```

然后，根据 URI 来执行对应的动作或者获取对应的数据表进行操作：

```
int match = URI_MATCHER.match(uri);
```


## ContentProvider 源码分析

<!--  


https://www.baidu.com/s?word=android++contentprovider&tn=50000021_hao_pg&ie=utf-8&sc=UWd1pgw-pA7EnHc1FMfqnHRYn1bLrjTYnW0knBuW5y99U1Dznzu9m1YznWR4nWDvPH61&ssl_sample=s_11%2Cs_109&srcqid=2634579022130942498&H123Tmp=nu
https://blog.csdn.net/zhang_jun_ling/article/details/51288441
https://blog.csdn.net/f2006116/article/details/78386870
https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653579366&idx=1&sn=49361b16d275db8e7f18af44c894e2e8&chksm=84b3ba61b3c4337789c87a7249eb6185765b1fbe80846661965c44c9084976b7e26bdea607ec&mpshare=1&scene=23&srcid=10134b55yuq4SR3vfPyzdGuI#rd

-->
