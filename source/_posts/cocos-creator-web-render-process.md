---
title: Cocos Creator --  Web游戏渲染流程
categories: cocos
comments: true
tags: [cocos]
description:  Web游戏渲染流程
date: 2020-3-20 10:00:00
---

## 概述

前面我们介绍过，在 engine core/renderer/index.js 的 `initWebGL` 方法中，针对不同的平台，提供的渲染的方式也是不一样的。
本文介绍在 Web 平台的渲染方式，相关代码主要在 engine 的 cocos2d/core/renderer/ 、 cocos2d/renderer/ 和 cocos2d/core/components/中。

## 相关类

 - RenderComponent：所有的渲染组件都是继承自 cc.RenderComponent，例如cc.Sprite，cc.Label 等
 - Assembler：组件的 Assembler 主要负责组件数据的更新处理及填充，生成 RenderData，由于不同的渲染组件在数据内容及填充上也都不相同，所以每一个渲染组件都会对应拥有自己的 Assembler 对象，而所有的 Assembler 对象都是继承自  cc.Assembler。
 - Material：作为资源，主要记录渲染组件的渲染状态，使用的纹理及 Shader。控制渲染组件的视觉效果，有 Effect 属性。
 - RenderData：Assembler 中持有的渲染数据，用来保存顶点数据、顶点索引数据。
 - RenderFlow：渲染流，用来遍历场景下面的所有节点，根据每个节点的_renderFlag，处理节点的位置，颜色，透明度，更新并渲染。
 - ModelBatcher：用来管理渲染数据的 model，渲染批次合并，从而减少drawcall，提升新能。
 - ForwardRenderer：持有的 Device 使用 gl 的函数，将定点数据、纹理等绘制到屏幕上。在 `initWebGL` 根据平台的不同，创建不同的 ForwardRenderer。
 - Effect：可定义GLSL脚本。

## 初始化

```
├── _initRenderer  // engine CCGame.js
    ├── renderer.initWebGL  // engine core/renderer/index.js
        ├── new Device 
        ├── new renderer.Scene() 
        ├── _initBuiltins 
        ├── new ForwardRenderer // engien cocos2d/renderer/renderers/forward-renderer.js
            ├── _registerStage // engien cocos2d/renderer/core/base-renderer.js
        ├── new ModelBatcher
        ├── RenderFlow.init // engien cocos2d/core/renderer/render-flow.js
            ├── RenderFlow.getFlow // engien cocos2d/core/renderer/render-flow.js
                ├── RenderFlow.createFlow  // engien cocos2d/core/renderer/render-flow.js
            ├── RenderFlow.getFlow._func // 开始渲染流程
```

## 渲染流程

我们在《游戏启动运行流程》介绍到，每一帧的刷新都会调用 director.mainLoop 方法。
Web 平台的渲染流程如下：

```
├── CCDirector.mainLoop // engine CCDirector.js
    ├── renderer.render // engine cocos2d/core/renderer/index.js
        ├── RenderFlow.render // engien cocos2d/core/renderer/render-flow.js
            ├── RenderFlow.visitRootNode // engien cocos2d/core/renderer/render-flow.js
                ├── flows[rootNode._renderFlag]._func // 执行RenderFlow渲染流程，后面详细介绍
                ......
                    ├── RenderFlow._children
                        ├── RenderFlow._render
                        ......
            ├── ForwardRenderer.render // engien cocos2d/renderer/renderers/forward-renderer.js
                ├── Base._render // engien cocos2d/renderer/core/base-renderer.js
                    ├── _stage2fn[info.stage] // engien cocos2d/renderer/core/base-renderer.js
                        ├── _opaqueStage // engien cocos2d/renderer/renderers/forward-renderer.js
                            ├── _drawItems // engien cocos2d/renderer/renderers/forward-renderer.js
                                ├── Base.draw // engien cocos2d/renderer/core/base-renderer.js
                                    ├── Device.draw // engien cocos2d/renderer/gfx/device.js
```

## RenderComponent

相关代码在 cocos2d/core/components/CCRenderComponent.js

 - _resetAssembler：在组件创建的时候会去调用，会在组件生命周期方法之前执行，主要负责创建并初始化渲染组件的 Assembler 实例。
 - _activeMaterial：负责创建并设置渲染组件所使用的材质实例，会在组件启用及材质修改时调用。 
 - markForRender 、disableRender：控制渲染组件是否进行渲染
 - setVertsDirty：在渲染组件数据有更新时，设置标记
 - setMaterial：则是设置材质实例给渲染组件
 - _updateMaterial：更新材质
 - _updateColor：更新颜色
 - _checkBacth ：检测合批

## Assembler

组件的 Assembler 主要负责组件数据的更新处理及填充，生成 RenderData，由于不同的渲染组件在数据内容及填充上也都不相同，所以每一个渲染组件都会对应拥有自己的 Assembler 对象，而所有的 Assembler 对象都是继承自  cc.Assembler。
相关代码在 cocos2d/core/renderer/webgl/assemblers/ 和 cocos2d/core/renderer/ 下面。
Assembler 定义了下面的方法：

 - init 负责初始化渲染数据及一些局部参数
 - updateRenderData 负责在渲染组件的顶点数据有变化时进行更新修改
 - fillBuffers 负责在渲染时进行顶点数据的 Buffer 填充

2D渲染中，Assembler2D 类是一个重要的基础类，最常用的cc.Sprite的各种模式（Simple，平铺，九宫格）在内部都对应了不同的Assembler派生类。同样是一个四边形的节点，不同的Assembler可以将其转化成不同数量的顶点实现不同的渲染效果：

 - Simple模式下是常规的四边形，有4个顶点
 - 平铺模式下Assembler根据纹理的重复次数对节点进行“拆碎”，相当于每重复一次就产生1个四边形
 - 九宫格模式下Assembler将节点拆分为9个四边形，每个四边形对应纹理上的一个“格子”

源码在 cocos2d/core/renderer/assembler-2d.js

```
    fillBuffers (comp, renderer) {
        // 如果节点的世界坐标发生变化，重新从当前节点的世界坐标计算一次顶点数据
        if (renderer.worldMatDirty) {
            this.updateWorldVerts(comp);
        }
      	// 获取准备好的顶点数据
      	// vData包含pos、uv、color数据
      	// iData包含三角剖分后的顶点索引数据
        let renderData = this._renderData;
        let vData = renderData.vDatas[0];
        let iData = renderData.iDatas[0];
      	// 获取顶点缓存
        let buffer = this.getBuffer(renderer);
        // 获取当前节点的顶点数据对应最终buffer的偏移量
      	// 可以简单理解为当前节点和其他同格式节点的数据，都将按顺序追加到这个大buffer里
        let offsetInfo = buffer.request(this.verticesCount, this.indicesCount);

        // buffer data may be realloc, need get reference after request.

        // fill vertices
        let vertexOffset = offsetInfo.byteOffset >> 2,
            vbuf = buffer._vData;
      	// 将准备好的vData拷贝到VetexBuffer里。
        // 这里会判断如果buffer装不下了，vData会被截断一部分。
      	// 通常不会出现装不下这种情况，因为buffer.request中会分配足够大的空间；如果出现，则当前组件只能被渲染一部分
        if (vData.length + vertexOffset > vbuf.length) {
            vbuf.set(vData.subarray(0, vbuf.length - vertexOffset), vertexOffset);
        } else {
            vbuf.set(vData, vertexOffset);
        }

        // 将准备好的iData拷贝到IndiceBuffer里
        let ibuf = buffer._iData,
            indiceOffset = offsetInfo.indiceOffset,
            vertexId = offsetInfo.vertexOffset;
        for (let i = 0, l = iData.length; i < l; i++) {
            ibuf[indiceOffset++] = vertexId + iData[i];
        }
    }
```

这里为什么要需要准备顶点数据，而不是在 `fillBuffers()` 方法内直接计算后填入buffer呢?
因为 `fillBuffers()` 每帧都会被调用，是热点代码，需要关注效率。但是顶点数据不是每一帧都会更新，可以预先计算。

```
    getBuffer () {
        return cc.renderer._handle._meshBuffer;
    }
```

```
// cocos2d/core/renderer/index.js
this._handle = new ModelBatcher(this.device, this.scene);
```

这里 getBuffer 获取的是 ModelBatcher 的 _meshBuffer，也就是说数据被填充到了 _meshBuffer。
_meshBuffer 保存着所有顶点数据，包括不同的纹理。

### 顶点数据

这里介绍一下 RenderData 的顶点数据。
最常用的顶点格式是 vfmtPosUvColor ，也是Assembler2D默认使用的格式。
代码在 cocos2d/core/renderer/webgl/vertex-format.js

```
var vfmtPosUvColor = new gfx.VertexFormat([
    // 节点的世界坐标，占2个float32
    { name: gfx.ATTR_POSITION, type: gfx.ATTR_TYPE_FLOAT32, num: 2 },
    // 节点的纹理uv坐标，占2个float32
    { name: gfx.ATTR_UV0, type: gfx.ATTR_TYPE_FLOAT32, num: 2 },
    // 节点颜色值，cc.Sprite组件上可以设置。占4个uint8 = 1个float32
    { name: gfx.ATTR_COLOR, type: gfx.ATTR_TYPE_UINT8, num: 4, normalize: true },
]);
```

看下Assembler2D里的属性和顶点格式的对应关系：

```
cc.js.addon(Assembler2D.prototype, {
    // vfmtPosUvColor 结构占5个float32
    floatsPerVert: 5,
    // 一个四边形4个顶点
    verticesCount: 4,
    // 一个四边形按照对角拆分成2个三角形，2*3 = 6个顶点索引
    indicesCount: 6,
    // uv的值在vfmtPosUvColor结构里下标从2开始算
    uvOffset: 2,
    // color的值在vfmtPosUvColor结构里下标从4开始算
    colorOffset: 4,
});
```

了解了上面的顶点格式之后，顶点数据无非就是计算 pos、uv、color几个值。
在Assembler里分别有 updateVerts()、 updateUVs()、 updateColor() 方法来准备这几个值，并且临时存储在Assembler自己分配的数组里。

```
export default class Assembler2D extends Assembler {
    constructor () {
        super();

      	// renderData.vDatas用来存储pos、uv、color数据
        // renderData.iDatas用来存储顶点索引数据
        this._renderData = new RenderData();
        this._renderData.init(this);
        
        this.initData();
        this.initLocal();
    }

    get verticesFloats () {
      	// 当前节点的所有顶点数据总大小
        return this.verticesCount * this.floatsPerVert;
    }

    initData () {
        let data = this._renderData;
      	// 创建一个足够长的空间用来存储顶点数据 & 顶点索引数据
      	// 这个方法内部会初始化顶点索引数据
        data.createQuadData(0, this.verticesFloats, this.indicesCount);
    }
    
    updateUVs (sprite) {
      	// 获取当前cc.Sprite组件设置的spriteFrame对应的uv
      	// uv数组长度=8，分别表示4个顶点的uv.x, uv.y
      	// 按照左下、右下、左上、右上的顺序存储，注意这里的顺序和顶点索引的数据需要对应上
        let uv = sprite._spriteFrame.uv;
        let uvOffset = this.uvOffset;		// 之前提到过vfmtPosUvColor结构里uvOffset = 2
        let floatsPerVert = this.floatsPerVert; // floatsPerVert = vfmtPosUvColor结构大小 = 5
        let verts = this._renderData.vDatas[0];
        for (let i = 0; i < 4; i++) {
            // 2个1组取uv数据，写入renderData.vDatas对应位置
            let srcOffset = i * 2;
            let dstOffset = floatsPerVert * i + uvOffset;
            verts[dstOffset] = uv[srcOffset];
            verts[dstOffset + 1] = uv[srcOffset + 1];
        }
    }
}
```

### 顶点索引

除了pos、uv、color数据之外，为什么还需要计算顶点索引数据？
我们发送给GPU的数据，实际上表示的是三角形，而不是四边形。一个四边形需要剖分成2个三角形然后传给GPU。
在4个顶点数据的基础上，三角形的描述信息单独存在 IndiceBuffer (即renderData.iDatas)里，IndiceBuffer里的每个值表示其对应顶点数据的下标。通过索引可以合并掉多个三角形中相同的顶点数据，减少总数据大小。

常规四边形的索引数据准备，源码位置：cocos2d/core/renderer/webgl/render-data.js。

```
    initQuadIndices(indices) {
      	// 按照上述剖分方式得到的下标: [0,1,2] [1,3,2]
        // 6个一组（对应1个四边形）生成索引数据
        let count = indices.length / 6;
        for (let i = 0, idx = 0; i < count; i++) {
            let vertextID = i * 4;
            indices[idx++] = vertextID;
            indices[idx++] = vertextID+1;
            indices[idx++] = vertextID+2;
            indices[idx++] = vertextID+1;
            indices[idx++] = vertextID+3;
            indices[idx++] = vertextID+2;
        }
    }
```

## RenderFlow

源码在cocos2d/core/renderer/render-flow.js
可以参考 cocos 文档关于渲染流的介绍：https://docs.cocos.com/creator/manual/zh/advanced-topics/render-flow.html。
RenderFlow 即为渲染流，用来遍历场景下面的所有节点，根据每个节点的_renderFlag，处理节点的位置，颜色，透明度，更新并渲染。
在 2.X 的版本中，RenderFlow 会把渲染过程划分为多个渲染状态，比如Transform，Render，Children 等，每个渲染状态都对应了一个函数。在 RenderFlow 初始化的时候，会预先根据渲染状态创建好对应的渲染分支，这些分支会把对应的渲染状态链接在一起。
我们可以把它理解为一个函数链表。一个节点要被渲染前，会更新该节点的_renderFlag，然后会根据它的 _renderFlag 进行响应的分支处理。
比如：当节点上的位置、颜色、透明度变化之后，就需要同步修改 _renderFlag，这样在渲染时就会根据 _renderFlag 处理对应的流程。

```
RenderFlow.render = function (scene, dt) {
    _batcher.reset();
    _batcher.walking = true;
    // 遍历渲染场景节点的所有子节点
    RenderFlow.visitRootNode(scene);

    _batcher.terminate();
    _batcher.walking = false;
    // 将batcher中的数据渲染到屏幕
    _forward.render(_batcher._renderScene, dt);
};
```

在 RenderFlow.visitRootNode 方法内部，会开始执行 RenderFlow 的渲染流程，根据 _renderFlag 来执行相对应的渲染阶段。

_renderFlag 有下面几种：

```
const DONOTHING = 1 << FlagOfset++;
const BREAK_FLOW = 1 << FlagOfset++;
const LOCAL_TRANSFORM = 1 << FlagOfset++;
const WORLD_TRANSFORM = 1 << FlagOfset++;
const TRANSFORM = LOCAL_TRANSFORM | WORLD_TRANSFORM;
const UPDATE_RENDER_DATA = 1 << FlagOfset++;
const OPACITY = 1 << FlagOfset++;
const COLOR = 1 << FlagOfset++;
const OPACITY_COLOR = OPACITY | COLOR;
const RENDER = 1 << FlagOfset++;
const CHILDREN = 1 << FlagOfset++;
const POST_RENDER = 1 << FlagOfset++;
const FINAL = 1 << FlagOfset++;
```

通过 createFlow 绑定 _renderFlag 和对应的执行方法：

```
function createFlow (flag, next) {
    let flow = new RenderFlow();
    flow._next = next || EMPTY_FLOW;

    switch (flag) {
        case DONOTHING: 
            flow._func = flow._doNothing;
            break;
        case BREAK_FLOW:
            flow._func = flow._doNothing;
            break;
        case LOCAL_TRANSFORM: 
            flow._func = flow._localTransform;
            break;
        case WORLD_TRANSFORM: 
            flow._func = flow._worldTransform;
            break;
        case OPACITY:
            flow._func = flow._opacity;
            break;
        case COLOR:
            flow._func = flow._color;
            break;
        case UPDATE_RENDER_DATA:
            console.log('Test UPDATE_RENDER_DATA')
            flow._func = flow._updateRenderData;
            break;
        case RENDER: 
            flow._func = flow._render;
            break;
        case CHILDREN: 
            flow._func = flow._children;
            break;
        case POST_RENDER: 
            flow._func = flow._postRender;
            break;
    }

    return flow;
}
```

```
// 空方法，通常是渲染流的最后一个方法
_proto._doNothing = function () {
};
// 更新本地矩阵。节点位置通过本地坐标和世界坐标矩阵管理，
// 通过矩阵叉乘进行高效的矩阵变换
_proto._localTransform = function (node) {
    node._updateLocalMatrix();
    node._renderFlag &= ~LOCAL_TRANSFORM;
    this._next._func(node);
};
// 更新世界坐标矩阵
_proto._worldTransform = function (node) {
    _batcher.worldMatDirty ++;

    let t = node._matrix;
    let trs = node._trs;
    let tm = t.m;
    tm[12] = trs[0];
    tm[13] = trs[1];
    tm[14] = trs[2];

    node._mulMat(node._worldMatrix, node._parent._worldMatrix, t);
    node._renderFlag &= ~WORLD_TRANSFORM;
    this._next._func(node);

    _batcher.worldMatDirty --;
};
// 处理透明度
_proto._opacity = function (node) {
    _batcher.parentOpacityDirty++;

    node._renderFlag &= ~OPACITY;
    this._next._func(node);

    _batcher.parentOpacityDirty--;
};
// 更新颜色
_proto._color = function (node) {
    let comp = node._renderComponent;
    if (comp) {
        comp._updateColor();
    }

    node._renderFlag &= ~COLOR;
    this._next._func(node);
};
// 更新渲染数据
_proto._updateRenderData = function (node) {
    console.log('Test _updateRenderData')
    let comp = node._renderComponent;
    comp._assembler.updateRenderData(comp);
    node._renderFlag &= ~UPDATE_RENDER_DATA;
    this._next._func(node);
};
// 这个函数会调用 RenderComponent._checkBatch()，
// 此处会检查当前节点和上一个节点是否能够合批，
// 如果无法合批，那么此处会打断合批一次。
_proto._render = function (node) {
    let comp = node._renderComponent;
    // 检测合批
    comp._checkBacth(_batcher, node._cullingMask);
    // 填充数据
    comp._assembler.fillBuffers(comp, _batcher);
    this._next._func(node);
};

//这个函数里会遍历当前节点的子节点，这一过程递归进行，
// 相当于对层级树进行了 深度优先遍历。
// 由于父子节点之间无法合批，所以深度优先遍历会频繁打断合批。
// 如果能将 RENDER函数按 广度优先遍历执行就可以减少合批被打断次数。
_proto._children = function (node) {
    let cullingMask = _cullingMask;
    let batcher = _batcher;

    let parentOpacity = batcher.parentOpacity;
    let opacity = (batcher.parentOpacity *= (node._opacity / 255));

    let worldTransformFlag = batcher.worldMatDirty ? WORLD_TRANSFORM : 0;
    let worldOpacityFlag = batcher.parentOpacityDirty ? OPACITY_COLOR : 0;
    let worldDirtyFlag = worldTransformFlag | worldOpacityFlag;

    let children = node._children;
    for (let i = 0, l = children.length; i < l; i++) {
        let c = children[i];

        // Advance the modification of the flag to avoid node attribute modification is invalid when opacity === 0.
        c._renderFlag |= worldDirtyFlag;
        if (!c._activeInHierarchy || c._opacity === 0) continue;

        _cullingMask = c._cullingMask = c.groupIndex === 0 ? cullingMask : 1 << c.groupIndex;

        // TODO: Maybe has better way to implement cascade opacity
        let colorVal = c._color._val;
        c._color._fastSetA(c._opacity * opacity);
        flows[c._renderFlag]._func(c);
        c._color._val = colorVal;
    }

    batcher.parentOpacity = parentOpacity;

    this._next._func(node);
};

_proto._postRender = function (node) {
    let comp = node._renderComponent;
    comp._checkBacth(_batcher, node._cullingMask);
    comp._assembler.postFillBuffers(comp, _batcher);
    this._next._func(node);
};
```

来简单看一下一个图片的 render flow，这只是渲染图片时的一种渲染组合而已，具体情况不同，渲染组合也是不同的：

```
├── RenderFlow.init
    ├── _localTransform
        ├── _opacity
            ├── _color
                ├── _children
                    ├── init // 进入子节点
                        ├── _localTransform
                            ├── _worldTransform
                                ├── _opacity
                                    ├── _color
                                        ├── _children
                                            ├── _localTransform
                                                ├── _worldTransform
                                                    ├── _updateRenderData
                                                        ├── SimpleSpriteAssembler.updateRenderData
                                                            ├── updateUVs
                                                            ├── updateVerts
                                                        ├── _opacity
                                                            ├── _color
                                                                ├── _render
                                                                    ├── _checkBacth
                                                                    ├── _assembler.fillBuffers
                                                                    ├── _doNothing
                                        
```

## ModelBatcher

代码在 cocos2d/core/renderer/webgl/model-batcher.js。
用来管理渲染数据的 model，渲染批次合并，从而减少drawcall，提升新能。

在 RendererFlow 的 _render 方法中，会调用 CCRenderComponent 的 _checkBacth 方法：

```
    _checkBacth (renderer, cullingMask) {
        let material = this._materials[0];
        if ((material && material.getHash() !== renderer.material.getHash()) || 
            renderer.cullingMask !== cullingMask) {
            renderer._flush();
    
            renderer.node = material.getDefine('CC_USE_MODEL') ? this.node : renderer._dummyNode;
            renderer.material = material;
            renderer.cullingMask = cullingMask;
        }
    }
```

如果发现两个材质不同，就调用 ModelBatcher 的 _flush 方法创建一个新的渲染批次数据Model。

```
    _flush () {
        let material = this.material,
            buffer = this._buffer,
            indiceCount = buffer.indiceOffset - buffer.indiceStart;
        if (!this.walking || !material || indiceCount <= 0) {
            return;
        }

        let effect = material.effect;
        if (!effect) return;
        
        // Generate ia
        let ia = this._iaPool.add();
        ia._vertexBuffer = buffer._vb;
        ia._indexBuffer = buffer._ib;
        ia._start = buffer.indiceStart;
        ia._count = indiceCount;
        
        // Generate model
        let model = this._modelPool.add();
        this._batchedModels.push(model);
        model.sortKey = this._sortKey++;
        model._cullingMask = this.cullingMask;
        model.setNode(this.node);
        model.setEffect(effect);
        model.setInputAssembler(ia);
        //添加model
        this._renderScene.addModel(model);
        buffer.forwardIndiceStartToOffset();
    },
```

_flush 方法生成新的 Model 数据，并把数据加入到 _renderScene 中去，一个 Model 数据就是一个渲染批次，所以 Model 越多，DC（DrawCall）就越多。
在base-renderer.js 的 _render 方法中会获取 scene 中的 _models，提取数据转化为 drawItem：

```
  _render (view, scene) {
    ......
    for (let i = 0; i < scene._models.length; ++i) {
      let model = scene._models.data[i];

      // filter model by view
      if ((model._cullingMask & view._cullingMask) === 0) {
        continue;
      }

      let drawItem = this._drawItemsPools.add();
      model.extractDrawItem(drawItem);
    }
    ......
```

```
  extractDrawItem(out) {
    out.model = this;
    out.node = this._node;
    out.ia = this._inputAssembler;
    out.effect = this._effect;
  }
```

在 _checkBacth 合批完成后，就会调用 _assembler.fillBuffers 将 _renderData 数据到 ModelBatcher 的 _meshBuffer 。

```
// cocos2d/core/renderer/assembler-2d.js
    getBuffer () {
        return cc.renderer._handle._meshBuffer;
    }
```

```
// cocos2d/core/renderer/index.js
this._handle = new ModelBatcher(this.device, this.scene);
```

## ForwardRenderer

源码在 cocos2d/renderer/renderers/forward-renderer.js。
ForwardRender 继承自 Base，是和底层 gl 渲染接近的类。

 - _device：对应平台的绘制对象，调用 gl 接口进行渲染。
 - _programLib：管理 shader 定义、获取和检查等相关的变量。
 - _stage2fn：保存有不同渲染通道的名称与其对应的不同渲染方法。
 - _viewPools：单个相机的描述数据类（View）的对象池，一个View对应一个相机。
 - _drawItemsPools：渲染数据类的对象池，保存有每个渲染批次需要的model，effect等数据。
 - _stageItemsPools：单个渲染通道需要渲染的数据的对象池，对 _drawItemsPools 的数据按照不同的渲染通道进行了分类。

### 渲染通道

ForwardRenderer 意为前向渲染，泛指传统上只有opaque和transparent另个通道的渲染技术，cocos有三个渲染通道，
绑定 Stage 操作的相关方法：

```
export default class ForwardRenderer extends BaseRenderer {
  constructor(device, builtin) {
    ......

    this._registerStage('shadowcast', this._shadowStage.bind(this));
    this._registerStage('opaque', this._opaqueStage.bind(this));
    this._registerStage('transparent', this._transparentStage.bind(this));
  }
```

## Device

Device 源码在 cocos2d/renderer/gfx/device.js 。
Device 是和底层 OpenGL 渲染最接近的类，运行平台对应的绘制图形对象的实例，用于绘制图形到屏幕，内部的方法直接调用 _gl 的相关方法执行渲染任务。

Device 的构造方法中构造 gl 实例，默认采用 webgl 进行绘制：

```
  constructor(canvasEL, opts) {
    let gl;

    ......
    try {
      gl = canvasEL.getContext('webgl', opts)
        || canvasEL.getContext('experimental-webgl', opts)
        || canvasEL.getContext('webkit-3d', opts)
        || canvasEL.getContext('moz-webgl', opts);
    } catch (err) {
      console.error(err);
      return;
    }
```

来看一下Device 的 draw 方法：

```
// engien cocos2d/renderer/gfx/device.js

  draw(base, count) {
    console.log("TestRender engine Device.js draw")
    const gl = this._gl;
    let cur = this._current;
    let next = this._next;

    // commit blend
    _commitBlendStates(gl, cur, next);

    // commit depth
    _commitDepthStates(gl, cur, next);

    // commit stencil
    _commitStencilStates(gl, cur, next);

    // commit cull
    _commitCullMode(gl, cur, next);

    // commit vertex-buffer
    _commitVertexBuffers(this, gl, cur, next);

    // commit index-buffer
    if (cur.indexBuffer !== next.indexBuffer) {
      gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, next.indexBuffer && next.indexBuffer._glID !== -1 ? next.indexBuffer._glID : null);
    }

    // commit program
    let programDirty = false;
    if (cur.program !== next.program) {
      if (next.program._linked) {
        gl.useProgram(next.program._glID);
      } else {
        console.warn('Failed to use program: has not linked yet.');
      }
      programDirty = true;
    }

    // commit texture/sampler
    _commitTextures(gl, cur, next);

    // commit uniforms
    for (let i = 0; i < next.program._uniforms.length; ++i) {
      let uniformInfo = next.program._uniforms[i];
      let uniform = this._uniforms[uniformInfo.name];
      if (!uniform) {
        // console.warn(`Can not find uniform ${uniformInfo.name}`);
        continue;
      }

      if (!programDirty && !uniform.dirty) {
        continue;
      }

      uniform.dirty = false;

      // TODO: please consider array uniform: uniformInfo.size > 0

      let commitFunc = (uniformInfo.size === undefined) ? _type2uniformCommit[uniformInfo.type] : _type2uniformArrayCommit[uniformInfo.type];
      if (!commitFunc) {
        console.warn(`Can not find commit function for uniform ${uniformInfo.name}`);
        continue;
      }

      commitFunc(gl, uniformInfo.location, uniform.value);
    }

    if (count) {
      // drawPrimitives
      if (next.indexBuffer) {
        gl.drawElements(
          this._next.primitiveType,
          count,
          next.indexBuffer._format,
          base * next.indexBuffer._bytesPerIndex
        );
      } else {
        gl.drawArrays(
          this._next.primitiveType,
          base,
          count
        );
      }

      // update stats
      this._stats.drawcalls++;
    }

    // reset states
    cur.set(next);
    next.reset();
  }
```



## 相关文章

https://www.jianshu.com/u/01450ce9ecbf
https://www.jianshu.com/p/8653006df65d