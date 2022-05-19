---
title: FileProvider 的使用和源码分析
categories: Android
comments: true
tags: [FileProvider]
description: 介绍 FileProvider 的使用方法
date: 2018-8-22 10:00:00
---

## 概述

Android 7以后对于应用之间的文件访问做了限制，不再支持跨应用之间的 file:/// 格式文件访问，文件访问需要使用 content:/// 格式，官方推荐我们使用 FileProvider 来完成跨应用文件的读写。

## 使用

使用步骤：

 - manifest中声明 FileProvider
 - res/xml中定义对外暴露的文件夹路径
 - 生成content://类型的Uri
 - 给Uri授予临时权限
 - 使用Intent传递Uri


```
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_path" />
        </provider>
```

这里 `android:exported` 和 `android:grantUriPermissions` 必须设置为 false和true，否则编译时会抛出 SecurityException。后面源码分析会解释。

```
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <external-files-path
        name="test" path="testfolder"
        />
</paths>
```
name 是生成的 uri 中指代路径名称，path 是实际的路径名称。
paths 属性支持的类型在 FileProvider 源码中可以找到：

```
    String TAG_ROOT_PATH = "root-path";   // new File("/")
    String TAG_FILES_PATH = "files-path";  // context.getFilesDir()
    String TAG_CACHE_PATH = "cache-path";  // context.getCacheDir()
    String TAG_EXTERNAL = "external-path"; // Environment.getExternalStorageDirectory()
    String TAG_EXTERNAL_FILES = "external-files-path"; // ContextCompat.getExternalFilesDirs(context, null)
    String TAG_EXTERNAL_CACHE = "external-cache-path"; // ContextCompat.getExternalCacheDirs(context)
    String TAG_EXTERNAL_MEDIA = "external-media-path"; // context.getExternalMediaDirs()
```

```
        Intent intent = new Intent("****");
        intent.setPackage("com.*.*.*");
        //  生成Uri
        // content://com.example.heqiang.testsomething.fileprovider/test/doc
        // 即格式为content://authorities/定义的name属性/文件的相对路径
        Uri uri = FileProvider.getUriForFile(this,getPackageName()+".fileprovider",
                new File(getExternalFilesDir("testfolder").getAbsolutePath()+"/doc"));

        intent.putExtra("EXTRA_URL",uri.toString());
        // 赋予临时读权限
        // 这种方式不行
        //intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
        // 这种方式ok
        grantUriPermission("com.*.*.*", uri, Intent.FLAG_GRANT_READ_URI_PERMISSION);

        try {
            startActivity(intent);
        } catch (ActivityNotFoundException e) {
            Log.e("Test","launch error ",e);
        }
```

被启动的应用拿到文件流之后就可以对文件进行操作了，包括读和写。

```
InputStream in = context.getContentResolver().openInputStream(uri);
```


## 源码分析

`FileProvider` 是继承自 `ContentProvider`。

```
public class FileProvider extends ContentProvider {
```

对于增删改查接口仅支持 `query` 和 `delete`、以及 `openInputStream` 和 `openOutputStream`。

另外在 attachInfo 方法中检查了 `exported` 和 `grantUriPermissions` 属性：

```
    @Override
    public void attachInfo(@NonNull Context context, @NonNull ProviderInfo info) {
        super.attachInfo(context, info);

        // Sanity check our security
        if (info.exported) {
            throw new SecurityException("Provider must not be exported");
        }
        if (!info.grantUriPermissions) {
            throw new SecurityException("Provider must grant uri permissions");
        }

        mStrategy = getPathStrategy(context, info.authority);
    }
```

说明 FileProvider 不是公开的，不能像普通的 ContentProvider 那样的方式被外界的程序使用。

