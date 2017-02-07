---
title: React Native 入门
categories: React Native
comments: true
tags: [React Native]
date: 2017-01-18 15:00:00
---
## Helle World代码分析

```
Helle World代码分析

import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View,
} from 'react-native';

export default class HelloWorldAppp extends Component {
  render() {
    return (
      <View style={styles.container}>
        <Text style={styles.welcome}>
          Welcome to React Native!
        </Text>
        <Text style={styles.instructions}>
          To get started, edit index.android.js
        </Text>
        <Text style={styles.instructions}>
          Double tap R on your keyboard to reload,{'\n'}
          Shake or press menu button for dev menu
        </Text>
      </View>
    );
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#F5FCFF',
  },
  welcome: {
    fontSize: 20,
    textAlign: 'center',
    margin: 10,
  },
  instructions: {
    textAlign: 'center',
    color: '#333333',
    marginBottom: 5,
  },
});

AppRegistry.registerComponent('AwesomeProject', () => HelloWorldAppp);
```
初看这段代码，可能觉得并不像JavaScript——没错，这是“未来”的JavaScript.
首先你需要了解ES2015 （也叫作ES6）——这是一套对JavaScript的语法改进的官方标准。但是这套标准目前还没有在所有的浏览器上完整实现，所以目前而言web开发中还很少使用。React Native内置了对ES2015标准的支持，你可以放心使用而无需担心兼容性问题。上面的示例代码中的`import`、`from`、`class`、`extends`、以及`() =>`箭头函数等新语法都是ES2015中的特性。如果你不熟悉ES2015的话，可以看看阮一峰老师的书，[ES 6 标准入门](http://es6.ruanyifeng.com/)。
示例中的这一行`<Text>*****</Text>`恐怕很多人看起来也觉得陌生。这叫做JSX——是一种在JavaScript中嵌入XML结构的语法。很多传统的应用框架会设计自有的模板语法，让你在结构标记中嵌入代码。React反其道而行之，设计的JSX语法却是让你在代码中嵌入结构标记。初看起来，这种写法很像web上的HTML，只不过使用的并不是web上常见的标签如`<div>`或是`<span>`等，这里我们使用的是React Native的组件。上面的示例代码中，使用的是内置的`<Text>`组件，它专门用来显示文本。
上面的代码定义了一个名为`HelloWorldAppp`的新的组件（Component），并且使用了名为`AppRegistry`的内置模块进行了“注册”操作。你在编写React Native应用时，肯定会写出很多新的组件。而一个App的最终界面，其实也就是各式各样的组件的组合。组件本身结构可以非常简单——唯一必须的就是在render方法中返回一些用于渲染结构的JSX语句。`HelloWorldAppp`组件的名称你可以随意定，但`AppRegistry.registerComponent('AwesomeProject', () => HelloWorldAppp);`第一个参数必须和工程名称一致。
注意，组件类的第一个字母必须大写，否则会报错。另外，组件类只能包含一个顶层标签，否则也会报错。
`render()`方法为必须有的方法，实现组件渲染功能，返回JSX或其他组件来构成DOM，和Android的XML布局、WPF的XAML布局类似，只能返回一个顶级元素。

## JSX
React的核心机制之一就是可以在内存中创建虚拟的DOM元素。React利用虚拟DOM来减少对实际DOM的操作从而提升性能。
JSX就是Javascript和HTML结合的一种格式。可以看作JavaScript的拓展，看起来有点像HTML。React发明了JSX，利用HTML语法来创建虚拟DOM。HTML语言直接写在JavaScript语言之中，不加任何引号，这就是JSX的语法，它允许 HTML 与 JavaScript 的混写。遇到HTML标签（以 < 开头），JSX就当HTML解析，遇到代码块（以 { 开头），就当JavaScript解析。最后，每个HTML标签都会转化为JavaScript代码来运行。
使用React，不一定非要使用JSX语法，可以使用原生的JS进行开发。但是React作者强烈建议我们使用JSX，因为JSX在定义类似HTML这种树形结构时，十分的简单明了。简明的代码结构更利于开发和维护。 XML有着开闭标签，在构建复杂的树形结构时，比函数调用和对象字面量更易读。
看个直接的对比，如下面两段代码是等价的：
JS：

```
var child1 = React.createElement('li', null, 'First Text Content');
var child2 = React.createElement('li', null, 'Second Text Content');
var root = React.createElement('ul', { className: 'my-list' }, child1, child2);
```
JSX：

```
var root =(
  <ul className="my-list">
    <li>First Text Content</li>
    <li>Second Text Content</li>
  </ul>
);
```
后者将XML语法直接加入JS中,通过代码而非模板来高效的定义界面。之后JSX通过翻译器转换为纯JS再由浏览器执行。在实际开发中，JSX在产品打包阶段都已经编译成纯JavaScript，JSX的语法不会带来任何性能影响。另外，由于JSX只是一种语法，因此JavaScript的关键字class, for等也不能出现在XML中，而要如例子中所示，使用className, htmlFor代替，这和原生DOM在JavaScript中的创建也是一致的。JSX只是创建虚拟DOM的一种语法格式而已,除了用JSX,我们也可以用JS代码来创建虚拟DOM。
### 样式使用
```
<View style={{width: 50, height: 50, flex: 1,backgroundColor: 'powderblue'}} />
```
上面第一个大括号是JSX语法，第二个大括号是JavaScript语法。
### JSX中绑定事件
JSX让事件直接绑定在元素上。

```
<button onClick={this.checkAndSubmit.bind(this)}>Submit</button>
```
和原生HTML定义事件的唯一区别就是JSX采用驼峰写法来描述事件名称，大括号中仍然是标准的JavaScript表达式，返回一个事件处理函数。
可以看到这里需要调用bind方法，这个方法第一个参数主要用来设置作用域，从第二个参数开始便是我们想要传递的参数了。

## 调试
代码中添加打印可以使用`console.log("")`方法，在logcat中过滤ReactNativeJS TAG。

