---
title: OpenGL ES -- 图像处理
categories: OpenGL
comments: true
tags: [OpenGL ES]
description: 介绍 OpenGL ES 2.0 实现一些滤镜效果
date: 2020-5-6 10:00:00
---

## 概述

滤镜实际上就是在片元着色器对特定的像素点进行处理。现在的很多美颜相机都有滤镜的功能，都用到了 OpenGL 来实现滤镜。那么接下来我们就看看这些滤镜是如何实现的。
滤镜并不需要操作顶点着色器，所以不同滤镜顶点着色器代码是一致的。
[源码](https://github.com/heqiangflytosky/AndroidOpenGLESDemo)

## 黑白滤镜

黑白滤镜也叫灰色滤镜。我们都知道，一个像素点由4个部分组成：分别是R（红色通道）、G（绿色通道）、B（蓝色通道）、A（透明通道）。通过这4部分不同值的调配，组成了色彩斑斓的颜色。
但是在黑白图片上，每个像素点的RGB三个通道值应该是相等的。知道了这个，将彩色图片处理成黑白图片就非常简单了。
我们直接取出像素点的RGB三个通道，再选择合适的灰度算法对读取到的色彩值进行调整，将计算所得赋给 gl_FragColor。
黑白滤镜通常有下面的算法：

 - 平均值法：Gray = (R+G+B)/3
 - 仅取绿色：Gray = G
 - 权值方法：Gray = R * 0.3 + G * 0.59 + B * 0.11

一般由于人眼对不同颜色的敏感度不一样，所以三种颜色值的权重不一样，一般来说绿色最高，红色其次，蓝色最低，最合理的取值分别为Wr ＝ 30％，Wg ＝ 59％，Wb ＝ 11％，所以权值法相对效果更好一点。

片元着色器核心代码：

```
            vec4 nColor = texture2D(u_TextureUnit, v_TextureCoordinates);
            // 黑白效果
            float c=nColor.r*u_ChangeColor.r+nColor.g*u_ChangeColor.g+nColor.b*u_ChangeColor.b;
            gl_FragColor=vec4(c,c,c,nColor.a);
            // 使用内置函数dot实现点积
            //float c = dot(nColor.rgb, u_ChangeColor);
            //gl_FragColor = vec4(vec3(c), 1.0);
```

原图：

<img src="/images/opengl-es-image-effect/0.png" width="342" height="239"/>

黑白滤镜效果：

<img src="/images/opengl-es-image-effect/1.png" width="342" height="239"/>

## 暖色滤镜和冷色滤镜

我们知道，红色和绿色属于暖色系，而蓝色属于冷色系，因此实现冷暖滤镜其实也很简单，取出每个像素点的值，然后增加对应通道的值，就可以实现这样的效果。
冷色调就是单一增加蓝色通道的值，暖色调是增加红/绿通道的值。

片元着色器核心代码：

```
void modifyColor(vec4 color){
    color.r=max(min(color.r,1.0),0.0);
    color.g=max(min(color.g,1.0),0.0);
    color.b=max(min(color.b,1.0),0.0);
    color.a=max(min(color.a,1.0),0.0);
}

            // 冷色调({0.0f,0.0f,0.1f})、暖色调({0.1f,0.1f,0.0f})效果
            vec4 deltaColor=nColor+vec4(u_ChangeColor,0.0);
            modifyColor(deltaColor);
            gl_FragColor=deltaColor;
```

冷色滤镜效果：

<img src="/images/opengl-es-image-effect/2.png" width="342" height="239"/>

暖色滤镜效果：

<img src="/images/opengl-es-image-effect/3.png" width="342" height="239"/>

## 模糊效果

图片模糊处理相对上面的色调处理稍微复杂一点，通常图片模糊处理是采集周边多个点，然后利用这些点的色彩和这个点自身的色彩进行计算，得到一个新的色彩值作为目标色彩。模糊处理有很多算法，类似高斯模糊、径向模糊等等。

模糊滤镜效果：

<img src="/images/opengl-es-image-effect/4.png" width="342" height="239"/>

## 放大镜

放大镜效果相对模糊处理来说，处理过程也会相对简单一些。我们只需要将制定区域的像素点，都以需要放大的区域中心点为中心，向外延伸其到这个中心的距离即可实现放大效果。

放大镜效果：

<img src="/images/opengl-es-image-effect/5.png" width="342" height="239"/>

## 一半处理

Demo 中对以上的效果有一个选项，叫处理一半的效果，这个其实很简单，就是当x坐标小于0时不加任何效果，x坐标大于0时再做对应的效果。

<img src="/images/opengl-es-image-effect/6.png" width="342" height="239"/>

<img src="/images/opengl-es-image-effect/7.png" width="342" height="239"/>

<img src="/images/opengl-es-image-effect/8.png" width="342" height="239"/>

