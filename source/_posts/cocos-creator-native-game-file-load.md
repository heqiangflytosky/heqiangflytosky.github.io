---
title: Cocos Creator -- JSB 游戏文件加载流程
categories: cocos
comments: true
tags: [cocos]
description:  游戏文件加载流程
date: 2020-2-28 10:00:00
---

## 概述

前面引擎的初始化流程中我们提到了 jsb-builtin.js 和 main.js 文件的加载和运行，cocos 引擎运行游戏，首先要把游戏文件加载到内存中，在 HelloWorld 工程中，游戏资源文件是放在游戏包 的res目录中的，本文就了解一下文件的加载过程。
这部分主要介绍 runtime 中 FileUtils 的实现，在 Android 原生平台，本地资源文件的加载主要靠它来实现。

## 文件的加载过程

main.js 加载：

```
├── AppDelegate::applicationDidFinishLaunching()  // AppDelegate.cpp
    ├── jsb_run_script()  // jsb_global.cpp
        ├── ScriptEngine::runScript() // ScriptEngine.cpp
            ├── delegate.onGetStringFromFile // jsb_global.cpp
                ├── FileUtils::getStringFromFile // CCFileUtils.cpp
                    ├── FileUtilsAndroid::getContents  // CCFileUtils-android.cpp
```

文件加载的实现在 FileUtilsAndroid 中：

```
FileUtils::Status FileUtilsAndroid::getContents(const std::string& filename, ResizableBuffer* buffer)
{
    if (filename.empty())
        return FileUtils::Status::NotExists;
    //调用 `FileUtils::fullPathForFilename` 来获取文件的绝对路径。
    std::string fullPath = fullPathForFilename(filename);
    
    ......

    //如果 / 开头，就调用 FileUtils::getContents 打开文件
    if (fullPath[0] == '/')
        return FileUtils::getContents(fullPath, buffer);

    std::string relativePath;
    size_t position = fullPath.find(ASSETS_FOLDER_NAME);
    if (0 == position) {
        // "@assets/" is at the beginning of the path and we don't want it
        relativePath += fullPath.substr(strlen(ASSETS_FOLDER_NAME));
    } else {
        relativePath = fullPath;
    }

    if (obbfile)
    {
        if (obbfile->getFileData(relativePath, buffer))
            return FileUtils::Status::OK;
    }

    if (nullptr == assetmanager) {
        LOGD("... FileUtilsAndroid::assetmanager is nullptr");
        return FileUtils::Status::NotInitialized;
    }

    AAsset* asset = AAssetManager_open(assetmanager, relativePath.data(), AASSET_MODE_UNKNOWN);
    if (nullptr == asset) {
        LOGD("asset (%s) is nullptr", filename.c_str());
        return FileUtils::Status::OpenFailed;
    }

    auto size = AAsset_getLength(asset);
    buffer->resize(size);

    int readsize = AAsset_read(asset, buffer->buffer(), size);
    ......

    return FileUtils::Status::OK;
}
```

获取文件路径的过程：

```
std::string FileUtils::fullPathForFilename(const std::string &filename) const
{
    // 已经是绝对路径的情况
   if (isAbsolutePath(filename))
    {
        return normalizePath(filename);
    }

    // Already Cached ?
    auto cacheIter = _fullPathCache.find(filename);
    if(cacheIter != _fullPathCache.end())
    {
        return cacheIter->second;
    }

    // Get the new file name.
    const std::string newFilename( getNewFilename(filename) );

    std::string fullpath;
     // 从 _searchPathArray 获取文件的搜索路径
    for (const auto& searchIt : _searchPathArray)
    {
        for (const auto& resolutionIt : _searchResolutionsOrderArray)
        {
            fullpath = this->getPathForFilename(newFilename, resolutionIt, searchIt);

            if (!fullpath.empty())
            {
                // Using the filename passed in as key.
                _fullPathCache.insert(std::make_pair(filename, fullpath));
                return fullpath;
            }
        }
    }

    ......

    // The file wasn't found, return empty string.
    return "";
}
```

_searchPathArray 的初始化在：

```
bool FileUtils::init()
{
    _searchPathArray.push_back(_defaultResRootPath);
    _searchResolutionsOrderArray.push_back("");
    return true;
}
```

我们还可以通过 `FileUtils::setSearchPaths` 重新设置搜索路径，或者通过 `FileUtils::addSearchPath` 添加搜索路径。

另外，在 Android 平台上面，`_defaultResRootPath` 初始化为 `@assets/`。

```

#define  ASSETS_FOLDER_NAME          "@assets/"
bool FileUtilsAndroid::init()
{
    // 为 _defaultResRootPath 赋值
    _defaultResRootPath = ASSETS_FOLDER_NAME;
    
    std::string assetsPath(getApkPathJNI());
    if (assetsPath.find("/obb/") != std::string::npos)
    {
        obbfile = new ZipFile(assetsPath);
    }

    return FileUtils::init();
}
```

比如 jsb-builtin.js 放在 assets 中，它得到的完整路径为：`@assets/jsb-adapter/jsb-builtin.js`

另外：在 `Cocos2dxRenderer.nativeInit(this.mScreenWidth, this.mScreenHeight, mDefaultResourcePath); ` 中设置资源路径，但这个 `mDefaultResourcePath` 是空的。