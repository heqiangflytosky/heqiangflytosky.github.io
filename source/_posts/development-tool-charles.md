---
title: 抓包工具 Charles 的使用
categories: 开发工具
comments: true
tags: [开发工具, Charles]
description: 介绍 Charles 的使用
date: 2019-11-2 10:00:00
---

## 概述

Charles是一款代理服务器，通过成为电脑或者浏览器的代理，然后截取请求和请求结果达到分析抓包的目的。
本文就介绍一下在 Mac 上面安装和配置 Charles 的方法，以及在 Android 手机上对应的配置。
并且介绍了在 Android 7 以上手机上默认无法抓包的解决办法。
本文是基于 Charles 4.2.5 的版本进行介绍。

## 安装和破解

这部分可以搜索网络上的资料，这里就不再介绍。


## Mac 配置

### 配置 PC 端根证书

Help -> SSL Proxying -> Install Charles Root Certificate，安装后会弹出钥匙串访问界面，如图：

<img src="/images/development-tool-charles/1.png" width="438" height="298"/>

双击 Charles 证书，弹出证书详细界面，点击『信任』选项，然后将所有设置为始终信任，如图:

<img src="/images/development-tool-charles/2.png" width="331" height="228"/>

### 配置手机端根证书

Help -> SSL Proxying -> Install Charles Root Certificate On a Mobile Device or a Remote Browser，弹出如下界面：

<img src="/images/development-tool-charles/3.png" width="519" height="106"/>

然后根据提示在手机端设置Http代理，下面以魅族手机为例来介绍一下设置方法：

设置 -> 无线网络 -> 点击当前连接网络后面的高级选项 -> 代理设置界面 -> 手动设置代理，然后在代理设置界面输入上对话框显示的服务器地址和端口号。
输入后点击保存，Charles 会弹出下面对话框：

<img src="/images/development-tool-charles/4.png" width="520" height="106"/>

然后点击 Allow。
然后打开手机浏览器，访问chls.pro/ssl，下载证书并安装(证书名任意)。下面介绍安装方法：
设置 -> 指纹、面部和安全 -> 设备管理与凭证 -> 从存储设备安装，打开刚才下载的 charles-proxy-ssl-proxying-certificate.pem，随意为证书命名后点击确定。

<img src="/images/development-tool-charles/5.png" width="270" height="558"/>

魅族手机还要求必须设置安全锁屏功能才能使用凭据存储。
手机端配置完毕。点击 Charles 上面 刚才弹出的 OK 按钮，就可以使用了。

### 配置抓取规则

进入Charles的SSL代理设置，Proxy -> SSL Proxying Settigs...，

<img src="/images/development-tool-charles/6.png" width="398" height="296"/>

可以点击 Add 添加规则：

<img src="/images/development-tool-charles/7.png" width="293" height="123"/>

然后打开手机，开始网络相关请求，就可以在 Charles 看到网络请求的数据了。


## 解决 Android 7.0 以上无法抓包问题

上面配置好以后，我们开始网络请求，在 Android 7以上手机上可能会请求失败，原因是因为从 Android 7.0 开始，默认的网络安全性配置修改了，具体请阅读[官方文档网络安全性配置](https://developer.android.com/training/articles/security-config)。
在 Charles 上看到：

<img src="/images/development-tool-charles/8.png" width="613" height="76"/>

添加 `res/xmlnetwork_security_config.xml`:

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

在 AndroidManifest.xml 中配置：

```
    <application
        ......
        android:networkSecurityConfig="@xml/network_security_config"
        ......>
```

然后就可以用 Charles 代理进行正常网络请求和抓包了。

这种解决方式有一个安全风险：Release版的应用会有被他人抓包的风险。
我们可以使用只在debug模式下允许抓包来规避这个风险：

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- 支持 Android 7.0 以上调试时，信任 Charles 和 Fiddler 等用户信任的证书 -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

在 Android 9.0（API 28）以上允许部分 http 请求，最佳的解决方式肯定是全部使用 https 请求，安全性更高，如果有些请求或测试环境下还是需要使用 http 请求，需要在网络安全性配置添加白名单：

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- 支持 Android 9.0 以上使用部分域名时使用 http -->
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">sample.domain</domain>
    </domain-config>
    <!-- 支持 Android 7.0 以上调试时，信任 Charles 和 Fiddler 等用户信任的证书 -->
    <debug-overrides>
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

或者：

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
    </base-config>
    <debug-overrides>
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </debug-overrides>
</network-security-config>
```

## 常见问题

1. 配置好后无法打开APP
在我们抓取时碰到个别APP在配置代理后无法打开，这个主要是因为该APP做了防止抓取处理，比如校验https的证书是否合法等，这种解决方法可以通过反编译APP，查看源码解决，难度较大。

2. 抓取到的内容为乱码
有的APP为了防止抓取，在返回的内容上做了层加密，所以从Charles上看到的内容是乱码。这种情况下也只能反编译APP,研究其加密解密算法进行解密。

## 参考文档

https://developer.android.com/training/articles/security-config