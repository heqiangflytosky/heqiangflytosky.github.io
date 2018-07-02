---
title: Android 插件化 -- 资源的动态加载实践
categories: Android 插件化
comments: true
tags: [Android 插件化]
description: 类的动态加载源码分析
date: 2017-5-22 10:00:00
---

## 预备知识

### 资源类

[访问资源官方文档](https://developer.android.com/guide/topics/resources/accessing-resources)
我们知道，Android 程序中的每个资源在编译的时候，aapt 会生成 R 类，其中包含 res/ 目录中所有资源的资源 ID。 每个资源类型都有对应的 R 子类（例如，R.drawable 对应于所有可绘制对象资源），而该类型的每个资源都有对应的静态整型数（例如，R.drawable.icon）。这个整型数就是可用来检索资源的资源 ID。 
资源ID 是一个32bit的数字，格式是PPTTNNNN ， PP代表资源所属的包(package) ,TT代表资源的类型(type)，NNNN代表这个类型下面的资源的名称。 对于系统资源来说取值是0x00-0x02，对于应用程序的资源来说，PP的取值是0×7f。
TT 和NNNN 的取值是由AAPT工具随意指定的–基本上每一种新的资源类型的数字都是从上一个数字累加的（从1开始）
```
public final class R {
  public static final class anim {
    public static final int abc_fade_in=0x7f010000;
    ......
  }
  public static final class animator {
    public static final int design_appbar_state_list_animator=0x7f020000;
  }
  public static final class attr {
    public static final int actionBarDivider=0x7f030000;
    ......
  }
  public static final class bool {
    public static final int abc_action_bar_embed_tabs=0x7f040000;
    ......
  }
  public static final class color {
    public static final int abc_background_cache_hint_selector_material_dark=0x7f050000;
    ......
  }
  public static final class dimen {
    public static final int abc_action_bar_content_inset_material=0x7f060000;
    ……
    public static final int tooltip_y_offset_touch=0x7f06009b;
  }
  public static final class drawable {
    public static final int abc_ab_share_pack_mtrl_alpha=0x7f070007;
    ......
  }
  public static final class id {
    public static final int ALT=0x7f080000;
    ......
  }
  public static final class integer {
    public static final int abc_config_activityDefaultDur=0x7f090000;
    ......
  }
  public static final class layout {
    public static final int abc_action_bar_title_item=0x7f0a0000;
    ......
  }
  public static final class mipmap {
    public static final int ic_launcher=0x7f0b0000;
    ......
  }
  public static final class string {
    public static final int abc_action_bar_home_description=0x7f0c0000;
    .....
  }
  public static final class style {
    public static final int AlertDialog_AppCompat=0x7f0d0000;
    .....
  }
  public static final class styleable {
    public static final int[] ActionBar={
        0x7f030031, 0x7f030032, 0x7f030033, 0x7f030061, 
        0x7f030062, 0x7f030063, 0x7f030064, 0x7f030065, 
        0x7f030066, 0x7f03006d, 0x7f030071, 0x7f030072, 
        0x7f03007d, 0x7f03009d, 0x7f03009e, 0x7f0300a2, 
        0x7f0300a3, 0x7f0300a4, 0x7f0300a9, 0x7f0300af, 
        0x7f0300f5, 0x7f0300fe, 0x7f03010e, 0x7f030112, 
        0x7f030113, 0x7f030137, 0x7f03013a, 0x7f030166, 
        0x7f030170
      };

    public static final int[] ActionBarLayout={
        0x010100b3
      };
    ……
    public static final int View_theme=4;
  }
}

```

### createPackageContext 方法

再来了解一下 `Context` 中的 `createPackageContext` 方法，通过这个方法可以创建另外一个包的 `Context` 上下文，这样就可以访问该软件包的资源，甚至可以执行其它软件包的代码。

### 动态获取资源的两种方法

 - 通过 `getResources().getIdentifier(String name,String defType,String defPackage)` 获取。
 - 通过反射调用 R.java 类来获取。

## 动态加载已经安装的APK

### 代码实现

在测试的apk中添加字符串资源：

```
<resources>
    ......
    <string name="str_test_dynamic_load_res">测试动态加载资源</string>
</resources>
```

并将该apk安装到手机上面。
在另外一个apk中实现下面的代码：

```
    public void clickTestDynamicLoadRes(View v) {
        try {
            // 待加载apk的包名
            String packageName = "com.example.heqiang.myapplication";
            // 需要加载资源的类型
            String type = "string";
            // 需要加载字符串资源的名称
            String strName = "str_test_dynamic_load_res";

            Context context = this.createPackageContext(packageName,
                    Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY);
            Resources  res = context.getResources();
            ClassLoader classLoader = context.getClassLoader();

            // 获取资源ID
            String rClassName = packageName+".R$"+type;
            Class rClass = classLoader.loadClass(rClassName);
            int resID = (int) rClass.getField(strName).get(null);

            String str = res.getString(resID);
            Log.e("Test","str = "+str);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }  catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }
```

结果：

```
Test: str = 测试动态加载资源
```

成功实现字符串资源的动态加载，其它类型的资源实现方法一样。
上面例子中获取资源ID是通过反射R类来实现的，其实还有另外方法可以实现，通过 `Resources` 的 `getIdentifier` 方法类实现。

```
    public void clickTestDynamicLoadRes(View v) {
        try {
            // 待加载apk的包名
            String packageName = "com.example.heqiang.myapplication";
            // 需要加载资源的类型
            String type = "string";
            // 需要加载字符串资源的名称
            String strName = "str_test_dynamic_load_res";

            Context context = this.createPackageContext(packageName,
                    Context.CONTEXT_INCLUDE_CODE | Context.CONTEXT_IGNORE_SECURITY);
            Resources  res = context.getResources();

            int resID = res.getIdentifier(strName, type, packageName);

            String str = res.getString(resID);
            Log.e("Test","str = "+str);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
    }
```


## 动态加载未安装的APK

```
    public void clickTestDynamicLoadRes(View v) {
        try {
            // 待加载apk的路径
            String resPath = Environment.getExternalStorageDirectory()+"/app-debug.apk";
            // 需要加载资源的类型
            String type = "string";
            // 需要加载字符串资源的名称
            String strName = "str_test_dynamic_load_res";
            // 获取包信息。
            PackageInfo mInfo = getPackageManager().getPackageArchiveInfo(resPath, PackageManager.GET_ACTIVITIES);
            // 待加载apk的包名
            String packageName = mInfo.packageName;

            AssetManager assetManager = AssetManager.class.newInstance();
            Method addAssetPath = AssetManager.class.getDeclaredMethod("addAssetPath", String.class);
            addAssetPath.invoke(assetManager, resPath);
            Resources res = new Resources(assetManager, super.getResources().getDisplayMetrics(), super.getResources().getConfiguration());

            // 获取到资源的ID。
            int resID = res.getIdentifier(strName, type, packageName);

            String str = res.getString(resID);
            Log.e("Test","str = "+str);

        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
   
```
