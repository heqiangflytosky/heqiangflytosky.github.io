---
title: Hexo--Next主题优化
date: 2016-01-12 10:00:00
tags: [Hexo, Next Theme]
categories: Hexo
comments: true
---
按照上篇博客中更换Theme的方法，我们已经将主题改为Next，但是还是有地方我们可以优化和配置的。
一些常见的配置方法参考[文档](http://theme-next.iissnan.com/getting-started.html)即可。
<!-- more -->
### 语言设置
将languages目录下面的zh-Hans.yml修改为zh-CN.yml或者按照文档中的修改根目录配置文件也行。
### 修改样式
#### 网站标题栏背景颜色
网站标题栏背景颜色是黑色的，感觉不好看，可以在source/css/_schemes/Pisces/_brand.styl中修改
```
.site-meta {
  padding: 20px 0;
  color: white;
  background: $blue-dodger; //修改为自己喜欢的颜色

  +tablet() {
    box-shadow: 0 0 16px rgba(0,0,0,0.5);
  }
  +mobile() {
    box-shadow: 0 0 16px rgba(0,0,0,0.5);
  }
}
```
但是，我们一般不主张这样修改源码的，在`next/source/css/_custom`目录下面专门提供了`custom.styl`供我们自定义样式的，因此也可以在`custom.styl`里面添加：
```
// Custom styles.
.site-meta {
  background: $blue; //修改为自己喜欢的颜色
}
```
### 主页显示文章摘要
默认情况下在主页是把文章内容全部显示出来的，如果在根目录的`_config.yml`中配置分页:
```
per_page: 10
```
那么每页显示10篇文章，这样的花主页就看起来不是很精简了，可以通过下面的方法来配置主页只显示文章摘要：
在`themes/next/`的`_config.yml`中配置：
```
# Automatically Excerpt. Not recommand.
# Please use <!-- more --> in the post to control excerpt accurately.
auto_excerpt:
  enable: false 
  length: 150

```
把`enable`项配置为`true`就可以了。
但是我们也可以看到注释中是不推荐这样做的，因为这样会强制把文章前150个字符做为摘要的，会出现描述不完整的情况，而且有时候会把文章的源码显示出来。
那么我们就用推荐的在文章中用`<!-- more -->`这种方式来作为文章摘要的方式，可以根据每篇文章的不同情况自己来把控摘要的内容。
或者是在文章中配置`description`也是可以的。
```
---
title: Hexo+GitHub搭建个人博客
categories: Hexo
comments: true
keywords: Hexo, Blog, GitHub
tags: [Hexo, Blog, GitHub]
description: 使用Hexo在GitHub上搭建个人博客
date: 2017-01-010 13:00:00
---
```
### 阅读次数统计(使用LeanCloud)
参考[文档](http://theme-next.iissnan.com/third-party-services.html)
### 加入网站缩略图标
加入后就可以在浏览器的标签栏或者是收藏夹里面现实网站的缩略图标了。
在`themes/next/`的`_config.yml`中配置：
```
# Put your favicon.ico into `hexo-site/source/` directory.
favicon: /images/favicon.ico
```
然后把图表放到根目录的`source/images/`下面。
