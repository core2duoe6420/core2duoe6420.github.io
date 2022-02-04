---
title: "折腾在电脑上看IPTV"
date: 2022-02-03
slug: iptv-proxy
tags:
- network
---

老爸退休在家没有事做，最近不知为何喜欢上了在电脑上多开视频同时观看，问我能不能在电脑上看电视频道（主要是五星体育）。我之前就有看到过一些公共的IPTV频道可以直接用Potplayer播放，也知道有把电信的IPTV转发到电脑上的方案，只是我自己不看电视没有动力去折腾。最近算是尝试了一下。

<!--more-->

最简单的方案无疑是网上找公共的IPTV清单，比如这个https://github.com/iptv-org/iptv 是真的牛逼，全球的电视频道几乎都能看。但是很遗憾，有几个关键频道很卡，观看体验极差。于是只能折腾电信的IPTV。

基本原理也不复杂，只要有一个接口加入85 VLAN，通过DHCP拿到一个IP，就能访问到电信的IPTV专网，接下来就是想办法让电脑可以接收到IGMP流量。

## 获取专网IP

家里目前的网络结构是光猫改了桥接，光猫上的端口都绑定了85 VLAN：

![1643886244221](https://res.cloudinary.com/core2duoe6420/image/upload/v1643914587/posts/iptv-proxy/1643886244221_f3mmhk.png)

拨号由软路由负责，WAN口名字是`eth3`，软件是Koolshare的`LEDE v2.37`。在`LEDE`的网络配置里新增这么几行：

```
config interface 'VLAN85'
	option proto 'dhcp'
	option ifname 'eth3.85'
        option defaultroute '0'
	option multipath 'off'
	option peerdns '0'
```

保存后就看到新接口自动获取了电信的专网IP：

![1643886480817](https://res.cloudinary.com/core2duoe6420/image/upload/v1643914804/posts/iptv-proxy/1643914741498_pa5kqi.jpg)

## 获取IPTV流量

这块在网上的信息非常杂乱，好多文章把好几种方式混在一起说，其实完全不必。我总结了一下，其实主要有3种方法，任何一种都能正常看到IPTV，当然每种方案都有利有弊：

|方案|使用此方案的地址|优点|缺点|
|---|---|---|---|
|`udpxy`|使用http单播`http://192.168.1.1:4022/udp/239.45.3.210:5140`|配置方便，没有广播风暴|切频道的等待时间较长|
|`igmpproxy`|使用rtp组播`rtp://239.45.3.210:5140`|切频道的等待时间短|即使开启`IGMP Snooping`，若下流设备不支持依然会有广播风暴|
|使用回看地址|使用rtsp单播`rtsp://124.75.34.37:554/PLTV/...434_0.smil`|支持回看|不支持回看的频道就看不了了|

这三种方案我都试了一下，简要说下配置过程。

### `udpxy`

`udpxy`的原理是监听一个端口，协议为HTTP，根据URL的规则解析出要连接的组播地址和接口，然后进行流量转发。这里的地址将会是IPTV频道的IGMP组播地址。

`Openwrt`对`udpxy`提供了相当好的支持。首先通过以下命令安装：

```
opkg update
opkg install udpxy luci-app-udpxy
```

然后编辑`/etc/config/udpxy`

```
config udpxy
        option respawn '1'
        option verbose '0'
        option status '1'
        option port '4022'
        option disabled '0'
        option bind 'br-lan'
        option source 'eth3.85'
```

这里比较关键的是`bind`和`source`两个选项，`bind`是监听内网的接口，`source`就是配置专网IP的接口。`udpxy`启动后可以通过`http://192.168.1.254:4022/status`来查看状态。

使用`udpxy`的另一个优点是几乎不用动防火墙，不需要支持转发，不需要专网接口的NAT，只需要接受LAN网络的入站即可（这个一般都是开启的，否则根本访问不了`LEDE`）。

配置好以后就可以用Potplayer打开一个地址试试能否播放了。

### 回看地址

这个方案主要来自[这个仓库](https://github.com/lucifersun/China-Telecom-ShangHai-IPTV-list)。这个仓库里的地址在两三年前似乎是可以公网访问的，也就是说任何人只要有网都能看。但是后来被电信封杀了，只能走电信的IPTV专网才能访问到。另外，这些地址不是直播地址，而是回放地址，所以也不知道实际延时落后多少，以下说明来自作者原话：

>  这个播放列表使用IPTV的频道 回放 功能。IPTV直播用的是专网组播，无法直接通过Internet播放。因为不是所有频道都支持回放，所以这个列表里的频道 必然少于 IPTV的直播频道。

这个方案的原理也很简单，我们只要让需要访问的IP走IPTV专网就可以了。

首先，在`LEDE`的防火墙里，打开专网的“IP伪装”（NAT），设置转发为接受。设置LAN到专网的转发为接受。

然后需要添加几条路由让这些地址走IPTV专网而不是默认网关。命令格式如下：

```
ip route add 124.75.34.0/24 via 23.x.x.x dev eth3.85
```

其中`124.75.34.0/24`的部分有好多条，需要通过Wireshark抓包一次次跳转尝试，我这边只试了一个频道，跳转了2次，一共3个地址。[这个Issue](https://github.com/lucifersun/China-Telecom-ShangHai-IPTV-list/issues/28)里有人总结了几个地址可以作为参考。

`23.x.x.x`这个部分要用专网的网关代替，这个信息正常应该是从DHCP中获得的，但是我没有找到DHCP的日志，所以只能靠盲猜，使用本网段广播地址-1的最高地址，居然成功了。

这个方案要正式投入使用还需要一些工程方面的工作，例如收集IP段，写个脚本应对DHCP获得的地址变化的情况，所以并不会投入实际使用。


### igmpproxy

这个方案害我折腾了好久。它的大致原理在[Openwrt文档](https://openwrt.org/docs/guide-user/network/wan/udp_multicast)里写的挺清楚了，大致上是在路由器上跑一个`igmpproxy`，它使用`raw socket`监听所有IP协议为`2（IGMP）`的数据包，发现有请求加入IP组的包后，就由自己向专网发送加入IP组的包，然后添加一条IP路由，剩下的流量就由路由表负责了。

首先安装：

```
opkg update
opkg install igmpproxy
```

然后编辑配置文件`/etc/config/igmpproxy`

```
config igmpproxy
        option quickleave 1

config phyint
        option network VLAN85
        option zone VLAN85
        option direction upstream
        list altnet 0.0.0.0/0

config phyint
        option network lan
        option zone lan
        option direction downstream
        list whitelist 239.45.0.0/16 # 这个很坑，下面说
```

最后使用`service igmpproxy restart`启动即可。

关于这个配置文件，在文档里说明得比较详细了，这个`zone`不能省略，因为启动脚本会通过这个字段来添加几条iptables规则，使得UDP流量可以顺利通过。

防火墙方面，这个方案不需要IP伪装（NAT），正常情况下转发规则会在`igmpproxy`启动时（一定要通过脚本启动）被添加进防火墙，如果没法正常工作，那么可以尝试打开LAN和专网之间的转发。

配置好后，使用Potplayer打开`rtp://239.45.3.210:5140`应该就能播放了。

接下来说一下这个方案坑的地方。

首先是[这个Issue](https://github.com/pali/igmpproxy/issues/61)里说的问题，`igmpproxy`好像会转发`224.0.0.0/24`的流量，这个地址是本地流量不应该被转发。最新版的`igmpproxy`加入了对`whitelist`配置的支持，就是我上面配置文件中加入的那行，然而，`Openwrt`的配置工具没有支持这个选项，所以那行加了等于白加。~~这个问题导致了另一个问题就是Velop不认网络了，指示灯直接变红（联网并没有问题），垃圾领势软件做的太差。~~（2月5日更新：这个问题得到了解决，不是由`igmpproxy`造成的，之后再写文章说）

其次就是导致我折腾了半天的原因，但这其实不是`igmpproxy`本身的问题，而是我电脑系统的问题。`IGMP`在加入一个IP组时，会由客户端发出一个Membership Report的数据包，像这样：

![1643890616060](https://res.cloudinary.com/core2duoe6420/image/upload/v1643914587/posts/iptv-proxy/1643890616060_gvmmpg.png)

然而在我的电脑上这个包死活是不发的。我使用虚拟机启动了一台Windows和Ubuntu，测试后总算确认不是`igmpproxy`的配置问题，因为虚拟出来的系统都能正常工作，只有宿主机自己死活不发，关掉Windows防火墙也没有用。这个问题最终也没有解决，不想折腾了。

## EPG和KODI

总之最后还是决定使用`udpxy`的方案。在研究的过程中发现有人提到了EPG，全称Electronic Program Guide，配合KODI可以实现很不错的效果：

![1643890989724](https://res.cloudinary.com/core2duoe6420/image/upload/v1643914588/posts/iptv-proxy/1643890989724_jgk2ex.png)


这个主要就是些配置活，可以参考[KODI的文档](http://www.kodiplayer.cn/course/2925.html)，还有[这楼里](https://github.com/lucifersun/China-Telecom-ShangHai-IPTV-list/issues/28#issuecomment-778612495)老哥给出的频道列表。

## 参考资料

- 不能用KODI看的IPTV不是真4K https://post.smzdm.com/p/alx7n2ke/
- 电信公网疑似已屏蔽回放源IP https://github.com/lucifersun/China-Telecom-ShangHai-IPTV-list/issues/28
- 给Kodi直播电视添加电子节目单EPG 电视节目指南 http://www.kodiplayer.cn/course/2925.html
- 频道抓包结果（包括节目名称） https://github.com/lucifersun/China-Telecom-ShangHai-IPTV-list/issues/35
- 上海电信iptv最全的组播地址 12月5日新鲜出炉未分类 https://www.right.com.cn/forum/thread-724974-1-1.html
- Openwrt与IPTV之一——igmpproxy https://www.cnblogs.com/harryzwh/p/4279316.html
- Openwrt与IPTV之二——udpxy https://www.cnblogs.com/harryzwh/p/4279335.html
- IPTV共享之填坑篇+组播地址 https://wp.gxnas.com/4058.html
- 「IPTV 与互联网融合」全解析 igmpproxy udpxy IGMP RTSP 抓包 直播源 四川 电信 https://www.right.com.cn/forum/thread-2470633-1-1.html
- IPTV融合进普通网络一般步骤 https://www.right.com.cn/forum/thread-248400-1-1.html
- 利用OpenWrt系统拓宽IPTV使用范围 https://www.red-yellow.net/利用OpenWrt系统拓宽IPTV使用范围.html
- How IGMP operates https://techhub.hpe.com/eginfolib/networking/docs/switches/K-KA-KB/15-18/5998-8164_mrg/content/ch01s08.html
- IPTV / UDP multicast https://openwrt.org/docs/guide-user/network/wan/udp_multicast
- Filter all local multicast https://github.com/pali/igmpproxy/issues/61