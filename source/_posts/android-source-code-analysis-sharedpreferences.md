---
title: Android SharedPreferences 源码分析以及跨进程读写问题
categories: Android
comments: true
tags: [Android]
description: SharedPreferences 源码分析以及跨进程读写问题
date: 2016-9-12 10:00:00
---

## 概述

Android SharedPreferences 提供了下面的模式来支持跨进程读写数据问题。

```
    @Deprecated
    public static final int MODE_MULTI_PROCESS = 0x0004;
```

这种模式已经被官方标记为 `Deprecated`，关于废弃的原因，官方有解释：

```
     * @deprecated MODE_MULTI_PROCESS does not work reliably in
     * some versions of Android, and furthermore does not provide any
     * mechanism for reconciling concurrent modifications across
     * processes.  Applications should not attempt to use it.  Instead,
     * they should use an explicit cross-process data management
     * approach such as {@link android.content.ContentProvider ContentProvider}.
```

Google认为多个进程读同一个文件都是不安全的，不建议这么做。Android 不保证该模式总是能正确的工作，建议使用 ContentProvider 替代多进程之间文件的共享。
如果在某些条件下必须使用，则要注意下面介绍的几个坑。

## 测试案例

在测试的工程中创建一个 `Activity`，指定在另外一个进程和 Task 中启动：

```
            android:process=":second"
            android:launchMode="singleTask"
            android:taskAffinity="com.example.heqiang.testsomething.SecondActivity"
```

然后在 `MainActivity` 中启动 `SecondActivity` 把当前时间写入到 `SharedPreferences`：

```
    public void startSecondActivity(View v) {
        SharedPreferences preferences = getSharedPreferences("TEST_MULTI_PROCESS",Context.MODE_MULTI_PROCESS);
        String time = String.valueOf(System.currentTimeMillis());
        preferences.edit().putString("time", time).apply();
        Toast.makeText(this, time, Toast.LENGTH_SHORT).show();

        Intent intent = new Intent(this, SecondActivity.class);
        startActivity(intent);
    }
```

在 `SecondActivity` 中添加一个按钮，点击可以跨进程获取 SharedPreferences 中 time 的值。
先看第一种实现方案：

```
public class SecondActivity extends Activity {
    private SharedPreferences mSharedPreferences;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        mSharedPreferences = getSharedPreferences("TEST_MULTI_PROCESS", Context.MODE_MULTI_PROCESS);
        Toast.makeText(this,"onCreate",Toast.LENGTH_SHORT).show();
    }


    public void getData(View view) {
        String time = mSharedPreferences.getString("time","null");
        Toast.makeText(this,time,Toast.LENGTH_SHORT).show();
    }
}
```

这种情况下 `mSharedPreferences` 实例只在 `onCreate` 中被初始化一次。
测试方法，启动 `SecondActivity` 后获取一次数据，然后将 `SecondActivity` 切回到后台，再次写入数据然后打开 `SecondActivity` 再次读取数据，你会发现，每次读取到的数据都是第一次的数据，虽然此时已经写入了新的数据，造成了写入和读取数据不同步的问题。
下面来测试一下下面的情况：

```
public class SecondActivity extends Activity {
    private SharedPreferences mSharedPreferences;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        Toast.makeText(this,"onCreate",Toast.LENGTH_SHORT).show();
    }


    public void getData(View view) {
        mSharedPreferences = getSharedPreferences("TEST_MULTI_PROCESS", Context.MODE_MULTI_PROCESS);
        String time = mSharedPreferences.getString("time","null");
        Toast.makeText(this,time,Toast.LENGTH_SHORT).show();
    }
}
```

这种情况下 `mSharedPreferences` 的初始化放在了每次获取数据的时候。这样读取数据是正常的。
为什么会是这样的？我们从源码里面找答案。

## 源码分析

### 获取 SharedPreferences 文件

```
    private File getPreferencesDir() {
        synchronized (mSync) {
            if (mPreferencesDir == null) {
                mPreferencesDir = new File(getDataDir(), "shared_prefs");
            }
            return ensurePrivateDirExists(mPreferencesDir);
        }
    }
```

在 `ContextImpl.getPreferencesDir ` 方法中，会固定从 /data/data/<包名>/shared_prefs目录下获取对应名称的xml文件，如果想改变目录路径，则需要通过反射，在构造 `SharedPreferencesImpl` 时传入File参数来实现。

```
                    Class spiClass = Class.forName("android.app.SharedPreferencesImpl");
                    Constructor constructor = spiClass.getDeclaredConstructor(File.class, int.class);
                    constructor.setAccessible(true);
                    sp = (SharedPreferences) constructor.newInstance(file, mode);
```

### 获取 SharedPreferences 实例

```
    @Override
    public SharedPreferences getSharedPreferences(File file, int mode) {
        checkMode(mode);
        SharedPreferencesImpl sp;
        synchronized (ContextImpl.class) {
            final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
            sp = cache.get(file);
            if (sp == null) {
                sp = new SharedPreferencesImpl(file, mode);
                cache.put(file, sp);
                return sp;
            }
        }
        if ((mode & Context.MODE_MULTI_PROCESS) != 0 ||
            getApplicationInfo().targetSdkVersion < android.os.Build.VERSION_CODES.HONEYCOMB) {
            // If somebody else (some other process) changed the prefs
            // file behind our back, we reload it.  This has been the
            // historical (if undocumented) behavior.
            sp.startReloadIfChangedUnexpectedly();
        }
        return sp;
    }
    
    private ArrayMap<File, SharedPreferencesImpl> getSharedPreferencesCacheLocked() {
        if (sSharedPrefsCache == null) {
            sSharedPrefsCache = new ArrayMap<>();
        }

        final String packageName = getPackageName();
        ArrayMap<File, SharedPreferencesImpl> packagePrefs = sSharedPrefsCache.get(packageName);
        if (packagePrefs == null) {
            packagePrefs = new ArrayMap<>();
            sSharedPrefsCache.put(packageName, packagePrefs);
        }

        return packagePrefs;
    }
```

通过 `ContextImpl.getSharedPreferences` 方法可以看到，获取的 `SharedPreferences` 是 `SharedPreferencesImpl` 实例。这个实例保存在静态变量 `SharedPreferencesImpl` 中，因此，无论该包名的应用中有多少个 `ContextImpl`，他们共同使用同一个 `SharedPreferencesImpl` 实例。
在 `MODE_MULTI_PROCESS` 模式下，会调用 `startReloadIfChangedUnexpectedly()` 方法，区别也就在这里。
我们先来看一下 `SharedPreferencesImpl` 类。
该类就是一个简单的二级缓存，在启动时会将文件里的数据全部都加载到内存里。
先来看一下构造方法：

```
    SharedPreferencesImpl(File file, int mode) {
        mFile = file;
        mBackupFile = makeBackupFile(file);
        mMode = mode;
        mLoaded = false;
        mMap = null;
        startLoadFromDisk();
    }
    private void startLoadFromDisk() {
        synchronized (this) {
            mLoaded = false;
        }
        new Thread("SharedPreferencesImpl-load") {
            public void run() {
                loadFromDisk();
            }
        }.start();
    }

    private void loadFromDisk() {
        synchronized (SharedPreferencesImpl.this) {
            if (mLoaded) {
                return;
            }
            if (mBackupFile.exists()) {
                mFile.delete();
                mBackupFile.renameTo(mFile);
            }
        }

        ......

        Map map = null;
        StructStat stat = null;
        try {
            stat = Os.stat(mFile.getPath());
            if (mFile.canRead()) {
                BufferedInputStream str = null;
                try {
                    str = new BufferedInputStream(
                            new FileInputStream(mFile), 16*1024);
                    map = XmlUtils.readMapXml(str);
                } catch (XmlPullParserException | IOException e) {
                    Log.w(TAG, "getSharedPreferences", e);
                } finally {
                    IoUtils.closeQuietly(str);
                }
            }
        } catch (ErrnoException e) {
            /* ignore */
        }

        synchronized (SharedPreferencesImpl.this) {
            mLoaded = true;
            if (map != null) {
                mMap = map;
                mStatTimestamp = stat.st_mtime;
                mStatSize = stat.st_size;
            } else {
                mMap = new HashMap<>();
            }
            notifyAll();
        }
    }
```

在构造 `SharedPreferencesImpl` 实例时，会从xml文件中通过 `loadFromDisk()` 把所有数据读取到  Map 中。

### 数据读取

以 `getString` 为例子介绍：
先来看数据的读取：

```
    @Nullable
    public String getString(String key, @Nullable String defValue) {
        synchronized (this) {
            awaitLoadedLocked();
            String v = (String)mMap.get(key);
            return v != null ? v : defValue;
        }
    }
```

可以看到，数据的读取直接从内存的 Map 中获取，没有涉及到 xml 文件的读取。
这也就不难理解，跨进程读取和写入的时候，为什么会造成数据的不同步了，如果写入是在另外一个进程，写入后，如果读取进程不重新从加载xml文件到内存，那么读取进程的Map是不会更新的，就读取不到另外进程新写入的数据了。
get 方法使用了对象的同步锁，说明这个方法是线程安全的。

### 数据写入

再来看一下数据写入的情况，以 `putString` 为例子介绍：：

```
    public Editor edit() {
        synchronized (this) {
            awaitLoadedLocked();
        }

        return new EditorImpl();
    }
    
    public final class EditorImpl implements Editor {
        // 存储提交之前需要修改的数据
        private final Map<String, Object> mModified = Maps.newHashMap();
        private boolean mClear = false;

        public Editor putString(String key, @Nullable String value) {
            //也使用了对象的同步锁，说明这个方法是线程安全的。
            synchronized (this) {
                // 暂时存储到 mModified 中
                mModified.put(key, value);
                return this;
            }
        }
        
        ...
        
        public boolean commit() {
            // 写入到内存Map中
            MemoryCommitResult mcr = commitToMemory();
            // 写入到本地xml文件
            SharedPreferencesImpl.this.enqueueDiskWrite(
                mcr, null /* sync write on this thread okay */);
            try {
                // 等待写入到本地xml文件结束
                mcr.writtenToDiskLatch.await();
            } catch (InterruptedException e) {
                return false;
            }
            // 通知通过 registerOnSharedPreferenceChangeListener 注册的监听器
            notifyListeners(mcr);
            return mcr.writeToDiskResult;
        }
        
        public void apply() {
            // 写入到内存Map中
            final MemoryCommitResult mcr = commitToMemory();
            // awaitCommit 和 postWriteRunnable 用来做一些线程间的同步操作
            final Runnable awaitCommit = new Runnable() {
                    public void run() {
                        try {
                            // 等待写入到本地xml文件结束
                            mcr.writtenToDiskLatch.await();
                        } catch (InterruptedException ignored) {
                        }
                    }
                };

            QueuedWork.add(awaitCommit);

            Runnable postWriteRunnable = new Runnable() {
                    public void run() {
                        awaitCommit.run();
                        QueuedWork.remove(awaitCommit);
                    }
                };

            // 写入到本地xml文件
            SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);
            // 通知通过 registerOnSharedPreferenceChangeListener 注册的监听器
            notifyListeners(mcr);
        }
```

写入内存：

```
        private MemoryCommitResult commitToMemory() {
            MemoryCommitResult mcr = new MemoryCommitResult();
            synchronized (SharedPreferencesImpl.this) {
                // mDiskWritesInFlight > 0说明内存中有未写入磁盘的数据
                if (mDiskWritesInFlight > 0) {
                    // mMap 做一个 copy，保证写入磁盘过程中用的是不同的 Map 对象。
                    //因为写入磁盘是个异步过程，这样在后一次调用commitToMemory时，
                    //在更新mMap中的值时不会影响前一次的mapToWriteToDisk的写入磁盘。
                    mMap = new HashMap<String, Object>(mMap);
                }
                mcr.mapToWriteToDisk = mMap;
                // mDiskWritesInFlight 加1，表示多了一个写操作。在写入磁盘后减1
                mDiskWritesInFlight++;

                boolean hasListeners = mListeners.size() > 0;
                if (hasListeners) {
                    mcr.keysModified = new ArrayList<String>();
                    mcr.listeners =
                            new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
                }

                synchronized (this) {
                    // 是否需要清空 SharedPreferences 里面的数据
                    if (mClear) {
                        if (!mMap.isEmpty()) {
                            mcr.changesMade = true;
                            mMap.clear();
                        }
                        mClear = false;
                    }

                    for (Map.Entry<String, Object> e : mModified.entrySet()) {
                        String k = e.getKey();
                        Object v = e.getValue();
                        // "this" is the magic value for a removal mutation. In addition,
                        // setting a value to "null" for a given key is specified to be
                        // equivalent to calling remove on that key.
                        if (v == this || v == null) {
                            if (!mMap.containsKey(k)) {
                                continue;
                            }
                            mMap.remove(k);
                        } else {
                            if (mMap.containsKey(k)) {
                                Object existingValue = mMap.get(k);
                                if (existingValue != null && existingValue.equals(v)) {
                                    continue;
                                }
                            }
                            mMap.put(k, v);
                        }

                        mcr.changesMade = true;
                        if (hasListeners) {
                            mcr.keysModified.add(k);
                        }
                    }
                    // 清空临时缓存
                    mModified.clear();
                }
            }
            return mcr;
        }
```

整个方法用到了 `SharedPreferencesImpl` 类锁来同步。

写入磁盘：

```
    private void enqueueDiskWrite(final MemoryCommitResult mcr,
                                  final Runnable postWriteRunnable) {
        final Runnable writeToDiskRunnable = new Runnable() {
                public void run() {
                    synchronized (mWritingToDiskLock) {
                        // 写入数据到磁盘
                        writeToFile(mcr);
                    }
                    synchronized (SharedPreferencesImpl.this) {
                        //减1标示完成了一个写磁盘操作
                        mDiskWritesInFlight--;
                    }
                    if (postWriteRunnable != null) {
                        postWriteRunnable.run();
                    }
                }
            };
        // 如果是 commit 方法，postWriteRunnable为空，为同步操作
        final boolean isFromSyncCommit = (postWriteRunnable == null);

        if (isFromSyncCommit) {
            // 如果是同步操作，而且只有一次的内存写入操作
            // 直接在当前线程执行写入磁盘操作并返回
            boolean wasEmpty = false;
            synchronized (SharedPreferencesImpl.this) {
                wasEmpty = mDiskWritesInFlight == 1;
            }
            if (wasEmpty) {
                writeToDiskRunnable.run();
                return;
            }
        }
        // 如果是apply，或者是在commit的情况下有多个批次的写入等待写入磁盘
        // 就另起线程异步执行写入操作
        QueuedWork.singleThreadExecutor().execute(writeToDiskRunnable);
    }
```

### startReloadIfChangedUnexpectedly

```
    void startReloadIfChangedUnexpectedly() {
        synchronized (this) {
            // 判断是否有意外的修改，比如其他进程的修改，如果没有就不用reload
            if (!hasFileChangedUnexpectedly()) {
                return;
            }
            startLoadFromDisk();
        }
    }
```

这个方法主要是从新从磁盘上把数据加到到内存中，保存在前面提到的Map中。

```
    private boolean hasFileChangedUnexpectedly() {
        synchronized (this) {
            if (mDiskWritesInFlight > 0) {
                // 如果 mDiskWritesInFlight 说明是我们自己的修改，是预期的，直接返回，不用reload
                return false;
            }
        }

        final StructStat stat;
        try {
            BlockGuard.getThreadPolicy().onReadFromDisk();
            stat = Os.stat(mFile.getPath());
        } catch (ErrnoException e) {
            return true;
        }

        synchronized (this) {
            // 比较文件的更新时间和大小是否和本地的一致，如果不一致就要重新load
            return mStatTimestamp != stat.st_mtime || mStatSize != stat.st_size;
        }
    }
```

### apply 和 commit

前面我们也分析了这两个方法的源码，下面来看一下这两个方法的差异点：

 - apply 是没有返回值的，commit 有返回值
 - apply 写入文件的操作是异步的，而commit 的写入文件的操作是在当前线程同步执行的

综合性能考虑，如果在主线程操作且不需要返回值的情况下，优先使用 apply 来提交修改。

<!--  

https://www.jianshu.com/p/875d13458538

-->
