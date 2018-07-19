
---
title: Android 图形系统 -- Canvas使用
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 Android 图形系统
date: 2015-1-16 10:00:00
---

Canvas顾名思义就是一块画布，可以在上面画一些你想要的东西。
创建一个Canvas有一下两种方法：

 - Canvas():创建一个空的画布，可以使用setBitmap()方法来设置绘制的具体画布； 
 - Canvas(Bitmap bitmap):以bitmap对象创建一个画布，则将内容都绘制在bitmap上，bitmap不得为null; 

## 使用Canvas来截图

```
    private ImageView mImageView1;
    private ImageView mImageView2;
    private TextView mTextVew;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mImageView1 = (ImageView) findViewById(R.id.imageview1);
        mImageView2 = (ImageView) findViewById(R.id.imageview2);
        mTextVew = (TextView) findViewById(R.id.textView);
    }

    public void buttonClick(View v){
        Bitmap capture = Bitmap.createBitmap(300,300, Bitmap.Config.ARGB_8888);
        capture.eraseColor(Color.TRANSPARENT);
        Canvas c = new Canvas(capture);
        //c.scale(0.8f,0.8f);
        c.drawColor(Color.LTGRAY);//设置背景颜色
        mImageView1.draw(c);//绘制图片
        mTextVew.draw(c);//绘制文字

        mImageView2.setImageBitmap(capture);
    }
```

上面代码的效果就是：创建了一个尺寸是300*300的Bitmap，使用它作为Canvas操作的对象。首先使用了c.drawColor(Color.LTGRAY)绘制截图的背景颜色，mImageView1.draw(c)把图片绘制在Canvas上，mTextVew.draw(c)把自体绘制在Canvas上，最后用mImageView2把最终绘制的结果显示出来了，如果去掉c.drawColor(Color.LTGRAY)和mTextVew.draw(c)，其实就是mImageView1的一个截图，我们可以用截图来保存本地等其他操作。

![这里写图片描述](/images/android-graphic-system-canvas-sample/image1.png)

## 通过onDraw在View中实现各种操作

例子：

```
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.save();

        Paint paint = new Paint();
        paint.setColor(0xffff0000);
        paint.setStrokeWidth(6);
        paint.setTextSize(64);

        //画线
        canvas.drawLine(0, 50, 200, 50, paint);
        //画矩形
        canvas.drawRect(80, 80, 200, 200, paint);
        //画圆
        canvas.drawCircle(400,150,80,paint);
        //写字
        canvas.drawText("Hello",80, 300,paint);

        canvas.restore();
    }
```

效果：

![这里写图片描述](/images/android-graphic-system-canvas-sample/image2.png)

下面再看一个有意思的例子：

http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2012/1212/703.html

```
    @Override
    protected void onDraw(Canvas canvas) {
        Paint paint = new Paint(); //设置一个笔刷大小是3的黄色的画笔
        paint.setColor(Color.YELLOW);
        paint.setStrokeJoin(Paint.Join.ROUND);
        paint.setStrokeCap(Paint.Cap.ROUND);
        paint.setStrokeWidth(3);

        paint.setAntiAlias(true);
        paint.setStyle(Paint.Style.STROKE);
        canvas.translate(canvas.getWidth()/2, 200); //将位置移动画纸的坐标点:150,150
        canvas.drawCircle(0, 0, 100, paint); //画圆圈

        //使用path绘制路径文字
        canvas.save();
        canvas.translate(-75, -75);
        Path path = new Path();
        path.addArc(new RectF(0,0,150,150), -180, 180);
        Paint citePaint = new Paint(paint);
        citePaint.setTextSize(14);
        citePaint.setStrokeWidth(1);
        canvas.drawTextOnPath("http://blog.csdn.net/heqiangflytosky", path, 5, 0, citePaint);
        canvas.restore();

        Paint tmpPaint = new Paint(paint); //小刻度画笔对象
        tmpPaint.setStrokeWidth(1);

        float  y=100;
        int count = 60; //总刻度数

        for(int i=0 ; i <count ; i++){
            if(i%5 == 0){
                canvas.drawLine(0f, y, 0, y+12f, paint);
                canvas.drawText(String.valueOf(i/5+1), -4f, y+25f, tmpPaint);

            }else{
                canvas.drawLine(0f, y, 0f, y +5f, tmpPaint);
            }
            canvas.rotate(360/count,0f,0f); //旋转画纸
        }

        //绘制指针
        tmpPaint.setColor(Color.GRAY);
        tmpPaint.setStrokeWidth(4);
        canvas.drawCircle(0, 0, 7, tmpPaint);
        tmpPaint.setStyle(Paint.Style.FILL);
        tmpPaint.setColor(Color.YELLOW);
        canvas.drawCircle(0, 0, 5, tmpPaint);
        canvas.drawLine(0, 10, 0, -65, paint);
    }
```
效果：

![clock](/images/android-graphic-system-canvas-sample/image3.png)

## Canvas剪切操作

例子：

```
public class TestView extends View {

    public 
    TestView(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        canvas.save();

        Paint paint = new Paint();
        paint.setColor(0xffff0000);
        Rect rect = new Rect(0,0,300,300);
        Rect clipRect = new Rect(40,40,200,200);
        //显示clipRect与rect相交区域
        //canvas.clipRect(clipRect);
        //显示clipRect与rect相交以外区域
        canvas.clipRect(clipRect,Region.Op.XOR);
        canvas.drawRect(rect,paint);

        canvas.restore();
    }
}
```

效果：
不剪切

![noclip](/images/android-graphic-system-canvas-sample/image4.png)

剪切

![clip](/images/android-graphic-system-canvas-sample/image5.png)

XOR剪切

![这里写图片描述](/images/android-graphic-system-canvas-sample/image6.png)

Region.Op还提供一下参数供选择：

        DIFFERENCE(0),   //相减
        INTERSECT(1),
        UNION(2),        //并集
        XOR(3),          //异或
        REVERSE_DIFFERENCE(4),   //逆向差集
        REPLACE(5);

还可以参考APIDEMO

