---
title: Android：关于Drawable的缓存机制应该了解的知识
categories: Android
comments: true
tags: [Android]
description: 在Android中，出于对内存优化的考虑，对于图片的存储使用了缓存机制，资源id相同的图片使用了同一个位图信息，如果对这些机制不了解的话开发过程中就会造成一些困扰。本文通过实例和分析Drawable的缓存机制源码的方式来介绍一下Drawable的缓存机制，并且了解一下Drawable.mutate()的用法。
date: 2017-6-15 10:00:00
---

## 问题演示

下面我们通过一个实例来演示一个我们在使用Drawable过程中经常会遇到的一个问题。

首先贴出UI布局文件，这里放了两个 `ImageView`，它们的寬高不一样，而且对他们加以蓝色的背景。
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
              xmlns:mz="http://schemas.android.com/apk/res-auto"
              android:id="@+id/root"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:padding="40dp"
              android:orientation="vertical">
    <ImageView
        android:id="@+id/first"
        android:layout_width="100dp"
        android:layout_height="200dp"
        android:scaleType="fitXY"
        android:background="#1E90FF"/>

    <ImageView
        android:id="@+id/second"
        android:layout_width="200dp"
        android:layout_height="100dp"
        android:layout_marginTop="50dp"
        android:scaleType="fitXY"
        android:background="#1E90FF"/>
</LinearLayout>
```

### 实例1

首先我们给第一个`ImageView`设置一个显示图片。
```java
        final BitmapDrawable firstDrawable = (BitmapDrawable) getResources()
                .getDrawable(R.drawable.test_mutate);
        mFirstImage = (ImageView) findViewById(R.id.first);
        mSecondImage = (ImageView) findViewById(R.id.second);

        mFirstImage.setImageDrawable(firstDrawable);
```
看下面的效果，因为第二个我们没有设置前景图片，因此会现实背景图片。这个很正常，我们不会有什么疑问。

![效果图](/images/android-knowledge-point-drawable-cache/drawable-1.png)

### 实例2

接下来我们在原来代码的基础上添加下面代码，为第二个`ImageView`设置图片。

```java
        ......
        mSecondImage.setImageDrawable(firstDrawable);
```

看一下效果图，第一个图片的现实效果和**实例1**变的不一样了，你也许会感觉这个很正常，因为同一个`Drawable`对象设置给两个大小不同的`ImageView`，第二个尺寸改变以后第一个也跟着改变了。

![效果图](/images/android-knowledge-point-drawable-cache/drawable-2.png)

### 实例3

那么我们再实例化一个`Drawable`对象设置给第二个`ImageView`。

```java
        final BitmapDrawable firstDrawable = (BitmapDrawable) getResources()
                .getDrawable(R.drawable.test_mutate);
        final BitmapDrawable secondDrawable = (BitmapDrawable) getResources()
                .getDrawable(R.drawable.test_mutate);
        mFirstImage = (ImageView) findViewById(R.id.first);
        mSecondImage = (ImageView) findViewById(R.id.second);

        mFirstImage.setImageDrawable(firstDrawable);
        mSecondImage.setImageDrawable(secondDrawable);
```

看一下效果图，这下显示正常了，这个也可以理解，两个不同的`Drawable`对象设置给不同的`ImageView`，他们互不干涉。那么真的是这样的吗？再接着往下面看。

![效果图](/images/android-knowledge-point-drawable-cache/drawable-3.png)

### 实例4

我们在上面的代码的基础上把第二个`Drawable`的 alpha 设置为15 0 。

```java
        ......
        secondDrawable.setAlpha(150);
```

看下面效果图，奇怪的现象发生了，第一个图片也变成半透明的了，为什么呢？

![效果图](/images/android-knowledge-point-drawable-cache/drawable-4.png)

**问题1:**为什么设置第二个图片的 alpha 会对第一个图片有影响？

### 实例5

你也许听说过 `mutate()` 的作用，那么现在我们改一下代码：

```java
        ......
        secondDrawable.mutate().setAlpha(150);
```

看下面效果图，现在正常了。

![效果图](/images/android-knowledge-point-drawable-cache/drawable-5.png)

**问题2:** `mutate()` 方法是做什么的？

### 实例6

下面我们再对代码稍作修改：

```java
        final BitmapDrawable firstDrawable = (BitmapDrawable) getResources()
                .getDrawable(R.drawable.test_mutate);

        mFirstImage = (ImageView) findViewById(R.id.first);
        mSecondImage = (ImageView) findViewById(R.id.second);

        mFirstImage.setImageDrawable(firstDrawable);
        mSecondImage.setImageDrawable(firstDrawable.getConstantState().newDrawable());
```

这样两个图片也能正常显示出来了。

![效果图](/images/android-knowledge-point-drawable-cache/drawable-3.png)

修改一下最后一行代码：

```java
        Drawable drawable = firstDrawable.getConstantState().newDrawable();
        mSecondImage.setImageDrawable(drawable);
        drawable.setAlpha(150);
```

这样的效果仍然是两个图片都是半透明的。

也需要调用`drawable.mutate().setAlpha(150);`才能使第二个半透明，第一个没有半透明。

**问题3:** `Drawable.getConstantState().newDrawable()`又是怎么回事？

## 问题分析

首先通过**实例3**我们可以得到这样的结论：分别两次调用`getResources().getDrawable(R.drawable.test_mutate)`肯定不是指向同一个对象的。为了验证真实性，我们添加Log。

```
Log.e("Test","firstDrawable = "+firstDrawable+", secondDrawable = "+secondDrawable);
```

有下面打印：

```
3110  3110 E Test    : firstDrawable = android.graphics.drawable.BitmapDrawable@3109294, secondDrawable = android.graphics.drawable.BitmapDrawable@d2fb13d
```

那么，`firstDrawable`和`secondDrawable`肯定不是指向同一个对象了。

### 问题1

我们来分析**问题1**为什么设置第二个图片的 alpha 会对第一个图片有影响？
两个完全不同的`ImageView`因为设置了资源id相同的图片就产生了关联，现在我们可以猜想，`firstDrawable`和`secondDrawable`肯定存在某种联系的。此时我们可能立刻想到为了优化性能，Android内部是不是针对相同的资源使用了同一份位图信息呢？是不是有什么缓存机制呢？带着这个疑问我们先来分析`Resources`的源码。
在`Resources`类中，我们找到了`loadDrawable()`方法：

```java
        ...
        final DrawableCache caches;
        ...
            final Drawable cachedDrawable = caches.getInstance(key, theme);
            if (cachedDrawable != null) {
                return cachedDrawable;
            }
        ...
```

这里会从`caches`里面获取曾经加载过的资源，如果找到就直接返回缓存。具体这个缓存是怎么放进去的我们就不再详细分析了。前面我们也说了，`firstDrawable`和`secondDrawable`是不同的对象，那他们在这个缓存里肯定也不是同一个`Drawable`了。
再直接往下看，`DrawableCache`是什么呢？
`DrawableCache`继承自`ThemedResourceCache`。
下来看一下`DrawableCache`的`getInstance()`方法：

```java
    public Drawable getInstance(long key, Resources.Theme theme) {
        final Drawable.ConstantState entry = get(key, theme);
        if (entry != null) {
            return entry.newDrawable(mResources, theme);
        }

        return null;
    }
```

现在我们知道了，`caches`里面缓存的不是`Drawable`对象，而是`Drawable.ConstantState`对象。

```java
    public static abstract class ConstantState {
        public abstract Drawable newDrawable();

        public Drawable newDrawable(Resources res) {
            return newDrawable();
        }

        public Drawable newDrawable(Resources res, Theme theme) {
            return newDrawable(null);
        }

        public abstract int getChangingConfigurations();
        public int addAtlasableBitmaps(Collection<Bitmap> atlasList) {
            return 0;
        }

        protected final boolean isAtlasable(Bitmap bitmap) {
            return bitmap != null && bitmap.getConfig() == Bitmap.Config.ARGB_8888;
        }

        public boolean canApplyTheme() {
            return false;
        }
    }
```
`ConstantState`类是一个抽象类，`BitmapDrawable.BitmapState`便是它的实现类之一。由于`getResources().getDrawable(R.drawable.test_mutate)`得到的是`BitmapDrawable`，那么我们就重点分析这个类。

```java
    final static class BitmapState extends ConstantState {
        final Paint mPaint;

        int[] mThemeAttrs = null;
        Bitmap mBitmap = null;
        ColorStateList mTint = null;
        Mode mTintMode = DEFAULT_TINT_MODE;
        int mGravity = Gravity.FILL;
        float mBaseAlpha = 1.0f;
        Shader.TileMode mTileModeX = null;
        Shader.TileMode mTileModeY = null;
        int mTargetDensity = DisplayMetrics.DENSITY_DEFAULT;
        boolean mAutoMirrored = false;

        int mChangingConfigurations;
        boolean mRebuildShader;

        BitmapState(Bitmap bitmap) {
            mBitmap = bitmap;
            mPaint = new Paint(DEFAULT_PAINT_FLAGS);
        }

        BitmapState(BitmapState bitmapState) {
            mBitmap = bitmapState.mBitmap;
            mTint = bitmapState.mTint;
            mTintMode = bitmapState.mTintMode;
            mThemeAttrs = bitmapState.mThemeAttrs;
            mChangingConfigurations = bitmapState.mChangingConfigurations;
            mGravity = bitmapState.mGravity;
            mTileModeX = bitmapState.mTileModeX;
            mTileModeY = bitmapState.mTileModeY;
            mTargetDensity = bitmapState.mTargetDensity;
            mBaseAlpha = bitmapState.mBaseAlpha;
            mPaint = new Paint(bitmapState.mPaint);
            mRebuildShader = bitmapState.mRebuildShader;
            mAutoMirrored = bitmapState.mAutoMirrored;
        }

        @Override
        public boolean canApplyTheme() {
            return mThemeAttrs != null || mTint != null && mTint.canApplyTheme();
        }

        @Override
        public int addAtlasableBitmaps(Collection<Bitmap> atlasList) {
            if (isAtlasable(mBitmap) && atlasList.add(mBitmap)) {
                return mBitmap.getWidth() * mBitmap.getHeight();
            }
            return 0;
        }

        @Override
        public Drawable newDrawable() {
            return new BitmapDrawable(this, null);
        }

        @Override
        public Drawable newDrawable(Resources res) {
            return new BitmapDrawable(this, res);
        }

        @Override
        public int getChangingConfigurations() {
            return mChangingConfigurations
                    | (mTint != null ? mTint.getChangingConfigurations() : 0);
        }
    }
```

在`newDrawable()`方法里面返回的是一个新的`BitmapDrawable`对象，但是所有相同资源的`BitmapDrawable`对象共用同一个`BitmapState`对象。我们注意到`BitmapState`的`mBitmap`属性，这也验证了前面的猜想，它们共用同一个`Bitmap`。

```
    private BitmapDrawable(BitmapState state, Resources res) {
        mBitmapState = state;

        updateLocalState(res);
    }

    @Override
    public void setAlpha(int alpha) {
        final int oldAlpha = mBitmapState.mPaint.getAlpha();
        if (alpha != oldAlpha) {
            mBitmapState.mPaint.setAlpha(alpha);
            invalidateSelf();
        }
    }
```

那么我们`setAlpha()`操作实际改变的是`mBitmapState`的属性值，这也就不难理解**问题1**了，因为它们用的是同一个`BitmapState`对象。
为了验证这个结论，我们添加打印：

```java
        Log.e("Test","firstDrawable = "+firstDrawable.getConstantState()+", secondDrawable = "+secondDrawable.getConstantState());
```

打印如下：

```
4433  4433 E Test    : firstDrawable = android.graphics.drawable.BitmapDrawable$BitmapState@3109294, secondDrawable = android.graphics.drawable.BitmapDrawable$BitmapState@3109294
```

它们确实是指向同一个对象的。

它们的关系可以用下图表示：

![效果图](/images/android-knowledge-point-drawable-cache/drawable-6.jpg)

### 问题2

接下来再来分析**问题2:** `mutate()` 方法是做什么的？

我们先来看一下`Drawable`中对这个方法的解释：

```java
    /**
     * Make this drawable mutable. This operation cannot be reversed. A mutable
     * drawable is guaranteed to not share its state with any other drawable.
     * This is especially useful when you need to modify properties of drawables
     * loaded from resources. By default, all drawables instances loaded from
     * the same resource share a common state; if you modify the state of one
     * instance, all the other instances will receive the same modification.
     *
     * Calling this method on a mutable Drawable will have no effect.
     *
     * @return This drawable.
     * @see ConstantState
     * @see #getConstantState()
     */
    public Drawable mutate() {
        return this;
    }
```

`mutate()`返回的`Drawable`对象不再与同资源的其他`Drawable`共用 state，那么它的属性改变后就不再影响其他的`Drawable`了。

在`BitmapDrawable`的`mutate()`方法里面确实又新建了一个`BitmapState`对象。

```java
    /**
     * A mutable BitmapDrawable still shares its Bitmap with any other Drawable
     * that comes from the same resource.
     *
     * @return This drawable.
     */
    @Override
    public Drawable mutate() {
        if (!mMutated && super.mutate() == this) {
            mBitmapState = new BitmapState(mBitmapState);
            mMutated = true;
        }
        return this;
    }
```

它们的关系可以用下图表示：

![效果图](/images/android-knowledge-point-drawable-cache/drawable-7.jpg)

**注意：** mutate操作是不可逆转的，已经调用过`mutate()`方法的`BitmapDrawable`对象再调用`mutate()`是不起作用的。这点在代码中可以清楚的看到。

### 问题3
记下来分析**问题3:** `Drawable.getConstantState().newDrawable()`又是怎么回事？
经过上面的源码分析，这个很容易就理解了，它就是获得`Drawable`的`ConstantState`来重新实例化一个`Drawable`，两个`Drawable`还是共用一个`ConstantState`。
这个和重新调用`getResources().getDrawable(R.drawable.test_mutate)`原理是一样的。

### 附加问题

那为什么设置 alpha 两个图片互有影响，而在**实例3**中第二个`Drawable`大小尺寸改变却没有影响呢？
这就要附带分析一下`ImageView`的`ScaleType`原理。
我们从`ImageView.setImageDrawable`开始分析，这个方法的调用流程如图：

```
├── ImageView.setImageDrawable
     └── ImageView.updateDrawable
          └── configureBounds()
```

```java
    private void configureBounds() {
        ...
        if (dwidth <= 0 || dheight <= 0 || ScaleType.FIT_XY == mScaleType) {
            mDrawable.setBounds(0, 0, vwidth, vheight);
            mDrawMatrix = null;
        }
        ...
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        ...

        if (mDrawMatrix == null && mPaddingTop == 0 && mPaddingLeft == 0) {
            mDrawable.draw(canvas);
        } else {
            int saveCount = canvas.getSaveCount();
            canvas.save();
            
            if (mCropToPadding) {
                final int scrollX = mScrollX;
                final int scrollY = mScrollY;
                canvas.clipRect(scrollX + mPaddingLeft, scrollY + mPaddingTop,
                        scrollX + mRight - mLeft - mPaddingRight,
                        scrollY + mBottom - mTop - mPaddingBottom);
            }
            
            canvas.translate(mPaddingLeft, mPaddingTop);

            if (mDrawMatrix != null) {
                canvas.concat(mDrawMatrix);
            }
            mDrawable.draw(canvas);
            canvas.restoreToCount(saveCount);
        }
    }
```

在`configureBounds()`里面根据不同的`ScaleType`会进行不同的变换，包括设置绘制边界、缩放、位移、绘制是的矩阵变换等等。
在`onDraw()`方法中再把这个`Drawable`绘制到`Canvas`上，这些改变变化的只是`Drawable`本身，而对`ConstantState`不会有改变。
为了验证这个结论，我们在**实例3**代码基础上，添加一些Log。

```java
    public void refresh(View v){
        Log.e("Test","1 rect1 = "+mFirstImage.getDrawable().getBounds());
        Log.e("Test","2 rect2 = "+mSecondImage.getDrawable().getBounds());
    }
```

打印如下：

```
21313 21313 E Test    : 1 rect1 = Rect(0, 0 - 200, 400)
21313 21313 E Test    : 2 rect2 = Rect(0, 0 - 400, 200)
```

它们的`Drawable.mBounds`是不同的。

## 参考文章

https://android-developers.googleblog.com/2009/05/drawable-mutations.html

