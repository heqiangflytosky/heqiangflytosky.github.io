---
title: Cocos Creator -- EditBox 实现
categories: cocos
comments: true
tags: [cocos]
description:  EditBox 实现原理
date: 2020-3-23 10:00:00
---

## 概述

本文以 EditBox 的实现为例来介绍一些组件的实现逻辑。

## engine 源码介绍

EditBox 的实现在 engine 的 CCEditBox.js

```
let EditBox = cc.Class({
    name: 'cc.EditBox',
    extends: cc.Component,
    ......
    statics: {
        _ImplClass: EditBoxImplBase,  // implemented on different platform adapter
        KeyboardReturnType: KeyboardReturnType,
        InputFlag: InputFlag,
        InputMode: InputMode
    },

    _init () {
        this._upgradeComp();

        this._isLabelVisible = true;
        this.node.on(cc.Node.EventType.SIZE_CHANGED, this._syncSize, this);

        let impl = this._impl = new EditBox._ImplClass();
        impl.init(this);

        this._updateString(this._string);
        this._syncSize();
    },
    ......
    focus () {
        if (this._impl) {
            this._impl.setFocus(true);
        }
    },
    ......
```

从代码可以看到，EditBox 的调用主要是通过 `_impl` 实现的，`_impl` 是 `EditBox._ImplClass` 的实例。
`EditBox._ImplClass` 在不同的平台有不同的实现，下面分别来进行介绍。

### H5平台

如果是 Web 模式：

```
// CCEditBox.js
if (cc.sys.isBrowser) {
    require('./WebEditBoxImpl');
}
```

```
//WebEditBoxImpl.js

EditBox._ImplClass = WebEditBoxImpl;
......  
    endEditing () {
        if (this._elem) {
            this._elem.blur();
        }
    },
    // implement dom input
    _createInput () {
        this._isTextArea = false;
        this._elem = document.createElement('input');
    },

    _createTextArea () {
        this._isTextArea = true;
        this._elem = document.createElement('textarea');
    },

```

editbox 的主要操作时通过 `_elem` 来实现，`_elem` 是通过获取 'input' 组件来实现。HtmlInputElement 是 H5 原生的实现。

### 原生平台

在 Android 原生平台，`EditBox._ImplClass` 的实例化在 jsb-adapter 的 jsb-editbox.js 中进行。

```
// jsb-editbox.js
    EditBox._ImplClass = JsbEditBoxImpl;
    ......
        endEditing () {
            jsb.inputBox.offConfirm();
            jsb.inputBox.offInput();
            jsb.inputBox.offComplete();
            this._editing = false;
            if (!cc.sys.isMobile) {
                this._delegate._showLabels();
            }
            jsb.inputBox.hide();
            this._delegate.editBoxEditingDidEnded();
        },
    ......
```

EditBox 的操作时通过 `jsb.inputBox` 实现的。

```
// jsb-input.js
jsb.inputBox = {
	onConfirm: function(cb) {
		var newCb = callbackWrapper(cb);
		eventTarget.addEventListener('confirm', newCb);
		recordCallback('confirm', newCb);
	},
	offConfirm: function(cb) {
		removeListener('confirm', cb);
	},

	onComplete: function(cb) {
		var newCb = callbackWrapper(cb);
		eventTarget.addEventListener('complete', newCb);
		recordCallback('complete', newCb);
	},
	offComplete: function(cb) {
		removeListener('complete', cb);
	},

	onInput: function(cb) {
		var newCb = callbackWrapper(cb);
		eventTarget.addEventListener('input', newCb);
		recordCallback('input', newCb);
	},
	offInput: function(cb) {
		removeListener('input', cb);
	},

    /**
     * @param {string}		options.defaultValue
     * @param {number}		options.maxLength
     * @param {bool}        options.multiple
     * @param {bool}        options.confirmHold
     * @param {string}      options.confirmType
     * @param {string}      options.inputType
     * 
     * Values of options.confirmType can be [done|next|search|go|send].
     * Values of options.inputType can be [text|email|number|phone|password].
     */
	show: function(options) {
		cc_jsb.showInputBox(options);
	},
	hide: function() {
		cc_jsb.hideInputBox();
	},
};

// 输入内容时的回调
jsb.onTextInput = function(eventName, text) {
	var event = new Event(eventName);
	event.text = text;
	eventTarget.dispatchEvent(event);
};
```

然后通过 jsb 调用原生的实现。

```
// jsb_global.cpp

static bool JSB_showInputBox(se::State &s) {
    const auto &args = s.args();
    size_t argc = args.size();
    CC_UNUSED bool ok = true;
    if (argc == 1) {
        bool ok;
        se::Value tmp;
        const auto &obj = args[0].toObject();

        cocos2d::EditBox::ShowInfo showInfo;

        if (obj->getProperty("defaultValue", &tmp)) {
            SE_PRECONDITION2(tmp.isString(), false, "defaultValue is invalid!");
            showInfo.defaultValue = tmp.toString();
        }
        if (obj->getProperty("maxLength", &tmp)) {
            SE_PRECONDITION2(tmp.isNumber(), false, "maxLength is invalid!");
            showInfo.maxLength = tmp.toInt32();
        }

        if (obj->getProperty("multiple", &tmp)) {
            SE_PRECONDITION2(tmp.isBoolean(), false, "multiple is invalid!");
            showInfo.isMultiline = tmp.toBoolean();
        }


        if (obj->getProperty("confirmHold", &tmp)) {
            SE_PRECONDITION2(tmp.isBoolean(), false, "confirmHold is invalid!");
            if (!tmp.isUndefined())
                showInfo.confirmHold = tmp.toBoolean();
        }


        if (obj->getProperty("confirmType", &tmp)) {
            SE_PRECONDITION2(tmp.isString(), false, "confirmType is invalid!");
            if (!tmp.isUndefined())
                showInfo.confirmType = tmp.toString();
        }

        if (obj->getProperty("inputType", &tmp)) {
            SE_PRECONDITION2(tmp.isString(), false, "inputType is invalid!");
            if (!tmp.isUndefined())
                showInfo.inputType = tmp.toString();
        }


        if (obj->getProperty("originX", &tmp)) {
            SE_PRECONDITION2(tmp.isNumber(), false, "originX is invalid!");
            if (!tmp.isUndefined())
                showInfo.x = tmp.toInt32();
        }

        if (obj->getProperty("originY", &tmp)) {
            SE_PRECONDITION2(tmp.isNumber(), false, "originY is invalid!");
            if (!tmp.isUndefined())
                showInfo.y = tmp.toInt32();
        }

        if (obj->getProperty("width", &tmp)) {
            SE_PRECONDITION2(tmp.isNumber(), false, "width is invalid!");
            if (!tmp.isUndefined())
                showInfo.width = tmp.toInt32();
        }

        if (obj->getProperty("height", &tmp)) {
            SE_PRECONDITION2(tmp.isNumber(), false, "height is invalid!");
            if (!tmp.isUndefined())
                showInfo.height = tmp.toInt32();
        }

        EditBox::show(showInfo);

        return true;
    }
```

c++ 通过 jni 调用 Android 端来实现。

```
// EditBox-android.cpp
void EditBox::show(const cocos2d::EditBox::ShowInfo& showInfo)
{
    JniHelper::callStaticVoidMethod(JCLS_EDITBOX,
                                    "showNative",
                                    showInfo.defaultValue,
                                    showInfo.maxLength,
                                    showInfo.isMultiline,
                                    showInfo.confirmHold,
                                    showInfo.confirmType,
                                    showInfo.inputType);
}

void EditBox::hide()
{
    JniHelper::callStaticVoidMethod(JCLS_EDITBOX, "hideNative");
}

    se::Value textInputCallback;

    void getTextInputCallback()
    {
        if (!textInputCallback.isUndefined())
            return;

        auto global = se::ScriptEngine::getInstance()->getGlobalObject();
        se::Value jsbVal;
      //weichaohui edit jsb to mz_jsb
        if (global->getProperty("mz_jsb", &jsbVal) && jsbVal.isObject() &&  jsbVal.toObject()) //add by daixw for crash #923278
        {
            jsbVal.toObject()->getProperty("onTextInput", &textInputCallback);
            // free globle se::Value before ScriptEngine clean up
            se::ScriptEngine::getInstance()->addBeforeCleanupHook([](){
                textInputCallback.setUndefined();
            });
        }
    }

    void callJSFunc(const std::string& type, const jstring& text)
    {
        getTextInputCallback();

        se::AutoHandleScope scope;
        se::ValueArray args;
        args.push_back(se::Value(type));
        args.push_back(se::Value(cocos2d::JniHelper::jstring2string(text)));
        textInputCallback.toObject()->call(args, nullptr);
    }
```

Cocos2dxEditBox 在 Android 端是继承自 EditText。