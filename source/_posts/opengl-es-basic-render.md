---
title: OpenGL ES -- 基础绘制
categories: OpenGL
comments: true
tags: [OpenGL ES]
description: 介绍 OpenGL ES 2.0 的一些绘制方法
date: 2020-4-9 10:00:00
---


## 概述

在前面的文章 [Android 图形系统 -- GLSurfaceView 使用 ](http://www.heqiangfly.com/2016/09/18/android-graphic-system-glsurfaceview/) 一文中我们了解了 GLSurfaceView 的基本使用以及使用 Android 提供的 OpenGL ES 接口绘制了一个三角形，接下来本文会较为详细地介绍一下在 Android 环境下的 OpenGL ES 的基本操作。
前面也讲过，在 OpenGL 里面只能绘制点、直线和三角形，每个物体的绘制都是通过这三个基本绘制聚合而成的。那么今天我们就绘制一个矩形、一条直线和两个点来介绍这些基本绘制操作。

## 代码

先贴上代码，里面有部分注释，想了解细节的话可以参考下面的章节。

```
public class FirstRender implements GLSurfaceView.Renderer {
    // 每个顶点分量的个数，这里是2，也可以用3个分量来表示一个顶点
    private static final int POSITOIN_COMPONENT_COUNT = 2;
    private static final int BYTES_PER_FLOAT = 4;
    private static final String U_COLOR = "u_Color";
    private static final String A_POSITION = "a_Position";
    private static final float color_grey[] = { 0.5f, 0.5f, 0.5f, 1.0f };
    private static final float color_red[] = { 1.0f, 0.0f, 0.0f, 1.0f };

    private float[] tableVerticesWithTriangles = {
            // 第一个三角形
            -0.5f, -0.5f,
            0.5f, 0.5f,
            -0.5f, 0.5f,

            // 第二个三角形
            -0.5f, -0.5f,
            0.5f, -0.5f,
            0.5f, 0.5f,

            // 直线
            -0.5f, 0f,
            0.5f, 0f,

            // 两个点
            0f, -0.25f,
            0f, 0.25f};

    // 顶点着色器代码
    private final String vertexShaderCode =
            "attribute vec4 a_Position;" +
                    "void main() {" +
                    "  gl_Position = a_Position;" +
                    "  gl_PointSize = 10.0;" +
                    "}";
    // 片元着色器代码
    private final String fragmentShaderCode =
            "precision mediump float;" +
                    "uniform vec4 u_Color;" +
                    "void main() {" +
                    "  gl_FragColor = u_Color;" +
                    "}";

    private FloatBuffer vertexData;
    private int mPrograme;
    private int uColorLocation;
    private int aPosition;

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        GLES20.glClearColor(0f,0,0,0f);

        // 1. 为顶点数据分配内存
        vertexData = ByteBuffer.allocateDirect(tableVerticesWithTriangles.length * BYTES_PER_FLOAT)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        vertexData.put(tableVerticesWithTriangles);

        // 2. 编译着色器
        int vertexShader = ShaderHelper.compileVertexShader(vertexShaderCode);
        int fragmentShader = ShaderHelper.compileFragmentShader(fragmentShaderCode);

        // 3. 链接到 OpenGL 程序
        mPrograme = ShaderHelper.linkPrograme(vertexShader, fragmentShader);

        // 4. 验证 OpenGL 程序对象
        ShaderHelper.validatePrograme(mPrograme);

        // 5. 使用 OpenGL 程序，告诉 OpenGL 在绘制任何东西到屏幕上的时候要使用这里定义的程序
        GLES20.glUseProgram(mPrograme);

        // 6. 获取数据
        aPosition = GLES20.glGetAttribLocation(mPrograme, A_POSITION);
        uColorLocation = GLES20.glGetUniformLocation(mPrograme, U_COLOR);

        // 7. 关联属性与顶点数据的数组
        // 把当前位置设置在数组的开头
        vertexData.position(0);
        // 告诉 OpenGL，可以在 vertexData 中找到 a_Position 定义的数据
        GLES20.glVertexAttribPointer(aPosition, POSITOIN_COMPONENT_COUNT, GLES20.GL_FLOAT,
                false, 0, vertexData);

        // 8. 使能顶点数组
        GLES20.glEnableVertexAttribArray(aPosition);

    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0,0,width,height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);

        // 6.开始绘制图形
        // 6.1 更新颜色，画正方形
        GLES20.glUniform4f(uColorLocation, 0.5f, 0.5f, 0.5f, 1.0f);
        GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0 , 6);

        // 6.2 更新颜色，画直线
        GLES20.glUniform4f(uColorLocation, 1.0f, 0.0f, 0.0f, 1.0f);
        GLES20.glDrawArrays(GLES20.GL_LINES, 6 , 2);

        // 6.3 更新颜色，画点，颜色为蓝色
        GLES20.glUniform4f(uColorLocation, 0.0f, 0.0f, 1.0f, 1.0f);
        GLES20.glDrawArrays(GLES20.GL_POINTS, 8 , 1);

        // 6.4 更新颜色，画点，颜色为红色，这里用另外一个api glUniform4fv 指定Uniform变量的值
        GLES20.glUniform4fv(uColorLocation, 1, color_red, 0);
        GLES20.glDrawArrays(GLES20.GL_POINTS, 9 , 1);
    }
}
```

```
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
```

效果图：

<img src="/images/opengl-es-basic-render/result.png" width="209" height="397"/>

## 详解

### 定义顶点

了解顶点前，建议先了解一下 OpenGL 的坐标映射，该部分在[OpenGL ES -- 基础知识 ](http://www.heqiangfly.com/2020/04/04/opengl-es-basic-knowledge/)一文中。
前面我们说过，在 OpenGL 里面只能绘制点、直线和三角形，那么这里的长方形就是被分解为两个三角形来绘制的。
我们把两个三角形，一条直线和两个点的顶点信息放在一个数组里面。
这里我们用两个分量来标识一个顶点，在后面的章节里面，我们将会用三个分量来走进三维的世界。
另外，我们还可在顶点里面加入其它属性，比如颜色属性、纹理左边属性等，这些后面的文章会介绍，今天介绍的顶点里面，仅包含顶点的位置属性。

```
    private float[] tableVerticesWithTriangles = {
            // 第一个三角形
            -0.5f, -0.5f,
            0.5f, 0.5f,
            -0.5f, 0.5f,

            // 第二个三角形
            -0.5f, -0.5f,
            0.5f, -0.5f,
            0.5f, 0.5f,

            // 直线
            -0.5f, 0f,
            0.5f, 0f,

            // 两个点
            0f, -0.25f,
            0f, 0.25f};
```

当我们定义三角形时，我们总是以逆时针的顺序排列顶点，这称为卷曲顺序。因为在任何地方都使用这种一致的卷曲顺序，可以优化性能。使用卷曲顺序可以指出一个三角形属于任何给定物体的前面或者后面，OpenGL 可以忽略哪些无论如何都无法被看到的后面的三角形。


### 为顶点分配内存

OpenGL 是运行在 native 环境，因此我们需要为顶点创建本地内存缓冲来存储数据。

```
    private FloatBuffer vertexData;
    
        vertexData = ByteBuffer.allocateDirect(tableVerticesWithTriangles.length * BYTES_PER_FLOAT)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        vertexData.put(tableVerticesWithTriangles);
```

在 Java 中一个浮点数占 4 个字节，由此可以算出需要创建的缓冲区的大小。
`order(ByteOrder.nativeOrder())` 来告诉缓冲区按照本地字节顺序来存储内容。因为有的起始大端，有的是小端。
掉用 `vertexData.put` 就把数据从 Android 虚拟机的内存赋值到本地内存了。当进程结束时，这块内存会被释放掉。

### 创建着色器

着色器的代码我们可以放在一个单独的文件中，因为我们目前的着色器代码比较简单，可以直接定义在 Java 文件中。
另外推荐一个 Android Studio 插件：Android Studio GLSL Support。当我们把着色器代码写在单独文件中时，这个插件比较有用。

```
    // 顶点着色器代码
    private final String vertexShaderCode =
            "attribute vec4 aPosition;" +
                    "void main() {" +
                    "  gl_Position = aPosition;" +
                    "  gl_PointSize = 10.0;" +
                    "}";
    // 片元着色器代码
    private final String fragmentShaderCode =
            "precision mediump float;" +
                    "uniform vec4 uColor;" +
                    "void main() {" +
                    "  gl_FragColor = uColor;" +
                    "}";
```

对于我们定义过的每个单一的定点，顶点着色器都会被调用一次；当它被调用的时候，它会在 a_Position 属性里接收当前顶点的位置，这个属性被定义为vec4类型。
一个vec4是包含了4个分量的向量；在位置的上下文中，可以认为这4个分量是X,Y,Z和W坐标，X,Y和Z对应一个三维位置，而W是一个特殊的坐标，后面会专门讲解W坐标，现在暂时略过。如果没有指定，默认情况下，OpenGL都是把向量的前三个坐标设为0，并把最后一个坐标设为1。
一个顶点会有几个属性，比如颜色和位置。关键词"attribute"就是把这些属性放进着色器的手段。
之后，可以定义main()，这是着色器的主要入口点；它所做的就是把前面定义过的位置复制到指定的输出变量gl_Position；这个着色器一定要给gl_Position赋值；OpenGL会把gl_Position中存储的位置当作顶点的最终位置，并把这些顶点组装成点，直线和三角形。
在片段着色器我们定义了一个 uniform 类型的变量 u_Color。它不像前面用 attribute 定义顶点时，每个顶点都要设置一个值。一个 uniform 会让每个顶点都使用同一个值，除非我们在次改变它。如顶点着色器中的位置所使用的属性一样，u_Color 也是一个四分量向量，但是在颜色的上下文中，这四分量分别对应红色，绿色，蓝色和阿尔法。
后面在[]()一文中我们会对比介绍 varying 限定符。
接着我们定义了main()，它是这个着色器的住入口点，它把我们在uniform里定义的颜色复制到那个特殊的输出变量 gl_FragColor。着色器一定要给 gl_FragColor 赋值，OpenGL会使用这个颜色作为当前片段的最终颜色。

### 编译着色器

这里，用 `GLES20.glCreateShader` 调用创建了一个新的着色器对象，并把这个对象的ID存入变量 `shaderObjectId`。这个type可以代表定点着色器的 `GLES20.GL_VERTEX_SHADER`，或者是代表片段着色器的 `GLES20.GL_FRAGMENT_SHADER`。
然后检查着色器对象的有效性。
然后，调用 `GLES20.glShaderSource` 上传着色器代码，调用 `GLES20.glCompileShader` 编译着色器代码。
最后调用 `GLES20.glGetShaderiv` 取出编译状态并检查是否编译成功。
这是 Android 平台上的 OpenGL 的另一个通用模式。 为了取出一个值，我们通常会使用一个长度为1的数组，并把这个数组传进 OpenGL 调用。在一个调用中，我们告诉OpenGL把结果存进数组的第一个元素。

### 链接

既然我们已经加载并编译了一个顶点着色器和一个片段着色器，下一步就是把它们绑定在一起放入一个程序（program）里。
一个 OpenGL 程序就是把一个顶点着色器和一个片段着色器链接在一起变成单个对象。顶点着色器和片段着色器总是一起工作的。没有片段着色器，OpenGL 就不知道怎么绘制那些组成的每个点，直线和三角形片段；如果没有顶点着色器，OpenGL 就不知道在哪里绘制这些片段。顶点着色器和片段着色器一起合作生成屏幕上最终的图像。
虽然顶点着色器和片段着色器总是要一起工作的，但并不意味着它们必须是一对一匹配的，我们可以同时在多个程序中使用同一个着色器。
首先调用 `GLES20.glCreateProgram()` 新建程序对象，并把那个对象的ID存进programObjectId。
然后调用 `GLES20.glAttachShader` 把顶点着色器和片段着色器附加到程序对象上，然后通过 `GLES20.glLinkProgram` 把这些着色器联合起来。
然后可以再调用 `GLES20.glValidateProgram` 来验证一下这个程序对于当前的OpenGL状态是不是有效的。
验证通过后调用 `GLES20.glUseProgram()` 告诉OpenGL在绘制任何东西到屏幕上的时候要使用这个定义的程序。

### 关联数据

接下来就是要获得我们在着色语言中定义的属性和uniform的位置。我们通过 `GLES20.glGetAttribLocation` 和 `GLES20.glGetUniformLocation` 获得 a_Position 和 u_Color 的位置，并把它们存入 aPosition 和 uColorLocation。
接下来就是要告诉 OpenGL 去哪里寻找 aPosition 属性对应的数据了。对于 uColorLocation 因为我们没有在顶点数据中去定义，那么后面绘制是我们将会去手动更新它的值，这样来绘制出不同的颜色。
我们通过 `GLES20.glVertexAttribPointer` 告诉 OpenGL，可以在 vertexData 中找到 a_Position 定义的数据。

下面来介绍一下 glVertexAttribPointer 的参数：

| 1 | 2 |
|:-------------:|:-------------:|
| int index | 这个是位置属性，我们传入 aPosition |
| int size | 这是每个属性的数据的计数，或者是多对于这个属性，有多少个分量与每一个顶点关联。前面我们介绍过，可以是2个分量，也可以是3或4个分量 |
| int type | 这是数据的类型 |
| boolean normalized | 只有使用整形数据的时候，这个参数才有意义 |
| int stride | 只有当一个数组存储多于一个属性时，才有意义。这里只有一个属性，可以传 0 进去 |
| Buffer ptr| 这个参数告诉 OpenGL 去哪里读取数据。 |

### 使能顶点数组

尽管我们已经把数据属性链接起来了，在开始绘制之前，还需要调用 `GLES20.glEnableVertexAttribArray` 来使能顶点数组，这样，OpenGL 就知道去哪里寻找它所需要的数据了。

### 绘制

绘制的方法比较简单了，主要是通过 `GLES20.glDrawArrays` 来实现绘制三角形、点和线的。
由于我们定义顶点数据时没有指定颜色，那么我们在绘制图形前可以通过 `GLES20.glUniform4f` 来为他们设置颜色。

