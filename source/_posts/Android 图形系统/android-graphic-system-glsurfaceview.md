---
title: Android 图形系统 -- GLSurfaceView 使用
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 GLSurfaceView 的使用
date: 2016-9-18 10:00:00
---

## 概述

GLSurfaceView 继承自 SurfaceView，并且实现了 SurfaceHolder.Callback2 接口。在 SurfaceView 的基础上添加了 EGL 的管理，并自带了一个GLThread绘制线程。
这使得 GLSurfaceView 也拥有了 OpenGlES 所提供的图形处理能力，通过它定义的 Render 接口，使更改具体的 Render 的行为非常灵活性，只需要将实现了渲染函数的 Renderer 的实现类设置给 GLSurfaceView 即可。
GLSurfaceView 涉及到了OpenGL ES，使用 GLSurfaceView 必须开启硬件加速属性。
本文我们只简单介绍 GLSurfaceView 的使用，相关 OpenGL ES 2.0的使用后面会有专门的文章来介绍。

## GLSurfaceView 的特性

 1. 管理一个surface，这个surface就是一块特殊的内存，能直接排版到android的视图view上。
 2. 管理一个EGL display，它能让opengl把内容渲染到上述的surface上。
 3. 用户自定义渲染器(render)。
 4. 让渲染器在独立的线程里运作，和UI线程分离。
 5. 支持按需渲染(on-demand)和连续渲染(continuous)。
 6. 可以封装、跟踪并且排查渲染器的问题。

## GLSurfaceView 的方法

 - setRenderMode(Renderer renderer)：设置渲染模式。RENDERMODE_CONTINUOUSLY（每一帧都会触发渲染）和RENDERMODE_WHEN_DIRTY（需要时渲染，在创建和调用requestRender()时）
 - setEGLConfigChooser：设置配置选择器，进行颜色，深度，模板等等设置，在setRenderer之前调用
 - setEGLWindowSurfaceFactory：自定义EGLSurface工厂
 - setEGLContextClientVersion：设置 opengl es 版本
 - setRenderer(Renderer renderer)：设置渲染器。

## Renderer 接口

看一下渲染器接口定义的方法：

 - onSurfaceCreated(GL10 gl, EGLConfig config)：当 Surface 被创建的时候，GLSurfaceView 会调用这个方法；在应用程序第一次调用时会调用，当设备被唤醒或者用户从其他 Activity 切换回来时，也可能会被调用。
 - onSurfaceChanged(GL10 gl, int width, int height)：在 Surface 被创建以后，每次 Surface 尺寸变化时，这个方法都会被 GLSurfaceView 调用到。
 - onDrawFrame(GL10 gl)：当绘制一帧时，这个方法会被调用。在这个方法返回后，渲染缓冲区会被交换并显示在屏幕上。

为什么上面的方法都有个 GL01 呢？它是 OpenGL ES 1.0 的API遗留下来的，如果要编写使用 OpenGL ES 1.0 的渲染器，就要使用这个参数。但是，对于 OpenGL ES 2.0 来说，GLES20类提供了静态方法来存取。

## GLSurfaceView 的使用

下面介绍 GLSurfaceView 的基本使用，使用 GLSurfaceView 可以在 Android 中搭建一个基本的 OpenGL ES 开发环境。
下面的代码使用 OpengGL ES 2.0 的 API 绘制了一个三角形。

```
public class MyGLSurfaceView extends GLSurfaceView implements GLSurfaceView.Renderer {
    float triangleCoords[] = {
            0.5f,  0.5f, 0.0f, // top
            -0.5f, -0.5f, 0.0f, // bottom left
            0.5f, -0.5f, 0.0f  // bottom right
    };
    float color[] = { 1.0f, 0.0f, 0.0f, 1.0f }; //白色
    static final int COORDS_PER_VERTEX = 3;
    //顶点个数
    private final int vertexCount = triangleCoords.length / COORDS_PER_VERTEX;
    //顶点之间的偏移量
    private final int vertexStride = COORDS_PER_VERTEX * 4; // 每个顶点四个字节

    private FloatBuffer vertexBuffer;
    private int mProgram;
    private int mPositionHandle;
    private int mColorHandle;
    private final String vertexShaderCode =
            "attribute vec4 vPosition;" +
                    "void main() {" +
                    "  gl_Position = vPosition;" +
                    "}";

    private final String fragmentShaderCode =
            "precision mediump float;" +
                    "uniform vec4 vColor;" +
                    "void main() {" +
                    "  gl_FragColor = vColor;" +
                    "}";

    public MyGLSurfaceView(Context context) {
        this(context,null);
    }

    public MyGLSurfaceView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        setEGLContextClientVersion(2);
        setRenderer(this);
        setRenderMode(GLSurfaceView.RENDERMODE_WHEN_DIRTY);
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        //将背景设置为灰色
        GLES20.glClearColor(0.5f,0.5f,0.5f,1.0f);
        //清空屏幕，会擦除屏幕上所有的颜色，并用前面 glClearColor 调用定义的颜色填充屏幕
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        //开辟对应容量的缓冲，申请底层空间
        ByteBuffer bb = ByteBuffer.allocateDirect(
                triangleCoords.length * 4);
        // 设置字节顺序为本地操作系统顺序
        bb.order(ByteOrder.nativeOrder());
        // 将坐标数据转换为FloatBuffer，用以传入给OpenGL ES程序
        vertexBuffer = bb.asFloatBuffer();
        // 将数组中的顶点数据送入缓冲
        vertexBuffer.put(triangleCoords);
        // 设置缓冲起始位置
        vertexBuffer.position(0);
        int vertexShader = loadShader(GLES20.GL_VERTEX_SHADER,
                vertexShaderCode);
        int fragmentShader = loadShader(GLES20.GL_FRAGMENT_SHADER,
                fragmentShaderCode);

        //创建一个空的OpenGLES程序
        mProgram = GLES20.glCreateProgram();
        //将顶点着色器加入到程序
        GLES20.glAttachShader(mProgram, vertexShader);
        //将片元着色器加入到程序中
        GLES20.glAttachShader(mProgram, fragmentShader);
        //连接到着色器程序
        GLES20.glLinkProgram(mProgram);

    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0,0,width,height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        //将程序加入到OpenGLES2.0环境
        GLES20.glUseProgram(mProgram);

        // 获取顶点着色器的定点位置属性引用vPosition的值
        // mProgram 为采用的着色器程序id
        mPositionHandle = GLES20.glGetAttribLocation(mProgram, "vPosition");
        //启用定点位置数据
        GLES20.glEnableVertexAttribArray(mPositionHandle);
        //准备三角形的坐标数据，将顶点数据传送进渲染管线
        GLES20.glVertexAttribPointer(mPositionHandle, COORDS_PER_VERTEX,
                GLES20.GL_FLOAT, false,
                vertexStride, vertexBuffer);
        //获取片元着色器的vColor成员的句柄
        mColorHandle = GLES20.glGetUniformLocation(mProgram, "vColor");
        //设置绘制三角形的颜色
        GLES20.glUniform4fv(mColorHandle, 1, color, 0);
        //绘制三角形
        GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, vertexCount);
        //禁止顶点数组的句柄
        GLES20.glDisableVertexAttribArray(mPositionHandle);
    }

    public int loadShader(int type, String shaderCode){
        //根据type创建顶点着色器或者片元着色器
        int shader = GLES20.glCreateShader(type);
        //将资源加入到着色器中，并编译
        GLES20.glShaderSource(shader, shaderCode);
        GLES20.glCompileShader(shader);
        return shader;
    }
}
```