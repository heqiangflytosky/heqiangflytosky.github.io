---
title: SystemUI --  QSTile 动画实现之 QSAnimator
categories:  Android SystemUI
comments: true
tags: [SystemUI]
description: 介绍QSTile 动画实现原理
date: 2021-10-30 10:00:00
---


在上下滑动通知栏过程中，伴随着 QSTile 联动的一些动画，这些动画是由 QSAnimator 实现的。
在 QSFragment 中有个 QSAnimator 对象，当QS位置变化或者 QSAnimator onLayoutChange 时，会调用 QSAnimator 的  `setPosition()` 方法来更新 View 的属性来实现相应动画效果。

```
QSAnimator.onLayoutChange()
    QSAnimator.updateAnimators()
    QSAnimator.setCurrentPosition()
        QSAnimator.setPosition()
            TouchAnimator.setPosition()
```

TouchAnimator是对动画类的封装，而其内建的Builder是对动画参数的配置，build方法直接返回了一个TouchAnimator对象。
下面我们以亮度前的缩放为例来介绍一下。

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

updateAnimators() 方法中构造了 TouchAnimator 对象，设置了需要做动画的属性。getProperty() 方法根据设置的配置返回了 View 对象的属性，View 对象保存到了 mTargets ,FloatKeyframeSet 保存到 mValues 中。

```
//TouchAnimator.java

        public Builder addFloat(Object target, String property, float... values) {
            add(target, KeyframeSet.ofFloat(getProperty(target, property, float.class), values));
            return this;
        }

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
        @Override
        protected void interpolate(int index, float amount, Object target) {
            float firstFloat = mValues[index - 1];
            float secondFloat = mValues[index];
            mProperty.set((T) target, firstFloat + (secondFloat - firstFloat) * amount);
        }

```

关键的mProperty.set语句实际上就相当于：View.SCALE_Y.setValue(view, scale) -> BrightnessSliderView.setScaleY(scale);

