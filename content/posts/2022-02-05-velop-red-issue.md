---
title: "解决Linksys Velop指示灯变红但是网络通畅的问题"
date: 2022-02-05
slug: velop-red-issue
coverImage: https://res.cloudinary.com/core2duoe6420/image/upload/v1644055415/posts/velop-red-issue/velop_tztin8.jpg
coverMeta: out
thumbnailImagePosition: left
thumbnailImage: https://res.cloudinary.com/core2duoe6420/image/upload/v1644055415/posts/velop-red-issue/sf217443-002_en_v8_rsxhyt.png
---

在之前[折腾IPTV]({{< ref "2022-02-03-iptv-proxy" >}})的过程中，发现用来做AP的Linksys Velop的网络指示灯变成了红色，但是我电脑上的网络一切正常。起初以为是`igmpproxy`引起的问题，后来发现关闭`igmpproxy`后问题依旧。本文记录了解决此问题的过程。

<!--more-->

首先，经过一系列的关停重启操作后，发现问题出现的规律如下：

- 启动VLAN 85的接口`eth3.85`，大约5分钟后Velop的指示灯会变红
- 关闭`eth3.85`，大约15秒内Velop的指示灯会恢复成正常的蓝色

到此只能猜测Velop发出了某些网络探针，而开启`eth3.85`导致了这种探针失败。但是由于Velop的软件做的实在太烂，是个完全的黑盒子，默认的管理界面上几乎什么信息都没有，所以根本无从知道这种探针究竟是什么。最初的怀疑是探针访问的服务器落在了IPTV专网的`23.3.0.0/17`内，导致流量被引入了IPTV专网从而失败。

Google一番后，发现网络上关于Velop的资料实在太少了，仅在reddit上发现了下面几篇：

- [Red lights but all appears to be working](https://www.reddit.com/r/LinksysVelop/comments/iu31w3/red_lights_but_all_appears_to_be_working/)
- [Linksys Velop Red Light Despite Devices Connected Just Fine](https://www.reddit.com/r/pihole/comments/gn7rtf/linksys_velop_red_light_despite_devices_connected/)
- [linksys velop wired backhaul](https://www.reddit.com/r/LinksysVelop/comments/ip0h3k/linksys_velop_wired_backhaul/)

事实上，前两个帖子已经揭露了造成这个问题的原因很可能和`heartbeat.belkin.com`这个域名有关，不过我还是走了不少弯路。

上面的第三个链接中有提到`sysinfo.cgi`可以看到一些Velop的内部日志。我试了一下果然可以，然而这个cgi只是访问的时候跑一遍各种命令然后回显输出，最关键的`/var/log/message`只显示200行。为了追踪日志我写了个Python脚本轮询日志，脚本放在了[这里](https://gist.github.com/core2duoe6420/3361c0ed7d9a654bd056f3c953d10767)。

日志里我比较在意的有以下几点：

- service.devidentd Ethernet Agent: EthUtil returns multiple MACs 这条日志大量重复，然而Google后一无所获
- service.status Subscriber: Subject 'network/AC4C4927-CC4A-0BF1-B850-302303D68097/WLAN/neighbors' matches subscribed topic 'network/+/WLAN/neighbors'
- lldpd.status lldpd event lldp::update  received.

后面两条日志让我有了奇怪的猜测，会不会存在某种WAN通知机制让路由器告诉Velop，“我这边增加一个网络”，例如第三条日志里的LLDP？当然，事实证明这完全是扯淡🤣。

从日志中没有什么收获，只能寄希望于抓包，看看在打开和关闭`eth3.85`前后时间内，Openwrt和Velop到底进行了哪些通信。于是在与Velop连接的端口`eth0`上启动tcpdump，然后用Wireshark分析。

一开始我重点关注了上文提到的`heartbeat.belkin.com`这个域名，但是发现Velop指示灯变红的时候，对这个域名的DNS和ping都是正常的。

![20220205165652_bdlds6](https://res.cloudinary.com/core2duoe6420/image/upload/v1644051527/posts/velop-red-issue/20220205165652_bdlds6.png)

于是开始怀疑是这并不是我要找的探针。接下来花了一个多小时试图找出两种情况下流量的不同之处，然而并没有什么成果。直到我偶然间发现有一次对`heartbeat.belkin.com`发起的AAAA记录的查询居然有响应（其他相同的请求时没有的），于是最终还是怀疑到了DNS头上。我尝试取消勾选Openwrt里VLAN85接口的“自动获取DNS服务器”，然后保持`eth3.85`打开，此后Velop一直保持正常状态没有再变红。

![20220205170826_bvcqb6](https://res.cloudinary.com/core2duoe6420/image/upload/c_scale,w_500/v1644052201/posts/velop-red-issue/20220205170826_bvcqb6.png#center)

为了确认，把“自动获取DNS服务器”重新打开，几分钟后Velop指示灯又变红，并且在Openwrt的dnsmasq使用的上游DNS配置文件`/tmp/resolv.conf.d/resolv.conf.auto`中发现确实多出了IPTV专网DHCP获取到的DNS，并且是10开头的本地网段。经过测试后发现这个DNS服务器会响应请求，只是速度很慢，并且响应的地址会掺杂IPv6地址。我的电脑因为使用了旁路由，许多DNS请求被代理了，所以影响不大。爸妈那边也确实反映了网络不通畅，响应速度慢，只是我没有重视。

至此这个问题基本得到了解决。最后还是想吐槽Velop的软件做的是真的烂👎