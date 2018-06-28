---
title: Android 插件化 -- 类的动态加载源码分析
categories: Android 插件化
comments: true
tags: [Android 插件化]
description: 类的动态加载源码分析
date: 2017-5-18 10:00:00
---

## 流程图

```
├── BaseDexClassLoader
    └── DexPathList
        ├── DexPathList.makeDexElements
            └── DexPathList.makeElements
                └── DexPathList.loadDexFile
                    ├── DexPathList.optimizedPathFor
                    └── DexFile.loadDex
                        ├── new DexFile
                            └── DexFile.openDexFile
                                └── DexFile.openDexFileNative
        └── DexPathList.makePathElements
└── ClassLoader.loadClass
    └── ClassLoader.loadClass
        ├── ClassLoader.findLoadedClass
        └── BaseDexClassLoader.findClass
            └── DexPathList.findClass
                └── DexFile.loadClassBinaryName
                    └── DexFile.defineClass
```



## 构造方法

先来看一下 `BaseDexClassLoader` 的构造方法：

```
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, librarySearchPath, optimizedDirectory);
    }
```

这里除了调用父类 `ClassLoader` 的构造方法之外，就是创建了 `DexPathList` 对象。

```
    private ClassLoader(Void unused, ClassLoader parent) {
        this.parent = parent;
    }

    protected ClassLoader(ClassLoader parent) {
        this(checkCreateClassLoader(), parent);
    }
```

`ClassLoader` 的构造方法很简单，就是把参数中的父 `ClassLoader` 赋值给成员变量。

### DexPathList

下面来看一下 `DexPathList` 构造方法：

```
    public DexPathList(ClassLoader definingContext, String dexPath,
            String librarySearchPath, File optimizedDirectory) {
        // 这里省略一些参数校验的代码
        ......

        this.definingContext = definingContext;

        ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
        // 获取dex文件
        this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                           suppressedExceptions, definingContext);
        // Native libraries 可能在系统目录，也可能存在应用中，采用下面的搜索顺序：
        //   1. This class loader's library path for application libraries (librarySearchPath):
        //   1.1. Native library directories
        //   1.2. Path to libraries in apk-files
        //   2. The VM's library path from the system property for system libraries
        //      also known as java.library.path
        this.nativeLibraryDirectories = splitPaths(librarySearchPath, false);
        this.systemNativeLibraryDirectories =
                splitPaths(System.getProperty("java.library.path"), true);
        List<File> allNativeLibraryDirectories = new ArrayList<>(nativeLibraryDirectories);
        allNativeLibraryDirectories.addAll(systemNativeLibraryDirectories);
        // 获取Native libraries
        this.nativeLibraryPathElements = makePathElements(allNativeLibraryDirectories,
                                                          suppressedExceptions,
                                                          definingContext);

        if (suppressedExceptions.size() > 0) {
            this.dexElementsSuppressedExceptions =
                suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            dexElementsSuppressedExceptions = null;
        }
    }
```

下面来看一下 `DexPathList.makeElements` 代码：

```
    private static Element[] makeElements(List<File> files, File optimizedDirectory,
                                          List<IOException> suppressedExceptions,
                                          boolean ignoreDexFiles,
                                          ClassLoader loader) {
        Element[] elements = new Element[files.size()];
        int elementsPos = 0;
        // 打开所有的文件并加载里面所有的dex文件
        for (File file : files) {
            File zip = null;
            File dir = new File("");
            DexFile dex = null;
            String path = file.getPath();
            String name = file.getName();
            // zipSeparator = "!/"
            if (path.contains(zipSeparator)) {
                String split[] = path.split(zipSeparator, 2);
                zip = new File(split[0]);
                dir = new File(split[1]);
            } else if (file.isDirectory()) {
                // 支持从指定文件路径中加载dex和动态库
                // 如果是一个文件路径，那么创建一个 Element 对象
                elements[elementsPos++] = new Element(file, true, null, null);
            } else if (file.isFile()) {
                // DEX_SUFFIX = ".dex"
                // 这里调用 loadDexFile 加载dex文件或者zip/jar/apk文件
                if (!ignoreDexFiles && name.endsWith(DEX_SUFFIX)) {
                    // 原始的dex文件 (不是包含在 zip/jar 里).
                    try {
                        dex = loadDexFile(file, optimizedDirectory, loader, elements);
                    } catch (IOException suppressed) {
                        System.logE("Unable to load dex file: " + file, suppressed);
                        suppressedExceptions.add(suppressed);
                    }
                } else {
                    zip = file;

                    if (!ignoreDexFiles) {
                        try {
                            dex = loadDexFile(file, optimizedDirectory, loader, elements);
                        } catch (IOException suppressed) {
                            suppressedExceptions.add(suppressed);
                        }
                    }
                }
            } else {
                System.logW("ClassLoader referenced unknown path: " + file);
            }

            if ((zip != null) || (dex != null)) {
                // 将文件加入到数组中
                elements[elementsPos++] = new Element(dir, false, zip, dex);
            }
        }
        if (elementsPos != elements.length) {
            elements = Arrays.copyOf(elements, elementsPos);
        }
        return elements;
    }
```

`ClassLoader` 构造方法部分我们分析完了，其实整个过程就是调用了 `DexPathList` 的构造函数，把 dex 文件或者 Native libraries 加载到 `Element` 数组的过程。
体现在 `DexPathList` 中的 `dexElements` 和 `nativeLibraryPathElements` 成员变量，供 `DexPathList.findClass` 和 `DexPathList.findLibrary` 方法查询。

上面的方法中调用了 `DexPathList.loadDexFile` 方法：

```
    private static DexFile loadDexFile(File file, File optimizedDirectory, ClassLoader loader,
                                       Element[] elements)
            throws IOException {
        if (optimizedDirectory == null) {
            return new DexFile(file, loader, elements);
        } else {
            String optimizedPath = optimizedPathFor(file, optimizedDirectory);
            return DexFile.loadDex(file.getPath(), optimizedPath, 0, loader, elements);
        }
    }
```

这里可以看到，针对是否指定 `optimizedDirectory` 做了不同的处理。这里其实是分别调用了不同的 `DexFile` 构造方法。

### DexFile

```
    private static Object openDexFile(String sourceName, String outputName, int flags,
            ClassLoader loader, DexPathList.Element[] elements) throws IOException {
        // Use absolute paths to enable the use of relative paths when testing on host.
        return openDexFileNative(new File(sourceName).getAbsolutePath(),
                                 (outputName == null)
                                     ? null
                                     : new File(outputName).getAbsolutePath(),
                                 flags,
                                 loader,
                                 elements);
    }
```

`DexFile` 主要就是调用了 native 方法 `openDexFileNative` 来打开 dex 文件。

## 加载类

下面来看一下类的加载过程：

```
    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // 首先，看需要获取的类是否已经加载进来
            Class c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    // 如果没有找到，如果制定了父加载器，
                    // 就调用父加载器的loadClass方法去加载
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                    // 否则调用 findBootstrapClassOrNull 去加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                }

                if (c == null) {
                    // 如果还是没有加载，就调用 findClass 方法加载
                    // 在ClassLoader中findClass是个空实现，需要子类来实现
                    long t1 = System.nanoTime();
                    c = findClass(name);
                }
            }
            return c;
    }
```

`findClass` 方法其实调用的是 `BaseDexClassLoader` 的 `findClass` 方法：

```
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
```

其实调用的是 `DexPathList.findClass` 方法：

```
    public Class findClass(String name, List<Throwable> suppressed) {
        // 其实就是看前面加载的 dexElements 数组中是否存在这个类
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;
            // 如果找到了，就调用 DexFile.loadClassBinaryName 来加载类
            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
```

类的加载重点在 `DexFile.defineClass` 方法，这里面会调用 native 方法 `defineClassNative`。这里面的代码暂时不做分析。

## 总结

从上面分析我们可以看出，在构造 `DexClassLoader` 和 `PathClassLoader` 时，会加载 jar/apk/zip/dex 中的 dex 文件以及 Native libraries 并保存到 `Element` 数组中。后面动态加载类时都是从保存到数组中的 dex 文件中搜索，并调用 `DexFile` 的 Native 方法进行加载的过程。Native 方法这里暂不分析，有兴趣的大家可以自行研究。
