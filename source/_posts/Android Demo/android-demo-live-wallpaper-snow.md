---
title: Android 动态壁纸
categories: Android Demo
comments: true
tags: [Android Demo]
description: 通过具体例子来介绍Android动态壁纸的实现
date: 2017-3-10 10:00:00
---

[Github源码地址](https://github.com/heqiangflytosky/LiveWallPaper)

## 概述

动态壁纸，顾名思义就是有动画效果的壁纸，它是针对静态壁纸的一种表述方式，它不但可以实现动态效果，而且还可以与用户的触摸事件进行交互。
其实在Android的源码里面已经有一些动态壁纸的Demo，Android 内置的动态壁纸都在 packages/wallpapers/ 这个目录里，其中LivePicker目录里包含的是动态墙纸的选择列表的代码，也就是你在桌面选择添加动态墙纸时出现的系统里所有动态墙纸的那个列表的实现代码。
如果你想自己来实现一个动态壁纸也是一件简单的事情，本文中实现了一个雪花飞舞的动态壁纸效果，而且具有与用户交互的功能，当触摸屏幕时会在触摸点的位置生成一个雪花。

## 效果

<img src="https://github.com/heqiangflytosky/LiveWallPaper/raw/master/img/livewallpaper_snow.gif"/>

## 代码

下面先来看一下AndroidManifest.xml文件：
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest package="com.android.hq.livewallpaper"
          xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET"/>

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity"
                  android:label="@string/app_name"
                  android:hardwareAccelerated="true"

                  android:uiOptions="splitActionBarWhenNarrow"
                  android:theme="@style/AppTheme" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <service android:name=".SnowWallPaper"
                 android:label="雪花飞舞"
                 android:permission="android.permission.BIND_WALLPAPER">
            <intent-filter>
                <action android:name="android.service.wallpaper.WallpaperService" />
            </intent-filter>
            <meta-data android:name="android.service.wallpaper" android:resource="@xml/snow" />

        </service>
    </application>

</manifest>
```
 - `android:label="雪花飞舞"`可以为壁纸设置一个名称
 - `android:permission="android.permission.BIND_WALLPAPER"` 允许绑定壁纸权限，必须添加
 - `action android:name="android.service.wallpaper.WallpaperService"` 这个action把其当做一个动态壁纸加载进动态壁纸选择器的列表，必须添加

xml/snow.xml
```xml
<?xml version="1.0" encoding="utf-8"?>

<wallpaper xmlns:android="http://schemas.android.com/apk/res/android"
           android:thumbnail="@drawable/snow_thumbnail"
           android:author="@string/author"/>
```
`android:thumbnail`可以为壁纸设置一个缩略图，可以在选择动态壁纸的时候显示

WallpaperService.java
```java
public class SnowWallPaper extends WallpaperService {
    private final static String TAG = "WallpaperService";

    @Override
    public void onCreate() {
        super.onCreate();
    }

    // 必须实现这个方法来返回我们自定义引擎的实例
    @Override
    public Engine onCreateEngine() {
        return new SnowEngine();
    }

    //自定义引擎类
    public class SnowEngine extends Engine {
        private SnowUtils mSnowUtils;
        private Handler mHandler = new MyHandler();
        Bitmap mBackground;

        public static final int MSG_DRAW_FRAME = 1;
        public static final int MSG_PRODUCE_SNOW = 2;

        public SnowEngine() {
            super();
        }

        @Override
        public void onCreate(SurfaceHolder surfaceHolder) {
            super.onCreate(surfaceHolder);
            setTouchEventsEnabled(true);
        }

        // 手指触摸屏幕时会调用此方法
        @Override
        public void onTouchEvent(MotionEvent event) {
            super.onTouchEvent(event);
            if(event.getAction() == MotionEvent.ACTION_DOWN){
                mSnowUtils.produceInstantFlake((int)event.getX(), (int)event.getY());
            }
        }

        //壁纸由隐藏转换为显示状态时会调用这个方法
        @Override
        public void onVisibilityChanged(boolean visible) {
            super.onVisibilityChanged(visible);
            if (visible) {
                mHandler.obtainMessage(MSG_DRAW_FRAME).sendToTarget();
                mHandler.obtainMessage(MSG_PRODUCE_SNOW).sendToTarget();
            }
            //当壁纸隐藏时不再进行绘制
            else {
                mHandler.removeCallbacksAndMessages(null);
            }
        }

        @Override
        public void onSurfaceChanged(SurfaceHolder holder, int format, int width, int height) {
            super.onSurfaceChanged(holder, format, width, height);
            drawFrame();
        }

        @Override
        public void onOffsetsChanged(float xOffset, float yOffset, float xOffsetStep, float yOffsetStep, int xPixelOffset, int yPixelOffset) {
            super.onOffsetsChanged(xOffset, yOffset, xOffsetStep, yOffsetStep, xPixelOffset, yPixelOffset);
            drawFrame();
        }

        @Override
        public void onSurfaceDestroyed(SurfaceHolder holder) {
            super.onSurfaceDestroyed(holder);
            mHandler.removeCallbacksAndMessages(null);
        }

        // 具体的绘制都在这个方法里面实现
        private void drawFrame(){

            SurfaceHolder sh = getSurfaceHolder();
            final Rect frame = sh.getSurfaceFrame();
            final int dw = frame.width();
            final int dh = frame.height();

            if(mSnowUtils == null){
                mSnowUtils = new SnowUtils(getApplicationContext());
                mSnowUtils.init(dw, dh);
            }

            if (mBackground == null) {
                mBackground = BitmapFactory.decodeResource(getResources(),R.drawable.snow_bg);
            }

            Canvas c = sh.lockCanvas();
            if(c != null){
                try {
                    // 先绘制一个静态的背景图片
                    drawBackground(c,dw,dh);
                    // 绘制动态的雪花
                    mSnowUtils.draw(c);
                }finally {
                    sh.unlockCanvasAndPost(c);
                }
            }
        }

        private void drawBackground(Canvas c,int w, int h){
            if (mBackground != null) {
                RectF dest = new RectF(0, 0, w, h);
                c.drawBitmap(mBackground, null, dest, null);
            }
        }

        public class MyHandler extends Handler{
            @Override
            public void handleMessage(Message msg) {
                switch (msg.what){
                    case MSG_DRAW_FRAME:
                        drawFrame();
                        removeMessages(MSG_DRAW_FRAME);
                        sendMessageDelayed(obtainMessage(MSG_DRAW_FRAME), 50);
                        break;
                    case MSG_PRODUCE_SNOW:
                        mSnowUtils.produceSnowFlake();
                        removeMessages(MSG_PRODUCE_SNOW);
                        sendMessageDelayed(obtainMessage(MSG_PRODUCE_SNOW), 1000);
                        break;
                    default:
                        break;
                }
            }
        }
    }
}
```
从代码中我们可以看到，`drawFrame()`方法承担了具体的绘制工作，绘制工作分为两部分，一部分是绘制静态的背景，第二部分是绘制动态的雪花，如果去掉第二部分，就是一个静态壁纸了。
其实静态壁纸和动态壁纸是一样的，静态壁纸只是绘制一张图片，动态背景就要不断的绘制动画。
具体介绍参考我的博客[]()。
