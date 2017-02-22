---
title: React Native-flexbox
categories: React Native
comments: true
tags: [React Native]
date: 2017-01-19 15:00:00
---
## 介绍flexbox布局
flexbox是React Native应用开发中必不可少的内容，也是最常用的内容。flexbox是由W3C在2009年提出的一种新的布局方案，该布局可以简单快速地完成各种伸缩性的设计。
flexbox是Flexible Box的缩写，即为弹性盒子布局。
flexbox布局由伸缩容器和伸缩项目组成。采用Flex布局的元素称为伸缩容器。伸缩容器的子元素成为伸缩项目，伸缩项目使用伸缩布局模型来排版。
<!-- more -->
### 伸缩容器的属性
伸缩容器支持的属性有：

 - display
 - flex-direction
 - flex-wrap
 - flex-flow
 - justify-content
 - align-items
 - align-content

#### display
该属性用来制定元素是否为伸缩容器，其语法为：
display:flex | inline-flex

 - flex：这个值用于产生块级伸缩容器。
 - inline-flex：这个值用于产生行内级伸缩容器。

#### flex-direction
该属性用于指定主轴方向，其语法为：
flex-direction: row | row-reverse | column | column-reverse

 - row: 默认值，如果为横向伸缩，则无需定义。该属性指定如果伸缩容器为水平方向轴，排版方式为从左向右排列。
 - row-reverse: 如果伸缩容器为水平方向轴，排版方式为从右向左排列。
 - column: 如果伸缩容器为垂直方向轴，排版方式为从上到下排列。
 - column-reverse: 如果伸缩容器为垂直方向轴，排版方式为从下向上排列。

#### flex-wrap
该属性用于指定伸缩容器的主轴线方向空间不足的情况下，是否换行以及如何换行，其语法为：
flex-wrap: nowrap | wrap | wrap-reverse

 - nowrap: 默认值，即使空间不足，伸缩容器也不允许换行。
 - wrap:允许换行。若主轴为水平轴，换行方向为从上到下。
 - wrap-reverse:允许换行。若主轴为水平轴，换行方向为从下向上。

#### flex-flow
该属性是flex-direction属性和flex-wrap属性的简写形式，默认值为row nowrap。

#### justify-content
该属性用来定义伸缩项目沿主轴线的对齐方式，其语法为：
justify-content: flex-start | flex-end | center | space-between | space-around

 - flex-start: 默认值，伸缩项目向主轴线的起始位置靠齐。
 - flex-end: 伸缩项目向主轴线的结束位置靠齐。
 - center: 伸缩项目向主轴线的中间位置靠齐。
 - space-between: 伸缩项目平均地分布在主轴线里。第一个伸缩项目在主轴线的开始位置，最后一个伸缩项目在主轴线的终点位置。
 - space-around: 伸缩项目平均地分布在主轴线里。两端会保留间距一半的空间。

#### align-items
该属性用来定义伸缩项目在伸缩容器的交叉轴上的对其方式，其语法为：
align-items: flex-start | flex-end | center | baseline | stretch

 - flex-start: 伸缩项目向交叉轴的起始位置靠齐。
 - flex-end: 伸缩项目向交叉轴的结束位置靠齐。
 - center: 伸缩项目向交叉轴的中间位置靠齐。
 - baseline： 伸缩项目根据他们的基线对齐。
 - stretch： 默认值，伸缩项目在交叉轴方向拉伸填充整个伸缩容器。

#### align-content
该属性用来调整伸缩项目出现换行后在交叉轴上的对齐方式，类似于伸缩项目在主轴上使用justify-content。其语法为：
align-content: flex-start | flex-end | center | space-between | space-around | stretch

 - flex-start：伸缩项目向交叉轴的起点对齐。
 - flex-end：伸缩项目向交叉轴的终点对齐。
 - center：伸缩项目向交叉轴的中间位置对齐。
 - space-between：伸缩项目向交叉轴两端对齐，轴线之间的间隔平均分布。
 - space-around：每根轴线两侧的间隔都相等。所以，轴线之间的间隔比轴线与边框的间隔大一倍。
 - stretch（默认值）：伸缩项目会在交叉轴上伸展以占用剩余的空间。

### 伸缩项目的属性
伸缩项目支持的属性有：

 - order
 - flex-grow
 - flex-shrink
 - flex-basis
 - flex
 - align-self

#### order
该属性用来定义项目的排列顺序。数值越小，排列越靠前，默认为0。其语法为：
order: <integer>

#### flex-grow
该属性定义项目的放大比例，默认为0，即如果存在剩余空间，也不放大。如果所有项目的flex-grow属性都为1，则它们将等分剩余空间（如果有的话）。如果一个项目的flex-grow属性为2，其他项目都为1，则前者占据的剩余空间将比其他项多一倍。其语法为：
flex-grow: <number> /*默认值为0*/

#### flex-shrink
该属性定义了项目的缩小比例，默认为1，即如果空间不足，该项目将缩小。
flex-shrink: <number>; /* default 1 */
如果所有项目的flex-shrink属性都为1，当空间不足时，都将等比例缩小。如果一个项目的flex-shrink属性为0，其他项目都为1，则空间不足时，前者不缩小。
负值对该属性无效。

#### flex-basis
该属性定义了在分配多余空间之前，项目占据的主轴空间（main size）。浏览器根据这个属性，计算主轴是否有多余空间。它的默认值为auto，即项目的本来大小。
该属性相当于设置了伸缩项目的一个基准值，剩余的空间按比率进行伸缩。其语法为：
flex-basis: <length> | auto; /* default auto */

#### flex
该属性是flex-grow, flex-shrink 和 flex-basis的简写，默认值为0 1 auto。后两个属性可选。其语法为：
flex: none | <'flex-grow'> <'flex-shrink'> <'flex-basis'>

#### align-self
该属性允许单个项目有与其他项目不一样的对齐方式，可覆盖align-items属性。默认值为auto，表示继承父元素的align-items属性，如果没有父元素，则等同于stretch。
align-self: auto | flex-start | flex-end | center | baseline | stretch

 - atuo: 伸缩项目按自身设置的寬高显示，如果没有设置，则按stretch来计算其值。
 - flex-start: 伸缩项目向交叉轴的起始位置靠齐。
 - flex-end: 伸缩项目向交叉轴的结束位置靠齐。
 - center: 伸缩项目向交叉轴的中间位置靠齐。
 - baseline： 伸缩项目根据他们的基线对齐。
 - stretch： 伸缩项目在交叉轴方向拉伸填充整个伸缩容器。

## 在React Native中的使用flexbox
React Native主要支持以下属性：

 - alignItems
 - alignSelf
 - flex
 - flexDirection
 - flexWrap
 - justifyContent

这些属性的用法和语法和上面介绍的是一样的，区别在于需要用驼峰拼写法。

## 实例

