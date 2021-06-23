---
title: Android 图形系统 -- View 绘制流程
categories: Android 图形系统
comments: true
tags: [Android 图形系统]
description: 介绍 View 绘制流程
date: 2016-8-2 10:00:00
---

## 概述

View 的绘制流程可以分为 measure、layout、draw 三个阶段完成，ViewGroup 和 View 在三个阶段的处理流程又是不一样的，理解这三个阶段的处理流程将会对我们自定义 View 会有很大帮助。本文将对这三个阶段做一下简单介绍。
View 的绘制流程是从 ViewRootImpl 类的 performTraversals 方法开始的，依次经过 performMeasure、performLayout、performDraw 三个方法，最终分别调用 View 或者 ViewGroup 的 measure/onMeasure、layout/onLayout、draw/onDraw 方法来完成 View 的测量、布局和绘制工作。
这个流程其实很好理解，我们要绘制一个视图，必须首先知道视图大小，然后知道把它放到什么位置，然后绘制才能在屏幕上显示出来。
下面来看一下绘制流程的函数调用关系：

```
ViewRootImpl.performTraversals
    ├── ViewRootImpl.performMeasure
        ├── View.measure
            ├── View.onMeasure(ViewGroup.onMeasure)
    ├── ViewRootImpl.performLayout
        ├── View.layout
            ├── ViewGroup.onLayout
    ├── ViewRootImpl.performDraw
    　　├── ViewRootImpl.draw
    　　　　├── ViewRootImpl.drawSoftware
　　　　　　　　├── View.draw
　　　　　　　　　　├── View.onDraw

```


## 测量

measure 方法是 View 绘制流程的第一步，在测量过程中，measure() 方法被父 View 调用，在 measure() 中做一些优化工作后，就调用 onMeasure() 方法进行自我测量，根据自己的规格来测量出真实尺寸。
measure()为View的final方法，不可重写。
对于 onMeasure() 方法，View 和 ViewGroup 的实现会有所区别：

 - View：在 onMeasure() 根据自己的规格来计算自己的尺寸并保存。
 - ViewGroup：在 onMeasure() 中调用所有子 View 的measure() 方法让它们进行自我测量，然后根据子 View  计算出的期望尺寸来计算出它们的时机尺寸和位置并保存。在此过程中，子 View 也完成了自身的测量任务。

在学习具体的测量流程之前，我们先学习点有关测量的基础知识。

### MeasureSpec

View 的 onMeasure　方法传入的参数是　`onMeasure(int widthMeasureSpec, int heightMeasureSpec)`，widthMeasureSpec 和 heightMeasureSpec 就代表了两个 MeasureSpec 值，它用来把测量要求从父 View 传递到子 View。子 View 的大小是由父 View 的测量要求和子 View LayoutParams 共同决定的（这个规则会在介绍 getChildMeasureSpec 章节进行详细的介绍），这里父 View 的测量要求就是这个 MeasureSpec，它是一个32位的int值。

 - 高2位：SpecMode，测量模式，有三种类型：
  - UNSPECIFIED：父 View 不对子 View 做任何限制，需要多大给多大，这种情况多用于系统内部，表示一种测量状态。
  - EXACTLY：父 View 已经检测出子 View 所需要的精确大小，子View应该服从这个边界。这个时候子 View 的最终大小就是 MeasureSpec 低30位指定的值。它对应于 LayoutParams 中给出具体大小值和 match_parent 这两种情况。
  - AT_MOST：父 View 给出子 View 一个最大可用大小，子 View 自己去适应这个大小。它对应于 wrap_content 这种情况。
 - 低30位：SpecSize，在特定测量模式下的大小。

关于 MeasureSpec 的一些方法：

 - MeasureSpec.makeMeasureSpec(size,mode)：将大小和模式生成一个int的规格，高两位代表模式，低30位代表尺寸，实际使用的是位运算生成。
 - MeasureSpec.getMode(measureSpec)：取出一个规格的测量模式，实际使用的是位运算生成。
 - MeasureSpec.getSize(measureSpec)：取出一个规格的尺寸，也是位运算生成。

### 关于测量的几个方法

ViewGroup 提供了几个重要的基础方法来测量子 view，View 提供了 setMeasuredDimension() 来保存测量的宽高结果。

#### measureChildren()

```
    protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
        final int size = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < size; ++i) {
            final View child = children[i];
            if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
                measureChild(child, widthMeasureSpec, heightMeasureSpec);
            }
        }
    }
```

其实就是遍历子 View，调用 measureChild 方法(显示状态为GONE的view不会测量)。

#### measureChild()

```
    protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

调用 child 的 getLayoutParam 方法拿到布局参数，然后结合父view传来的规格(也就是measure时传来的两个参数)调用 getChildMeasureSpec() 方法生成子view的具体宽高规格，并调用 child.measure 方法进行子view的测量；getChildMeasureSpec 方法就是具体规格生成规则的方法。

#### measureChildWithMargins()

```
    protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

该方法只是在measureChild方法基础上考虑到了子view的margin属性，所以这里需要将child的LayoutParam强转成MarginLayoutParam使用其margin相关属性，一般情况下的ViewGroup都会生产一个继承自MarginLayoutParam的LayoutParam类，所以在测量时可以调用此方法快捷测量，但是如果自定义的某个ViewGroup没有使用继承MarginLayoutParam的LayoutParam，那么就不能调用此方法，否则会强转时异常;

#### getChildMeasureSpec(int spec, int padding, int childDimension)

```
    //spec参数   表示父View的MeasureSpec
    //padding参数    父View的Padding+子View的Margin，父View的大小减去这些边距，才能精确算出子View的MeasureSpec的size
    //childDimension参数  表示该子View内部LayoutParams属性的值（lp.width或者lp.height）
    //可以是wrap_content、match_parent、一个精确值，比如20dp
    public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);//获得父View的mode 
        int specSize = MeasureSpec.getSize(spec);//获得父View的大小

        //父View的大小-自己的Padding+子View的Margin，得到值才是子View的大小。
        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //根据上面逻辑条件获取的mode和size构建MeasureSpec对象
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

这就是前面说的根据父类要求的规格和自身的 LayoutParams 计算出实际规格的方法，第一个参数是父view要求的规格(宽度或高度的)；第二个参数是在该属性(宽度或高度)上已使用的值，计算时应该刨除；第三个是在该属性上子view自己想要的大小(具体值、MATCH_PARENT、WRAP_CONTENT)。
这里用一个表格来直观的表示上面的规则：

| 父View的测量规格 | 子View的宽高属性 | 得到的子控件的测量规格 | 说明 |
|:-------------:|:-------------:|:-------------:|:-------------:|
| EXACTLY | 具体大小（20dp）/ match_parent | EXACTLY |  |
|  | wrap_content | AT_MOST |  |
| AT_MOST | 具体大小（20dp） | EXACTLY |  |
|  | match_parent / wrap_content | AT_MOST |  |
| UNSPECIFIED | 具体大小（20dp） | EXACTLY |  |
|  | match_parent / wrap_content | UNSPECIFIED |  |

#### resolveSizeAndState(size,measureSpec,state)

```
    public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
        final int specMode = MeasureSpec.getMode(measureSpec);
        final int specSize = MeasureSpec.getSize(measureSpec);
        final int result;
        switch (specMode) {
            case MeasureSpec.AT_MOST:
                if (specSize < size) {
                    result = specSize | MEASURED_STATE_TOO_SMALL;
                } else {
                    result = size;
                }
                break;
            case MeasureSpec.EXACTLY:
                result = specSize;
                break;
            case MeasureSpec.UNSPECIFIED:
            default:
                result = size;
        }
        return result | (childMeasuredState & MEASURED_STATE_MASK);
    }
```

该方法是根据测量得到的尺寸与原有的规格限制，决定最终的尺寸，也就是调整测量得到的尺寸，使其满足原有规格的限制。

#### setMeasuredDimension()

```
    protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;

            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }
    
    private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```

onMeasure 测量后，一定要记得调用 setMeasuredDimension 来保存测量的宽高结果。

### 测量流程

在Andrond常用的几种布局中，FrameLayout 的测量流程是比较简单的，我们先来看一下 FrameLayout 的测量流程，它覆盖了 View 的 onMeasure 方法来实现自己的测量逻辑：

```
├── FrameLayout.onMeasure
    ├── MeasureSpec.getMode
    ├── ViewGroup.measureChildWithMargins
        ├── View.measure(child.measure)
    ├── View.setMeasuredDimension
    ├── MeasureSpec.makeMeasureSpec
    ├── ViewGroup.getChildMeasureSpec
    ├── View.measure(child.measure)
```

```
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();
	// 有非EXACTLY mode的MeasureSpec（某个方向或者某两个方向尺寸不确定）
        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
		//遍历自己的子View，只要不是GONE的都会参与测量
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
		//根据测量后子view的大小，得出子view的最大宽高
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {
                    // 保存设置了MATCH_PARENT的子view
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        // Account for padding too
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        // Check against our minimum height and width
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        // Check against our foreground's minimum height and width
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }
	//所有的孩子测量之后，经过一系类的计算之后通过setMeasuredDimension设置自己的宽高，
	//对于FrameLayout 可能用最大的字View的大小
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));
	//如果有子view设置了MATCH_PARENT，重新测量子view
        //因为前面的测量对于设置了MATCH_PARENT的View，无法给出确切的值，所以要再次调用子View的measure方法，传入正确的值。
        count = mMatchParentChildren.size();
        if (count > 1) {
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);
                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

                final int childWidthMeasureSpec;
                if (lp.width == LayoutParams.MATCH_PARENT) {
                    final int width = Math.max(0, getMeasuredWidth()
                            - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                            - lp.leftMargin - lp.rightMargin);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            width, MeasureSpec.EXACTLY);
                } else {
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }

                final int childHeightMeasureSpec;
                if (lp.height == LayoutParams.MATCH_PARENT) {
                    final int height = Math.max(0, getMeasuredHeight()
                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                            - lp.topMargin - lp.bottomMargin);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            height, MeasureSpec.EXACTLY);
                } else {
                    childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                            getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                            lp.topMargin + lp.bottomMargin,
                            lp.height);
                }

                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }
    }
```

再来看一下　TextView 的测量流程：

```
├── TextView.onMeasure
    ├── MeasureSpec.getMode
    ├── MeasureSpec.getSize
    ├── setMeasuredDimension
```

下面我们来详细看一下　View 的　onMeasure 方法。

```
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    
    public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
    
    protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
```

View 根据自身的测量规格计算自己实际的宽高，并保存宽高数据。
在默认的实现 getDefaultSize 方法中，如果父 View 的测量要求是 MeasureSpec.AT_MOST 和 MeasureSpec.EXACTLY，子 View 的大小都是父 View 给出的 MeasureSpec 值里面保存的期望值大小，这是如果子 View 的属性设置的是精确值，此时测量结果就是给出的精确值，如果设置的是 match_parent 或者 wrap_content 那么给到子 View 的值就是 MeasureSpec 里面保存的父 View给的一个最大值，那么子 View 的表现就是都会填充父 View。
根据上面的代码，自定义的 View 如果我们只是要设置具体的宽高，或者是要填充父 View ，这种情况下可以不覆盖 onMeasure 方法，如果要设置 wrap_content ，那么是必须要覆盖一下 onMeasure 方法，为空间设置合适的尺寸，否则 wrap_content 会和 match_parent一样填充父组件。

在 measure 流程进行完后，我们就可以通过 `getMeasureHeight()/getMeasuredWidth()` 来获取宽高。但是在某些场景下，系统需要进行多次的 measure 才能确定最终的宽高，因此在 onMeasure() 中拿到的宽高很可能是不一样的，比较好的做法是在 measure 进行完后在 onLayout() 方法中再通过 `getMeasureHeight()/getMeasuredWidth()` 获取宽高。

### 总结

测量控件大小是父控件发起的。
父控件要测量子控件大小，需要重写onMeasure方法，然后调用 measureChildren 或者 measureChildWithMargins 方法。
View 的 onMeasure 方法的参数是通过 getChildMeasureSpec 生成的
如果我们自定义控件需要使用 wrap_content 属性,我们需要重写 onMeasure 方法。
测量控件的步骤：
父控件 onMeasure -> measureChildren/measureChildWithMargin -> getChildMeasureSpec -> 子控件的measure -> onMeasure -> setMeasureDimension -> 父控件 onMeasure 结束调用 setMeasureDimension 最后保存自己的大小。

## 布局

进行布局的时候，layout() 方法被父 View 调用，在 layout() 方法中它会保存父 View 传递进来的自己的位置和尺寸，并且调用 onLayout() 来进行内部实际的布局操作。对于 onLayout() View 和 ViewGroup 也是有所区别的：

 - View：由于没有子 View，它的　onLayout　方法是个空实现。
 - ViewGroup：它的　onLayout　方法是个抽象方法，继承自　ViewGroup　的类（比如　LinearLayout 和　FrameLayout）　必须实现这个方法。在 onLayout() 方法中会调用自己所有的子 View 的 layout() 方法，把它们的尺寸传递给它们，让它们完成自我的内部布局。

来看一下 View 的 layout() 方法的调用流程：

```
├── View.layout
    ├── View.setFrame
    ├── onLayout
    ├── onLayoutChange
```

这个流程其实很简单，首先调用 setFrame 设置View的四个顶点位置，然后调用 onLayout 确定子元素的位置，后面调用 onLayoutChange 回调。
ViewGroup 的 layout() 方法是个 final 方法，内部实现比较简单，主要调用 `super.layout(l, t, r, b)`。
onLayout() 依赖于具体的布局，因此 View和ViewGroup都没有实现这个方法，我们来看看 FrameLayout 的实现。

```
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }

    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        // 获取子 View 的个数
        final int count = getChildCount();
        // 计算一下当前 View 的内边距
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();
        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();
        // 循环遍历子 View
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            // 判断是否需要进行布局
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                // 获取测量高度
                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();

                int childLeft;
                int childTop;
                // 获取 gravity 属性
                int gravity = lp.gravity;
                if (gravity == -1) {
                    gravity = DEFAULT_CHILD_GRAVITY;
                }
                // 获取布局方向
                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;
                // 计算 左上角的位置
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT:
                        if (!forceLeftGravity) {
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    case Gravity.LEFT:
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }

                switch (verticalGravity) {
                    case Gravity.TOP:
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL:
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM:
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:
                        childTop = parentTop + lp.topMargin;
                }

                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
```

布局子 View 的时候，首先根据 gravity 属性计算出左上角位置，然后调用 chuild 的layout 方法，进行各自的布局。
FrameLayout 的布局实现其实挺简单的，因为 FrameLayout 的子 View 都是叠加在一起的，不会竞争空间，彼此不存在影响，因此，在支持布局操作时，我们只要考虑子 View 自身相对于 FrameLayout 的举例即可。

## 绘制

在确定好 View 的位置之后，就要就行绘制操作了，我们先看一下 View.draw() 方法的调用流程，ViewGroup没有重写 View 的这个方法。
draw 方法其实就是在传入的参数 Cavas 上面进行一些绘制。

```
├── View.draw
    ├── drawBackground
    ├── saveUnclippedLayer
    ├── onDraw
    ├── dispatchDraw
    ├── drawRect
    ├── onDrawForeground
```

具体的步骤在源码里面注释里写的非常清楚：

  - Step 1, draw the background, if needed
  - Step 2, save the canvas' layers
  - Step 3, draw the content
  - Step 4, draw the children
  - Step 5, draw the fade effect and restore layers
  - Step 6, draw decorations (foreground, scrollbars)
  - Step 7, draw the default focus highlight

自定义 View 可以在 onDraw 方法里面自定义自己的实现。

### 绘制顺序

另外，我们再介绍一个小技巧：修改子View的绘制顺序。
这个需求的使用场景是什么呢？比如有时我们想把某个下面的子View全部显示出来，不被其他View遮挡，这时就需要调整绘制顺序了。我们先看一下 `dispatchDraw()` 的源码：

```
    @Override
    protected void dispatchDraw(Canvas canvas) {
        ......
        // Only use the preordered list if not HW accelerated, since the HW pipeline will do the
        // draw reordering internally
        // 看是否有子View设置Z轴，只有在关闭硬件加速时才会生效
        final ArrayList<View> preorderedList = usingRenderNodeProperties
                ? null : buildOrderedChildList();
        // 是否启用自定义绘制顺序，只有当 preorderedList 为空而且使能自定义顺序时才生效
        final boolean customOrder = preorderedList == null
                && isChildrenDrawingOrderEnabled();
        for (int i = 0; i < childrenCount; i++) {
            while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
                final View transientChild = mTransientViews.get(transientIndex);
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                        transientChild.getAnimation() != null) {
                    more |= drawChild(canvas, transientChild, drawingTime);
                }
                transientIndex++;
                if (transientIndex >= transientCount) {
                    transientIndex = -1;
                }
            }
            // 获取需要绘制view的索引
            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            // 根据索引来获取需要绘制的View
            final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        while (transientIndex >= 0) {
            // there may be additional transient views after the normal views
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                more |= drawChild(canvas, transientChild, drawingTime);
            }
            transientIndex++;
            if (transientIndex >= transientCount) {
                break;
            }
        }
        if (preorderedList != null) preorderedList.clear();

        ......
```

我们先来看一下 `buildOrderedChildList ` 方法：

```
    ArrayList<View> buildOrderedChildList() {
        final int childrenCount = mChildrenCount;
        if (childrenCount <= 1 || !hasChildWithZ()) return null;

        if (mPreSortedChildren == null) {
            mPreSortedChildren = new ArrayList<>(childrenCount);
        } else {
            // callers should clear, so clear shouldn't be necessary, but for safety...
            mPreSortedChildren.clear();
            mPreSortedChildren.ensureCapacity(childrenCount);
        }

        final boolean customOrder = isChildrenDrawingOrderEnabled();
        for (int i = 0; i < childrenCount; i++) {
            // add next child (in child order) to end of list
            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            final View nextChild = mChildren[childIndex];
            final float currentZ = nextChild.getZ();

            // insert ahead of any Views with greater Z
            int insertIndex = i;
            while (insertIndex > 0 && mPreSortedChildren.get(insertIndex - 1).getZ() > currentZ) {
                insertIndex--;
            }
            mPreSortedChildren.add(insertIndex, nextChild);
        }
        return mPreSortedChildren;
    }
```

`buildOrderedChildList` 的逻辑就是按照 Z 轴调整 children 顺序，Z 轴值相同则参考 customOrder 的配置。
`getAndVerifyPreorderedIndex ` 方法是获取获取需要绘制view的索引

```
    private int getAndVerifyPreorderedIndex(int childrenCount, int i, boolean customOrder) {
        final int childIndex;
        // 如果开启自定义绘制顺序，就根据自定义顺序来绘制，否则就按照默认顺序绘制
        if (customOrder) {
            final int childIndex1 = getChildDrawingOrder(childrenCount, i);
            if (childIndex1 >= childrenCount) {
                throw new IndexOutOfBoundsException("getChildDrawingOrder() "
                        + "returned invalid index " + childIndex1
                        + " (child count is " + childrenCount + ")");
            }
            childIndex = childIndex1;
        } else {
            childIndex = i;
        }
        return childIndex;
    }

    protected int getChildDrawingOrder(int childCount, int i) {
        return i;
    }
```

因此，我们可以得到自定义绘制顺序的两种方法：

 - 修改子view的Z轴大小，可以通过 `view.setZ(float z);`z值越大，越靠近顶端。只有在不开启硬件加速时才生效
 - 调用 `setChildrenDrawingOrderEnable(true)` 开启自定义绘制顺序，并且重写 `getChildDrawingOrder()` 修改子 View 的取值索引。

其实，还有一种方法，调用 `view.bringToFront()`，但是这种方式是系统先将view移除出当前viewGroup，然后再添加进来，会导致重新绘制当前界面。

```
    @Override
    public void bringChildToFront(View child) {
        final int index = indexOfChild(child);
        if (index >= 0) {
            removeFromArray(index);
            addInArray(child, mChildrenCount);
            child.mParent = this;
            requestLayout();
            invalidate();
        }
    }
```

其实，`RecyclerView` 给我们提供了一个回调来自定义子View的绘制顺序，它就是重写了 `getChildDrawingOrder` 方法：

```
    public interface ChildDrawingOrderCallback {
        /**
         * Returns the index of the child to draw for this iteration. Override this
         * if you want to change the drawing order of children. By default, it
         * returns i.
         *
         * @param i The current iteration.
         * @return The index of the child to draw this iteration.
         *
         * @see RecyclerView#setChildDrawingOrderCallback(RecyclerView.ChildDrawingOrderCallback)
         */
        int onGetChildDrawingOrder(int childCount, int i);
    }

    @Override
    protected int getChildDrawingOrder(int childCount, int i) {
        if (mChildDrawingOrderCallback == null) {
            return super.getChildDrawingOrder(childCount, i);
        } else {
            return mChildDrawingOrderCallback.onGetChildDrawingOrder(childCount, i);
        }
    }
```

## 一些知识点

### getMeasureHeight 和 getHeight 的区别

关于 View 的宽高，可以由两对值得到：`getMeasureHeight()/getMeasuredWidth()` 和 `getHeight()/getWidth()` 。
`getMeasureHeight()/getMeasuredWidth()` 表示该view在它的父view里期望的值，在 measure 后可以得到。
`getHeight()/getWidth()` 表示该 View 在屏幕上的实际大小，在 layout 方法后可以得到。
实际上在当屏幕可以包裹内容的时候，他们的值相等，只有当 View 超出屏幕后，才能看出他们的区别： `getMeasuredHeight()/getMeasuredWidth()` 是实际 View 的大小，与屏幕无关，而 `getHeight()/getWidth()` 的大小此时则是屏幕的大小。当超出屏幕后，`getMeasureHeight()/getMeasuredWidth()`等于 `getHeight()/getWidth()` 加上屏幕之外没有显示的大小。

## 推荐阅读

[绘制流程小细节，如何修改 View绘制的顺序？](https://mp.weixin.qq.com/s/xAl1yPs4al2czSXsw6__hQ)
