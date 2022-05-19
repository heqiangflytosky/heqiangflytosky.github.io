---
title: Android -- 单元测试
categories: Android
comments: true
tags: [单元测试]
description: 如何在 Android 开发中进行单元测试
date: 2016-7-6 10:00:00
---


## 概述

单元测试是我们应用程序测试中基本的测试，通过对代码进行单元测试，可以轻松发现我们开发的程序是否存在问题。通过在开发过程中把一个大的需求进行分解测试，可以有助于我们及时发现问题，保证代码质量。单元测试程序代码跟随我们的需求进行版本迭代，不断完善我们的单元测试案例，也方便我们在代码重构时验证代码质量。
[Android 官方文档](https://developer.android.com/studio/test)也为我们如何进行单元测试进行了指导。

## 单元测试分类

Android 中的单元测试基于 JUnit，可分为 **本地测试** 和 **instrumented 测试**。

| 类型 | 本地测试 | instrumented 测试|
|:-------------:|:-------------:|:-------------:|
| 代码路径 | src/test/java/ | src/androidTest/java/ |
| 特点 | 运行在本地JVM上，不需要设备或模拟器的支持，但是无法直接运行含有android系统API引用的测试代码。 | 需要运行在android设备或模拟器下面，因此可以使用android系统的API |
| 优点 | 速度快，使用简单 | 可以测试Android相关API |
| 缺点 | 无法测试Android相关API | 速度慢 |
| 框架 | UIAutomator，Robotium，Espresso，Macaca，Appium | Mockito,EasyMockito,Jmockit Powermock,Robolectric |

## 如何进行单元测试

在我们新建一个Android工程时，Android Studio 会默认为我们添加单元测试相关的依赖和创建一些单元测试目录和代码：

```
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'com.android.support.test:runner:1.0.1'
    androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.1'
```

<img src="/images/android-unit-test/dir.png" width="415" height="82"/>

### 新建单元测试

如何为一个类或者一个类中的某些方法创建单元测试案例呢？

 1.打开包含要测试的代码的 Java 文件。
 2.点击要测试的类或方法，然后按 Ctrl+Shift+T 键，在弹出菜单中点击 Create New Test...。或者点击右键，在弹出菜单中选择 Generate...，再在弹出菜单中选择 Test...。
 3.然后再弹出对话框中选择要生成的任何方法。点击OK。
 <img src="/images/android-unit-test/create-test.png" width="511" height="486"/>
 4.在弹出的对话框中选择生成测试类的路径。点击OK。
 <img src="/images/android-unit-test/choose-dir.png" width="411" height="426"/>

我们还可以手动生成一些通用的测试案例，即不是依托某个类来生成。

 1.选择 androidTest 文件夹或者 test 文件夹，右键->New->Java class，和生成普通的类相同。
 2.添加测试方法。测试方法要记得加上相应的注解。
 
### 运行单元测试

#### IDE 中运行

 - 运行单个方法的单元测试：选中方法名，右键单击，选择 Run 或者 Debug
 - 运行单个类的单元测试：在类文件中选择类名，或者在Project 中选择类，右键单击，选择 Run 或者 Debug
 - 运行一个目录下的所有测试类：选择这个目录，右键单击，选择 Run 或者 Debug

#### 命令行运行


#### 查看测试结果

在 Run 窗口中查看测试结果


## JUnit 介绍

### 常用注解

| 注解名称 | 解释 |
|:-------------:|:-------------:|
|@Test |	表示此方法为测试方法|
|@Before |	在每个测试方法前执行，可做初始化操作|
|@After |	在每个测试方法后执行，可做释放资源操作|
|@Ignore |	忽略的测试方法|
|@BeforeClass |	在类中所有方法前运行。此注解修饰的方法必须是static void|
|@AfterClass |	在类中最后运行。此注解修饰的方法必须是static void|
|@RunWith |	指定该测试类使用某个运行器|
|@Parameters |	指定测试类的测试数据集合|
|@Rule 	|重新制定测试类中方法的行为|
|@FixMethodOrder |	指定测试类中方法的执行顺序|

## 测试异步方法

由于 JUnit 本身的设计，当每个测试方法执行结束时，该方法的运行线程会一并 kill 掉, 因此对于异步调用的方法，子线程会一并回收，回调函数也无法执行。
我们可以采取下面的方法来把异步调用改为同步调用。

### 使用 CountDownLatch

CountDownLatch 的使用参考我的博客[Java 并发编程 -- CountDownLatch ](http://www.heqiangfly.com/2015/07/25/java-thread-how-to-use-countdownlatch/)。

```
    @Test
    public void getAppInfo() {
        if (mCountDownLatch != null && mCountDownLatch.getCount() > 0) {
            return;
        }
        mCountDownLatch = new CountDownLatch(1);
        Context appContext = InstrumentationRegistry.getTargetContext();

        AppHelper.getAppInfo(appContext, "com.hq.game", new CallBack<AppBean>() {
            @Override
            public void onSuccess(AppBean appBean) {
                Log.e("Test",appBean.getName());
                if (mCountDownLatch != null && mCountDownLatch.getCount() > 0) {
                    mCountDownLatch.countDown();
                }

            }

            @Override
            public void onFail(Throwable e) {
                Log.e("Test","error",e);
            }
        });
        Log.e("Test","getAppInfo");
        try {
            mCountDownLatch.await(2, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            mCountDownLatch = null;
        }
        Log.e("Test","end");
    }
```

