
---
title: Android 插件化框架Small解析 -- 类的动态加载源码分析
categories: Android 插件化
comments: true
tags: [Android 插件化]
description: 类的动态加载源码分析
date: 2017-5-20 10:00:00
---

在博客 [Android 插件化 -- 类的动态加载源码分析 ](http://www.heqiangfly.com/2017/05/18/android-plugins-dynamic-load-source-code-analysis/) 中我们从源码角度了解了类的动态加载，那么现在我们再来看一下 Small 框架中是怎么来实现类的动态加载的。

## ClassLoader Dex 扩展

在 `ApkBundleLauncher.loadBundle` 中：

```
    public void loadBundle(Bundle bundle) {
        ...

            // Load dex
            final LoadedApk fApk = apk;
            Bundle.postIO(new Runnable() {
                @Override
                public void run() {
                    try {
                        fApk.dexFile = DexFile.loadDex(fApk.path, fApk.optDexFile.getPath(), 0);
                    } catch (IOException e) {
                        throw new RuntimeException(e);
                    }
                }
            });

            // Extract native libraries with specify ABI
            String libDir = parser.getLibraryDirectory();
            if (libDir != null) {
                apk.libraryPath = new File(apk.packagePath, libDir);
            }
            sLoadedApks.put(packageName, apk);
        }

        ...
```

这里直接调用了 `DexFile.loadDex` 来加载 dex 文件并保存到 `LoadedApk` 对象的 `dexFile` 变量中。
把 native libraries 路径保存到 `LoadedApk.libraryPath` 中
然后在 `ApkBundleLauncher.postSetUp()` 方法中：

```
    public void postSetUp() {
        ...
        String[] dexPaths = new String[N];
        DexFile[] dexFiles = new DexFile[N];
        // 这里把所有的dex文件保存到一个数组中
        for (LoadedApk apk : apks) {
            dexPaths[i] = apk.path;
            dexFiles[i] = apk.dexFile;
            if (Small.getBundleUpgraded(apk.packageName)) {
                if (apk.optDexFile.exists()) apk.optDexFile.delete();
                Small.setBundleUpgraded(apk.packageName, false);
            }
            i++;
        }
        // 扩展当前ClassLoader的DexPathList，把前面解析到的dex添加到 DexPathList.dexElements 中去
        ReflectAccelerator.expandDexPathList(cl, dexPaths, dexFiles);

        // 把 native library 也保存到数组中
        List<File> libPathList = new ArrayList<File>();
        for (LoadedApk apk : apks) {
            if (apk.libraryPath != null) {
                libPathList.add(apk.libraryPath);
            }
        }
        // 扩展当前ClassLoader的DexPathList，把前面解析到的 native library 添加到 DexPathList.nativeLibraryDirectories 中
        if (libPathList.size() > 0) {
            ReflectAccelerator.expandNativeLibraryDirectories(cl, libPathList);
        }

        ...
    }
```

### 扩展 DexPathList

这里主要是调用反射扩展当前 ClassLoader 的 DexPathList 的 dexElements 变量。

```
    public static boolean expandDexPathList(ClassLoader cl, String[] dexPaths, DexFile[] dexFiles) {
        if (Build.VERSION.SDK_INT < 14) {
            return V9_13.expandDexPathList(cl, dexPaths, dexFiles);
        } else {
            return V14_.expandDexPathList(cl, dexPaths, dexFiles);
        }
    }
    
        public static boolean expandDexPathList(ClassLoader cl,
                                                String[] dexPaths, DexFile[] dexFiles) {
            try {
                int N = dexPaths.length;
                Object[] elements = new Object[N];
                // 这里相当于做了DexPathList.makeElements的工作，生成 Element 数组
                for (int i = 0; i < N; i++) {
                    String dexPath = dexPaths[i];
                    File pkg = new File(dexPath);
                    DexFile dexFile = dexFiles[i];
                    elements[i] = makeDexElement(pkg, dexFile);
                }
                // 把生成的  Element 数组添加到DexPathList.dexElements中
                fillDexPathList(cl, elements);
            } catch (Exception e) {
                e.printStackTrace();
                return false;
            }
            return true;
        }
        
        private static Object makeDexElement(File pkg, boolean isDirectory, DexFile dexFile) throws Exception {
            if (sDexElementClass == null) {
                sDexElementClass = Class.forName("dalvik.system.DexPathList$Element");
            }
            if (sDexElementConstructor == null) {
                sDexElementConstructor = sDexElementClass.getConstructors()[0];
            }
            Class<?>[] types = sDexElementConstructor.getParameterTypes();
            switch (types.length) {
                case 3:
                    if (types[1].equals(ZipFile.class)) {
                        // Element(File apk, ZipFile zip, DexFile dex)
                        ZipFile zip;
                        try {
                            zip = new ZipFile(pkg);
                        } catch (IOException e) {
                            throw e;
                        }
                        try {
                            return sDexElementConstructor.newInstance(pkg, zip, dexFile);
                        } catch (Exception e) {
                            zip.close();
                            throw e;
                        }
                    } else {
                        // Element(File apk, File zip, DexFile dex)
                        return sDexElementConstructor.newInstance(pkg, pkg, dexFile);
                    }
                case 4:
                default:
                    // Element(File apk, boolean isDir, File zip, DexFile dex)
                    if (isDirectory) {
                        return sDexElementConstructor.newInstance(pkg, true, null, null);
                    } else {
                        return sDexElementConstructor.newInstance(pkg, false, pkg, dexFile);
                    }
            }
        }
        
        private static void fillDexPathList(ClassLoader cl, Object[] elements)
                throws NoSuchFieldException, IllegalAccessException {
            if (sPathListField == null) {
                sPathListField = getDeclaredField(DexClassLoader.class.getSuperclass(), "pathList");
            }
            Object pathList = sPathListField.get(cl);
            if (sDexElementsField == null) {
                sDexElementsField = getDeclaredField(pathList.getClass(), "dexElements");
            }
            expandArray(pathList, sDexElementsField, elements, true);
        }
```

### 扩展 NativeLibraryDirectories

这里主要是调用反射扩展当前 ClassLoader 的 DexPathList 的 nativeLibraryDirectories 变量。

```
    public static void expandNativeLibraryDirectories(ClassLoader classLoader, List<File> libPath) {
        int v = Build.VERSION.SDK_INT;
        if (v < 14) {
            V9_13.expandNativeLibraryDirectories(classLoader, libPath);
        } else if (v < 23) {
            V14_22.expandNativeLibraryDirectories(classLoader, libPath);
        } else {
            V23_.expandNativeLibraryDirectories(classLoader, libPath);
        }
    }
    
        public static void expandNativeLibraryDirectories(ClassLoader classLoader,
                                                          List<File> libPaths) {
            if (sPathListField == null) return;

            Object pathList = getValue(sPathListField, classLoader);
            if (pathList == null) return;

            if (sDexPathList_nativeLibraryDirectories_field == null) {
                sDexPathList_nativeLibraryDirectories_field = getDeclaredField(
                        pathList.getClass(), "nativeLibraryDirectories");
                if (sDexPathList_nativeLibraryDirectories_field == null) return;
            }

            try {
                // List<File> nativeLibraryDirectories
                List<File> paths = getValue(sDexPathList_nativeLibraryDirectories_field, pathList);
                if (paths == null) return;
                paths.addAll(libPaths);

                // Element[] nativeLibraryPathElements
                if (sDexPathList_nativeLibraryPathElements_field == null) {
                    sDexPathList_nativeLibraryPathElements_field = getDeclaredField(
                            pathList.getClass(), "nativeLibraryPathElements");
                }
                if (sDexPathList_nativeLibraryPathElements_field == null) return;

                int N = libPaths.size();
                Object[] elements = new Object[N];
                for (int i = 0; i < N; i++) {
                    Object dexElement = makeDexElement(libPaths.get(i));
                    elements[i] = dexElement;
                }

                expandArray(pathList, sDexPathList_nativeLibraryPathElements_field, elements, false);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
```

## 类的动态加载

把 ClassLoader 扩展以后，就可以在宿主apk里面正常调用插件中的类了。插件类的入口 Activity 我们还是要通过反射来调用的，[启动插件Activity流程](http://www.heqiangfly.com/2017/05/12/open-source-android-plugins-small-principle/)在另外一篇博客中有介绍。

