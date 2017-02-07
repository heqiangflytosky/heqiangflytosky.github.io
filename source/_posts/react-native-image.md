---
title: React Native-Image控件
categories: React Native
comments: true
tags: [React Native]
date: 2017-01-19 12:00:00
---
## 基本用法
### 加载本地图片

```
<Image source={require('./img/baidu.png')}/>
```
### 加载App内资源图片

```
<Image source={{uri: 'ic_launcher'}} style={{width: 140, height: 140}} />
```
### 加载网络图片

```
<Image source={{uri:'http://172.17.137.68/heqiang/2.jpg'}} style={{width: 200, height: 200}}/>
```
资源图片和网络图片必须声明图片寬高，否则不显示。
### 适配不同平台
有时我们希望在不同平台之间用不同的图片
比如baidu.android.png，baidu.ios.png，代码中只需要写baidu.png，便可以适配android和ios平台
baidu@2x.png，baidu@3x.png还可以适配不同分辨率的机型。如果没有图片恰好满足屏幕分辨率，则会自动选中最接近的一个图片。这点是和Android中是类似的。
### 代码

```
import React, { Component } from 'react';
import {
  AppRegistry,
  StyleSheet,
  Text,
  View,
  Image
} from 'react-native';

class HelloWorldAppp extends Component{
  render() {
    console.log("render()");
    return (
      <View>
        <Text style={styles.title_text}>本地图片</Text>
        <Image source={require('./img/baidu.png')}/>
        <Text style={styles.title_text}>资源图片</Text>
        <Image source={{uri: 'ic_launcher'}} style={{width: 140, height: 140}} />
        <Text style={styles.title_text}>网络图片</Text>
        <Image source={{uri:'http://*******.jpg'}} style={{width: 200, height: 200}}/>
    );
  }
}

AppRegistry.registerComponent('AwesomeProject', () => HelloWorldAppp);

const styles = StyleSheet.create({
  title_text:{
    fontSize:18,
  }

});
```

### 效果图
![效果图](/images/react-native-image/image1.png)

## 回调函数和属性

 - onLayout：layout时调用，与View组件的onLayout函数类似
 - onLoadStart：开始加载时调用
 - onLoadEnd加载结束时调用
 - onLoad：成功读取图片时调用
```
        <Image source={{uri:'http://172.17.137.68/heqiang/23.jpg'}} style={{width: 200, height: 200}} 
          onLoad={function(){console.log("onLoad");}}
          onLayout={function(){console.log("onLayout");}}
          onLoadStart={function(){console.log("onLoadStart");}}
          onLoadEnd={function(){console.log("onLoadEnd");}}
          />
```
 - resizeMode
  - cover：在显示比例不失真的情况下填充整个显示区域。可以对图片进行放大或者缩小，超出显示区域的部分不显示，也就是说，图片可能部分会显示不了。
  - contain：要求显示整张图片，可以对它进行等比缩小，图片会显示完整，可能会露出Image控件的底色。如果图片宽高都小于控件宽高，则不会对图片进行放大。
  - stretch：不考虑保持图片原来的宽高比，填充整个Image定义的显示区域，这种模式显示的图片可能会畸形和失真。
  - center：居中不缩放
 
 resizeMode也可以定义在style中，但在属性上定义的优先级比style中高。比如下面设置中最终生效的是Image.resizeMode.center。
```
        <Image source={{uri:'http://172.17.137.68/heqiang/test.png'}} 
          style={{width: 200, height: 200, backgroundColor: 'grey',resizeMode: Image.resizeMode.contain}} 
          resizeMode={Image.resizeMode.center}
          />
```
## 样式风格
 - FlexBox 支持弹性盒子风格
 - Transforms 支持属性动画
 - resizeMode 设置缩放模式
 - backgroundColor 背景颜色
 - borderColor 边框颜色
 - borderWidth 边框宽度
 - borderRadius 边框圆角
 - overflow 设置图片尺寸超过容器可以设置显示或者隐藏('visible','hidden')
 - tintColor 颜色设置
 - opacity 设置不透明度0.0(透明)-1.0(完全不透明)



