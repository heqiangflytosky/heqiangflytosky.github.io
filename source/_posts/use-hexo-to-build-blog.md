---
title: Hexo+GitHub搭建个人博客
comments: true
keywords: Hexo, Blog, GitHub
description: 使用Hexo在GitHub上搭建个人博客
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
### 修改网站相关信息
修改根目录下面的_config.yml文件
```
title: 寒江蓑笠的个人博客 #网站title
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
### 主题
博客的整体配置在D:\hexo\_config.yml文件中进行。默认使用的主题是landscape，主题的配置在D:\hexo\themes\landscape\_config.yml。

## 发表博客
### 文章信息
```
title: Hexo+GitHub搭建个人博客
layout: post
date: 2017-01-010 15:00:00
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














