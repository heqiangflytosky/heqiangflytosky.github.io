---
title: Hexo博客配置优化
date: 2016-01-11 10:00:00
tags: Hexo
categories: Hexo
comments: true
description: 介绍一些Hexo的配置技巧
---
[Hexo 文档](https://hexo.io/docs/)
### 创建hexo分支
为了实现能在更换环境（比如更换电脑）的情况下我们仍然能发布博客，我们创建一个hexo分支用来存放hexo的文件。
``` bash
$ git checkout --orphan hexo
$ git rm -rf .
```
将hexo代码全部copy过来
```
$ git add . -A
$ git commit -m "hexo init"
$ git push origin hexo
```
这样就用hexo分支来存放网站的原始文件，master分支用来存放生成的静态网页。
<!-- more -->
### 添加README.md到github
众所周知hexo会把文件夹内的所有md文件解析成html，而github的readme只支持MD格式，但是我们可以使用下面方式来规避。
修改_config.yml文件：
skip_render: README.md
在source目录下创建README.md文件。
其他几种情况下的写法：
 - 单个文件夹下全部文件：skip_render: demo/*
 - 单个文件夹下指定类型文件：skip_render: demo/*.html
 - 单个文件夹下全部文件以及子目录:skip_render: demo/**
 - 多个文件夹以及各种复杂情况：
 ```
 skip_render:
    - 'demo/*.html'
    - 'demo/**'
 ```
### 修改网站相关信息
修改根目录下面的_config.yml文件
```
title: 孤舟蓑笠翁，独钓寒江雪 #网站title
subtitle: 天道酬勤  #副标题，网站名下面
description: 技术博客     //网站描述，便于搜索引擎用关键词检索
author: QH
language: zh-CN
timezone: Asia/Shanghai
```
### 添加RSS
安装RSS插件
```
$ npm install hexo-generator-feed --save
```
### 添加百度sitemap
站点地图，方便搜索引擎的收录
```
$ npm install hexo-generator-baidu-sitemap --save
```
我们在百度里面搜索site:heqiangfly.com，发现没有我们的博客并没有被百度收录，也就是说你的博客别人可能会看不到，下面来解决这个问题。
进入[链接提交](http://zhanzhang.baidu.com/linksubmit/url)，然后[验证网站所有权](http://zhanzhang.baidu.com/site/siteadd)，选择文件验证，下面baidu_verify_IIJFGFbbEX.html文件到source/目录下面。
修改_config.yml
```
skip_render: 
  - README.md
  - baidu_verify_IIJFGFbbEX.html
```
注意-后面要加个空格。
按照说明完成验证。
在百度站长平台里面的站点管理里面看到是否验证成功。
上面进行步骤成功之后，进入站点信息->网页抓取->链接提交->详情，按照说明进行设置。
完成后等一段时间，在百度里面搜索site:heqiangfly.com，有记录说明是被收录了。
### 添加Google收录
```
$ npm install hexo-generator-sitemap --save
```
谷歌操作比较简单，就是向[Google站长工具](https://www.google.com/webmasters/tools/home?hl=zh-CN)提交sitemap。
类似百度，通过HTML文件方式验证通过后，在站点里选择 抓取->站点地图里 添加/测试站点地图。
完成后等一段时间（大概一天时间）在Google里面搜索site:heqiangfly.com，就可以看到搜索结果了。
### 站点访问量统计
#### 添加CNZZ统计
首先要在[CNZZ网站](http://i.umeng.com/signup)注册一个帐号，复制一种你喜欢的统计格式的代码，在themes/landscape/layout/_partial/新建文件cnzz.ejs，加入代码：
```
<% if (theme.cnzz){ %>
<script type="text/javascript">var cnzz_protocol = (("https:" == document.location.protocol) ? " https://" : " http://");document.write(unescape("%3Cspan id='cnzz_stat_icon_1261134288'%3E%3C/span%3E%3Cscript src='" + cnzz_protocol + "s95.cnzz.com/z_stat.php%3Fid%3D1261134288' type='text/javascript'%3E%3C/script%3E"));</script>
<% } %>
```
把第一行与最后一行之间的代码替换成你自己的代码。
然后，在页面的某个位置添加你期望站长统计出现的位置，比如我是在footer.ejs里面加上以下代码：
```
<%- partial('cnzz') %>
```
然后在themes/landscape/_config.yml里面打开统计开关：
添加代码：
```
#### Analytics
cnzz: true
```
就会在页面左下角出现站长统计了。
#### 其他方法
参考[不蒜子](http://busuanzi.ibruce.info/)的方法。
### 文章访问量统计
使用LeanCloud，参考[文档](https://notes.wanghao.work/2015-10-21-%E4%B8%BANexT%E4%B8%BB%E9%A2%98%E6%B7%BB%E5%8A%A0%E6%96%87%E7%AB%A0%E9%98%85%E8%AF%BB%E9%87%8F%E7%BB%9F%E8%AE%A1%E5%8A%9F%E8%83%BD.html#%E9%85%8D%E7%BD%AELeanCloud)
### 修改日期显示
打开根目录下的_config.yml文件，修改：
```
date_format: YYYY-MM-DD
```
即可。
### 创建关于页面
```
hexo new page about
```
会在source/about中生成index.md，然后自己可以随意编辑。
在主题配置文件中添加
menu:
  about: /about
### 创建分类
```
hexo new page categories
```
会在source/categories中生成index.md，然后修改index.md。
```
---
title: 分类
date: 2017-01-17 17:53:50
type: "categories"
comments: false
---
```
在博客文章中配置，比如：
```
---
title: Hexo+GitHub搭建个人博客
categories: Hexo
comments: true
keywords: Hexo, Blog, GitHub
description: 使用Hexo在GitHub上搭建个人博客
date: 2017-01-010 13:00:00
---
```
那么这篇博客就添加到了Hexo分类中了。
在主题配置文件中添加
menu:
  categories: /categories
### 创建标签
```
hexo new page tags
```
会在source/tags中生成index.md，然后修改index.md。
```
---
title: tags
date: 2017-01-17 17:39:48
type: "tags"
comments: false
---
```
然后在文章中配置
```
tags: Hexo
```
就可以了。
多个标签：
```
tags: [标签1,标签2,标签3]
```
下面的写法也是可以的：
```
tags: 
 - Hexo
 - Blog
 - GitHub
```
在主题配置文件中添加
menu:
  tags: /tags

### 创建404页面

在 source 目录下创建 404.html 文件，页面内容可以自己定义，比如：

```

<html>
    <head>
         <meta charset="UTF-8" />
         <title>404</title>                                                                                                                                        
    </head>
    <body>
         <p> 有可能博客调整导致您所访问的文章地址发生变化，在此深感抱歉！ </p>
         <p> 请在在首页、归档或者分类里面找您需要的文章。  </p>
         <p> 
             <a href="http://www.heqiangfly.com/">回到首页</a>
             <a href="http://www.heqiangfly.com/archives/">归档</a>
             <a href="http://www.heqiangfly.com/categories/">分类</a>   
         </p>
    </body>
</html>

```

### 主题
[官方推荐的一些主题](https://hexo.io/themes/) 。
在 Hexo 中有两份主要的配置文件，其名称都是 _config.yml。博客的整体配置在hexo\_config.yml文件中进行。默认使用的主题是landscape，主题的配置在hexo\themes\landscape\_config.yml。
下面就把默认的landscape主题切换为next主题：
#### 下载主题
```
cd hexo\themes
git clone https://github.com/iissnan/hexo-theme-next next
```
注意：next为一个git仓库，可以采用submodule方式来管理，也可以把next下面的.git删除，然后再提交。
#### 启用主题
打开next根目录下面的配置文件_config.yml，修改
```
theme: next
```
#### 验证主题
```
hexo g
hexo s
```
浏览器中打开 http://localhost:4000/ 。
Next主题的具体其他的配置请参考[文档](http://theme-next.iissnan.com/getting-started.html)
然后再把自己自定义的一些东西，比如CNZZ统计迁移过来即可。
