---
title: SystemUI --  QSTile 动画实现之 QSAnimator
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍QSTile 动画实现原理
date: 2021-10-30 10:00:00
---


在上下滑动通知栏过程中，伴随着 QSTile 联动的一些动画，这些动画是由 QSAnimator 实现的。
QSAnimator 实现了 OnLayoutChangeListener 接口，并且在初始化时为 QSContainerImpl 添加了监听器。

```
qs.getView().addOnLayoutChangeListener(this)
```

在 QSFragment 中有个 QSAnimator 对象，它会在 updateAnimators() 方法中构建所有的 `TouchAnimator.Builder()` 对象，并且绑定需要做动画的属性。当QS布局变化时，会调用 QSAnimator 的  `setPosition()` 方法来更新所有 View 的属性来实现相应动画效果。

```
QSAnimator.onLayoutChange()
    QSAnimator.updateAnimators()
    QSAnimator.setCurrentPosition()
        QSAnimator.setPosition()
            TouchAnimator.setPosition()
```

TouchAnimator是对动画类的封装，而其内建的Builder是对动画参数的配置，build方法直接返回了一个 TouchAnimator 对象。
下面我们以亮度条的缩放为例来介绍一下。

```
// QSAnimator.java
    private TouchAnimator mBrightnessAnimator;
    private void updateAnimators() {
        ......
                mBrightnessAnimator = new TouchAnimator.Builder()
                        .addFloat(brightness, "alpha", 0, 1)
                        .addFloat(brightness, "sliderScaleY", 0.3f, 1)
                        .setInterpolator(Interpolators.ALPHA_IN)
                        .setStartDelay(0.3f)
                        .build();
```

updateAnimators() 方法中构造了 TouchAnimator 对象，设置了需要做动画的属性。

```
//TouchAnimator.java
        // add 方法将View 对象保存到了 Builder.mTargets ,FloatKeyframeSet 保存到 Builder.mValues 中。values 中包含了初始值和最终值
        public Builder addFloat(Object target, String property, float... values) {
            add(target, KeyframeSet.ofFloat(getProperty(target, property, float.class), values));
            return this;
        }
        // getProperty() 方法根据设置的配置返回了 View 对象的属性
        private static Property getProperty(Object target, String property, Class<?> cls) {
            if (target instanceof View) {
                switch (property) {
                    case "translationX":
                        return View.TRANSLATION_X;
                    case "translationY":
                        return View.TRANSLATION_Y;
                    case "translationZ":
                        return View.TRANSLATION_Z;
                    case "alpha":
                        return View.ALPHA;
                    case "rotation":
                        return View.ROTATION;
                    case "x":
                        return View.X;
                    case "y":
                        return View.Y;
                    case "scaleX":
                        return View.SCALE_X;
                    case "scaleY":
                        return View.SCALE_Y;
                }
            }
            if (target instanceof TouchAnimator && "position".equals(property)) {
                return POSITION;
            }
            return Property.of(target.getClass(), cls, property);
        }

        // 构造一个FloatKeyframeSet
        public static KeyframeSet ofFloat(Property property, float... values) {
            return new FloatKeyframeSet((Property<?, Float>) property, values);
        }
        
        private void add(Object target, KeyframeSet keyframeSet) {
            mTargets.add(target);
            mValues.add(keyframeSet);
        }
        
        public TouchAnimator build() {
            return new TouchAnimator(mTargets.toArray(new Object[mTargets.size()]),
                    mValues.toArray(new KeyframeSet[mValues.size()]),
                    mStartDelay, mEndDelay, mInterpolator, mListener);
        }
```

FloatKeyframeSet 继承自 KeyframeSet，可以为View的一些属性设置float类型的值。它保存了我们为动画设置的View属性的名称、关键帧的个数、以及属性的数值。
比如 `addFloat(brightness, "alpha", 0, 1)` 我们就为 brightness 进行 alpha 属性的变化，关键帧的个数为2个，只设置了初始值和目标值，分别为 0 和 1 。后面就会根据QS的位置以及view的alpha的初始值和目标值计算出当前应该为brightness设置的alpha的值。通过这样来实现根据QS位置的变化来达到brightness透明度变化以及SCALE_Y变化的目的。


再来看看动画的执行：

```
QSAnimator.setPosition()
    mBrightnessAnimator.setPosition()
        KeyframeSets[i].setValue()
            FloatKeyframeSet.interpolate()
                mProperty.set()
```

```
//TouchAnimator.java

        void setValue(float fraction, Object target) {
            //constrain是求百分比的函数，比如 MathUtils.constrain(5, 0, 100) 表示 （100-0）*5。
            int i = MathUtils.constrain((int) Math.ceil(fraction / mFrameWidth), 1, mSize - 1);
            float amount = (fraction - mFrameWidth * (i - 1)) / mFrameWidth;
            interpolate(i, amount, target);
        }

        @Override
        protected void interpolate(int index, float amount, Object target) {
            float firstFloat = mValues[index - 1];
            float secondFloat = mValues[index];
            // 根据初始值，目标值和当前的 amount 计算处属性的当前值。
            mProperty.set((T) target, firstFloat + (secondFloat - firstFloat) * amount);
        }

```

关键的mProperty.set语句实际上就相当于：View.SCALE_Y.setValue(view, scale) -> BrightnessSliderView.setScaleY(scale)，为View的某个属性设置一个数值。

