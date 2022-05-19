---
title: Vue -- 封装组件及通信
categories: JavaScript
comments: true
tags: [Vue]
description: 介绍一下Vue组件的封装和通信
date: 2019-1-12 10:00:00
---

## 概述

组件是可以复用的Vue实例，在我们开发过程中我们可以把经常重复使用的功能封装成为组件，打开快捷开发的目的。

## 组件封装

### 全局注册

使用 `Vue.component` 函数创建一个组件。

main.js

```
Vue.component('button-counter', {
  data: function () {
    return {
      count: 0
    }
  },
  template: '<div><button v-on:click="count++">You clicked me {{ count }} times.</button><div><div>tttt</div><slot>default slot</slot></div</div>'
})
```

使用：

```
      <button-counter>自定义组件</button-counter>
```

这种全局注册组件在一些大型的项目中会带来诸多的不便，下面将会介绍模块系统来构建单文件组件。

### 局部注册、模块系统与单文件组件

components/Custom.vue

```
<template>
  <div>
    <div>{{name}}</div>
  </div>
</template>
<style>
</style>
<script>
export default {
  props: ['name']
}
</script>
```

使用：

```
<template>
  <div class="hello">
    <custom :name="msg" ></custom>
  </div>
</template>

<script>
import custom from "@/components/Custom";
export default {
  name: "HelloWorld",
  components: {
    custom
  },
  data () {
    return {
      msg: "Welcome to Your Vue.js App"
    };
  },
  methods: {

  }
}
</script>

<style scoped>

</style>

```

## 组件通信

有时为了做到解耦，一些组件只负责数据的展示，而不负责数据的生产，这就涉及到组件之间的通信和数据传递问题，那么接下来就介绍集中组件之间通信的方法。

### `props`/`$emit` 实现

#### 父组件和子组件通信

像前面例子中展示那样父组件可以通过 props 属性来将数据传递给子组件。
props 属性可以通过下面的方式声明：
第一种方式：

```
  props: ['name'],
```

第二种方式：

```
  props: {
    name: String // 这里指定了字符串类型，如果类型不一致会警告
  },
```

第三种方式：

```
  props: {
    name: {
      type: String,
      default: 'John'
    }
  },
```

props 传入参数，不建议对它进行操作，如果要操作，请先在子组件深拷贝。
在 vue 中 props 遵循的是单向数据流，因为数据流向的单一才能保证数据变化的可追踪性，原则上子组件修改 props 是不被允许的。
但是当 props 传递的参数为对象或者数组的时候，是通过引用传入的，所以对于一个引用类型的 prop 来说，在子组件中改变这个参数本身将会影响到父组件的数据状态。这就会导致父组件的 data 混乱，而且难以捕捉。
对于这种情况可以这样处理：在子组件中声明新变量，然后把 prop 深拷贝赋值给新变量，之后子组件就使用新变量。
但是这种情况下父组件改变参数时，子组件无法更新参数，需要时可以通过watch或者computed来实现实时更新。

#### 子组件和父组件通信

可以通过事件的形式来是实现子组件向父组件传递消息：
子组件模拟一个按钮点击时向父组件传递消息

```
// 子组件

<template>
  <div>
    ...
    <button v-on:click="click">点我</button>
  </div>
</template>

<script>
export default {
  ......
  methods: {
    click () {
      this.$emit('changeText', 'from custom')
    }
  }
}
</script>
```

父组件接收消息：

```
// 父组件
<template>
  <div class="hello">
    ......
    <div >{{text}}</div>
    <custom :name="msg" @changeText='change'></custom>
  </div>
</template>
export default {
  ......
  methods: {
    change (msg) {
      this.text = msg
    }
  }
}
</script>
```

### `$emit`/`$on` 实现

通过一个 Vue 实例作为中央事件总线（事件中心），用它来触发事件和监听事件，可以实现了任何组件间的通信，包括父子、兄弟、跨级。

```


//vue原型链挂载总线
Vue.prototype.$myEvent = new Vue();
//子组件发送数据
this.$myEvent.$emit("change",data);
//子组件接收数据
this.$myEvent.$on("change",function(data){
})

```

```
// 注册总线
import Vue from 'vue'
Vue.prototype.$myEvent = new Vue()
```

在定义的全局变量前面加$，用以区分全局变量和组件内部局部变量，也防止命名冲突。

```
// A 组件 发送消息

<template>
  <div>
    <button v-on:click="click">TestA点我</button>
  </div>
</template>

<script>
export default {
  methods: {
    click () {
      this.$myEvent.$emit("changeText", "from A")
    }
  }
};
</script>
```

```
// B 组件 接收消息

<template>
  <div>
    <div>{{test}}</div>
  </div>
</template>

<script>
export default {
  data () {
    return {
      test: 'test emit'
    }
  },
  created() {
    var _this = this
    this.$myEvent.$on("changeText", function(msg) {
      console.log(msg)
      _this.test = msg
    })
  }
};
</script>
```

### sync 修饰符

[sync 文档](https://cn.vuejs.org/v2/guide/components-custom-events.html#sync-%E4%BF%AE%E9%A5%B0%E7%AC%A6)
props是单向数据流 一般子组件是不能改变props的值的，update:myPropName模式却可以实现props的“双向绑定”。Vue 在2.3.0版本以后新增了sync语法糖。
sync 只是作为一个编译时的语法糖存在。它会被扩展为一个自动更新父组件属性的 v-on 监听器。

```
    <testA ref="testA" :visible.sync="testBool">
      <span slot="a">Hello</span>
      <span slot="b">Hello b</span>
    </testA>
```

会被扩展为：

```
    <testA ref="testA" :visible="testBool" @update:visible="val => testBool = val">
      <span slot="a">Hello</span>
      <span slot="b">Hello b</span>
    </testA>
```

当子组件需要更新 visible 的值时，它需要显式地触发一个更新事件：

```
this.$emit('update:visible', false);
```

sync 提供了一种子组件与父组件沟通的思路，如果我们不用.sync，我们想做上面的功能，我们也可以props传初始值，然后事件监听，也是同样可以实现的，但是使用 sync 会更加简洁高效。

### slot 的应用

有时我们需要在不同的场景为组件配置不同的展示，那么我们也可以用插槽来解决。
在子组件位置留一个 slot 出来：

```
// 封装 testA 组件
<template>
  <div>
    <button v-on:click="click"><slot></slot></button>
  </div>
</template>
```

然后在父组件中写入：

```
<testA>Hello 点我</testA>
```

顺便了解一下具名插槽的使用：

```
// 封装 testA 组件
<template>
  <div>
    <button v-on:click="click"><slot name='a'>a</slot></button>
    <div><slot name='b'>b</slot></div>
  </div>
</template>
```

```
    <testA> <span  slot="a">Hello</span><span  slot="b">Hello b</span></testA>
```

### refs 的应用

可以通过 `$refs` 获取组件实例的方式进行组件之间的通信，可以用于父子组件通信以及兄弟组件之间的通信。

```
// testA 组件
<template>
  <div v-show="visible">
    <button v-on:click="click"><slot name='a'>a</slot></button>
    <div><slot name='b'> b</slot></div>
  </div>
</template>
......
export default {
  data () {
    return {
      visible: true,
    }
  },
......
  methods: {
    .......
    testMethod () {
      this.visible = !this.visible
      console.log("testMethod "+this.visible)
    }
  }
};
```

```
// 应用
    <div>
      <button v-on:click="$refs.testA.testMethod()">点我</button>
    </div>
    
    <testA ref="testA">
      <span slot="a">Hello</span>
      <span slot="b">Hello b</span>
    </testA>
```

或者直接调用visible属性：

```
    <div>
      <button v-on:click="$refs.testA.visible=!$refs.testA.visible">点我</button>
    </div>
```

## 推荐文章

[组件基础--Vue官方文档](https://cn.vuejs.org/v2/guide/components.html)