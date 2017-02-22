---
title: React Native-组件Component 
categories: React Native
comments: true
tags: [React Native]
date: 2017-01-19 10:00:00
---
## 创建一个Component
一个组件类可以像前面Hello World工程中那样通过 `class HelloWorldAppp extends Component` 来创建，或者通过`React.createClass`来创建，并且提供一个render方法以及其他可选的生命周期函数、组件相关的事件或方法定义。
<!-- more -->
因此，HelloWorldAppp和下面的实现方法是等价的:
```
var HelloWorldAppp = React.createClass({
  render() {
    return (
      <View

      </View>
    );
  },
});
```
通过继承`Component`实现的组件中如果实现`getDefaultProps` `getInitialState`等方法时，会有下面警告：
```
Warning: getDefaultProps was defined on HelloWorldAppp, a plain JavaScript class. This is only supported for classes created using React.createClass. Use a static property to define defaultProps instead.
```
## React组件生命周期
先来看一段代码：

```
import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View,
  Image,
  TouchableHighlight
} from 'react-native';

var clickTime = 0;
var HelloWorldAppp = React.createClass({
  getDefaultProps(){
    console.log("getDefaultProps")
    return {title:"HelloWorld"}
  },
  getInitialState(){
    console.log("getInitialState")
   return {content:"点击屏幕任意位置"}
  },
  componentWillMount(){
    console.log("componentWillMount")
  },
  componentDidMount(){
    console.log("componentDidMount")
  },
  shouldComponentUpdate(nextProps,nextState){
    console.log("shouldComponentUpdate")
    return true
  },
  componentWillUpdate(nextProps,nextState){
    console.log("componentWillUpdate")
  },
  componentDidUpdate(prevProps,prevState){
    console.log("componentDidUpdate")
  },
  render() {
    console.log("render")
    return (
      <TouchableHighlight
        onPress={() => this.backgorundClicked()}
        underlayColor = '#ddd'
        style = {styles.container}
        >
        <Text style={styles.welcome}>{clickTime > 0 ? this.state.content : this.props.title + " \n " + this.state.content}</Text>
      </TouchableHighlight>
    );
  },
  backgorundClicked(){
    clickTime++
    this.setState({
      content:"第"+clickTime+"次点击"
    });
  }
});

AppRegistry.registerComponent('AwesomeProject', () => HelloWorldAppp);

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
  title_text:{
    fontSize:18,
  }

});
```

然后认识一下组件中的一些函数：

 - getDefaultProps： 用来设置组件属性的默认值。通常会将固定的内容放在这个过程中进行初始化和赋值，一个控件可以利用`this.props`获取在这里初始化它的属性，由于组件初始化后，再次使用该组件不会调用`getDefaultProps`函数，所以组件自己不可以自己修改props（即：props可认为是只读的），只可由其他组件调用它时在外部修改。
`getDefaultProps`并不是在组件实例化时被调用，而是在`createClass`时被调用，返回值会被缓存。也就是说，不能在`getDefaultProps`中使用任何特定的实例数据。
 - getInitialState： 这里是对控件的一些状态进行初始化，由于该函数不同于`getDefaultProps`，在以后的过程中，会再次调用，所以可以将控制控件的状态的一些变量放在这里初始化，如控件上显示的文字，可以通过`this.state`来获取值，通过`this.setState`来修改state值，修改方式如下：
```
    this.setState({
      content:"第"+clickTime+"次点击"
    });
```

值得注意的是，一旦调用了`this.setState`方法，控件必将调用render方法，对控件进行再次的渲染，不过，React框架会自动根据DOM的状态来判断是否需要真正的渲染。

 - render：上面已经说过render是一个组件必须有的方法，形式为一个函数，并返回JSX或其他组件来构成DOM，和Android的XML布局、WPF的XAML布局类似，只能返回一个顶级元素。

### 生命周期函数

 - 装载组件
  - componentWillMount：这个方法被调用时期是组件将要被加载在视图上之前，功能比较少，即：render一个组件前最后一次修改state的机会。
  - componentDidMount：即调用了render方法后，组件加载成功并被成功渲染出来以后所执行的hook函数，一般会将网络请求等加载数据的操作，放在这个函数里进行，来保证不会出现UI上的错误.
 - 更新组件状态
存在期主要是用来处理与用户的交互，如：点击事件，都比较简单，也不是很常用，具体有以下几个过程：
  - componentWillReceiveProps：指父元素对组件的props或state进行了修改。
  - shouldComponentUpdate：一般用于优化，可以返回false或true来控制是否进行渲染
  - componentWillUpdate：组件刷新前调用，类似`componentWillMount`
  - componentDidUpdate：更新后的hook
 - 卸载（删除）组件
 销毁期，用于清理一些无用的内容，如：点击事件Listener。
  - componentWillUnmount

上面函数的调用顺序是：

 - 创建时
getDefaultProps
getInitialState
componentWillMount
render
componentDidMount
 - 更新时
shouldComponentUpdate
componentWillUpdate
render
componentDidUpdate

总得来讲，React Native组件的生命周期，经历了Mount->Update->Unmount这三个大的过程，即从创建到销毁的过程，如果借助Android和iOS的开发思想，那么React Native组件的生命周期就更容易理解了。那么，我们构建一个React Native控件也就能够知道如何下手，如何控制和优化。经过一层一层的封装和调用，一个完整的React Native应用也就构建出来了。

## Props（属性）和 State（状态）
Props 就是组件的属性，由外部通过 JSX 属性传入设置，一旦初始设置完成，就可以认为 `this.props` 是不可更改的，所以不要轻易更改设置 `this.props` 里面的值（虽然对于一个 JS 对象你可以做任何事）。
```
class MyText extends Component{
  render() {
    return(
      <Text style={this.props.style}>{this.props.content}</Text>
      );
  }
}

class HelloWorldAppp extends Component{
  render() {
    return (
      <MyText style={styles.welcome} content="HelloWorld"/>
    );
  }
}
```

State 是组件的当前状态，可以把组件简单看成一个“状态机”，根据状态 state 呈现不同的 UI 展示。一旦状态（数据）更改，组件就会自动调用 render 重新渲染 UI，这个更改的动作会通过 `this.setState` 方法来触发。
```
class HelloWorldAppp extends Component{
  constructor(props) {
    super(props);
    this.state = {showText:"Test"};
  }
  render() {
    return (
      <View>
          <Text style={styles.title_text}>{this.state.showText}</Text>
      </View>

    );
  }
}
```
## refs
ref是React中的一种属性，当render函数返回某个组件的实例时，可以给render中的某个虚拟DOM节点添加一个ref属性。要获取一个React组件的引用，可以使用ref来获取你拥有的子组件的引用。

```
var HelloWorldAppp = React.createClass({
    render() {
      console.log("render");
    return (
      <View style={styles.container}>
        <TextInput style={styles.welcome}  ref="mytestinput"></TextInput>

        <Text style={styles.welcome} onPress={this.blur}>清除焦点</Text>
        <Text style={styles.welcome} onPress={this.focus}>获取焦点</Text>
      </View>
    );
  },
  focus(){
    if(this.refs.mytestinput != null){
      this.refs.mytestinput.focus();
    }
  },
  blur(){
    if(this.refs.mytestinput != null){
      this.refs.mytestinput.blur();
    }
  }
});
```
在这个例子中，我们通过`this.refs.mytestinput`获取输入框实例，来实现获取焦点和清除焦点。

## 组件之间通信
通过上面的讲解我们可以知道：

 - 子组件调用父组件：可以通过`this.props`方法。
 - 父组件调用子组件：可以通过`ref`来实现。


