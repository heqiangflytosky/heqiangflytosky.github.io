---
title: JavaScript 基础 -- export 和 import
categories: JavaScript
comments: true
tags: [JavaScript]
description: 介绍import和export的使用
date: 2018-5-15 10:00:00
---

[MDN export 文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/export)
[MDN import 文档](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/import)

## export

export 语句用于从模块中导出函数、对象或原始值，以便其他程序可以通过 import 语句使用它们。

### 命名导出

```
// test.js
function print() {
    console.log('print function')
}

var person = {
    name: 'James',
    say: function() {
        console.log('I am James')
    }
}

const URL = "http://www.heqiangfly.com"

export {print, person,URL}
```

命名导出可以导出多个值：
或者：

```
// test.js
export function print() {
    console.log('print function')
}

export var person = {
    name: 'James',
    say: function() {
        console.log('I am James')
    }
}

export const URL = "http://www.heqiangfly.com"
```

导入方式：

```
// main.js
import {print, person,URL} from './test'

      print()
      person.say()
      console.log(URL)

```

命名导出对导出多个值很有用。在导入期间，必须使用相应对象的相同名称。

### 默认导出

```
function print() {
    console.log('print function')
}

var person = {
    name: 'James',
    say: function() {
        console.log('I am James')
    }
}

const URL = "http://www.heqiangfly.com"

export default {print, person,URL}
```

导出：

```
import api from './test'

      api.print()
      api.person.say()
      console.log(api.URL)
```

可以使用任何名称导入默认导出，而且不需要用大括号括起来，但是，只能有一个默认导出。

```
export default function print() {
    console.log('print function')
}
```

```
import api from './test'

api()
```

注意，不能使用 var、let 或 const 用于导出默认值 export default。

### 两种方式同时使用

```
function print() {
    console.log('print function')
}
const title = "A"

const url = "http://www.heqiangfly.com"
const name = "James"

export {url, name}
export default{title,print}
```

导出：

```
import api, {url, name} from './test'

      console.log(api.title)
      api.print()

      console.log(url)
      console.log(name)
```

### 重命名

我们还可以在导出模块时进行重命名：

```
const url = "http://www.heqiangfly.com"
const name = "James"

export {url as webSite, name}
```

导入：

```
import {webSite, name} from './test'

      console.log(webSite)
      console.log(name)
```

## import

### 导入默认值

导入由默认导出（export daulft）方式导出的值，具体例子可看前面的默认导出一节

### 导入单个导出

`import {myExport} from '/modules/my-module.js';`
下面通过命名导出三个值：

```
// test.js
function print() {
    console.log('print function')
}

var person = {
    name: 'James',
    say: function() {
        console.log('I am James')
    }
}

const URL = "http://www.heqiangfly.com"

export {print, person,URL}
```

如果我们不需要全部导入，可以这样做：

```
import {URL} from './test'

  console.log(URL)

```

### 导入整个模块的内容

使用 `import * as name from "module-name"` 可以将 module-name.js 文件中导出的所有模块插入当前作用域。
针对上面的test.js代码，可以使用下面的导入方法：

```
  import * as api from './test'
  
      console.log(api.URL)
      api.print()
```

### 导入多个导出

`import {foo, bar} from '/modules/my-module.js';`
在 emport 章节的命名导出一节中已经有所介绍。

### 重命名导入

`import {reallyReallyLongModuleExportName as shortName} from '/modules/my-module.js';`
导入时可以重命名导出，可以解决导出命名冲突的问题。

```
import {person, URL as webSite} from './test'

    console.log(webSite)
```

### 导入代码

这将运行模块中的全局代码, 但实际上不导入任何值。

```
import '/modules/my-module.js';
```

### 动态导入

可以将关键字 import 称作动态导入模块的函数，当使用它的时候，会返回一个 promise。

```
import('/modules/my-module.js')
  .then((module) => {
    // Do something with the module.
  });
```

这种使用方式也支持 await 关键字。

```
let module = await import('/modules/my-module.js');
```
