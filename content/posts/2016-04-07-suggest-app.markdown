---
title:  "安利几个App"
date: 2016-04-07
slug: suggest-app
---

好久没有更新过博客了，因为没有记录事情的习惯，好多时候解决了一个问题，想写下来分享，但是想想又没多少人看，也就罢了。不过自从被国飞顾逼着看论文以来，我发现不做笔记是不行了，不然看完下一篇忘了前一篇，根本抓不住重点。正好最近换了iPhone，为了提高效率找了不少App来用，在此推荐推荐，顺便再讲讲去年双十一时候买的NAS。

<!--more-->

## NAS

先讲NAS。很早以前因为硬盘不够用，再加上用台式机共享电影不方便，看完片还得远程操作关下机，我就萌发了买NAS的念头。去年双十一京东搞白条活动，24期免息，价格虽然还是比海淘贵点，但也能接受，我就咬咬牙入了QNAP 453-Pro，外加两块4T红盘。

到现在已经用了快半年了，虽说最初的需求只是SMB共享影片，但后来发现NAS的用途远大于此，以至于现在我几乎要离不开它了。SMB就不多说了，我介绍一下我用的最多的3种功能。

#### BitTorrent Sync

BTSync是一款同步软件。一开始我用QNAP自己的QSync，发现其烂无比，经常不停的在扫描硬盘，有文件改动的时候反而没有反应。后来我在App Center里发现了这个BTSync，同步效果堪称完美，文件有改动时响应速度也很快，百万级以下文件数量完全没有问题（我现在的文件数量十万级都没到）。现在我在NAS、家里的台式机、笔记本电脑以及实验室里的电脑都装上BTSync。平时在实验室里的改动都会立刻同步到NAS上，回家开台式机后会立刻从NAS上同步过来，实在是居家旅行必备。

#### WebDAV

WebDAV是NAS自带的功能，可供远程主机访问NAS里的文件。其实这种功能FTP也可以做到，奈何iOS上的工具都不支持FTPS（FTP over SSL），只支持SFTP，但SFTP跟FTP其实是两种东西，NAS上开启SFTP也要求开启SSH，我不想平时都开着SSH，也不想让FTP在互联网上明文传输，所以只能放弃FTP。最后选用WebDAV是因为它既被iOS上的绝大多数应用支持，也能用HTTPS安全传输。如果说BTSync用于电脑与NAS之间的同步，WebDAV则是用于移动设备与NAS之间的同步。例如，我用GoodReader在iPad上看一篇论文，期间做了一些标记，之后我用GoodReader的同步功能与NAS进行同步，然后再在手机上同步一下，修改过的论文就传过来了，另一方面，电脑上存储的文件也会通过BTSync同步过来。

#### Container Station

Container Station是QNAP对Docker这样的容器技术的一个包装。最开始我用的是Virutalization Station，也就是虚拟机，但是NAS性能一般，虚拟机速度较慢，而且虚拟机的管理成本高众所周知。后来发现了Container Station之后我花了一番精力把虚拟机上的所有功能都转移到了Container Station上。现在容器上主要跑2个功能：nginx反向代理，以及squid HTTP代理。nginx反向代理是因为NAS上有些应用不支持HTTPS（比如transmission和BTSync的Web控制页面），我现在完全不能容忍HTTP明文传输，必须加密，所以就用nginx在外面套一层HTTPS。squid实际上是一个HTTP缓冲服务器，但我是拿它当代理来用，因为实验室网络太渣，但连我家里的网速度还不错，所以我现在平时在实验室就用家里的HTTP代理服务器上网，还有一个好处，就是我路由器上刷的梅林固件自带了Shadowsocks，可以透明翻墙，所以用了squid就可以直接翻墙了。

![Container Station](https://res.cloudinary.com/core2duoe6420/image/upload/v1643913767/posts/container-station_bqpnki.jpg)

为了能在控制台上直接看到squid的日志，我当初还费了一番精力。借助Docker的Volume功能，可以把配置文件放在BTSync的同步文件夹中，这样如果我在实验室想改squid的配置，只需在本地修改完配置文件，等一会儿BTSync就会把修改同步到NAS，最后在Container Station上重启一下容器就行。为了安全起见，我还做了两个版本的squid，一个是digest用户验证，一个是basic验证，后者只有在用一些不支持digest验证的软件时才会启动。

## iOS上的效率App

前面说过，我最近才换了iPhone，所以对iOS上的App热情比较高。之前虽然也有iPad，但是毕竟需求不一样，动力也不一样，这里先介绍几个我最近发现的，以后有好的再说。

#### 印象笔记+马克飞象

印象笔记就不多说了，大家都知道，虽然近两年似乎呼声没有以前那么高了，但是还算好用，OneNote因为渣一般的同步速度实在是没法用。马克飞象是我最近才开始用的东西。我原来一直用作业部落的CmdMarkdown作为Markdown编辑器，CmdMarkdown升级高级会员后也能同步印象笔记，但相比马克飞象它有一个致命弱点：不支持直接粘贴图片。仅此一项就有足够的理由用马克飞象了。用了马克飞象之后，做论文笔记的时候可以直接把公式输入进去了，神清气爽！

#### Papers

Papers是一个论文管理软件，可以在各个数据库中根据不同的字段搜索论文，可以在不同设备同步收藏的论文，还有Mac OS的版本。不过我并没有深入使用，一来是因为我还是主用GoodReader管理PDF，我也不是真的想搞research，我是被逼的！二来学校的网我真是无力吐槽，WIFI还好点，实验室的网能不能连上那些文献数据库完全是看它心情，大大影响体验……

#### AppZapp

AppZapp的功能很简单，就是App降价提醒，再也不会错过限免App了！

#### Wunderlist奇妙清单

这个App现在是我在逛知乎找App的时候知道的。一开始是想尝试一下著名的GTD工具omniFocus，奈何太贵，而且看评论似乎不适合我这样的散人。于是试了下Wunderlist，感觉还不错，就当做是简单的清单，把要看的电影、书、论文都列上去，看完一个划掉一个，特别爽。

#### iThoughts

国飞顾在这周的Meeting上叫我做总结，于是我就想到了之前看到的思维导图，用来做MTD的总结再好不过。知乎大神上推荐各种思维导图App：iThoughts、MindNoe、SimpleMind、MindManager等等，随手试了下iThoughts，感觉完全能满足我的需求，就不试别的了。iThoughts无论在哪个平台上都要花钱，而且不便宜，我现在Mac上用的是盗版的iThoughts（不要骂我），iOS上用的是iThoughts2go，这是iThoughts的免费版，相比完全版，只在节点数量上做了限制，不能超过20个，其它功能一样，所以我在Mac上先编辑完，再同步到手机和iPad上。我把iThoughts放到了AppZapp的提醒列表上，一等它降价就买（最低降到过12元啊……）。晒一张用iThoughts做的MTD总结图。

![MTD](https://res.cloudinary.com/core2duoe6420/image/upload/v1643913768/posts/MTD_mxdvva.png)

## 最后，关于博客

一年前我就说想给博客换肤了，结果直到今天才搞定，非常惭愧。新的系统基于Jekyll，用的是Hux的模板，在此表示感谢。评论系统换成了Disquz，我自己添加了按需加载MathJax的功能。新的模板比原来那个是好看多了。我没有把博客放在Github上然后通过CNAME指向它，因为这样没法用HTTPS，前面说过，我现在对非加密的HTTP简直深恶痛绝。现在这样也只是多了一步写完文章去VPS上手工运行Jekyll Build一下的操作，也不碍事。

本来这篇文章是想写些关于最近的事。最近实在是压力太大，就像我跟朋友说的那样：“论文发不出毕不了业，工作找不到，妹子也找不到，打个LOL我还定位到了青铜，实在是life loser。”后来想想还是算了，就在此一笔带过，不再散发负能量了。生活总得继续，只要不死，总有希望。

