
---
title: Android 插件化框架Small解析--动态注册组件原理及启动插件Activity流程
categories: Android开源项目
comments: true
tags: [Android开源项目, 插件化框架, Small]
description: 结合 Activity 的启动流程来分析如何启动插件 Activity，以此来介绍Small插件化一个核心技术--动态注册组件
date: 2017-5-12 10:00:00
---
## 动态注册组件原理

前面我们说过，替换 `Instrumentation` 对象和 `ActivityThreadHandlerCallback` 是插件化工作中的重头戏。
`Instrumentation`类看一下[官方文档对这个类的解释](https://developer.android.com/reference/android/app/Instrumentation.html?hl=zh-cn)，该类跟踪 `Application` 及 `Activity` 的整个生命周期，它的一些方法在 `Application` 及 `Activity` 所有生命周期函数的调用中，都会先调用这些方法，因此，得到了这个对象，我们就可以进入并跟踪 `Application` 和 `Activity` 的生命周期流程。
Small 想要做到动态注册 `Activity`，首先在宿主 Manifest 中注册一个命名特殊的占坑 `Activity` 来欺骗 `startActivityForResult` 以获得生命周期，再欺骗 `performLaunchActivity` 来获得插件 `Activity` 实例，又为了处理之间的信息传递，因此有了后面的 `ActivityThreadHandlerCallback`。

接下来我们就在 `ApkBundleLauncher.InstrumentationWrapper` 来看一下这些是如何实现的。
先来看一下 `execStartActivity` 方法：
`execStartActivity`方法有两个实现，一个是API Level 20以前的，一个是API Level 20以后的，仅仅是参数不同而已。
通过`Activity.startActivityForResult`源码可以看到，当我们调用 `startActivityForResult` 方法时，会调用 `mInstrumentation.execStartActivity` 来启动 `Activity`，此时便开启了 `Activity` 的整个生命周期流程。

```java
    public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {
        ……
        if (mParent == null) {
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
        ……
```

由于前面我们用 `ApkBundleLauncher.InstrumentationWrapper` 替换了 `mInstrumentation`，因此会调用到 `ApkBundleLauncher.InstrumentationWrapper` 中的 `execStartActivity()` 方法。该方法做的工作后面再详细介绍。主要是把需要启动的真实 `Activity` 替换为占坑 `Activity`。
熟悉 `Activity` 流程的同学都知道，真正启动 `Activity` 时，`ActivityManagerService` 调用 `ApplicationThread.scheduleLaunchActivity` 接口，通知相应的进程执行启动 `Activity` 的操作，`ApplicationThread` 把这个启动 `Activity` 的操作转发给 `ActivityThread`，`ActivityThread` 通过 `ClassLoader` 导入相应的 `Activity` 类，然后把它启动。
具体的在 `ActivityThread.ApplicationThread.scheduleLaunchActivity` 方法中会调用 `sendMessage(H.LAUNCH_ACTIVITY, r)`，该消息由 `ActivityThread` 中的消息处理对象 `mH` 来处理，由于我们把 `mH` 的 `mCallback` 替换为了`ActivityThreadHandlerCallback`，因此也会对 `LAUNCH_ACTIVITY` 消息进行拦截处理，处理完后再由`mH` 来处理正常的流程。
这里解释一下为什么要调用 `ensureInjectMessageHandler` 来反射替换 `ActivityThread` 的 `Handler` 对象 `mH` 的 `mCallback` 呢？
先看一下 `Handler` 源码中的 `dispatchMessage()` 方法。

```java
    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

这里会先执行 `mCallback` 的 `handleMessage()`，在返回false的情况下再执行自身的 `handleMessage()`。
这样就可以在 `ActivityThreadHandlerCallback` 中处理一些事情，然后在调用 `mH` 的方法。

## 启动插件Activity流程

这里以 `ApkBundleLauncher` 来作为 `Bundle` 的 `BundleLauncher` 为例进行说明。

```
├── Small.openUri()
     ├── Bundle.getLaunchableBundle(uri)
     │    └── Bundle.matchesRule()
     └── Bundle.launchFrom(context)
          └──ApkBundleLauncher.launchBundle()
               ├── ApkBundleLauncher.prelaunchBundle()
               │    └── Bundle.getActivityName()
               └── BundleLauncher.launchBundle()
                    └── Activity.startActivityForResult()
                         └── InstrumentationWrapper.execStartActivity()
                              ├── InstrumentationWrapper.wrapIntent()
                              └── InstrumentationWrapper.dequeueStubActivity
----------------------------------消息处理----------------------------------
├── ActivityThreadHandlerCallback.LAUNCH_ACTIVITY
     └── redirectActivity()
----------------------------------消息处理----------------------------------
├── InstrumentationWrapper.callActivityOnCreate()
```

### 根据URI匹配Bundle

匹配插件中注册的 `Activity`

```java
    protected static Bundle getLaunchableBundle(Uri uri) {
        if (sPreloadBundles != null) {
            for (Bundle bundle : sPreloadBundles) {
                if (bundle.matchesRule(uri)) {
                    if (bundle.mApplicableLauncher == null) {
                        break;
                    }

                    if (!bundle.enabled) return null; // Illegal bundle (invalid signature, etc.)
                    return bundle;
                }
            }
        }

        // 如果没有找到就用webview来显示这个uri
        if (uri.getScheme() != null) {
            Bundle bundle = new Bundle();
            try {
                bundle.url = new URL(uri.toString());
            } catch (MalformedURLException e) {
                e.printStackTrace();
            }
            bundle.prepareForLaunch();
            bundle.setQuery(uri.getEncodedQuery()); // Fix issue #6 from Spring-Xu.
            bundle.mApplicableLauncher = new WebBundleLauncher();
            bundle.mApplicableLauncher.prelaunchBundle(bundle);
            return bundle;
        }
        return null;
    }

    private Boolean matchesRule(Uri uri) {
        /* e.g.
         *  input
         *      - uri: http://base/abc.html
         *      - self.uri: http://base
         *      - self.rules: abc.html -> AbcController
         *  output
         *      - target => AbcController
         */
        // 将uri和Bundle.uriString进行匹配，如果不匹配直接返回false
        String uriString = uri.toString();
        if (this.uriString == null || !uriString.startsWith(this.uriString)) return false;

        // 截取类似 http://code.wequick.net/small-sample/detail?from=app.home 后面的字段?from=app.home进行解析
        String srcPath = uriString.substring(this.uriString.length());
        // 获取 ? 后面的字符
        String srcQuery = uri.getEncodedQuery();
        if (srcQuery != null) {
            srcPath = srcPath.substring(0, srcPath.length() - srcQuery.length() - 1);
        }

        ......

        this.path = dstPath;
        this.query = dstQuery;
        return true;
    }
```
`BundleLauncher.launchBundle()` 启动 `Activity`
```java
    public void launchBundle(Bundle bundle, Context context) {
        if(bundle.isLaunchable()) {
            if(context instanceof Activity) {
                Activity activity = (Activity)context;
                if(this.shouldFinishPreviousActivity(activity)) {
                    activity.finish();
                }
                activity.startActivityForResult(bundle.getIntent(), 10000);
            } else {
                context.startActivity(bundle.getIntent());
            }

        }
    }
```

### 启动真实的Activity

```java
    public void prelaunchBundle(Bundle bundle) {
        super.prelaunchBundle(bundle);
        Intent intent = new Intent();
        bundle.setIntent(intent);

        // 获取该插件的入口Activity
        String activityName = bundle.getActivityName();
        // 判断一下ActivityLauncher中的sActivityClasses是否包含该Activity，这里面只包含宿主app和Small框架里面的Activity
        if (!ActivityLauncher.containsActivity(activityName)) {
            // 一般启动插件Activity的情况下面是会走到这里的
            // sLoadedActivities 包含的是插件里面定义的Activity，在启动初始化时解析的
            if (sLoadedActivities == null) {
                throw new ActivityNotFoundException("Unable to find explicit activity class " +
                        "{ " + activityName + " }");
            }

            if (!sLoadedActivities.containsKey(activityName)) {
                if (activityName.endsWith("Activity")) {
                    throw new ActivityNotFoundException("Unable to find explicit activity class " +
                            "{ " + activityName + " }");
                }
                // 再次进行一些模糊匹配
                String tempActivityName = activityName + "Activity";
                if (!sLoadedActivities.containsKey(tempActivityName)) {
                    throw new ActivityNotFoundException("Unable to find explicit activity class " +
                            "{ " + activityName + "(Activity) }");
                }

                activityName = tempActivityName;
            }
        }
        // 设置ComponentName
        intent.setComponent(new ComponentName(Small.getContext(), activityName));

        // Intent extras - params
        String query = bundle.getQuery();
        if (query != null) {
            intent.putExtra(Small.KEY_QUERY, '?'+query);
        }
    }
```

### 启动流程拦截，启动占坑位Activity

```java
    protected static class InstrumentationWrapper extends Instrumentation
            implements InstrumentationInternal {

        ......

        public ActivityResult execStartActivity(
                Context who, IBinder contextThread, IBinder token, Activity target,
                Intent intent, int requestCode, android.os.Bundle options) {
            // 将intent 中真正的Activity替换为占坑位的Activity
            wrapIntent(intent);
            // 通过反射替换ActivityThread 的 Message Handler mH变量的 mCallback 为ActivityThreadHandlerCallback
            // 用于恢复Activity Info 到真实的Activity
            ensureInjectMessageHandler(sActivityThread);
            // 反射调用启动Activity
            return ReflectAccelerator.execStartActivity(mBase,
                    who, contextThread, token, target, intent, requestCode, options);
        }
        ......

        private void wrapIntent(Intent intent) {
            // 此处为插件中注册的真正的Activity
            ComponentName component = intent.getComponent();
            String realClazz;
            if (component == null) {
                // 如果component为空，交给宿主来处理这个intent
                component = intent.resolveActivity(Small.getContext().getPackageManager());
                if (component != null) {
                    // 系统或者宿主处理掉了，直接返回
                    return;
                }

                // 如果Action没有处理掉，看一下插件注册的Activity能否处理
                realClazz = resolveActivity(intent);
                if (realClazz == null) {
                    // 如果插件也不能处理，就直接返回，无能为力了……
                    return;
                }
            } else {
                realClazz = component.getClassName();
                if (realClazz.startsWith(STUB_ACTIVITY_PREFIX)) {
                    // 如果这个Activity已经是占坑位的Activity，进行解开回原来的Activity
                    realClazz = unwrapIntent(intent);
                }
            }

            if (sLoadedActivities == null) return;
            // 从sLoadedActivities中获得真正Activity的信息
            ActivityInfo ai = sLoadedActivities.get(realClazz);
            if (ai == null) return;

            // 把真实的Activity放到Category中并用'>'进行标识
            intent.addCategory(REDIRECT_FLAG + realClazz);
            // 获取一个占坑位的Activity
            String stubClazz = dequeueStubActivity(ai, realClazz);
            // 将真正需要启动的Activity替换为占坑位的Activity
            intent.setComponent(new ComponentName(Small.getContext(), stubClazz));
        }

        private String dequeueStubActivity(ActivityInfo ai, String realActivityClazz) {
            if (ai.launchMode == ActivityInfo.LAUNCH_MULTIPLE) {
                // 如果是普通的启动模式，用 A和A1 Activity，这两个Activity是可以重复使用的
                // 看是否有windowIsTranslucent属性
                Resources.Theme theme = Small.getContext().getResources().newTheme();
                theme.applyStyle(ai.getThemeResource(), true);
                TypedArray sa = theme.obtainStyledAttributes(
                        new int[] { android.R.attr.windowIsTranslucent });
                boolean translucent = sa.getBoolean(0, false);
                sa.recycle();
                // 如果有就用A1,没有就用A
                return translucent ? STUB_ACTIVITY_TRANSLUCENT : STUB_ACTIVITY_PREFIX;
            }

            // 根据启动模式匹配合适的占坑位Activity
            int availableId = -1;
            int stubId = -1;
            int countForMode = STUB_ACTIVITIES_COUNT;
            int countForAll = countForMode * 3; // 3=[singleTop, singleTask, singleInstance]
            if (mStubQueue == null) {
                mStubQueue = new String[countForAll];
            }
            int offset = (ai.launchMode - 1) * countForMode;
            for (int i = 0; i < countForMode; i++) {
                String usedActivityClazz = mStubQueue[i + offset];
                if (usedActivityClazz == null) {
                    if (availableId == -1) availableId = i;
                } else if (usedActivityClazz.equals(realActivityClazz)) {
                    stubId = i;
                }
            }
            if (stubId != -1) {
                availableId = stubId;
            } else if (availableId != -1) {
                mStubQueue[availableId + offset] = realActivityClazz;
            } else {
                Log.e(TAG, "Launch mode " + ai.launchMode + " is full");
            }
            return STUB_ACTIVITY_PREFIX + ai.launchMode + availableId;
        }
    }
````

### 消息拦截

由于前面介绍的反射替换 `ActivityThread` 的 `mH` 对象的 `mCallback`，这里拦截了四种消息：

```java
    private static class ActivityThreadHandlerCallback implements Handler.Callback {

        private static final int LAUNCH_ACTIVITY = 100;
        private static final int CREATE_SERVICE = 114;
        private static final int CONFIGURATION_CHANGED = 118;
        private static final int ACTIVITY_CONFIGURATION_CHANGED = 125;

        private Configuration mApplicationConfig;

        @Override
        public boolean handleMessage(Message msg) {
            switch (msg.what) {
                case LAUNCH_ACTIVITY:
                    redirectActivity(msg);
                    break;

                case CREATE_SERVICE:
                    ensureServiceClassesLoadable(msg);
                    break;

                case CONFIGURATION_CHANGED:
                    recordConfigChanges(msg);
                    break;

                case ACTIVITY_CONFIGURATION_CHANGED:
                    return relaunchActivityIfNeeded(msg);

                default:
                    break;
            }
            // 这里返回false，不影响 mH 对消息的处理
            return false;
        }
    }
```

#### LAUNCH_ACTIVITY

对 `LAUNCH_ACTIVITY` 的处理主要是将启动的占坑位的 `Activity` 重新替换为真实的 `Activity` 

```java

        private void redirectActivity(Message msg) {
            Object/*ActivityClientRecord*/ r = msg.obj;
            // 通过反射获取intent
            Intent intent = ReflectAccelerator.getIntent(r);
            // 解开intent，获取真实的Activity名称
            String targetClass = unwrapIntent(intent);
            boolean hasSetUp = Small.hasSetUp();
            if (targetClass == null) {
                // 在宿主中注册的Activity
                if (hasSetUp) return; // nothing to do

                if (intent.hasCategory(Intent.CATEGORY_LAUNCHER)) {
                    // 带CATEGORY_LAUNCHER属性的Activity
                    return;
                }

                // Launching an activity in remote process. Set up Small for it.
                Small.setUpOnDemand();
                return;
            }

            if (!hasSetUp) {
                // Restarting an activity after application recreated,
                // maybe upgrading or somehow the application was killed in background.
                Small.setUp();
            }

            // 替换为真正的 activityInfo
            ActivityInfo targetInfo = sLoadedActivities.get(targetClass);
            ReflectAccelerator.setActivityInfo(r, targetInfo);
        }
    }
```

### 拦截OnCreate方法

对 `Activity` 的 `OnCreate` 调用的拦截由 `InstrumentationWrapper.callActivityOnCreate()` 来进行处理。

```java
        @Override
        public void callActivityOnCreate(Activity activity, android.os.Bundle icicle) {
            do {
                if (sLoadedActivities == null) break;
                ActivityInfo ai = sLoadedActivities.get(activity.getClass().getName());
                if (ai == null) break;
                //用来设置Activity的一些转屏和键盘状态
                applyActivityInfo(activity, ai);
                ReflectAccelerator.ensureCacheResources();
            } while (false);

            // 重新设置 mInstrumentation ，防止被改变
            if (sBundleInstrumentation != null) {
                try {
                    Field f = Activity.class.getDeclaredField("mInstrumentation");
                    f.setAccessible(true);
                    Object instrumentation = f.get(activity);
                    if (instrumentation != sBundleInstrumentation) {
                        f.set(activity, sBundleInstrumentation);
                    }
                } catch (NoSuchFieldException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
            // 再调用 mInstrumentation 的callActivityOnCreate方法
            sHostInstrumentation.callActivityOnCreate(activity, icicle);
        }

```

另外还有对 `OnStop`，`OnDestroy`等其他生命周期方法的拦截，这里就不一一介绍了。

