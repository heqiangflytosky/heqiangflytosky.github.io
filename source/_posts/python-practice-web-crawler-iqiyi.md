
---
title: Python实战 -- 网络爬虫之解析爱奇艺视频
categories: Python
comments: true
tags: [Python]
description: 介绍使用 Python 来抓取爱奇艺视频网页内容来解析视频播放地址的方法
date: 2018-03-20 10:00:00
---

## 概述

说到 Python 实战示例，肯定不能不提网络爬虫，Python 简介的语法和强大的数据处理能力以及一些成熟的框架成为做爬虫项目工程师的最爱，那么本篇博客就介绍利用 Python 来解析视频网站视频源播放地址。
说到这个就再顺便介绍一下 Kodi，其实就是 XBMC 的前身，我最早接触 Python 其实也是缘由一个 关于 XBMC 的项目，Kodi的很多插件就是用 Python 来写的，本篇博客的爱奇艺视频解析原理也是参考了 Kodi 的插件，为了便于理解，做了一些简化，只保留了一些基本的功能。
大家可以参考这个开源的 [Kodi插件](https://github.com/taxigps/xbmc-addons-chinese)。
前面我也写过一片关于如何分析爱奇艺视频播放地址的文章，大家也可以参考一下[XBMC视频插件开发系列--网页数据解析](https://blog.csdn.net/heqiangflytosky/article/details/8931905)

## 正文

废话不多说，现在开始进入正题。
做爬虫，网络请求是必不可少的，那么现在我们就先写一个网络数据请求的方法出来。

### 网络请求方法

```
def getHttpData(url):
    req = urllib2.Request(url)

    try:
        reponse = urllib2.urlopen(req)
        httpData = reponse.read()
    except:
        print "get http data error"
        return ""

    return httpData
```

用百度主页来测试一下，OK，可以抓取到数据。

###  获取分类列表

我们进入[爱奇艺视频主页](http://www.iqiyi.com/)，我们会看到视频分类列表，点击这个列表，比如电影，就进入电影的分类页面，这个时候再点击一个检索条件，比如近期热门，就进入了分类检索页面。那么我们的解析工作也就从这个页面开始。
首先我们点击电影，发现网址变为：http://list.iqiyi.com/www/1/----------------iqiyi--.html。
再点击电影频道下方的免费分类：http://list.iqiyi.com/www/1/----------0---11-1-1-iqiyi--.html。
再点击电视剧频道：http://list.iqiyi.com/www/2/----------------iqiyi--.html。
多点击几个你就会得到这样的规律：
频道对应的是 http://list.iqiyi.com/www/<频道>/----------------iqiyi--.html，后面的 - 分割的各种检索条件，比如资费、地区和类型等等。
通过这种方式我们可以把频道对应的频道ID、排序方式ID和付费类型ID给整理好：

```
CHANNEL_LIST = [['电影', '1'], ['电视剧', '2'], ['纪录片', '3'], ['动漫', '4'], ['音乐', '5'], ['综艺', '6'], ['娱乐', '7'],['旅游', '9'], ['片花', '10'], ['教育', '12'], ['时尚', '13']]
ORDER_LIST = [['4', '更新时间'], ['11', '热门']]
PAYTYPE_LIST = [['', '全部影片'], ['0', '免费影片'], ['1', '会员免费'], ['2', '付费点播']]
```

那么就可以把获取视频类型的方法给写出来：

```
def rootList():
    for name, id in CHANNEL_LIST:
        u = sys.argv[0] + "?mode=1&name=" + urllib.quote_plus(name) + "&id=" + urllib.quote_plus(
            id) + "&cat=" + urllib.quote_plus("") + "&area=" + urllib.quote_plus("") + "&year=" + urllib.quote_plus(
            "") + "&order=" + urllib.quote_plus("11") + "&page=" + urllib.quote_plus(
            "1") + "&paytype=" + urllib.quote_plus("0")
        print name
        print u
```

通过这个方法，组装了各个分类视频需要的一些参数，这里把他们打印出来，根据需要再调用下面的方法获取各个视频。

```
电影
/Users/*****/iqiyi_custom.py?mode=1&name=%E7%94%B5%E5%BD%B1&id=1&cat=&area=&year=&order=11&page=1&paytype=0
电视剧
/Users/*****/iqiyi_custom.py?mode=1&name=%E7%94%B5%E8%A7%86%E5%89%A7&id=2&cat=&area=&year=&order=11&page=1&paytype=0
......
```

### 获取视频列表

有了视频分类列表，那么下一步就是获取你需要的视频列表以及每个视频所对应的播放页面地址了，先看代码吧，这里时主要来获取电影分类的，解析原理请看代码中的注释：

```python
#         id   c1   c2   c3   c4   c5     c11  c12   c14
# 电影     1 area  cat                paytype year order
# 电视剧   2 area  cat                paytype year order
# 纪录片   3  cat                     paytype      order
# 动漫     4 area  cat  ver  age      paytype      order
# 音乐     5 area lang       cat  grp paytype      order
# 综艺     6 area  cat                paytype      order
# 娱乐     7       cat area           paytype      order
# 旅游     9  cat area                paytype      order
# 片花    10      area       cat      paytype      order
# 教育    12            cat           paytype      order
# 时尚    13                      cat paytype      order
def progList(name, id, page, cat, area, year, order, paytype):
    c1 = ''
    c2 = ''
    c3 = ''
    c4 = ''
    if id == '7':  # 娱乐
        c3 = area
    elif id in ('9', '10'):  # 旅游&片花
        c2 = area
    elif id != '3':  # 非纪录片
        c1 = area
    if id in ('3', '9'):  # 纪录片&旅游
        c1 = cat
    elif id in ('5', '10'):  # 音乐&片花
        c4 = cat
    elif id == '12':  # 教育
        c3 = cat
    elif id == '13':  # 时尚
        c5 = cat
    else:
        c2 = cat
    # 根据分类信息组装url
    url = 'http://list.iqiyi.com/www/' + id + '/' + c1 + '-' + c2 + '-' + c3 + '-' + c4 + '-------' + \
          paytype + '-' + year + '--' + order + '-' + page + '-1-iqiyi--.html'
    print url
    currpage = int(page)
    # 抓取网页数据
    link = getHttpData(url)
    # 根据 data-key=<数字> 这个标记字符串来获取页数，匹配出来多少项就有多少页
    match1 = re.compile('data-key="([0-9]+)"').findall(link)
    if len(match1) == 0:
        totalpages = 1
    else:
        totalpages = int(match1[len(match1) - 1])

    # <div class="wrapper-piclist 和 <!-- 页码 开始 --> 中间的内容是有关视频列表的内容，用正则匹配获取相关内容
    match = re.compile('<div class="wrapper-piclist"(.+?)<!-- 页码 开始 -->', re.DOTALL).findall(link)
    if match:
        # 匹配前面内容中的 <li> </li>列表项，匹配多少项代表当前页有多少个视频
        # 正则表达式里面的 <li[^>]*> 中[^>]*非>字符的0个或多个字符，防止<li>标签内带一些属性值
        match = re.compile('<li[^>]*>(.+?)</li>', re.DOTALL).findall(match[0])

    print '-------'
    for i in range(0, len(match)):
        # 匹配alt=""冒号内的字符串，代码视频名称
        p_name = re.compile('alt="(.+?)"').findall(match[i])[0]
        # 匹配src = ""冒号内的字符串，代表视频海报地址，\s* 代表0个或多个空白字符
        p_thumb = re.compile('src\s*=\s*"(.+?)"').findall(match[i])[0]
        # 匹配href="([^"]*)"冒号内非"的0个或多个字符串，代表视频的播放界面，这个是后面解析视频播放地址的基础
        p_id = re.compile('href="([^"]*)"').search(match[i]).group(1)

        try:
            # 根据 data-qidanadd-episode 是否为1来判断是否是多集的影视剧
            p_episode = re.compile('data-qidanadd-episode="(\d)"').search(match[i]).group(1) == '1'
        except:
            p_episode = False
        # 匹配 <span class="icon-vInfo"> 和 </span>中间的内容
        match1 = re.compile('<span class="icon-vInfo">([^<]+)</span>').search(match[i])
        if match1:
            # 去掉匹配内容的空格
            msg = match1.group(1).strip()
            if (msg.find('更新至') == 0) or (msg.find('共') == 0):
                # 如果内容里面有'更新至' 或者 '共' 这样字符，表示是连续剧
                p_episode = True

        if p_episode:
            # 如果是连续剧 mode 设置为2
            mode = 2
            # 获取视频id
            p_id = re.compile('data-qidanadd-albumid="(\d+)"').search(match[i]).group(1)
        else:
            # 否则 mode 设置为3，进入progList 方法
            mode = 3
        # 组装url
        u = sys.argv[0] + "?mode=" + str(mode) + "&name=" + urllib.quote_plus(p_name) + "&id=" + urllib.quote_plus(
            p_id) + "&thumb=" + urllib.quote_plus(p_thumb)
        print p_name
        print u
```

调用方法：

```
name = '电影'
id = '1'
cat = ''
area = ''
year = ''
order = '11'
page = '1'
paytype = '0'

print name, id, page, cat, area, year, order, paytype
progList(name, id, page, cat, area, year, order, paytype)
```

结果：

```
反转人生
/Users/****/iqiyi_custom.py?mode=3&name=%E5%8F%8D%E8%BD%AC%E4%BA%BA%E7%94%9F&id=http%3A%2F%2Fwww.iqiyi.com%2Fv_19rr7ph8kw.html%23vfrm%3D2-4-0-1&thumb=%2F%2Fpic9.qiyipic.com%2Fimage%2F20180220%2Fbe%2F7a%2Fv_112880060_m_601_m2_180_236.jpg
密战
/Users/****/iqiyi_custom.py?mode=3&name=%E5%AF%86%E6%88%98&id=http%3A%2F%2Fwww.iqiyi.com%2Fv_19rrebyuec.html%23vfrm%3D2-4-0-1&thumb=%2F%2Fpic4.qiyipic.com%2Fimage%2F20180220%2F38%2F76%2Fv_114269181_m_601_m2_180_236.jpg
......
```

### 获取视频播放地址

通过上一个方法我们可以得到一个视频列表以及该视频的播放页面地址，这里我们就选择《密战》这部电影来介绍吧。

```
def getVMS(tvid, vid):
    t = int(time.time() * 1000)
    src = '76f90cbd92f94a2e925d83e8ccd22cb7'
    key = 'd5fb4bd9d50c4be6948c97edd7254b0e'
    sc = hashlib.md5(str(t) + key + vid).hexdigest()
    vmsreq = 'http://cache.m.iqiyi.com/tmts/{0}/{1}/?t={2}&sc={3}&src={4}'.format(tvid, vid, t, sc, src)
    print 'vms地址'
    print vmsreq
    return simplejson.loads(getHttpData(vmsreq))

def PlayVideo(name, id, thumb):
    id = id.split(',')
    if len(id) == 1:
        try:
            if ("http:" in id[0]):
                link = getHttpData(id[0])
                tvId = re.compile('data-player-tvid="(.+?)"', re.DOTALL).findall(link)[0]
                videoId = re.compile('data-player-videoid="(.+?)"', re.DOTALL).findall(link)[0]
            else:
                url = 'http://cache.video.qiyi.com/avlist/%s/' % (id[0])
                link = getHttpData(url)
                data = link[link.find('=') + 1:]
                json_response = simplejson.loads(data)
                tvId = str(json_response['data']['vlist'][0]['id'])
                videoId = json_response['data']['vlist'][0]['vid'].encode('utf-8')
        except:
            print '未能获取视频地址'
            return
    else:
        tvId = id[0]
        videoId = id[1]

    # 根据一定规则获取视频流信息
    info = getVMS(tvId, videoId)
    if info["code"] != "A00000":
        print '无法播放此视频'
        return

    vs = info["data"]["vidl"]
    sel = selResolution([x['vd'] for x in vs])
    if sel == -1:
        return
    # 根据分辨率获取对应地址
    video_links = vs[sel]["m3u"]

    print '视频播放url'
    print video_links
```

这里获得的url`http://cache.m.iqiyi.com/mus/870139400/ab59ce8ca82428c9a62cbaf9596d061c/afbe8fd3d73448c9//20171214/02/26/28e78579e4bfa8dcb7266c0f7487670a.m3u8?qd_originate=tmts_py&tvid=870139400&bossStatus=0&qd_vip=0&px=&qd_src=3_31_312&prv=&previewType=&previewTime=&from=&qd_time=1521815439747&qd_p=0e7bff72&qd_asc=2d335d24c5a9412e7da82ee3e003155c&qypid=870139400_04022000001000000000_4&qd_k=cd5ab91d76256006d732d157ce8ee616&isdol=0&code=2&qd_s=otv&vf=5299931f645288f5664953a7f30d65ea&np_tag=nginx_part_tag`就是视频地址的url，根据这个url取得数据是m3u8格式的播放地址。这里可能要根据所使用播放器的m3u8的支持形式做一些转化。
其实这些分段的地址单个拿出来都是可以播放的：

```
http://dx.data.video.qiyi.com/videos/v0/20171214/02/26/d2c83af6c58ccbd91525ef8eee950886.ts?qdv=1&qypid=870139400_04022000001000000000_4&start=0&end=334466&hsize=2161&tag=0&v=&contentlength=285384&qd_uid=&qd_vip=0&qd_src=3_31_312&qd_tm=1521816060630&qd_ip=0e7bff72&qd_p=0e7bff72&qd_k=95668376716c9dcf32e2fbd6597c6f40&sgti=&dfp=&qd_sc=947aee71a7bc4e65d241aeb5a4657439
······
```

好了，关于爱奇艺视频爬虫就介绍到这里。

## 参考

https://github.com/taxigps/xbmc-addons-chinese
