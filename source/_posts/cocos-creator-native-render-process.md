---
title: Cocos Creator -- JSB 游戏渲染流程
categories: cocos
comments: true
tags: [cocos]
description:  游戏渲染流程
date: 2020-3-15 10:00:00
---

## 概述

本文主要介绍在 Android 原生平台的渲染流程。

## 渲染初始化

```
├──  _initRenderer  // engine CCGame.js
    ├── renderer.initWebGL  // engine core/renderer/index.js
        ├── new renderer.Scene()  // native cocos/renderer/renderer/Scene.cpp
        ├── _initBuiltins  // 
        ├── new renderer.ForwardRenderer // 
            ├── js_renderer_ForwardRenderer_init // native jsb_renderer_auto.cpp
                ├── ForwardRenderer::init // native ForwardRenderer.cpp
                ├── registerStage  // native ForwardRenderer.cpp
        ├── new renderer.RenderFlow  // 
            ├── js_renderer_RenderFlow_constructor // native jsb_renderer_auto.cpp
                ├── RenderFlow::RenderFlow // native RenderFlow.cpp
                    ├── _batcher = new ModelBatcher(this)
                    ├── ParallelTask.init
        ├── RenderFlow.init // jsb-adapter engine/scene/render-flow.js
            ├── EventTarget.call // engien 
```

```
    _initRenderer () {
        // Avoid setup to be called twice.
        if (this._rendererInitialized) return;

        let el = this.config.id,
            width, height,
            localCanvas, localContainer;

        if (CC_JSB || CC_RUNTIME) {
            this.container = localContainer = document.createElement("DIV");
            this.frame = localContainer.parentNode === document.body ? document.documentElement : localContainer.parentNode;
            localCanvas = window.__canvas;
            this.canvas = localCanvas;
        }
        else {
            var element = (el instanceof HTMLElement) ? el : (document.querySelector(el) || document.querySelector('#' + el));

            if (element.tagName === "CANVAS") {
                width = element.width;
                height = element.height;

                //it is already a canvas, we wrap it around with a div
                this.canvas = localCanvas = element;
                this.container = localContainer = document.createElement("DIV");
                if (localCanvas.parentNode)
                    localCanvas.parentNode.insertBefore(localContainer, localCanvas);
            } else {
                //we must make a new canvas and place into this element
                if (element.tagName !== "DIV") {
                    cc.warnID(3819);
                }
                width = element.clientWidth;
                height = element.clientHeight;
                this.canvas = localCanvas = document.createElement("CANVAS");
                this.container = localContainer = document.createElement("DIV");
                element.appendChild(localContainer);
            }
            localContainer.setAttribute('id', 'Cocos2dGameContainer');
            localContainer.appendChild(localCanvas);
            this.frame = (localContainer.parentNode === document.body) ? document.documentElement : localContainer.parentNode;

            function addClass (element, name) {
                var hasClass = (' ' + element.className + ' ').indexOf(' ' + name + ' ') > -1;
                if (!hasClass) {
                    if (element.className) {
                        element.className += " ";
                    }
                    element.className += name;
                }
            }
            addClass(localCanvas, "gameCanvas");
            localCanvas.setAttribute("width", width || 480);
            localCanvas.setAttribute("height", height || 320);
            localCanvas.setAttribute("tabindex", 99);
        }

        this._determineRenderType();
        // WebGL context created successfully
        if (this.renderType === this.RENDER_TYPE_WEBGL) {
            var opts = {
                'stencil': true,
                // MSAA is causing serious performance dropdown on some browsers.
                'antialias': cc.macro.ENABLE_WEBGL_ANTIALIAS,
                'alpha': cc.macro.ENABLE_TRANSPARENT_CANVAS
            };
            // 初始化WebGL，不同的平台渲染模式是不一样的
            renderer.initWebGL(localCanvas, opts);
            this._renderContext = renderer.device._gl;
            
            // Enable dynamic atlas manager by default
            if (!cc.macro.CLEANUP_IMAGE_CACHE && dynamicAtlasManager) {
                dynamicAtlasManager.enabled = true;
            }
        }
        if (!this._renderContext) {
            this.renderType = this.RENDER_TYPE_CANVAS;
            // Could be ignored by module settings
            // 初始化 Canvas，一般不会走这里
            renderer.initCanvas(localCanvas);
            this._renderContext = renderer.device._ctx;
        }

        this.canvas.oncontextmenu = function () {
            if (!cc._isContextMenuEnable) return false;
        };

        this._rendererInitialized = true;
    },
```

渲染环境的初始化：

```
// engine core/renderer/index.js

    initWebGL (canvas, opts) {
        console.log("TestStart engine core/renderer/index.js initWebGL")

        require('./webgl/assemblers');
        const ModelBatcher = require('./webgl/model-batcher');

        this.Texture2D = gfx.Texture2D;
        this.canvas = canvas;
        this._flow = cc.RenderFlow;
        
        if (CC_JSB && CC_NATIVERENDERER) {
            // 原生平台走这里，目前使用Android 平台 native 渲染
            // native codes will create an instance of Device, so just use the global instance.
            this.device = gfx.Device.getInstance();
            this.scene = new renderer.Scene();
            let builtins = _initBuiltins(this.device);
            // 初始化 ForwardRenderer 和 RenderFlow，这里两个类都在c++ 实现
            // 因此，原生平台的渲染流程都在 c++ 实现
            this._forward = new renderer.ForwardRenderer(this.device, builtins);
            let nativeFlow = new renderer.RenderFlow(this.device, this.scene, this._forward);
            this._flow.init(nativeFlow);
        }
        else {
            // 浏览器环境走这里，默认走 webgl渲染
            let Scene = require('../../renderer/scene/scene');
            let ForwardRenderer = require('../../renderer/renderers/forward-renderer');
            this.device = new gfx.Device(canvas, opts);
            this.scene = new Scene();
            let builtins = _initBuiltins(this.device);
            this._forward = new ForwardRenderer(this.device, builtins);
            this._handle = new ModelBatcher(this.device, this.scene);
            this._flow.init(this._handle, this._forward);
        }
    },
```

```
// jsb-adapter engine/scene/render-flow.js

RenderFlow.init = function (nativeFlow) { engine/scene/render-flow.js")
    cc.EventTarget.call(this);
    // 赋值 _nativeFlow
    this._nativeFlow = nativeFlow;
};
```


绑定 Stage 操作的相关方法：

```
bool ForwardRenderer::init(DeviceGraphics* device, std::vector<ProgramLib::Template>& programTemplates, Texture2D* defaultTexture, int width, int height)
{
    BaseRenderer::init(device, programTemplates, defaultTexture);
    registerStage("opaque", std::bind(&ForwardRenderer::opaqueStage, this, std::placeholders::_1, std::placeholders::_2));
    registerStage("shadowcast", std::bind(&ForwardRenderer::shadowStage, this, std::placeholders::_1, std::placeholders::_2));
    registerStage("transparent", std::bind(&ForwardRenderer::transparentStage, this, std::placeholders::_1, std::placeholders::_2));
    return true;
}
```

## 绘制流程

关于绘制，大体可以分为两类，图标绘制和字体绘制。
关于字体的绘制流程参考下一节。
绘制过程一个关键的方法就是 `CCTexture2D.handleLoadedTexture`，这个是贴图加载事件处理器。用于加载图片或者已经绘制好的字体的纹理数据。

当我们在代码中为 CCTexture2D 设置好 _nativeAsset 后，就开始进行绘制纹理流程：

```
├── _nativeAsset.set
    ├── initWithElement
        ├── handleLoadedTexture
            ├── new Texture2D.Texture2D
            ├── Texture2D.update // jsb-adapter renderer/jsb-gfx.js
                ├── Texture2D.updateNative 
                    ├── js_gfx_Texture2D_update // cocos2d-x-lite jsb-gfx-auto.cpp
                        ├── seval_to_TextureOptions
                        ├── Texture2D::update // cocos2d-x-lite Texture2D.cpp
                            ├── glActiveTexture
                            ├── glBindTexture
                            ├── Texture2D::setMipmap
                                ├── Texture2D::setImage
                                    ├── glCompressedTexImage2D
                                    ├── glTexImage2D
                            ├── Texture2D::setTexInfo
                            ├── DeviceGraphics::restoreTexture // cocos2d-x-lite DeviceGraphics.cpp
                                ├── glBindTexture
            
```

```
void Texture2D::update(const Options& options)
{
    RENDERER_LOGD("TestRender TestLabel Texture2D::update: %p", this);
    bool genMipmap = _hasMipmap;

    _width = options.width;
    _height = options.height;
    _anisotropy = options.anisotropy;
    _minFilter = options.minFilter;
    _magFilter = options.magFilter;
    _mipFilter = options.mipFilter;
    _wrapS = options.wrapS;
    _wrapT = options.wrapT;
    _glFormat = options.glFormat;
    _glInternalFormat = options.glInternalFormat;
    _glType = options.glType;
    _compressed = options.compressed;
    _bpp = options.bpp;

    // check if generate mipmap
    _hasMipmap = options.hasMipmap;
    genMipmap = options.hasMipmap;

    if (options.images.size() > 1)
    {
        genMipmap = false; //REFINE: is it true here?
        uint16_t maxLength = options.width > options.height ? options.width : options.height;
        if (maxLength >> (options.images.size() - 1) != 1)
            RENDERER_LOGE("texture-2d mipmap is invalid, should have a 1x1 mipmap.");
    }

    // NOTE: get pot after _width, _height has been assigned.
    bool pot = isPow2(_width) && isPow2(_height);
    if (!pot)
        genMipmap = false;
    //将纹理与GL_TEXTURE0关联起来。
    GL_CHECK(glActiveTexture(GL_TEXTURE0));
    //绑定当前纹理，后续对纹理的操作都将作用在此纹理上。
    GL_CHECK(glBindTexture(GL_TEXTURE_2D, _glID));
    //调用 glTexImage2D 将内存中的数据拷贝到OpenGL纹理单元（显存）中
    if (!options.images.empty())
        setMipmap(options.images, options.flipY, options.premultiplyAlpha);

    setTexInfo();

    if (genMipmap)
    {
        GL_CHECK(glHint(GL_GENERATE_MIPMAP_HINT, GL_NICEST));
        GL_CHECK(glGenerateMipmap(GL_TEXTURE_2D));
    }
    //重置纹理
    _device->restoreTexture(0);
}
```

```
void DeviceGraphics::restoreTexture(uint32_t index)
{
    RENDERER_LOGW("TestRender TestLabel DeviceGraphics::restoreTexture");
    // 获取当前 State 保存的对应需要更新的纹理
    auto texture = _currentState->getTexture(index);
    // 绑定当前纹理，后续对纹理的操作都将作用在此纹理上。
    if (texture)
    {
        GL_CHECK(glBindTexture(texture->getTarget(), texture->getHandle()));
    }
    else
    {
        GL_CHECK(glBindTexture(GL_TEXTURE_2D, 0));
    }
}
```

## 渲染流程

前面章节我们介绍过，启动流程结束后，引擎会按帧率刷新，调用 CCDirector.mainLoop 来执行渲染流程。

```
├── CCDirector.mainLoop // engine core/CCDirector.js
    ├── renderer.renderer // engine core/renderer/index.js
        ├── RenderFlow.render // jsb-adapter engine/scene/render-flow.js
            ├── RenderFlow.validateRenderers() // engine core/renderer/render-flow.js
            ├── assembler._updateRenderData()
            ├── _nativeFlow.render(js_renderer_RenderFlow_render) //cocos2d-x-lit jsb-renderer_auto.cpp 
                ├── RenderFlow::render // cocos2d-x-lit RenderFlow.cpp
                    ├── ModelBatcher::startBatch()
                    ├── ModelBatcher::terminateBatch // cocos2d-x-lit ModelBatcher.cpp
                        ├── ModelBatcher::flush()
                        ├── ModelBatcher::flushIA()
                    ├── ForwardRenderer::render // cocos2d-x-lit ForwardRenderer.cpp
                        ├── BaseRenderer::render //cocos2d-x-lit BaseRenderer.cpp 
                            ├── ForwardRenderer::opaqueStage //cocos2d-x-lit BaseRenderer.cpp
                                ├── ForwardRenderer::drawItems //cocos2d-x-lit ForwardRenderer.cpp
                                    ├── BaseRenderer::draw // cocos2d-x-lit BaseRenderer.cpp
                                        ├── ProgramLib::switchProgram
                                            ├── Program::link()
                                                ├── _createShader
                                                    ├── glCreateShader
                                                ├── glCreateProgram
                                        ├── DeviceGraphics::draw // cocos2d-x-lit DeviceGraphics.cpp
                                            ├── glDrawElements
```

```
RenderFlow.render = function (scene) {
    _rendering = true;
    // 渲染前，确认需要重新渲染的组件，具体介绍在下面
    RenderFlow.validateRenderers();
    
    for (let i = 0, l = _dirtyTargets.length; i < l; i++) {
        let node = _dirtyTargets[i];
        node._inJsbDirtyList = false;

        let comp = node._renderComponent;
        if (!comp) continue;
        let assembler = comp._assembler;
        if (!assembler) continue;

        let flag = node._dirtyPtr[0];
        // 通过Flag来更新相关渲染数据
        // 该 Flag 为渲染流的某个状态
        // 该 Flag 通过在组件中设置 _renderFlag 得到
        if (flag & RenderFlow.FLAG_UPDATE_RENDER_DATA) {
            // 通过 assembler 更新各级渲染对象的顶点信息
            node._dirtyPtr[0] &= ~RenderFlow.FLAG_UPDATE_RENDER_DATA;
            assembler._updateRenderData && assembler._updateRenderData();
        }
        if (flag & RenderFlow.FLAG_COLOR) {
            node._dirtyPtr[0] &= ~RenderFlow.FLAG_COLOR;
            comp._updateColor && comp._updateColor();
        }

    }

    _dirtyTargets.length = 0;
    // 调用 native 方法 RenderFlow::render
    this._nativeFlow.render(scene._proxy, director._deltaTime);

    _dirtyTargets = _dirtyWaiting.slice(0);
    _dirtyWaiting.length = 0;
    // 更新渲染标志位
    _rendering = false;
};
```

```
RenderFlow.validateRenderers = function () {
    // 遍历所有添加的组件，然后调用这些组件的 _validateRender 方法，确认是否需要渲染
    // _validateList 的添加在 RenderFlow.registerValidate
    for (let i = 0, l = _validateList.length; i < l; i++) {
        let renderComp = _validateList[i];
        if (!renderComp.isValid) continue;
        if (!renderComp.enabledInHierarchy) {
            renderComp.disableRender();
        }
        else {
            renderComp._validateRender();
        }
        renderComp._inValidateList = false;
    }
    _validateList.length = 0;
};

// validate whether render component is ready to be rendered.
let _validateList = [];
RenderFlow.registerValidate = function (renderComp) {
    if (renderComp._inValidateList) return;
    _validateList.push(renderComp);
    renderComp._inValidateList =  true;
};
```

`RenderFlow.registerValidate ` 的调用流程如下，在各个子组件自行调用：

```
├── RenderComponent.markForRender
    ├──RenderComponent.markForValidate
       ├── RenderFlow.registerValidate
```

```
void RenderFlow::render(NodeProxy* scene, float deltaTime)
{
    //__android_log_print(ANDROID_LOG_DEBUG, "TestStart", "TestStart cocos2d-x-lite RenderFlow::render renderer/scene/RenderFlow.cpp");
    if (scene != nullptr)
    {
        
#if USE_MIDDLEWARE
        // udpate middleware before render
        middleware::MiddlewareManager::getInstance()->update(deltaTime);
#endif
        
#if SUB_RENDER_THREAD_COUNT > 0

        int mainThreadTid = RENDER_THREAD_COUNT - 1;

        NodeMemPool* instance = NodeMemPool::getInstance();
        auto& commonList = instance->getCommonList();
        if (commonList.size() < LocalMat_Use_Thread_Unit_Count)
        {
            _parallelStage = ParallelStage::NONE;
            calculateLocalMatrix();
        }
        else
        {
            _parallelStage = ParallelStage::LOCAL_MAT;
            _paralleTask->beginAllThreads();
            calculateLocalMatrix(mainThreadTid);
            _paralleTask->waitAllThreads();
        }

        _curLevel = 0;
        for(auto count = _levelInfoArr.size(); _curLevel < count; _curLevel++)
        {
            auto& levelInfos = _levelInfoArr[_curLevel];
            if (levelInfos.size() < WorldMat_Use_Thread_Node_count)
            {
                _parallelStage = ParallelStage::NONE;
                calculateLevelWorldMatrix();
            }
            else
            {
                _parallelStage = ParallelStage::WORLD_MAT;
                _paralleTask->beginAllThreads();
                calculateLevelWorldMatrix(mainThreadTid);
                _paralleTask->waitAllThreads();
            }
        }
#else
        calculateLocalMatrix();
        calculateWorldMatrix();
#endif
        
        _batcher->startBatch();

#if USE_MIDDLEWARE
        // render middleware
        middleware::MiddlewareManager::getInstance()->render(deltaTime);
#endif
        scene->resetGlobalRenderOrder();
        
        auto traverseHandle = scene->traverseHandle;
        traverseHandle(scene, _batcher, _scene);
        _batcher->terminateBatch();

        _forward->render(_scene);
    }
}

void ForwardRenderer::render(Scene* scene)
{
    __android_log_print(ANDROID_LOG_DEBUG, "TestStart", "TestStart TestRender cocos2d-x-lite ForwardRenderer::render renderer/renderer/ForwardRenderer.cpp");
    resetData();
    updateLights(scene);
    scene->sortCameras();
    auto& cameras = scene->getCameras();
    auto &viewSize = Application::getInstance()->getViewSize();
    for (auto& camera : cameras)
    {
        View* view = requestView();
        camera->extractView(*view, viewSize.x, viewSize.y);
    }
    __android_log_print(ANDROID_LOG_DEBUG, "TestStart", "TestRender ForwardRenderer::render %i", _views->getLength());
    for (size_t i = 0, len = _views->getLength(); i < len; ++i) {
        //if (i == 1) break;
        const View* view = _views->getData(i);
        BaseRenderer::render(*view, scene);
    }
    
    scene->removeModels();
}
```

```
void BaseRenderer::render(const View& view, const Scene* scene)
{
    // setup framebuffer
    _device->setFrameBuffer(view.frameBuffer);
    
    // setup viewport
    _device->setViewport(view.rect.x,
                         view.rect.y,
                         view.rect.w,
                         view.rect.h);
    
    // setup clear
    Color4F clearColor;
    if (ClearFlag::COLOR & view.clearFlags)
        clearColor = view.color;
    _device->clear(view.clearFlags, &clearColor, view.depth, view.stencil);
    
    // get all draw items
    // 获取当前场景中所有需要绘制的元素
    _drawItems->reset();
    for (const auto& model : scene->getModels())
    {
        int modelMask = model->getCullingMask();
        if ((modelMask & view.cullingMask) == 0)
            continue;
        
        DrawItem* drawItem = _drawItems->add();
        model->extractDrawItem(*drawItem);
    }
    
    // dispatch draw items to different stage
    _stageInfos->reset();
    StageItem stageItem;
    for (const auto& stage : view.stages)
    {
        StageInfo* stageInfo = _stageInfos->add();
        stageInfo->stage = stage;
        stageInfo->items.clear();
        // 添加当前 Stage 需要绘制的元素
        for (size_t i = 0, len = _drawItems->getLength(); i < len; i++)
        {
            const DrawItem* item = _drawItems->getData(i);
            
            stageItem.passes.clear();
            for (const Pass* p : item->effect->getPasses())
            {
                if (p->getStage() == stage)
                {
                    stageItem.passes.push_back(p);
                }
            }
            if (stageItem.passes.size() == 0)
            {
                continue;
            }
            
            stageItem.model = item->model;
            stageItem.ia = item->ia;
            stageItem.effect = item->effect;
            stageItem.sortKey = -1;
            
            stageInfo->items.push_back(stageItem);
        }
    }
    
    // render stages
    std::unordered_map<std::string, const StageCallback>::iterator foundIter;
    for (size_t i = 0, len = _stageInfos->getLength(); i < len; i++)
    {
        StageInfo* stageInfo = _stageInfos->getData(i);
        foundIter = _stage2fn.find(stageInfo->stage);
        __android_log_print(ANDROID_LOG_DEBUG, "TestStart", "TestStart TestRender cocos2d-x-lite BaseRenderer::render %s",stageInfo->stage.c_str());
        if (_stage2fn.end() != foundIter)
        {
            auto& fn = foundIter->second;
            fn(view, stageInfo->items);
        }
    }
}
```

`ForwardRenderer::drawItems` 用来绘制当前 Stage 的所有显示元素。

```
void ForwardRenderer::drawItems(const std::vector<StageItem>& items)
{
    size_t count = _shadowLights.size();
    __android_log_print(ANDROID_LOG_DEBUG, "TestStart","TestRender %i",count);
    if (count == 0 && _numLights == 0)
    {
        for (size_t i = 0, l = items.size(); i < l; i++)
        {
            draw(items.at(i));
        }
    }
    else
    {
        for (const auto& item : items)
        {
            for(int i = 0; i < count; i++)
            {
                Light* light = _shadowLights.at(i);
                _device->setTexture(cc_shadow_map[i], light->getShadowMap(), allocTextureUnit());
            }
            draw(item);
        }
    }
}
```


## Label 字体绘制渲染流程

下面以 Lable 字体的渲染来了解一下 2d 渲染流程。

```
├── CCLabel._applyFontTexture() // engine CCLabel.js
    ├── CCSpriteFrame.onTextureLoaded // 如果是BitmapFont会走这里。
    ├── Texture2D.initWithElement
    ├── TTFAssembler.updateRenderData  //engine core/renderer/utils/label/ttf.js
        ├── TTFAssembler._updateTexture  //engine ttf.js
            ├── _context.fillText  //engine ttf.js
                ├── js_engine_CanvasRenderingContext2D_fillText  jsb_cocos2d_auto.cpp
                    ├── CanvasRenderingContext2D.fillText  CCCanvasRenderingContext2D-android.cpp
                        ├── CanvasRenderingContext2D.recreateBufferIfNeeded()
                        ├── CanvasRenderingContext2DImpl.fillText   //CCCanvasRenderingContext2D-android.cpp
                            ├── JniHelper::callObjectVoidMethod
                                ├── CanvasRenderingContext2DImpl.fillText  //CanvasRenderingContext2DImpl.java
                            ├── CanvasRenderingContext2DImpl.fillData() //CCCanvasRenderingContext2D-android.cpp
                        ├── _canvasBufferUpdatedCB()
            ├── CCTexture2D.handleLoadedTexture // engine CCTexture2D.js
                ├── Texture2D.update // jsb-adapter renderer/jsb-gfx.js
                    ├── Texture2D.updateNative 
                        ├── js_gfx_Texture2D_update // cocos2d-x-lite jsb-gfx-auto.cpp
                            ├── seval_to_TextureOptions
                            ├── Texture2D::update // cocos2d-x-lite Texture2D.cpp
                                ├── glActiveTexture
                                ├── glBindTexture
                                ├── Texture2D::setMipmap
                                    ├── Texture2D::setImage
                                        ├── glCompressedTexImage2D
                                        ├── glTexImage2D
                                ├── Texture2D::setTexInfo
                                ├── DeviceGraphics::restoreTexture // cocos2d-x-lite DeviceGraphics.cpp
                                    ├── glBindTexture
                        
```

获取Canvas：

```
    _getAssemblerData () {
        _sharedLabelData = Label._canvasPool.get();
        _sharedLabelData.canvas.width = _sharedLabelData.canvas.height = 1;
        console.log('Test label ttf _getAssemblerData')
        return _sharedLabelData;
    }
```

### 创建buffer

创建一个 Bitmap 绑定 Canvas：

```
    private void recreateBuffer(float w, float h) {
         Log.d(TAG, "TestRender CanvasRenderingContext2DImpl recreateBuffer:" + w + ", " + h);
        if (mBitmap != null) {
            mBitmap.recycle();
        }
        mBitmap = Bitmap.createBitmap((int)Math.ceil(w), (int)Math.ceil(h), Bitmap.Config.ARGB_8888);
        mCanvas.setBitmap(mBitmap);
    }
```

### 绘制字体

```
// CanvasRenderingContext2DImpl.java

    private void fillText(String text, float x, float y, float maxWidth) {
        Log.d(TAG, "TestRender this: " + this + ", fillText: " + text + ", " + x + ", " + y + ", " + ", " + maxWidth);
        createTextPaintIfNeeded();
        mTextPaint.setARGB(mFillStyleA, mFillStyleR, mFillStyleG, mFillStyleB);
        mTextPaint.setStyle(Paint.Style.FILL);
        scaleX(mTextPaint, text, maxWidth);
        Point pt = convertDrawPoint(new Point(x, y), text);
        mCanvas.drawText(text, pt.x, pt.y, mTextPaint);
    }
```

### 获取纹理数据

绘制完字体后获取数据：

```
// CCCanvasRenderingContext2D-android.cpp

    void fillText(const std::string& text, float x, float y, float maxWidth)
    {
        SE_LOGD("CanvasRenderingContext2DImpl fillText");
        if (text.empty() || _bufferWidth < 1.0f || _bufferHeight < 1.0f)
            return;
        JniHelper::callObjectVoidMethod(_obj, JCLS_CANVASIMPL, "fillText", text, x, y, maxWidth);
        fillData();
    }
    

    void fillData()
    {
        jbyteArray arr = JniHelper::callObjectByteArrayMethod(_obj, JCLS_CANVASIMPL, "getDataRef");
        jsize len  = JniHelper::getEnv()->GetArrayLength(arr);
        jbyte* jbarray = (jbyte *)malloc(len * sizeof(jbyte));
        JniHelper::getEnv()->GetByteArrayRegion(arr,0,len,jbarray);
        unMultiplyAlpha( (unsigned char*) jbarray, len);
        _data.fastSet((unsigned char*) jbarray, len); //IDEA: DON'T create new jbarray every time.
        JniHelper::getEnv()->DeleteLocalRef(arr);
    }
```

`getDataRef` 方法的实现也在 CanvasRenderingContext2DImpl.java，把 mBitmap 的数据进行复制。

```

//CanvasRenderingContext2DImpl.java

    private byte[] getDataRef() {
        Log.d(TAG, "TestRender this: " + this + ", getDataRef ...");
        if (mBitmap != null) {
            final int len = mBitmap.getWidth() * mBitmap.getHeight() * 4;
            final byte[] pixels = new byte[len];
            final ByteBuffer buf = ByteBuffer.wrap(pixels);
            buf.order(ByteOrder.nativeOrder());
            mBitmap.copyPixelsToBuffer(buf);

            return pixels;
        }

        Log.e(TAG, "getDataRef return null");
        return null;
    }

```

_canvasBufferUpdatedCB 就是前面在 `HTMLCanvasElement.getContext()` 方法中创建的回调函数，Android端通过C++端把数据回传回来。

## 更新数据

绘制完成后，就开始通过 handleLoadedTexture 把绘制完成的纹理更新到场景中，等待下个tick周期进行渲染到屏幕上。


## 推荐文章

[基于WebGL的Canvas元素2D绘制加速](http://www.doc88.com/p-0901392998074.html)
[Cocos Creator 2.2 的渲染流程(原生渲染）]( https://zhuanlan.zhihu.com/p/96475666)：这部分在2.3版本后已经变更，native 渲染部分在 c++ 里面实现。
[cocos creator2.1渲染流程分析](https://www.jianshu.com/p/a678028fc583)
https://blog.csdn.net/6346289/article/details/99626005?depth_1-utm_source=distribute.pc_relevant.none-task-blog-OPENSEARCH-2&utm_source=distribute.pc_relevant.none-task-blog-OPENSEARCH-2
https://blog.csdn.net/wjf1616/article/details/105250690/