---
title: Hexo--Next主题优化
date: 2017-01-17 19:47:22
tags: [Hexo, Next Theme]
categories: Hexo
comments: true
---
按照上篇博客中更换Theme的方法，我们已经将主题改为Next，但是还是有地方我们可以优化和配置的。
一些常见的配置方法参考[文档](http://theme-next.iissnan.com/getting-started.html)即可。
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
### 阅读次数统计(使用LeanCloud)
参考[文档](http://theme-next.iissnan.com/third-party-services.html)
