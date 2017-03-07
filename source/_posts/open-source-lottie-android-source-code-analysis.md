---
title: 开源项目- Lottie 源码分析
categories: Android开源项目
comments: true
tags: [Android, Lottie]
date: 2017-02-07 12:00:00
---
Lottie的基本用法其实还是非常简单的，不熟悉的同学请阅读我的博客[开源项目-Lottie简介](http://www.heqiangfly.com/2017/02/07/open-source-lottie-android-introduction/)。接下来我们就从源码角度分析一下这么强大的功能是怎么实现的。
<!-- more -->
## 实现思路
Lottie使用json文件来作为动画数据源，然后把解析这些数据源出来，建立数据到对象的映射关系，根据里面的数据建立合适的`Drawable`绘制到`View`上面。
## 源码分析
下面我们就从`LottieAnimationView`作为切入点来一步一步分析。
### LottieAnimationView
`LottieAnimationView`继承自`AppCompatImageView`，封装了一些动画的操作：
```
public void playAnimation()
public void cancelAnimation()
public void pauseAnimation()
public void setProgress(@FloatRange(from = 0f, to = 1f)
public float getProgress()
public long getDuration()
public boolean isAnimating()
```
等等；
`LottieAnimationView`有两个很重要的成员变量：
```
@Nullable private LottieComposition.Cancellable compositionLoader;
private final LottieDrawable lottieDrawable = new LottieDrawable();
```
`LottieComposition`和`LottieDrawable`将会在下面专门进行分析，他们分别进行了两个重要的工作：json文件的解析和动画的绘制。
`compositionLoader`进行了动画解析工作，得到`LottieComposition`。
我们看到的动画便是在`LottieDrawable`上面绘制出来的，`lottieDrawable`在`setComposition`方法中被添加到`LottieAnimationView`上面最终显示出来。
```
setImageDrawable(lottieDrawable);
```

### 解析JSON文件
#### JSON文件
其实在 Bodymovin 插件这里也是比较神奇的，它是怎么生成json文件的呢？这个后面有时间再研究。解析出来的json文件是这样子的：
```
{
  "assets": [
    
  ],
  "layers": [
    {
      "ddd": 0,
      "ind": 0,
      "ty": 1,
      "nm": "MASTER",
      "ks": {
        "o": {
          "k": 0
        },
        "r": {
          "k": 0
        },
        "p": {
          "k": [
            164.457,
            140.822,
            0
          ]
        },
        "a": {
          "k": [
            60,
            60,
            0
          ]
        },
        "s": {
          "k": [
            100,
            100,
            100
          ]
        }
      },
      "ao": 0,
      "sw": 120,
      "sh": 120,
      "sc": "#ffffff",
      "ip": 12,
      "op": 179,
      "st": 0,
      "bm": 0,
      "sr": 1
    },
    ……
  ],
  "v": "4.4.26",
  "ddd": 0,
  "ip": 0,
  "op": 179,
  "fr": 30,
  "w": 325,
  "h": 202
}
```
重要的数据都在`layers`里面，后面会介绍。
#### LottieComposition
Lottie使用`LottieComposition`来作为存储json文件的对象，即把json文件映射到`LottieComposition`，`LottieComposition`中提供了解析json文件的几个静态方法：

```
public static Cancellable fromAssetFileName(Context context, String fileName, OnCompositionLoadedListener loadedListener);
public static Cancellable fromInputStream(Context context, InputStream stream, OnCompositionLoadedListener loadedListener);
public static LottieComposition fromFileSync(Context context, String fileName);
public static Cancellable fromJson(Resources res, JSONObject json, OnCompositionLoadedListener loadedListener);
public static LottieComposition fromInputStream(Resources res, InputStream file);
public static LottieComposition fromJsonSync(Resources res, JSONObject json);
```

其实上面这些函数最终的解析工作是在`public static LottieComposition fromJsonSync(Resources res, JSONObject json)`里面进行的。进行了动画几个属性的解析以及`Layer`解析。
下面看一下`LottieComposition`里面的几个变量：

```
    private final LongSparseArray<Layer> layerMap = new LongSparseArray<>();
    private final List<Layer> layers = new ArrayList<>();
```

`layers`存储json文件中的`layers`数组里面的数据，`Layer`就是对应了做图中图层的概念，一个完整的动画就是由这些图层叠加起来的，具体到下面再介绍。
`layerMap`存储了`Layer`和其`id`的映射关系。
下面几个是动画里面常用的几个属性：
```
    private Rect bounds;
    private long startFrame;
    private long endFrame;
    private int frameRate;
    private long duration;
    private boolean hasMasks;
    private boolean hasMattes;
    private float scale;
```

#### Layer
`Layer`就是对应了做图中图层的概念，一个完整的动画就是由这些图层叠加起来的。
`Layer`里面有个静态方法：
```
static Layer fromJson(JSONObject json, LottieComposition composition)；
```
它解析json文件的数据并转化为`Layer`对象，
```
  private final List<Object> shapes = new ArrayList<>();

  private String layerName;
  private long layerId;
  private LottieLayerType layerType;
  private long parentId = -1;
  private long inFrame;
  private long outFrame;
  private int frameRate;

  private final List<Mask> masks = new ArrayList<>();

  private int solidWidth;
  private int solidHeight;
  private int solidColor;

  private AnimatableIntegerValue opacity;
  private AnimatableFloatValue rotation;
  private IAnimatablePathValue position;

  private AnimatablePathValue anchor;
  private AnimatableScaleValue scale;

  private boolean hasOutAnimation;
  private boolean hasInAnimation;
  private boolean hasInOutAnimation;
  @Nullable private List<Float> inOutKeyFrames;
  @Nullable private List<Float> inOutKeyTimes;

  private MatteType matteType;

```
一些成员变量一一对应json文件`layers`数组中的属性，动画就是由他们组合而来的。

### 数据转换
#### LottieDrawable
`LottieDrawable`继承自`AnimatableLayer`，关于`AnimatableLayer`我们后面再分析。
`AnimatableLayer`还有其他的子类，`LottieDrawable`可以理解为根布局，里面包含着其他的`AnimatableLayer`的子类，他们的关系可以理解为`ViewGroup`以及`View`的关系，`ViewGroup`里面可以包含`ViewGroup`以及`View`。这部分暂且不细说，下面会详细介绍。
`LottieDrawable`会通过`buildLayersForComposition(LottieComposition composition)`进行动画数据到动画对象的映射。
会根据`LottieComposition`里面的每一个`Layer`生成一个对应的`LayerView`。
#### LayerView
`LayerView`也是`AnimatableLayer`的子类，它在`setupForModel()`里面会根据`Layer`里面的数据生成不同的`AnimatableLayer`的子类，添加到变量`layers`中去。
```
      else if (item instanceof ShapePath) {
        ShapePath shapePath = (ShapePath) item;
        ShapeLayerView shapeLayer =
            new ShapeLayerView(shapePath, currentFill, currentStroke, currentTrimPath,
                new ShapeTransform(composition), getCallback());
        addLayer(shapeLayer);
      } else if (item instanceof RectangleShape) {
        RectangleShape shapeRect = (RectangleShape) item;
        RectLayer shapeLayer =
            new RectLayer(shapeRect, currentFill, currentStroke, new ShapeTransform(composition),
                getCallback());
        addLayer(shapeLayer);
      } else if (item instanceof CircleShape) {
        CircleShape shapeCircle = (CircleShape) item;
        EllipseShapeLayer shapeLayer =
            new EllipseShapeLayer(shapeCircle, currentFill, currentStroke, currentTrimPath,
                new ShapeTransform(composition), getCallback());
        addLayer(shapeLayer);
      }
```
#### AnimatableLayer
AnimatableLayer的子类，分别对应着json文件中的不同数据：
```
Drawable (android.graphics.drawable)
    AnimatableLayer (com.airbnb.lottie)
        ShapeLayerView (com.airbnb.lottie)
        LottieDrawable (com.airbnb.lottie)
        LayerView (com.airbnb.lottie)
        RectLayer (com.airbnb.lottie)
        RoundRectLayer in RectLayer (com.airbnb.lottie)
        MaskLayer (com.airbnb.lottie)
        EllipseShapeLayer (com.airbnb.lottie)
        ShapeLayer (com.airbnb.lottie)
        GroupLayerView (com.airbnb.lottie)

```
### 绘制

`LottieDrawable`的`animator`来触发整个动画的绘制，最终会调用`LottieAnimationView`的`public void invalidateDrawable(Drawable dr)`方法进行视图的更新和重绘。
绘制工作基本是由`LottieDrawable`来完成的，具体实在其父类`AnimatableLayer`的`public void draw(@NonNull Canvas canvas)`方法中进行：
```
  @Override
  public void draw(@NonNull Canvas canvas) {
    int saveCount = canvas.save();
    applyTransformForLayer(canvas, this);

    int backgroundAlpha = Color.alpha(backgroundColor);
    if (backgroundAlpha != 0) {
      int alpha = backgroundAlpha;
      if (this.alpha != null) {
        alpha = alpha * this.alpha.getValue() / 255;
      }
      solidBackgroundPaint.setAlpha(alpha);
      if (alpha > 0) {
        canvas.drawRect(getBounds(), solidBackgroundPaint);
      }
    }
    for (int i = 0; i < layers.size(); i++) {
      layers.get(i).draw(canvas);
    }
    canvas.restoreToCount(saveCount);
  }
```
先绘制了本层的内容，然后开始绘制包含的`layers`的内容，这个过程类似与界面中`ViewGroup`嵌套绘制。如此完成各个`Layer`的绘制工作。

### 总结
由上面的分析我们得到了Lottie绘制动画的思路：
 1. 创建 `LottieAnimationView lottieAnimationView`
 2. 在`LottieAnimationView`中创建`LottieDrawable lottieDrawable`
 3. 在`LottieAnimationView`中创建`compositionLoader`，进行json文件解析得到`LottieComposition`，完成数据到对象`Layer`的映射。
 4. 解析完后通过`setComposition`方法把`LottieComposition`给`lottieDrawable`，`lottieDrawable`在`setComposition`方法中把`Layer`转换为`LayerView`，为绘制做好准备。
 5. 在`LottieAnimationView`中把`lottieDrawable`设置`setImageDrawable`，
 6. 然后开始动画`lottieDrawable.playAnimation()`。

## Lottie的性能

## 参考资料
http://www.jianshu.com/p/81be1bf9600c


