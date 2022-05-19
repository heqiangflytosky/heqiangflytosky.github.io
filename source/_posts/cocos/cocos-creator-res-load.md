---
title: Cocos Creator --  资源文件加载流程
categories: cocos
comments: true
tags: [cocos]
description:  资源文件加载流程
date: 2020-3-10 10:00:00
---

## 概述

资源的加载主要包含游戏包内的 src/project.dev.js 、res目录下的文件以及一些网络的资源。
engine 部分资源文件的管理主要在 cocos2d/core/assets/CCAsset.js 和 cocos2d/core/platform/CCAssetLibrary.js。
engine 部分资源文件的加载和释放代码在 cocos2d/core/load-pipeline 目录下面，主要有 CCloader.js、pipeline.js、loader.js、loading-items.js、downloader.js、text-download.js、asset-loader.js
如果是原生平台，runtiem 部分资源文件的加载相关代码在 HttpClient-android.cpp、jsb_xmlhttprequest.cpp、Cocos2dxHttpURLConnection.java，主要是文件下载的实现部分。
如果是原生平台，还包括 jsb-adapter/engine/jsb-loader.js，这部分包括对原生平台对 Downloader 和 Loader 的重新实现。

AssetLibrary 初始化在 main.js 中进行，配置资源文件的路径：

```
    // init assets
    cc.AssetLibrary.init({
        libraryPath: 'res/import',
        rawAssetsBase: 'res/raw-',
        rawAssets: settings.rawAssets,
        packedAssets: settings.packedAssets,
        md5AssetsMap: settings.md5AssetsMap,
        subpackages: settings.subpackages
    });
```

## 框架简介

 - CCLoader：继承于 Pipeline，CCLoader提供了友好的资源管理接口（加载、获取、释放）以及一些辅助接口（如自动释放、对Pipeline的修改）。
 - Pipeline：包含了多个Pipe和多个LoadingItems，这里实现了一个Pipe到Pipe衔接流转的过程，以及Pipe和LoadingItems的管理接口。pipeline 自身实现的是资源缓存和让资源依次流过管道，即**每个资源依次经过每个 pipe 的处理**。
 - Pipe：这里的 pipe 是指实现了加载资源的单位（如 loader，assetLoader，downloader），pipeline 对资源的处理最终是调用 pipe.handle 实现的。每一种Pipe都会对资源进行特定的加工。
 - LoadingItems：LoadingItems 是一个加载对象队列，可以用来输送加载对象到加载管线中。它继承于CallbackInvoker，管理着LoadingItem，一个 LoadingItem 就是资源从开始加载到加载完成的上下文。这里说的上下文，指的是与加载该资源相关的变量的集合，比如当前加载的状态、url、依赖哪些资源、以及加载完成后的对象等等。LoadingItems 配合 pipeline，实现了加载状态维护、依赖资源入队等内容。


## 资源加载

资源加载采用了 PipeLine 模式。
一次资源的加载流程底层的调用函数如下：

 1. cc.loader.loadRes()
 2. CCLoader.load()
 3. loading-items.append
 4. pipeline.flowIn
 5. pipeline.flow
 6. pipe.handle（pipe 对应于 asset-loader、downloader、loader）
 7. 加载完成，进行回调

加载流程中其实还有一种情况需要注意：loader 其实会根据资源类型将加载任务分发给不同的类，如 uuid 类型会交给 uuid-loader。uuid-loader 加载时会加载该资源的依赖资源，其过程为：

 1. uuid-loader.loadUuid
 2. uuid-loader.loadDepends
 3. pipeline.flowInDeps
 4. loading-items.append
 5. 后续流程和正常流程一致

### 加载流程

### CCLoader

CCLoader 继承自 pipeline：

```
// engine CCLoader.js
js.extend(CCLoader, Pipeline);
```

创建 CCLoader 时，会把三个 Pipe 提供给 PipeLine：

```
    Pipeline.call(this, [
        assetLoader,
        downloader,
        loader
    ]);
```

CCLoader 是用户可以直接调用来加载、释放资源的类，它对 pipeline、loading-items、loader 进行了整合封装，并提供了 `load`、`loadRes`、`loadResDir`、`release`、`releaseRes` 等方法供用户使用。资源加载相关的便是 `load`、`loadRes` 等方法，而所有方法最终是调用 `load` 方法来进行资源加载的。
`load` 和其它接口的最大区别在于，`load` 可以用于加载绝对路径的资源（比如一个sd卡的绝对路径、或者网络上的一个url），而 `loadRes` 等只能加载 res 目录下的资源。

先来看一下 `loadRes` 方法：

```
// engine CCLoader.js
proto.loadRes = function (url, type, mount, progressCallback, completeCallback) {
    if (arguments.length !== 5) {
        completeCallback = progressCallback;
        progressCallback = mount;
        mount = 'assets';
    }

    var args = this._parseLoadResArgs(type, progressCallback, completeCallback);
    type = args.type;
    progressCallback = args.onProgress;
    completeCallback = args.onComplete;

    var self = this;
    var uuid = self._getResUuid(url, type, mount);
    if (uuid) {
        this.load(
            {
                type: 'uuid',
                uuid: uuid
            },
            progressCallback,
            function (err, asset) {
                if (asset) {
                    // should not release these assets, even if they are static referenced in the scene.
                    self.setAutoReleaseRecursively(uuid, false);
                }
                if (completeCallback) {
                    completeCallback(err, asset);
                }
            }
        );
    }
    else {
        self._urlNotFound(url, type, completeCallback);
    }
};
```

loadRes 工作流程如下：

 - 调用_getResUuid查询uuid，该方法会调用AssetTable的getUuid方法查询资源的uuid。从网络上加载的资源以及SD卡中我们存储的资源，Creator并没有为它们生成uuid。所以这些不是在Creator项目中生成的资源不能使用loadRes来加载。
 - 调用this.load方法加载资源。
 - 在回调方法中设置加载完成后的处理，在加载完成后，该资源以及其引用的资源都会被标记为禁止自动释放（在场景切换的时候，Creator会自动释放下个场景不使用的资源）。

load 方法如下：

```
// engine CCLoader.js
roto.load = function(resources, progressCallback, completeCallback) {
    if (CC_DEV && !resources) {
        return cc.error("[cc.loader.load] resources must be non-nil.");
    }

    if (completeCallback === undefined) {
        completeCallback = progressCallback;
        progressCallback = this.onProgress || null;
    }

    var self = this;
    var singleRes = false;
    var res;
    if (!(resources instanceof Array)) {
        if (resources) {
            singleRes = true;
            resources = [resources];
        } else { 
            resources = [];
        }
    }

    _sharedResources.length = 0;
    for (var i = 0; i < resources.length; ++i) {
        var resource = resources[i];
        // Backward compatibility
        if (resource && resource.id) {
            cc.warnID(4920, resource.id);
            if (!resource.uuid && !resource.url) {
                resource.url = resource.id;
            }
        }
        // 获取资源对应的 url
        res = getResWithUrl(resource);
        if (!res.url && !res.uuid)
            continue;
        var item = this._cache[res.url];
        console.log("TestRes CCLoader cc.load url = "+res.url)
        _sharedResources.push(item || res);
    }
    // 创建一个 LoadingItems
    var queue = LoadingItems.create(this, progressCallback, function (errors, items) {
        callInNextTick(function () {
            if (completeCallback) {
                if (singleRes) {
                    let id = res.url;
                    completeCallback.call(self, errors, items.getContent(id));
                }
                else {
                    completeCallback.call(self, errors, items);
                }
                completeCallback = null;
            }

            if (CC_EDITOR) {
                for (let id in self._cache) {
                    if (self._cache[id].complete) {
                        self.removeItem(id);
                    }
                }
            }
            items.destroy();
        });
    });
    LoadingItems.initQueueDeps(queue);
    queue.append(_sharedResources);
    _sharedResources.length = 0;
};
```

load 方法中，所做的就是创建一个 LoadingItem，然后对其进行初始化，之后调用其 append 方法，把一个加载项 LoadingItems 队列。这里 LoadingItem 简单来看就是对待加载资源的一层封装。
所有要加载的资源最后会被添加到_sharedResources中（不论该资源是否已加载，如果已加载会push它的item，未加载会push它的res对象，res对象是通过 `getResWithUrl` 方法从 `AssetLibrary` 中查询出来的）。

### loading-tiems

loading-items 主要是完善了每个资源对象的属性，包括资源内容、url 地址等，同时维护每个资源的加载状态、依赖关系等。

```
// engine  loading-tiems.js
roto.append = function (urlList, owner) {
    if (!this.active) {
        return [];
    }
    if (owner && !owner.deps) {
        owner.deps = [];
    }

    this._appending = true;
    var accepted = [], i, url, item;
    for (i = 0; i < urlList.length; ++i) {
        url = urlList[i];

        // Already queued in another items queue, url is actually the item
        if (url.queueId && !this.map[url.id]) {
            this.map[url.id] = url;
            // Register item deps for circle reference check
            owner && owner.deps.push(url);
            // Queued and completed or Owner circle referenced by dependency
            if (url.complete || checkCircleReference(owner, url)) {
                this.totalCount++;
                // console.log('----- Completed already or circle referenced ' + url.id + ', rest: ' + (this.totalCount - this.completedCount-1));
                this.itemComplete(url.id);
                continue;
            }
            // Not completed yet, should wait it
            else {
                var self = this;
                var queue = _queues[url.queueId];
                if (queue) {
                    this.totalCount++;
                    LoadingItems.registerQueueDep(owner || this._id, url.id);
                    // console.log('+++++ Waited ' + url.id);
                    queue.addListener(url.id, function (item) {
                        // console.log('----- Completed by waiting ' + item.id + ', rest: ' + (self.totalCount - self.completedCount-1));
                        self.itemComplete(item.id);
                    });
                }
                continue;
            }
        }
        // Queue new items
        if (isIdValid(url)) {
            item = createItem(url, this._id);
            var key = item.id;
            // No duplicated url
            if (!this.map[key]) {
                this.map[key] = item;
                this.totalCount++;
                // Register item deps for circle reference check
                owner && owner.deps.push(item);
                LoadingItems.registerQueueDep(owner || this._id, key);
                accepted.push(item);
                // console.log('+++++ Appended ' + item.id);
            }
        }
    }
    this._appending = false;

    // Manually complete
    if (this.completedCount === this.totalCount) {
        // console.log('===== All Completed ');
        this.allComplete();
    }
    else {
        this._pipeline.flowIn(accepted);
    }
    return accepted;
};
```

append 所做的是：先对资源列表 urlList 里每一个元素做字段完善（create）、依赖处理（registerQueueDep），然后调用 pipeline.flowIn 将所有未加载完成的资源放入 pipeline 之中。

### pipeline

pipeline 包含多个 pipe，pipe 存储在 _pipes 数组中，而且数组中的 nextPipe 赋值给了 lastPipe.next，以此将 pipe 链接成单向链表结构。这里的 pipe 是指实现了加载资源的单位（如 loader），pipeline 对资源的处理最终是调用 pipe.handle 实现的，pipeline 自身实现的是资源缓存和让资源依次流过管道，即每个资源依次经过每个 pipe 的处理。在 CCLoader 初始化 pipeline 时，为它添加了 AssetLoader、Downloader、Loader 三个 pipe，每个资源都会依次经过这三个 pipe 的处理。pipeline 中的 _cache 数组是对每个资源的缓存。

```
// asset-loader pipeline.js
proto.flowIn = function (items) {
    var i, pipe = this._pipes[0], item;
    if (pipe) {
        // Cache all items first, in case synchronous loading flow same item repeatly
        for (i = 0; i < items.length; i++) {
            item = items[i];
            this._cache[item.id] = item;
        }
        for (i = 0; i < items.length; i++) {
            item = items[i];
            flow(pipe, item);
        }
    }
    else {
        for (i = 0; i < items.length; i++) {
            //flowOut方法流出资源
            this.flowOut(items[i]);
        }
    }
};
```

lowIn 方法所做其实就是：先将每个资源 item 缓存到 _cache 中，然后对每个 item 调用 flow 方法。这里要注意的是 flow 的 pipe 参数为 _pipes[0]，即先将资源交给 pipeline 的第一个 pipe。以下是 flow 方法：

```
function flow (pipe, item) {
    var pipeId = pipe.id;
    var itemState = item.states[pipeId];
    var next = pipe.next;
    var pipeline = pipe.pipeline;

    if (item.error || itemState === ItemState.WORKING || itemState === ItemState.ERROR) {
        return;
    }
    else if (itemState === ItemState.COMPLETE) {
        if (next) {
            flow(next, item);
        }
        else {
            // flowOut方法流出资源
            pipeline.flowOut(item);
        }
    }
    else {
        item.states[pipeId] = ItemState.WORKING;
        // Pass async callback in case it's a async call
        var result = pipe.handle(item, function (err, result) {
            if (err) {
                item.error = err;
                item.states[pipeId] = ItemState.ERROR;
                pipeline.flowOut(item);
            }
            else {
                // Result can be null, then it means no result for this pipe
                if (result) {
                    item.content = result;
                }
                item.states[pipeId] = ItemState.COMPLETE;
                if (next) {
                    flow(next, item);
                }
                else {
                    pipeline.flowOut(item);
                }
            }
        });
        // If result exists (not undefined, null is ok), then we go with sync call flow
        if (result instanceof Error) {
            item.error = result;
            item.states[pipeId] = ItemState.ERROR;
            pipeline.flowOut(item);
        }
        else if (result !== undefined) {
            // Result can be null, then it means no result for this pipe
            if (result !== null) {
                item.content = result;
            }
            item.states[pipeId] = ItemState.COMPLETE;
            if (next) {
                flow(next, item);
            }
            else {
                pipeline.flowOut(item);
            }
        }
    }
}
```

flow 方法中对资源的处理分为三部分：

 - 如果资源状态为 ERROR，则直接返回
 - 如果资源状态为 COMPLETE 已完成，如果后续还有 pipe 就将其交给下一个 pipe 处理
 - 如果资源状态为其他状态，则调用 pipe.handle 方法对其进行加载。加载完成后，先将加载结果交给 item，即item.content = result，然后会对其加载状态 states 进行修改，最后如果后续还有 pipe 就将其交给后续 pipe 处理。

### Pipe

加下来介绍一下 pipe 对资源的处理，pipe-line 中包含了三个 pipe：assetLoader、downloader 和 loader。

#### asset-loader

asset-loader 是Pipeline的第一个Pipe，这个Pipe的职责是进行初始化，从cc.AssetLibrary中取出该资源的完整信息，获取该资源的类型，对rawAsset类型进行设置type，方便后面的pipe执行不同的处理，而非rawAsset则执行callback进入下一个Pipe处理。

```
// engine asset-loader.js
AssetLoader.prototype.handle = function (item, callback) {
    var uuid = item.uuid;
    if (!uuid) {
        return item.content || null;
    }

    var self = this;
    cc.AssetLibrary.queryAssetInfo(uuid, function (error, url, isRawAsset) {
        if (error) {
            callback(error);
        }
        else {
            item.url = item.rawUrl = url;
            item.isRawAsset = isRawAsset;
            if (isRawAsset) {
                var ext = cc.path.extname(url).toLowerCase();
                if (!ext) {
                    callback(new Error(debug.getError(4931, uuid, url)));
                    return;
                }
                ext = ext.substr(1);
                var queue = LoadingItems.getQueue(item);
                reusedArray[0] = {
                    queueId: item.queueId,
                    id: url,
                    url: url,
                    type: ext,
                    error: null,
                    alias: item,
                    complete: true
                };
                if (CC_EDITOR) {
                    self.pipeline._cache[url] = reusedArray[0];
                }
                queue.append(reusedArray);
                // Dispatch to other raw type downloader
                item.type = ext;
                callback(null, item.content);
            }
            else {
                item.type = 'uuid';
                callback(null, item.content);
            }
        }
    });
};
```

#### downloader

downloader 是对资源处理的第二道工序，负责对资源的下载，比如从磁盘读取，从网络读取等。
不同类型的资源在不同的平台下有不同的获取方式：

 - 比如脚本在原生平台使用 require 方法获取，而在H5平台则使用动态添加的` <script>` HTML标签，指定src进行加载。
 - 又比如json在原生平台使用 jsb.fileutils 进行加载，而在H5平台则使用XMLHttpRequest从网络下载。

关于 jsb.fileutils 的实现，参考前面的《游戏文件加载流程》一文。
原生平台在 jsb-adapter/engine/jsb-loader.js 对 Downloader 的方法进行了重新实现。

```
// engine downloader.js
Downloader.prototype.handle = function (item, callback) {
    var self = this;
    // 根据不同的type 获取不同的 downloadFunc
    var downloadFunc = this.extMap[item.type] || this.extMap['default'];
    var syncRet = undefined;
    // 并发限制，默认最多同时下载64个资源，超过的会进入队列，等待前面的资源加载完成后再依次进行加载
    if (this._curConcurrent < cc.macro.DOWNLOAD_MAX_CONCURRENT) {
        this._curConcurrent++;
        // 交由对应的 downloadFunc 处理
        syncRet = downloadFunc.call(this, item, function (err, result) {
            self._curConcurrent = Math.max(0, self._curConcurrent - 1);
            self._handleLoadQueue();
            callback && callback(err, result);
        });
        if (syncRet !== undefined) {
            this._curConcurrent = Math.max(0, this._curConcurrent - 1);
            this._handleLoadQueue();
            return syncRet;
        }
    }
    else if (item.ignoreMaxConcurrency) {
        // 如果item的ignoreMaxConcurrency为true则无视该并发限制
        syncRet = downloadFunc.call(this, item, callback);
        if (syncRet !== undefined) {
            return syncRet;
        }
    }
    else {
        // 放到队列中
        this._loadQueue.push({
            item: item,
            callback: callback
        });
    }
};
```

在 handle 方法中根据资源类型选择不同的方法加载。Downloader的 this.extMap 记录了各种资源类型的下载方式，在 Web 平台所有的类型最终都对应 downloader.js 提供的这6个下载方法：

```
// engine downloader.js
var defaultMap = {
    // JS
    'js' : downloadScript,

    // Images
    'png' : downloadImage,
    'jpg' : downloadImage,
    'bmp' : downloadImage,
    'jpeg' : downloadImage,
    'gif' : downloadImage,
    'ico' : downloadImage,
    'tiff' : downloadImage,
    'webp' : downloadImage,
    'image' : downloadImage,
    'pvr': downloadBinary,
    'pkm': downloadBinary,

    // Audio
    'mp3' : downloadAudio,
    'ogg' : downloadAudio,
    'wav' : downloadAudio,
    'm4a' : downloadAudio,

    // Txt
    'txt' : downloadText,
    'xml' : downloadText,
    'vsh' : downloadText,
    'fsh' : downloadText,
    'atlas' : downloadText,

    'tmx' : downloadText,
    'tsx' : downloadText,

    'json' : downloadText,
    'ExportJson' : downloadText,
    'plist' : downloadText,

    'fnt' : downloadText,

    // Font
    'font' : skip,
    'eot' : skip,
    'ttf' : skip,
    'woff' : skip,
    'svg' : skip,
    'ttc' : skip,

    // Deserializer
    'uuid' : downloadUuid,

    // Binary
    'binary' : downloadBinary,
    'bin' : downloadBinary,
    'dbbin' : downloadBinary,
    'skel': downloadBinary,

    'default' : downloadText
};
```

在初始化 Downloader 时加到 extMap：

```
this.extMap = js.mixin(extMap, defaultMap);
```

在原生平台：

```
// jsb-adapter jsb-loader.js
cc.loader.addDownloadHandlers({
    // JS
    'js' : downloadScript,
    'jsc' : downloadScript,

    // Images
    'png' : downloadImage,
    'jpg' : downloadImage,
    'bmp' : downloadImage,
    'jpeg' : downloadImage,
    'gif' : downloadImage,
    'ico' : downloadImage,
    'tiff' : downloadImage,
    'webp' : downloadImage,
    'image' : downloadImage,
    'pvr' : downloadImage,
    'pkm' : downloadImage,

    // Audio
    'mp3' : downloadAudio,
    'ogg' : downloadAudio,
    'wav' : downloadAudio,
    'mp4' : downloadAudio,
    'm4a' : downloadAudio,

    // Text
    'txt' : downloadText,
    'xml' : downloadText,
    'vsh' : downloadText,
    'fsh' : downloadText,
    'atlas' : downloadText,

    'tmx' : downloadText,
    'tsx' : downloadText,

    'json' : downloadText,
    'ExportJson' : downloadText,
    'plist' : downloadText,

    'fnt' : downloadText,

    'binary' : downloadBinary,
    'bin' : downloadBinary,
    'dbbin': downloadBinary,
    'skel': downloadBinary,

    'default' : downloadText
});
```

##### downloadScript

如果是微信或者原生平台，只是对脚本进行require（CommonJS模块化规范），这里主要是web平台的处理，原生平台的处理在后面统一介绍，web平台是通过创建一个script的HTML标签，指定标签的src，添加事件监听，通过这种HTML的方式下载脚本，使其生效。

```
// jsb-adapter jsb-loader.js
function downloadScript (item, callback) {
    require(item.url);
    return null;
}
```

```
// engine downloader.js
function downloadScript (item, callback, isAsync) {
    var url = item.url,
        d = document,
        s = document.createElement('script');

    if (window.location.protocol !== 'file:') {
        s.crossOrigin = 'anonymous';
    }

    s.async = isAsync;
    s.src = urlAppendTimestamp(url);
    function loadHandler () {
        s.parentNode.removeChild(s);
        s.removeEventListener('load', loadHandler, false);
        s.removeEventListener('error', errorHandler, false);
        callback(null, url);
    }
    function errorHandler() {
        s.parentNode.removeChild(s);
        s.removeEventListener('load', loadHandler, false);
        s.removeEventListener('error', errorHandler, false);
        callback(new Error(debug.getError(4928, url)));
    }
    s.addEventListener('load', loadHandler, false);
    s.addEventListener('error', errorHandler, false);
    d.body.appendChild(s);
}
```

##### downloadImage

```
// jsb-adapter jsb-loader.js
function downloadImage(item, callback) {
    let img = new Image();
    img.src = item.url;
    img.onload = function (info) {
        callback(null, img);
    };
    img.onerror = function (event) {
        callback(new Error('load image fail:' + img.src), null);
    };
    // Don't return anything to use async loading.
}
```

对于 Web 平台下载图片的方法，注意对于跨域的处理。
由于浏览器同源策略，凡是发送请求url的协议、域名、端口三者之间任意一与当前页面地址不同即为跨域，以下为跨域的详细描述表格。在web端，使用webgl模式无法直接使用跨域图片，需要服务器配合设置Access-Control-Allow-Origin（Canvas模式允许使用跨域图片）。

```
// engine downloader.js

function downloadImage (item, callback, isCrossOrigin, img) {
    if (isCrossOrigin === undefined) {
        isCrossOrigin = true;
    }

    var url = urlAppendTimestamp(item.url);
    console.log("TestRes download.js downloadImage url="+url)
    img = img || new Image();
    // 对跨域的处理
    if (isCrossOrigin && window.location.protocol !== 'file:') {
        img.crossOrigin = 'anonymous';
    }
    else {
        img.crossOrigin = null;
    }

    if (img.complete && img.naturalWidth > 0 && img.src === url) {
        return img;
    }
    else {
        function loadCallback () {
            img.removeEventListener('load', loadCallback);
            img.removeEventListener('error', errorCallback);

            img.id = item.id;
            callback(null, img);
        }
        function errorCallback () {
            img.removeEventListener('load', loadCallback);
            img.removeEventListener('error', errorCallback);
            // 在加载失败的情况下，如果是https就不重试了，因为就算加载了到了图片也无法渲染
            // 如果是开启跨域支持，则将crossOrigin设置为null再重试一次
            if (window.location.protocol !== 'https:' && img.crossOrigin && img.crossOrigin.toLowerCase() === 'anonymous') {
                downloadImage(item, callback, false, img);
            }
            else {
                callback(new Error(debug.getError(4930, url)));
            }
        }

        img.addEventListener('load', loadCallback);
        img.addEventListener('error', errorCallback);
        // 赋值 src 开始加载
        img.src = url;
    }
}
```

##### downloadText

```
// jsb-adapter jsb-loader.js
function downloadText (item) {
    var url = item.url;
    // 用 fileUtils 来实现
    var result = cc_jsb.fileUtils.getStringFromFile(url);
    if (typeof result === 'string' && result) {
        return result;
    }
    else {
        return new Error('Download text failed: ' + url);
    }
}
```

```
// engine text-downloader.js

module.exports = function (item, callback) {
    var url = item.url;
    url = urlAppendTimestamp(url);
    var xhr = cc.loader.getXMLHttpRequest(),
        errInfo = 'Load text file failed: ' + url;
    xhr.open('GET', url, true);
    if (xhr.overrideMimeType) xhr.overrideMimeType('text\/plain; charset=utf-8');
    xhr.onload = function () {
        if (xhr.readyState === 4) {
            if (xhr.status === 200 || xhr.status === 0) {
                callback(null, xhr.responseText);
            }
            else {
                callback({status:xhr.status, errorMessage:errInfo + '(wrong status)'});
            }
        }
        else {
            callback({status:xhr.status, errorMessage:errInfo + '(wrong readyState)'});
        }
    };
    xhr.onerror = function(){
        callback({status:xhr.status, errorMessage:errInfo + '(error)'});
    };
    xhr.ontimeout = function(){
        callback({status:xhr.status, errorMessage:errInfo + '(time out)'});
    };
    // 发起下载请求
    xhr.send(null);
};
```

##### 资源下载

对于原生平台不论是图片还是文本的加载最终会调用 runtime 中的 HttpClient-android 的相关 api 来进行下载。
在 jsb_xmlhttprequest.cpp 中实现 js 中 `XMLHttpRequest` 的方法，资源的下载就靠它们来完成。

```
// jsb_xmlhttprequest.cpp
bool register_all_xmlhttprequest(se::Object* global)
{
    se::Class* cls = se::Class::create("XMLHttpRequest", global, nullptr, _SE(XMLHttpRequest_constructor));
    cls->defineFinalizeFunction(_SE(XMLHttpRequest_finalize));

    cls->defineFunction("open", _SE(XMLHttpRequest_open));
    cls->defineFunction("abort", _SE(XMLHttpRequest_abort));
    cls->defineFunction("send", _SE(XMLHttpRequest_send));
    cls->defineFunction("setRequestHeader", _SE(XMLHttpRequest_setRequestHeader));
    cls->defineFunction("getAllResponseHeaders", _SE(XMLHttpRequest_getAllResponseHeaders));
    cls->defineFunction("getResponseHeader", _SE(XMLHttpRequest_getResonpseHeader));
    cls->defineFunction("overrideMimeType", _SE(XMLHttpRequest_overrideMimeType));
    cls->defineProperty("__mimeType", _SE(XMLHttpRequest_getMIMEType), nullptr);

    cls->defineProperty("readyState", _SE(XMLHttpRequest_getReadyState), nullptr);
    cls->defineProperty("status", _SE(XMLHttpRequest_getStatus), nullptr);
    cls->defineProperty("statusText", _SE(XMLHttpRequest_getStatusText), nullptr);
    cls->defineProperty("responseText", _SE(XMLHttpRequest_getResponseText), nullptr);
    cls->defineProperty("responseXML", _SE(XMLHttpRequest_getResponseXML), nullptr);
    cls->defineProperty("response", _SE(XMLHttpRequest_getResponse), nullptr);
    cls->defineProperty("timeout", _SE(XMLHttpRequest_getTimeout), _SE(XMLHttpRequest_setTimeout));
    cls->defineProperty("responseType", _SE(XMLHttpRequest_getResponseType), _SE(XMLHttpRequest_setResponseType));
    cls->defineProperty("withCredentials", _SE(XMLHttpRequest_getWithCredentials), nullptr);

    cls->install();

    JSBClassType::registerClass<XMLHttpRequest>(cls);

    __jsb_XMLHttpRequest_class = cls;

    se::ScriptEngine::getInstance()->clearException();

    return true;
}
```

#### loader

loader 一般是对资源最后的一道处理工序，比如对下载完之后的图片进行处理等。
这部分在不同平台的处理也是不一样的。
原生平台在 jsb-adapter/engine/jsb-loader.js 对 Downloader 的方法进行了重新实现。
Web 平台的实现在 load-pipeline/loader.js。

Web 平台：

```
// engine loader.js
Loader.prototype.handle = function (item, callback) {
    var loadFunc = this.extMap[item.type] || this.extMap['default'];
    return loadFunc.call(this, item, callback);
};
```

```
// engine loader.js
var defaultMap = {
    // Images
    'png' : loadImage,
    'jpg' : loadImage,
    'bmp' : loadImage,
    'jpeg' : loadImage,
    'gif' : loadImage,
    'ico' : loadImage,
    'tiff' : loadImage,
    'webp' : loadImage,
    'image' : loadImage,
    'pvr' : loadPVRTex,
    'pkm' : loadPKMTex,

    // Audio
    'mp3' : loadAudioAsAsset,
    'ogg' : loadAudioAsAsset,
    'wav' : loadAudioAsAsset,
    'm4a' : loadAudioAsAsset,

    // json
    'json' : loadJSON,
    'ExportJson' : loadJSON,

    // plist
    'plist' : loadPlist,

    // asset
    'uuid' : loadUuid,
    'prefab' : loadUuid,
    'fire' : loadUuid,
    'scene' : loadUuid,

    // binary
    'binary' : loadBinary,
    'dbbin' : loadBinary,
    'bin' : loadBinary,
    'skel' : loadBinary,

    // Font
    'font' : fontLoader.loadFont,
    'eot' : fontLoader.loadFont,
    'ttf' : fontLoader.loadFont,
    'woff' : fontLoader.loadFont,
    'svg' : fontLoader.loadFont,
    'ttc' : fontLoader.loadFont,

    'default' : loadNothing
};
```

原生Android平台

```
// jsb-adapter jsb-loader.js
cc.loader.addLoadHandlers({
    // Font
    'font' : loadFont,
    'eot' : loadFont,
    'ttf' : loadFont,
    'woff' : loadFont,
    'svg' : loadFont,
    'ttc' : loadFont,

    // Audio
    'mp3' : loadAudio,
    'ogg' : loadAudio,
    'wav' : loadAudio,
    'mp4' : loadAudio,
    'm4a' : loadAudio,

    // compressed texture
    'pvr': loadCompressedTex,
    'pkm': loadCompressedTex,
});
```


## 资源释放

CCLoader 对于资源释放提供了 release、releaseRes、releaseAsset、releaseResDir、releaseAll 方法，但实际上所有方法最终是调用 release 方法来实现资源释放的，在 release 调用之前所做的就是获取资源对应的 uuid，然后调用 release 方法。

## 相关文章：

https://blog.csdn.net/u013647453/article/details/88564966
https://www.cnblogs.com/ybgame/p/10576884.html
https://www.cnblogs.com/ybgame/p/10576986.html