---
title: OpenGL ES -- 混合
categories: OpenGL
comments: true
tags: [OpenGL ES]
description: 介绍 OpenGL ES 混合
date: 2020-4-30 10:00:00
---

## 概述

Blend 混合是将源色和目标色以某种方式混合生成特效的技术。混合常用来绘制透明或半透明的物体。在混合中起关键作用的α值实际上是将源色和目标色按给定比率进行混合，以达到不同程度的透明。α值为0则完全透明，α值为1则完全不透明。混合操作只能在RGBA模式下进行，颜色索引模式下无法指定α值。物体的绘制顺序会影响到OpenGL的混合处理。

## API 介绍

```
glEnable( GL_BLEND );   // 启用混合
glDisable( GL_BLEND );  // 禁用关闭混合
```
获得混合的信息：

```
glGet( GL_BLEND_SRC );
glGet( GL_BLEND_DST );
glIsEnable( GL_BLEND );

glBlendFunc( GLenum sfactor , GLenum dfactor );         // 混合函数
sfactor 源混合因子
dfactor 目标混合因子
```

混合因子枚举：

|||
| ------------- |:-------------:| 
| GL_DST_ALPHA | ( Ad , Ad , Ad , Ad ) |
| GL_DST_COLOR | ( Rd , Gd , Bd , Ad ) |
| GL_ONE | (1,1,1,1) |
| GL_ONE_MINUS_DST_ALPHA | (1,1,1,1) - (Ad,Ad,Ad,Ad) |
| GL_ONE_MINUS_DST_COLOR | (1,1,1,1) - (Rd,Gd,Bd,Ad) |
| GL_ONE_MINUS_SRC_ALPHA | (1,1,1,1) - (As,As,As,As) |
| GL_SRC_ALPHA | ( As , As , As , As ) |
| GL_SRC_ALPHA_SATURATE | (f,f,f,1) : f = min(As,1-Ad) |
| GL_ZERO | (0, 0, 0, 0) |


glBlendFunc( GL_ONE , GL_ZERO );        // 源色将覆盖目标色
glBlendFunc( GL_ZERO , GL_ONE );        // 目标色将覆盖源色
glBlendFunc( GL_SRC_ALPHA , GL_ONE_MINUS_SRC_ALPHA ); // 是最常使用的

若源色为 ( 1.0 , 0.9 , 0.7 , 0.8 )
源色使用 GL_SRC_ALPHA
即 0.8*1.0 , 0.8*0.9 , 0.8*0.8 , 0.8*0.7
结果为 0.8 , 0.72 , 0.64 , 0.56

目标色为 ( 0.6 , 0.5 , 0.4 , 0.3 )
目标色使用GL_ONE_MINUS_SRC_ALPHA
即 1 - 0.8 = 0.2
0.2*0.6 , 0.2*0.5 , 0.2*0.4 , 0.2*0.3
结果为 0.12 , 0.1 , 0.08 , 0.06
由此而见，使用这个混合函数，源色的α值决定了结果颜色的百分比。
这里源色的α值为0.8，即结果颜色中源色占80%，目标色占20%。

## 绘制两个矩形

我们在[http://www.heqiangfly.com/2020/04/20/opengl-es-smoonth-shade/](http://www.heqiangfly.com/2020/04/20/opengl-es-smoonth-shade/) 一节代码的基础上，来实现一下混合的效果。
这里每个矩形的各个顶点的颜色设置为一样，这样就去掉了前面的平滑着色的效果。
修改顶点数据，我们绘制一个大的矩形，然后再绘制一个小矩形，并为各个顶点增加 Alpha 值。

```
    private float[] tableVerticesWithTriangles = {
            // Order of coordinates: X,Y,R,G,B,A
            // 大矩形
            0f,   0f,     0f,1f,0f,1f,
            -0.5f, -0.5f, 0f,1f,0f,1f,
            0.5f, -0.5f,  0f,1f,0f,1f,
            0.5f, 0.5f,   0f,1f,0f,1f,
            -0.5f, 0.5f,  0f,1f,0f,1f,
            -0.5f, -0.5f, 0f,1f,0f,1f,

            // 小矩形
            0.25f,-0.25f,1f,0f,0f,0.5f,
            0f, -0.5f,   1f,0f,0f,0.5f,
            0.5f, -0.5f, 1f,0f,0f,0.5f,
            0.5f, 0f,    1f,0f,0f,0.5f,
            0f, 0f,      1f,0f,0f,0.5f,
            0f, -0.5f,   1f,0f,0f,0.5f,

    };
```

由于表示颜色的分量变成了 4 位，因此要修改 ：

```
    private static final int BYTES_PER_FLOAT = 4;
```

修改 onDrawFrame 方法，增加对小矩形的绘制：

```

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);

        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_FAN, 0 , 6);
        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_FAN, 6 , 6);

    }
```

我们先来看一下绘制后的效果：

<img src="/images/opengl-es-color-blend/disable-blend.png" width="200" height="400"/>

可见，小矩形并未实现透明混合的效果。

## 实现混合效果

我们在 `onSurfaceCreated` 中增加下面的代码，来开启混合：

```
        GLES20.glEnable(GL_BLEND);
        GLES20.glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA);
```

来看一下绘制后的效果：

<img src="/images/opengl-es-color-blend/enable-blend.png" width="200" height="400"/>

已经实现了颜色的混合。