---
title: "Samba配置问题导致Kodi播放卡顿"
date: 2022-02-11
slug: samba-kodi
tags:
- Kodi
---

最近在电视上用Kodi看The Good Doctor的时候，卡顿及其严重，根本无法观看。起初我以为是片源的问题，可能是使用了某些高压缩率的编码参数。虽然很困惑，因为之前放HEVC 4K HDR视频都没有问题，但是我也没多想。后来发现几乎所有的片子都没法正常播放了，那就肯定是哪里出问题了，于是又开始一轮排障。

<!--more-->

首先想了想我最近这段时间做的变更：

1. 我更换了samba的镜像，从`dperson/samba`换到了`ghcr.io/jtagcat/samba`，原因是前者已经好久没有更新了，samba的一些bugfix和安全补丁都没有，而后者是前者的fork，支持维护中
2. Kodi版本升级到了19.3

考虑到升级过Kodi版本，我去设置里兜了一圈看看有没有新的设置。然而并没有什么发现。根据网上的一些说法，把SMB最小协议设置成SMBv2也没有作用。

看到网上有人说Kodi使用WebDAV的性能比SMB好很多，于是想先换成WebDAV来确认Wifi连接本身的速率可以满足播放的要求。想办法启动了一个HTTP的WebDAV服务器，配到Kodi上后，发现可以正常播放，那就说明问题确实出在SMB协议上。

为了验证Kodi的SMB客户端本身是否有问题，我在Windows上开启了一个共享文件夹，把视频文件放进去用Kodi播放，也没问题。那最终问题就定位到了NAS使用的samba服务器上。

我记得在我刚配好NAS那阵子Kodi是可以正常播放的，而那时候用的还是`dperson/samba`这个镜像，于是查看了一下新老镜像的Dockerfile有没有什么区别：

```bash
diff \
    <(curl -s https://raw.githubusercontent.com/dperson/samba/master/Dockerfile) \
    <(curl -s https://raw.githubusercontent.com/jtagcat/samba/main/samba/Dockerfile)
```

果然发现了一些猫腻：

```diff
42a42
>     echo '   smb encrypt = desired' >>$file && \
55,56c55
```

新版本多了个一个`smb encrypt = desired`的配置，查阅[samba官方文档](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#SERVERSMBENCRYPT)文档后得知，默认和desired选项区别是：

- Leaving it as default, explicitly setting default, or setting it to if_required globally will enable negotiation of encryption but will not turn on data encryption globally or per share.
- Setting it to desired globally will enable negotiation and will turn on data encryption on sessions and share connections for those clients that support it.

很有可能是这个选项导致Kodi的卡顿问题，于是在容器的配置里加一条环境变量`GLOBAL4: 'smb encrypt = default'`（这是镜像支持的一种配置方式）把这个设置还原回去，重建容器后，再用Kodi打开视频，流畅播放，问题解决！