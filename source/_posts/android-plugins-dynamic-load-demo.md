---
title: Android 插件化 -- 类的动态加载实践
categories: Android 插件化
comments: true
tags: [Android 插件化]
description: 通过 Demo 来实现类的动态加载
date: 2017-5-15 10:00:00
---

实现 Android 类的动态加载，我们一般通过两个类实现 `DexClassLoader` 和 `PathClassLoader`。
这两个类的区别在很多类似的文章里面也有介绍：

 - DexClassLoader：可以加载jar、apk、dex，支持从 SD 卡中加载。
 - PathClassLoader：只能加载已经安装到系统中的Apk文件，也就是/data/app目录下的apk文件。

下面我们来看一下这两个类的源码：

```
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), librarySearchPath, parent);
    }
}
```

```
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
```

可以看到，这两个类都是继承自 `BaseDexClassLoader`，不同的是构造函数接受的参数不同：
先来看一下 `DexClassLoader` 的参数：

 - dexPath：包含 class.dex 的 apk、jar 文件路径 ，多个用文件分隔符 `File.pathSeparator` 分隔。
 - optimizedDirectory：用来缓存优化的 dex 文件的路径，即从 apk 或 jar 文件中提取出来的 dex 文件。该路径不可以为空。
 - librarySearchPath：存储 C/C++ 库文件的路径。
 - parent：父类加载器，可以通过 `ClassLoader.getSystemClassLoader()` 或者 `Context.getClassLoader()` 获取。

再来看一下 `PathClassLoader` 的参数，少了 `optimizedDirectory` 参数，也证实了前面说的 `PathClassLoader` 只能加载安装过即已经进行过 odex 优化过的文件。
这里指出一下，dex 和 apk 是可以直接加载的，因为它们都是或者内部有 dex 文件，而原始的 jar 是不行的，必须转换成 dalvik 所能识别的字节码文件，转换工具可以使用android sdk中platform-tools目录下的 dx 工具：

```
dx --dex --output=dest.jar src.jar
```

## 实例

### 动态加载的类

我们在另外一个apk中实现下的类：

```
public class TestDynamicLoad {
    public String getString() {
        Log.e("Test","pid = "+android.os.Process.myPid() );
        return "This is a test case";
    }
    public int add(int a, int b) {
        Log.e("Test","pid = "+android.os.Process.myPid() );
        return a + b;
    }
    // 主要为了测试一个类中实例话另外一个类的情况
    public void testAnotherClass() {
        TestDynamicLoad2 dynamicLoad2 = new TestDynamicLoad2();
        dynamicLoad2.printStr();
    }
}
```

```
public class TestDynamicLoad2 {
    public void printStr(){
        Log.e("Test","This is TestDynamicLoad2" );
    }
}
```

然后编译生成apk后push到手机的SD卡上。
然后在另外一个apk中分别通过 `DexClassLoader` 和 `PathClassLoader` 来实现类的动态加载。

### DexClassLoader 加载

由于需要从SD卡中读取apk文件，需要加上SD卡读取权限：`<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>`。

```
        String className = "com.example.hq.myapplication.TestDynamicLoad";
        String dexPath = Environment.getExternalStorageDirectory()+"/app-debug.apk";
        File dexOutputDir = this.getDir("dex", 0);
        final String dexOutputPath = dexOutputDir.getAbsolutePath();
        Log.e("Test", "dexOutputPath = " + dexOutputPath);
        ClassLoader localClassLoader = getClassLoader();
        DexClassLoader dexClassLoader = new DexClassLoader(dexPath,
                dexOutputPath, null, localClassLoader);

        try {
            Class<?> localClass = dexClassLoader.loadClass(className);
            Constructor<?> localConstructor = localClass.getConstructor(new Class[]{});
            Object instance = localConstructor.newInstance(new Object[] {});
            Log.e("Test", "instance = " + instance);

            Method getString = localClass.getMethod("getString");
            getString.setAccessible(true);
            String str = (String) getString.invoke(instance);
            Log.e("Test","Dynamic load class getString = "+str);

            Method add = localClass.getMethod("add", int.class, int.class);
            add.setAccessible(true);
            int addResult = (int) add.invoke(instance, 6,8);
            Log.e("Test","Dynamic load class add = "+addResult);

            Method print = localClass.getMethod("testAnotherClass");
            print.setAccessible(true);
            print.invoke(instance);

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
```

代码执行后在手机的`/data/user/0/com.example.hq.testsomething/app_dex`目录生成 app-debug.dex 文件。

下面是执行的打印结果：

```
process id 30702
dexOutputPath = /data/user/0/com.example.hq.testsomething/app_dex
instance = com.example.hq.myapplication.TestDynamicLoad@5ff77f5
pid = 30702
Dynamic load class getString = This is a test case
pid = 30702
Dynamic load class add = 14
This is TestDynamicLoad2
```

### PathClassLoader 加载

```
        String className = "com.example.hq.myapplication.TestDynamicLoad";
        String dexPath = "/data/app/com.example.hq.myapplication-1/base.apk";
        ClassLoader localClassLoader = getClassLoader();
        PathClassLoader pathClassLoader = new PathClassLoader(dexPath, null, localClassLoader);
        Log.e("Test", "pathClassLoader = " + pathClassLoader);
        try {
            Class<?> localClass = pathClassLoader.loadClass(className);
            Constructor<?> localConstructor = localClass.getConstructor(new Class[]{});
            Object instance = localConstructor.newInstance(new Object[] {});
            Log.e("Test", "instance = " + instance);

            Method getString = localClass.getMethod("getString");
            getString.setAccessible(true);
            String str = (String) getString.invoke(instance);
            Log.e("Test","Dynamic load class getString = "+str);

            Method add = localClass.getMethod("add", int.class, int.class);
            add.setAccessible(true);
            int addResult = (int) add.invoke(instance, 6,8);
            Log.e("Test","Dynamic load class add = "+addResult);

            Method print = localClass.getMethod("testAnotherClass");
            print.setAccessible(true);
            print.invoke(instance);

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
```

结果：

```
process id 32385
pathClassLoader = dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.example.hq.myapplication-1/base.apk"],nativeLibraryDirectories=[/system/lib64, /vendor/lib64]]]
instance = com.example.hq.myapplication.TestDynamicLoad@5ff77f5
pid = 32385
Dynamic load class getString = This is a test case
pid = 32385
Dynamic load class add = 14
This is TestDynamicLoad2
```

可见，只要把类动态加载进来，`TestDynamicLoad` 中实例化 `TestDynamicLoad2` 以及调用它的方法和正常的调用没什么区别。
