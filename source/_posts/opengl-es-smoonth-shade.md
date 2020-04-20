---
title: OpenGL ES -- 平滑着色
categories: OpenGL
comments: true
tags: [OpenGL ES]
description: 介绍 OpenGL ES 如何实现平滑着色
date: 2020-4-20 10:00:00
---

## 概述

在前面一章我们介绍了使用使用 uniform 用单一的颜色绘制一种物体，本章我们将会引人 varying 类型的变量介绍如何实现平滑着色。

## 三角形扇

我们在介绍平滑着色以前，我们先来介绍一下三角形扇。然后我们将会实现一个中间亮色，然后沿对角线逐渐变色的效果。

<img src="/images/opengl-es-smoonth-shade/triangle.png" width="270" height="260"/>

我们在桌子中间加入一个新的点（0，0），最终看拿到了四个三角形，而不是原来的两个。我们修改一下正方形的顶点数据：

```
    private float[] tableVerticesWithTriangles = {
            // 矩形
            0f,   0f, 
            -0.5f, -0.5f, 
            0.5f, -0.5f, 
            0.5f, 0.5f, 
            -0.5f, 0.5f, 
            -0.5f, -0.5f, 
```

然后在 `onDrawFrame` 中将绘制的模式改为：

```
GLES20.glDrawArrays(GLES20.GL_TRIANGLE_FAN, 0 , 6);
```

这样就可以绘制三角形扇了。

## 平滑着色

下面我们开始介绍平滑着色。
在前面的基础知识一文中我们介绍过，可以在顶点中添加其他属性，那么我们就再添加一个颜色来作为第二个属性。

```
    private float[] tableVerticesWithTriangles = {
            // 矩形
            0f,   0f, 1f, 1f, 1f,
            -0.5f, -0.5f, 0.7f,0.7f,0.7f,
            0.5f, -0.5f, 0.7f,0.7f,0.7f,
            0.5f, 0.5f, 0.7f,0.7f,0.7f,
            -0.5f, 0.5f, 0.7f,0.7f,0.7f,
            -0.5f, -0.5f, 0.7f,0.7f,0.7f,

            // 直线
            -0.5f, 0f, 1f,0f,0f,
            0.5f, 0f,1f,0f,0f,

            // 两个点
            0f, -0.25f,0f,0f,1f,
            0f, 0.25f,1f,0f,0f
    };
```

我们为顶点增加了颜色分量，现在来需要修改顶点着色器的代码：

```
uniform mat4 u_Matrix;

attribute vec4 a_Position;
attribute vec4 a_Color;

varying vec4 v_Color;

void main(){
    v_Color = a_Color;

    gl_Position =  a_Position;
    gl_PointSize = 10.0;
}
```

我们加入了一个新的属性 `a_Color`,也加入了一个叫做 `v_Color` 的新 `varying`。前面已经讲过 `varying` 是一个特殊的变量类型，它把给它的值进行混合并把这些混合的值发送给片段着色器。

我们把varying也加入片段着色器：

```
precision mediump float;
varying vec4 v_Color;

void main() {
    gl_FragColor = v_Color;
}
```

我们用varying变量v_Color替换了原来代码中的uniform。如果这个片段属于一条直线，那个OpenGL就会用构成那条直线的两个顶点计算其混合后的颜色。

下面来看一下全部的 Render 代码：

```
public class SecondRender implements GLSurfaceView.Renderer {
    private Context mContext;
    private static final int POSITOIN_COMPONENT_COUNT = 2;
    private static final int COLOR_COMPONENT_COUNT = 3;
    private static final int BYTES_PER_FLOAT = 4;
    private static final int STRIDE = (POSITOIN_COMPONENT_COUNT + COLOR_COMPONENT_COUNT) * BYTES_PER_FLOAT;
    private static final String A_COLOR = "a_Color";
    private static final String U_COLOR = "u_Color";
    private static final String A_POSITION = "a_Position";

    private float[] tableVerticesWithTriangles = {
            // 矩形
            0f,   0f, 1f, 1f, 1f,
            -0.5f, -0.5f, 0.7f,0.7f,0.7f,
            0.5f, -0.5f, 0.7f,0.7f,0.7f,
            0.5f, 0.5f, 0.7f,0.7f,0.7f,
            -0.5f, 0.5f, 0.7f,0.7f,0.7f,
            -0.5f, -0.5f, 0.7f,0.7f,0.7f,

            // 直线
            -0.5f, 0f, 1f,0f,0f,
            0.5f, 0f,1f,0f,0f,

            // 两个点
            0f, -0.25f,0f,0f,1f,
            0f, 0.25f,1f,0f,0f
    };

    private FloatBuffer vertexData;
    private int mPrograme;
    private int uColorLocation;
    private int aPosition;
    private int aColorLocation;

    public SecondRender(Context context) {
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

    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);

        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_FAN, 0 , 6);

        GLES20.glDrawArrays(GLES20.GL_LINES, 6 , 2);

        GLES20.glDrawArrays(GLES20.GL_POINTS, 8 , 1);

        GLES20.glDrawArrays(GLES20.GL_POINTS, 9 , 1);
    }
}
```

```
public class ShaderHelper {
    public static int compileVertexShader(String shaderCode){
        // 编译顶点着色器
        return compileShader(GLES20.GL_VERTEX_SHADER, shaderCode);
    }

    public static int compileFragmentShader(String shaderCode) {
        // 编译片元着色器
        return compileShader(GLES20.GL_FRAGMENT_SHADER, shaderCode);
    }

    public static int compileShader(int type, String shaderCode) {
        // 创建一个新的着色器对象
        final int sharderObjectId = GLES20.glCreateShader(type);
        // 检查是否创建成功
        if (sharderObjectId == 0) {
            Log.e("Test","create shader error");
            return 0;
        }
        // 上传着色器源码
        GLES20.glShaderSource(sharderObjectId, shaderCode);
        // 编译着色器源码
        GLES20.glCompileShader(sharderObjectId);

        // 获取编译状态
        final int[] compileStatus = new int[2];
        GLES20.glGetShaderiv(sharderObjectId, GLES20.GL_COMPILE_STATUS, compileStatus,0);
        // 取出日志信息
        Log.e("Test","compile: "+GLES20.glGetShaderInfoLog(sharderObjectId));

        if (compileStatus[0] == 0) {
            // 编译失败时删除该着色器对象
            GLES20.glDeleteShader(sharderObjectId);
            return 0;
        }
        // 返回着色器id
        return sharderObjectId;
    }


    public static int linkPrograme(int vertexShaderId,int fragmentShaderId) {
        // 创建一个 OpenGL 程序对象
        final int programeObjectId = GLES20.glCreateProgram();
        if (programeObjectId == 0) {
            Log.e("Test","create programe error");
            return 0;
        }

        // 把顶点着色器和片段着色器附加到程序对象上
        GLES20.glAttachShader(programeObjectId, vertexShaderId);
        GLES20.glAttachShader(programeObjectId, fragmentShaderId);

        // 链接程序
        GLES20.glLinkProgram(programeObjectId);

        // 验证链接的状态
        final int[] linkStatus = new int[2];
        GLES20.glGetProgramiv(programeObjectId, GLES20.GL_LINK_STATUS, linkStatus,0);
        // 打印链接信息
        Log.e("Test","link programe: "+GLES20.glGetProgramInfoLog(programeObjectId));

        if (linkStatus[0] == 0) {
            GLES20.glDeleteProgram(programeObjectId);
            Log.e("Test","lint programe error");
            return 0;
        }

        return programeObjectId;
    }

    public static boolean validatePrograme(int programeObjectId) {
        GLES20.glValidateProgram(programeObjectId);
        final int[] validateStatus = new int[2];
        GLES20.glGetProgramiv(programeObjectId, GLES20.GL_VALIDATE_STATUS, validateStatus,0);
        Log.e("Test","validate programe: "+GLES20.glGetProgramInfoLog(programeObjectId));
        return validateStatus[0] != 0;
    }
}
```

新增了 `STRIDE` 常量，这个被称为跨距的常量告诉 OpenGL 每个位置之间有多少个字节，这样它就知道需要跳过多少了。
新增 `aColorLocation` 变量，把它和顶点着色器中的 `a_Color` 关联起来。
在 `onDrawFrame` 方法中把 `GLES20.glUniform4f` 方法去掉，因为我们已经把顶点数据和`a_Color` 关联起来了，就不再需要它们来设置颜色了。只需要调用 `GLES20.glDrawArrays` 即可，OpenGL 会自动从顶点数据里面读入颜色属性。

## 效果

<img src="/images/opengl-es-smoonth-shade/smoonth.png" width="337" height="599"/>


