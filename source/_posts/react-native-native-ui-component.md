---
title: React Native封装原生UI组件
categories: React Native
comments: true
tags: [React Native]
description:
date: 2017-01-15 10:00:00
---
在React Native开发过程中，有时我们想要使用原生的一个UI组件或者是JS比较难以实现的动画效果时，我们可以在React Naitve应用程序中封装和植入已有的原生组件。
比如开源项目[Lottie](https://github.com/airbnb/lottie-android)在Android上能够非常简单的实现一些复杂的动画效果，如果我们想在JS中也实现这样的效果呢？很简单，我们可以自己构建一个原生UI组件。
<!-- more -->
接下来就以此为例来进行介绍。Lottie官方已经提供了[React Native版本的Lottie](https://github.com/airbnb/lottie-react-native)了，但这里我们作为例子再来介绍一下。
[官方中文文档](http://reactnative.cn/docs/0.42/native-component-android.html#content)
先看一下效果图：
<img src="/images/react-native-native-ui-compent/lottie.gif" width="360" height="620"/>
## 封装组件
### 创建ViewManager的子类
创建`LottieViewManager`类，并实现`getName()`和`createViewInstance`方法：
```
public class LottieViewManager extends SimpleViewManager<LottieAnimationView> {
    private static final String REACT_CLASS = "LottieAnimationView";
    @Override
    public String getName() {
        return REACT_CLASS;
    }

    @Override
    protected LottieAnimationView createViewInstance(ThemedReactContext reactContext) {
        return new LottieAnimationView(reactContext);
    }
}
```

### 导出属性的设置方法
要导出给JavaScript使用的属性，需要申明带有`@ReactProp`（或`@ReactPropGroup`）注解的设置方法。方法的第一个参数是要修改属性的视图实例，第二个参数是要设置的属性值。方法的返回值类型必须为`void`，而且访问控制必须被声明为`public`。JavaScript所得知的属性类型会由该方法第二个参数的类型来自动决定。支持的类型有：`boolean`, `int`, `float`, `double`, `String`, `Boolean`, `Integer`, `ReadableArray`, `ReadableMap`。
`@ReactProp`注解必须包含一个字符串类型的参数`name`。这个参数指定了对应属性在JavaScript端的名字。
除了`name`，`@ReactProp`注解还接受这些可选的参数：`defaultBoolean`, `defaultInt`, `defaultFloat`。这些参数必须是对应的基础类型的值（也就是`boolean`, `int`, `float`），这些值会被传递给`setter`方法，以免JavaScript端某些情况下在组件中移除了对应的属性。注意这个"默认"值只对基本类型生效，对于其他的类型而言，当对应的属性删除时，`null`会作为默认值提供给方法。
使用`@ReactPropGroup`来注解的设置方法和`@ReactProp`不同。请参见`@ReactPropGroup`注解类源代码中的文档来获取更多详情。
**注意**： 在ReactJS里，修改一个属性会引发一次对设置方法的调用。有一种修改情况是，移除掉之前设置的属性。在这种情况下设置方法也一样会被调用，并且“默认”值会被作为参数提供（对于基础类型来说可以通过`defaultBoolean`、`defaultFloat`等`@ReactProp`的属性提供，而对于复杂类型来说参数则会设置为`null`）

```
    @ReactProp(name = "sourceName")
    public void setSourceName(LottieAnimationView view, String name) {
        view.setAnimation(name);
    }

    @ReactProp(name = "progress", defaultFloat = 0f)
    public void setProgress(LottieAnimationView view, float progress) {
        view.setProgress(progress);
    }

    @ReactProp(name = "loop")
    public void setLoop(LottieAnimationView view, boolean loop) {
        view.loop(loop);
    }
```

### 导出一些命令
```
    private static final int COMMAND_PLAY = 1;
    private static final int COMMAND_RESET = 2;

    @Override
    public Map<String, Integer> getCommandsMap() {
        return MapBuilder.of(
                "play", COMMAND_PLAY,
                "reset", COMMAND_RESET
        );
    }

    @Override
    public void receiveCommand(final LottieAnimationView view, int commandId, @Nullable ReadableArray args) {
        switch (commandId) {
            case COMMAND_PLAY: {
                new Handler(Looper.getMainLooper()).post(new Runnable() {
                    @Override public void run() {
                        if (ViewCompat.isAttachedToWindow(view)) {
                            view.setProgress(0f);
                            view.playAnimation();
                        }
                    }
                });
            }
            break;
            case COMMAND_RESET: {
                new Handler(Looper.getMainLooper()).post(new Runnable() {
                    @Override public void run() {
                        if (ViewCompat.isAttachedToWindow(view)) {
                            view.cancelAnimation();
                            view.setProgress(0f);
                        }
                    }
                });
            }
            break;
        }
    }
```

### 注册ViewManager
实现`ReactPackage`的子类`LottiePackage`，这和[原生模块的注册方法](http://www.heqiangfly.com/2017/01/14/react-native-native-modules/)类似，唯一的区别是我们把它放到`createViewManagers`方法里：
```
public class LottiePackage implements ReactPackage {
    @Override
    public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
        return Collections.emptyList();
    }

    @Override
    public List<Class<? extends JavaScriptModule>> createJSModules() {
        return Collections.emptyList();
    }

    @Override
    public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
        return Arrays.<ViewManager>asList(
                new LottieViewManager()
        );
    }
}
```
### 添加组件
在`Application`的`getPackages()`方法中添加模块：
```
        @Override
        protected List<ReactPackage> getPackages() {
            return Arrays.<ReactPackage>asList(
                    new MainReactPackage(), new LottiePackage()
            );
        }
```
或者是在`Activity`的`onCreate`中：
```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mReactRootView = new ReactRootView(this);
        mReactInstanceManager = ReactInstanceManager.builder()
                .setApplication(getApplication())
                .setBundleAssetName("index.android.bundle")
                .setJSMainModuleName("index.android")
                .addPackage(new MainReactPackage())
                .addPackage(new LottiePackage())
                .setUseDeveloperSupport(BuildConfig.DEBUG)
                .setInitialLifecycleState(LifecycleState.RESUMED)
                .build();
        mReactRootView.startReactApplication(mReactInstanceManager, "HelloWorld", null);
        setContentView(mReactRootView);
    }
```

### 实现对应的JavaScript模块
创建Lottie.js
```
import React, { PropTypes } from 'react';
import { 
    requireNativeComponent, 
    View,
    UIManager,
    findNodeHandle,
    ReactNative,
    Platform } from 'react-native'; 

/*
var LottieView = {
  name: 'LottieView',
  defaultProps: {
   progress: 0,
   loop: true,
  },
  propTypes: {
      sourceName :PropTypes.string,
      progress: PropTypes.number,
      loop: PropTypes.bool,
    ...View.propTypes // 包含默认的View的属性
  },
};

module.exports = requireNativeComponent('LottieAnimationView', LottieView);
*/

const Lottie = requireNativeComponent('LottieAnimationView', LottieView);

class LottieView extends React.Component {
  constructor(props) {
    super(props);
  }

  play() {
    this.runCommand('play');
  }

  reset() {
    this.runCommand('reset');
  }

  runCommand(name, args = []) {
    return Platform.select({
      android: () => UIManager.dispatchViewManagerCommand(
        this.getHandle(),
        UIManager.LottieAnimationView.Commands[name],
        args
      ),
      ios: () => LottieViewManager[name](this.getHandle(), ...args),
    })();
  }
  getHandle() {
      return findNodeHandle(this.refs.lottieView);
  }

  render() {
    return <Lottie ref="lottieView" {...this.props} loop={true} />;
  }
}

module.exports = LottieView;
```
`UIManager.dispatchViewManagerCommand`把调用命令分发到Native端对应的组件类型的ViewManager，再通过ViewManager调用View组件实例的对应方法，这部分后面再介绍。

## 调用组件
在index.android.js里面调用：
```
import LottieView from './Lottie'
……
  onClicked(){
    this.refs.lottie.play();
  }
  render() {
    return (
      <TouchableWithoutFeedback onPress={() => this.onClicked()}>
        <View style={styles.lottieContainer} >
          <LottieView ref="lottie" style={styles.lottie} sourceName='LogoSmall.json' loop={true}>  
          </LottieView>
        </View>
      </TouchableWithoutFeedback>

    )
  }
```

