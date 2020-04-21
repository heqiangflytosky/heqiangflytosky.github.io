---
title: OpenGL ES -- 调整屏幕宽高比
categories: OpenGL
comments: true
tags: [OpenGL ES]
description: 介绍 OpenGL ES 如何适配屏幕宽高比
date: 2020-4-25 10:00:00
---

## 概述

在前面我们介绍的例子的基础上，我们旋转手机屏幕，你会发现，在横屏模式下，图像被压扁了。
之所以出现这样的问题，是因为我们之前把坐标传递给 OpenGL 时，没有考虑屏幕的宽高比。
在 OpenGL 里面我们都会归一化坐标设备，我们要渲染的一切物体都要映射到 x 轴和 y 轴上 [-1, 1]的范围内，独立于屏幕的实际尺寸或者形状。这种情况下，当屏幕的宽高比发生变化时，物体的形状就会发生变化。
这个问题有个通用的解决方案：在 OpenGL 里，我们可以使用投影把真实世界的一部分映射到屏幕上，以这种方式映射会使它在不同的屏幕尺寸或方向上看起来总是正确的。

## 适应宽高比

我们需要调整坐标空间，以使它把屏幕的形状考虑在内，可行的一个方法是把较小的范围固定在[-1,1]内，而按屏幕尺寸的比例调整较大的范围。
举例来说，在竖屏情况下，其宽度是720，而高度是1280，因此我们可以把宽度范围限定在[-1,1]，并把高度范围调整为[-1280/720,1280/720]或[-1.78,1.78]。同理在横屏模式情况下，把高度范围设为[-1.78,1.78]，而把高度范围设为[-1,1]。
通过调整已有的坐标空间，最终会改变我们可用的空间。
通过这个方法，不论是竖屏模式还是横屏模式，物体看起来就都一样了。
这里就要使用一种正交投影的方法。
理论知识这里不做过多的介绍。

## 更新程序

### 更新着色器

顶点着色器：

```
uniform mat4 u_Matrix;

attribute vec4 a_Position;
attribute vec4 a_Color;

varying vec4 v_Color;

void main(){
    v_Color = a_Color;

    gl_Position = u_Matrix * a_Position;
    gl_PointSize = 10.0;
}
```

我们这里添加了一个新的uniform定义的“u_Matrix”，并把它定义为一个mat4类型，意思是这个uniform代表一个4*4的矩阵。
传递位置与一个矩阵的乘积，意味着顶点数组不用再被翻译为归一化设备的坐标了，其将被理解为存在于这个矩阵所定义的虚拟坐标空间中。这个矩阵会把坐标从虚拟坐标空间变化回归一化设备坐标。
在 render 代码中新增一个顶点数组存储矩阵：

```
private final float[] projectionMatrix = new float[16];
```

增加 `uMatrixLocation` 用于保存矩阵uniform的位置。

### 创建正交投影矩阵

在 `onSurfaceChanged` 中创建一个正交投影矩阵：

```
    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0,0,width,height);
        final float aspectRatio = width > height ?
                (float)width/(float)height : (float)height/(float)width;
        if (width > height) {
            Matrix.orthoM(projectionMatrix, 0, -aspectRatio,aspectRatio,-1f,1f,-1f,1f);
        } else {
            Matrix.orthoM(projectionMatrix, 0, -1f,1f, -aspectRatio,aspectRatio,-1f,1f);
        }
    }
```

### 传递矩阵给着色器

最重要的一步是需要把我们生成的正交投影矩阵给着色器，在 `onDrawFrame` 中加入：

```
GLES20.glUniformMatrix4fv(uMatirxLocation, 1, false, projectionMatrix, 0);
```

完成后再旋转屏幕进行观察，发现横竖屏模式下会得到一样的图形了。

## 完整代码

```
public class ThirdRender implements GLSurfaceView.Renderer {
    private Context mContext;
    private static final int POSITOIN_COMPONENT_COUNT = 2;
    private static final int COLOR_COMPONENT_COUNT = 3;
    private static final int BYTES_PER_FLOAT = 4;
    private static final int STRIDE = (POSITOIN_COMPONENT_COUNT + COLOR_COMPONENT_COUNT) * BYTES_PER_FLOAT;
    private static final String A_COLOR = "a_Color";
    private static final String U_COLOR = "u_Color";
    private static final String A_POSITION = "a_Position";
    private static final String U_MATRIX = "u_Matrix";

    private float[] tableVerticesWithTriangles = {
            // x,y, r,g,b
            0f,   0f, 1f, 1f, 1f,
            -0.5f, -0.8f, 0.7f,0.7f,0.7f,
            0.5f, -0.8f, 0.7f,0.7f,0.7f,
            0.5f, 0.8f, 0.7f,0.7f,0.7f,
            -0.5f, 0.8f, 0.7f,0.7f,0.7f,
            -0.5f, -0.8f, 0.7f,0.7f,0.7f,

            -0.5f, 0f, 1f,0f,0f,
            0.5f, 0f,1f,0f,0f,

            0f, -0.4f,0f,0f,1f,
            0f, 0.4f,1f,0f,0f
    };
    private final float[] projectionMatrix = new float[16];

    private FloatBuffer vertexData;
    private int mPrograme;
    private int uColorLocation;
    private int aPosition;
    private int aColorLocation;
    private int uMatirxLocation;

    public ThirdRender(Context context) {
        mContext = context;
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        GLES20.glClearColor(0.0f,0,0,0);

        vertexData = ByteBuffer.allocateDirect(tableVerticesWithTriangles.length * BYTES_PER_FLOAT)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();

        vertexData.put(tableVerticesWithTriangles);

        String vertexShaderCode = Utils.fileLoader(mContext, R.raw.sample_airhoockey_vertex_shader);
        String fragmentShaderCode = Utils.fileLoader(mContext, R.raw.sample_airhoockey_fragment_shader);
        int vertexShader = ShaderHelper.compileVertexShader(vertexShaderCode);
        int fragmentShader = ShaderHelper.compileFragmentShader(fragmentShaderCode);
        mPrograme = ShaderHelper.linkPrograme(vertexShader, fragmentShader);

        ShaderHelper.validatePrograme(mPrograme);
        GLES20.glUseProgram(mPrograme);

        aPosition = GLES20.glGetAttribLocation(mPrograme, A_POSITION);
        aColorLocation = GLES20.glGetAttribLocation(mPrograme, A_COLOR);
        uMatirxLocation = GLES20.glGetUniformLocation(mPrograme, U_MATRIX);

        vertexData.position(0);
        GLES20.glVertexAttribPointer(aPosition, POSITOIN_COMPONENT_COUNT, GLES20.GL_FLOAT,
                false, STRIDE, vertexData);

        GLES20.glEnableVertexAttribArray(aPosition);

        vertexData.position(POSITOIN_COMPONENT_COUNT);
        GLES20.glVertexAttribPointer(aColorLocation, COLOR_COMPONENT_COUNT, GLES20.GL_FLOAT,
                false, STRIDE, vertexData);
        GLES20.glEnableVertexAttribArray(aColorLocation);
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0,0,width,height);
        final float aspectRatio = width > height ?
                (float)width/(float)height : (float)height/(float)width;
        if (width > height) {
            Matrix.orthoM(projectionMatrix, 0, -aspectRatio,aspectRatio,-1f,1f,-1f,1f);
        } else {
            Matrix.orthoM(projectionMatrix, 0, -1f,1f, -aspectRatio,aspectRatio,-1f,1f);
        }
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);

        GLES20.glUniformMatrix4fv(uMatirxLocation, 1, false, projectionMatrix, 0);

        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_FAN, 0 , 6);
        GLES20.glDrawArrays(GLES20.GL_LINES, 6 , 2);
        GLES20.glDrawArrays(GLES20.GL_POINTS, 8 , 1);
        GLES20.glDrawArrays(GLES20.GL_POINTS, 9 , 1);

    }
}
```

着色器代码：

```
uniform mat4 u_Matrix;

attribute vec4 a_Position;
attribute vec4 a_Color;

varying vec4 v_Color;

void main(){
    v_Color = a_Color;

    gl_Position = u_Matrix * a_Position;
    gl_PointSize = 10.0;
}
```

```
precision mediump float;
varying vec4 v_Color;

void main() {
    gl_FragColor = v_Color;
}
```