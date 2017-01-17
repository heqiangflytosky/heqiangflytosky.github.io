---
title: Hexo+GitHub搭建个人博客
comments: true
keywords: Hexo, Blog, GitHub
description: 使用Hexo在GitHub上搭建个人博客
date: 2017-01-010 13:00:00
keywords: Hexo, Blog, GitHub
---
## 本地环境搭建
### 安装git
### 安装node.js
去[官网](https://nodejs.org/en/download/)下载node-v5.0.0-linux-x64解压即可

## 配置Github
### 建立Repository
首先在github上面建一个仓库username.github.io
### 域名配置
首先要购买一个域名，然后进入域名解析页面，添加一个CNAME记录，指向username.github.io即可。配置好一般等待TTL缓存的时间过后，博客建好后，访问你的域名heqiangfly.com，你会发现已经调整到你的博客页面了。

## 安装Hexo
### 安装Hexo
``` bash
$ npm install -g hexo
```
### 初始化hexo
``` bash
$ hexo init
```
### 生成静态页面
``` bash
$ hexo g
```
### 启动本地服务，进行文章预览调试
``` bash
$ hexo s
```
浏览器输入http://localhost:4000/，查看搭建效果。
### 配置Hexo
编辑更目录下面的_config.yml文件
```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: https://github.com/heqiangflytosky/heqiangflytosky.github.io.git
  branch: master
  name: heqiang
  email: heqiangfly@163.com
```
注意：每一项的填写，其:后面都要保留一个空格，下同。
### 发布到github
``` bash
$ hexo d
```
如果有下面报错：
```
ERROR Deployer not found: git
```
执行下面命令：
``` bash
$ npm install hexo-deployer-git --save
```
再次执行 hexo d，输入github用户名和秘密，就已经发布到github对应仓库的master分支了。
### 绑定域名
在source文件夹中新建一个CNAME文件
写入www.heqiangfly.com
生成提交
``` bash
$ hexo g
$ hexo d
```
然后访问www.heqiangfly.com你会发现已经跳转到博客页面了。
## Hexo优化
### 创建hexo分支
为了实现能在更换环境（比如更换电脑）的情况下我们仍然能发布博客，我们创建一个hexo分支用来存放hexo的文件。
``` bash
$ git checkout --orphan hexo
$ git rm -rf .
$ git add . -A
$ git push origin hexo
```
这样就用hexo分支来存放网站的原始文件，master分支用来存放生成的静态网页。
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
使用LeanCloud https://notes.wanghao.work/2015-10-21-%E4%B8%BANexT%E4%B8%BB%E9%A2%98%E6%B7%BB%E5%8A%A0%E6%96%87%E7%AB%A0%E9%98%85%E8%AF%BB%E9%87%8F%E7%BB%9F%E8%AE%A1%E5%8A%9F%E8%83%BD.html#%E9%85%8D%E7%BD%AELeanCloud
### 修改日期显示
打开根目录下的_config.yml文件，修改：
```
date_format: YYYY-MM-DD
```
即可。
### 主题
[官方推荐的一些主题](https://hexo.io/themes/) 。
在 Hexo 中有两份主要的配置文件，其名称都是 _config.yml。博客的整体配置在hexo\_config.yml文件中进行。默认使用的主题是landscape，主题的配置在hexo\themes\landscape\_config.yml。

## 发表博客
### 文章信息
```
title: Hexo+GitHub搭建个人博客
layout: post
date: 2017-01-010 13:00:00
comments: true
categories: Blog
tags: 
keywords: Hexo, Blog, GitHub
description: 使用Hexo在GitHub上搭建个人博客
```
### 设置摘要
```
以上是文章摘要 <!--more--> 以下是余下全文
```

## Hexo常用命令
- hexo generate = hexo g          #生成
- hexo server = hexo s            #启动服务预览
- hexo deploy = hexo d            #部署
- hexo new "博客"= hexo n "博客"   #新建文章
- hexo clean                      #清除缓存,清除缓存文件 db.json 和已生成的静态文件 public。 网页正常情况下可以忽略此条命令
- hexo new "postName"             #新建文章
- hexo new page "pageName"        #新建页面

好了，我的GitHub个人博客[孤舟蓑笠翁，独钓寒江雪](www.heqiangfly.com)就是这样炼成的，欢迎大家访问，博客持续更新中。














