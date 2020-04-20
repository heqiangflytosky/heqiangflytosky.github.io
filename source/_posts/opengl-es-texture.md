---
title: OpenGL ES -- 纹理
categories: OpenGL
comments: true
tags: [OpenGL ES]
description: 介绍 OpenGL ES 2.0 纹理绘制
date: 2020-4-15 10:00:00
---

## 概述

现在我们已经完成了简单图形和颜色的绘制，如果我们想在图像上增加一些精致的细节，那么就需要使用纹理（texture）。
纹理就是一个图像、照片，甚至由一个数学算法生成的分形数据。

## 纹理坐标

每个二维纹理都有其自己的坐标空间，其范围是从一个拐角的（0，0）到另一个拐角的（1，1）。按照惯例，一个维度叫做S，而另一个称为T。当我们想要把一个纹理应用于一个三角形或一组三角形的时候，我们要为每个顶点指定一组ST纹理坐标，以便OpenGL知道需要用那个纹理的哪个部分画到每个三角形上。这些纹理坐标有时也会被称为UV纹理坐标。

<img src="/images/opengl-es-texture/v.png" width="223" height="187"/>

对一个OpenGL纹理来说，它没有内在的方向性，因此我们可以使用不同的坐标把它定向到任何我们喜欢的方向上。然而，大多数计算机图像都有一个默认的方向，它们通常被规定为Y轴向下，Y的值随着向图像的底部移动而增加。

## 环绕方式

GL_CLAMP_TO_EDGE：纹理坐标会被约束在0到1之间，超出的部分会重复纹理坐标的边缘，产生一种边缘被拉伸的效果
GL_REPEAT：对纹理的默认行为。重复纹理图像。

## 纹理过滤

当纹理的大小被扩大或者缩小时，我们还需要使用纹理过滤明确说明会发生什么。当我们在渲染表面上绘制一个纹理时，那个纹理的纹理元素可能无法精确地映射到OpenGL生成的片段上。有两种情况：缩小和放大。当我们尽力把几个纹理元素挤进一个片段时，缩小就发生了；当我们把一个纹理元素扩展到许多片段时，方法就发生了。针对每一种情况，我们可以配置OpenGL使用一个纹理过滤器。

首先，讲述两个基本的过滤模式：最近邻过滤和双线性插值。

 - GL_NEAREST 邻近过滤，是OpenGL默认的纹理过滤方式。当设置为GL_NEAREST的时候，OpenGL会选择中心点最接近纹理坐标的那个像素。
 - GL_LINEAR 线性过滤，它会基于纹理坐标附近的纹理像素，计算出一个插值，近似出这些纹理像素之间的颜色。 

## 代码

顶点着色器代码：

```
attribute vec4 a_Position;

attribute vec2 a_TextureCoordinates;
varying vec2 v_TextureCoordinates;

void main()
{
    v_TextureCoordinates = a_TextureCoordinates;
    gl_Position = a_Position;
}
```

片段着色器代码：

```
precision mediump float;

uniform sampler2D u_TextureUnit;
varying vec2 v_TextureCoordinates;

void main()
{
    gl_FragColor = texture2D(u_TextureUnit, v_TextureCoordinates);
}
```

```
public class TestRender implements GLSurfaceView.Renderer {
    private Context mContext;
    private static final float[] vertexData = {
            // 第一个三角形
            -0.5f, -0.5f,  0f, 1f,
            0.5f, 0.5f,   1f, 0f,
            -0.5f, 0.5f,  0f, 0f,

            // 第二个三角形
            -0.5f, -0.5f, 0f, 1f,
            0.5f, -0.5f, 1f, 1f,
            0.5f, 0.5f,1f, 0f
    };
    private static final int POSITION_COMPONENT_COUNT=2;
    private static final int TEXTURE_COORDINATES_COMPONENT_COUNT=2;
    private static final int STRIDE=(POSITION_COMPONENT_COUNT+TEXTURE_COORDINATES_COMPONENT_COUNT)* Constants.BYTES_PER_FLOAT;
    protected static final String U_TEXTURE_UNIT = "u_TextureUnit";
    protected static final String A_POSITION = "a_Position";
    protected static final String A_TEXTURE_COORDINATES = "a_TextureCoordinates";

    private FloatBuffer vertexDataBuffer;
    private int textureId;
    private int programeId;

    private int uTextureUnitLocation;
    private int aPositionLocation;
    private int aTextureCoordinatesLocation;

    public TestRender(Context context) {
        mContext = context;
    }
    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        GLES20.glClearColor(0.0f,0,0,0);

        vertexDataBuffer = ByteBuffer
                .allocateDirect(vertexData.length * 4)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer()
                .put(vertexData);

        int vertexShader= ShaderHelper.compileVertexShader(Utils.fileLoader(mContext,R.raw.test_vertex_shader));
        int fragmentShader=ShaderHelper.compileFragmentShader(Utils.fileLoader(mContext,R.raw.test_fragment_shader));

        programeId = ShaderHelper.linkPrograme(vertexShader,fragmentShader);
        ShaderHelper.validatePrograme(programeId);

        this.uTextureUnitLocation=GLES20.glGetUniformLocation(programeId,U_TEXTURE_UNIT);
        this.aPositionLocation=GLES20.glGetAttribLocation(programeId, A_POSITION);
        this.aTextureCoordinatesLocation= GLES20.glGetAttribLocation(programeId,A_TEXTURE_COORDINATES);

        GLES20.glUseProgram(programeId);

        // 加载纹理
        textureId = TextureHelper.loadTexture(mContext, R.drawable.test);

        GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
        // 绑定纹理到这个纹理单元
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId);
        // 把选定的纹理单元传给片段着色器中的 u_TextureUnit，
        GLES20.glUniform1i(uTextureUnitLocation, 0);

        // 传递矩形顶点坐标
        vertexDataBuffer.position(0);
        GLES20.glVertexAttribPointer(aPositionLocation, POSITION_COMPONENT_COUNT, GLES20.GL_FLOAT, false, 16, vertexDataBuffer);
        GLES20.glEnableVertexAttribArray(aPositionLocation);
        // 传递纹理顶点坐标
        vertexDataBuffer.position(POSITION_COMPONENT_COUNT);
        GLES20.glVertexAttribPointer(aTextureCoordinatesLocation, TEXTURE_COORDINATES_COMPONENT_COUNT, GLES20.GL_FLOAT, false, 16, vertexDataBuffer);
        GLES20.glEnableVertexAttribArray(aTextureCoordinatesLocation);
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0,0,width,height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);

        GLES20.glDrawArrays(GLES20.GL_TRIANGLES,0,6);
    }
}
```

```
public class TextureHelper {
    public static int loadTexture(Context context, int resourceId){
        final int[] textureObjectIds = new int[2];
        GLES20.glGenTextures(1, textureObjectIds, 0);

        if (textureObjectIds[0] == 0){
            Log.e("Test","gen texture fail");
            return 0;
        }

        final BitmapFactory.Options options=new BitmapFactory.Options();
        options.inScaled=false;
        final Bitmap bitmap=BitmapFactory.decodeResource(context.getResources(),resourceId,options);
        if(bitmap==null){
            Log.e("Test","load bitmap fial");
            GLES20.glDeleteTextures(1,textureObjectIds,0);
            return 0;
        }

        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D,textureObjectIds[0]);

        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D,GLES20.GL_TEXTURE_MIN_FILTER,GLES20.GL_LINEAR_MIPMAP_LINEAR);
        GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D,GLES20.GL_TEXTURE_MAG_FILTER,GLES20.GL_LINEAR);

        GLUtils.texImage2D(GLES20.GL_TEXTURE_2D,0,bitmap,0);

        bitmap.recycle();

        GLES20.glGenerateMipmap(GLES20.GL_TEXTURE_2D);
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D,0);

        return textureObjectIds[0];
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

```
public class Utils {
    public static String fileLoader(Context context, int id) {
        StringBuilder body = new StringBuilder();

        try {
            InputStream inputStream = context.getResources().openRawResource(id);
            InputStreamReader inputStreamReader = new InputStreamReader(inputStream);
            BufferedReader bufferedReader = new BufferedReader(inputStreamReader);
            String nextLine;
            while ((nextLine = bufferedReader.readLine()) != null) {
                body.append(nextLine);
                body.append('\n');
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return body.toString();
    }
}
```

图片绘制效果：

<img src="/images/opengl-es-texture/r2.png" width="258" height="487"/>

## 裁剪纹理

可以看到图片有些拉伸，下面我们把图片进行一些裁剪，修改一下顶点坐标：

```
    private static final float[] vertexData = {
            // 第一个三角形
            -0.5f, -0.5f,  0.5f, 1f,
            0.5f, 0.5f,   1f, 0f,
            -0.5f, 0.5f,  0.5f, 0f,

            // 第二个三角形
            -0.5f, -0.5f, 0.5f, 1f,
            0.5f, -0.5f, 1f, 1f,
            0.5f, 0.5f,1f, 0f
    };
```

裁剪后的效果：

<img src="/images/opengl-es-texture/r1.png" width="238" height="457"/>
