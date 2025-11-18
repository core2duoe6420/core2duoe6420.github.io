---
title: "OpenWrt 硬件流量卸载导致MAC地址被缓存"
date: 2025-11-18
slug: flowtable-mac-issue
hide_site_title: true
tags:
- openwrt
- flowtable
---

又双叒叕踩坑了。这回的坑是我尝试把原本Proxmox的虚拟机ImmortalWrt（OpenWrt的一个编译版本）迁移到Docker中，同时保留IP地址不变的过程中遇到的。这个ImmortalWrt里跑着我的WireGuard服务。迁移很顺利，启动Docker容器后，可以正常ping通，我的手机也能连上WireGuard，唯独一个24小时开机的节点死活连不上，表现为无法握手，WireGuard里一直显示接收为0KB，一个包都收不到。

<!--more-->

尝试打开WireGuard的内核日志：

```bash
echo "module wireguard +p" | sudo tee /sys/kernel/debug/dynamic_debug/control
```

但是日志里只显示超时导致握手失败，并没有额外的信息。

这个容器跑在我[上文](../docker-host-bridge-network/)介绍的桥接的宿主网络中，整个网络拓扑包含一个bridge，一对veth，以及一个macvlan，IP地址为`192.168.1.240`。尝试用tcpdump在链路中的各个节点上抓包，发现其实是有响应包的，在macvlan的parent网卡上都能抓到响应包，但是一进容器包就消失地无影无踪。

百思不得其解后问了ChatGPT，GPT说如果MAC地址与任何macvlan的虚拟网卡的MAC地址不符，就会被丢包。我当时想的是这怎么可能，毕竟ping是正常的，而且有设备连接成功，说明ARP工作正常。事后证明ChatGPT是对的，这确实是造成WireGuard连不上的原因，而我浪费了大量排障时间，直到实在无计可施，死马当活马医比对了一下MAC地址，才发现了端倪。

之前运行tcpdump时没有加上`-e`参数，加上之后就能很明显看到，来回数据包的MAC地址是不一致的：

```
$ sudo tcpdump -i veth-macvlan1 -n -vvvv port 51820 -e
tcpdump: listening on veth-macvlan1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
13:16:12.183767 3a:ea:10:f9:a1:2e > bc:24:11:2b:dd:87, ethertype IPv4 (0x0800), length 190: (tos 0x88, ttl 64, id 7184, offset 0, flags [none], proto UDP (17), length 176)
    192.168.1.240.51820 > x.x.x.x.51820: [bad udp cksum 0x11d5 -> 0xecab!] UDP, length 148
13:16:12.201030 bc:24:11:2b:dd:87 > b6:01:52:44:16:fa, ethertype IPv4 (0x0800), length 134: (tos 0x0, ttl 52, id 32270, offset 0, flags [none], proto UDP (17), length 120)
    x.x.x.x.51820 > 192.168.1.240.51820: [udp sum ok] UDP, length 92
```

这就很奇怪了，其它连接都正常，唯独这个节点有问题，于是问题定位到发出这个数据包的主路由，另一台ImmortalWrt上找问题。又由于问题仅出现在这条连接上，怀疑是不是又和conntrack有关。在主路由上看conntrack：

```
root@r-main:~# conntrack -L | grep 51820
udp      17 src=x.x.x.x dst=192.168.0.254 sport=51820 dport=51820 packets=5418 bytes=2996332 src=192.168.1.240 dst=x.x.x.x sport=51820 dport=51820 packets=3649 bytes=1587316 [OFFLOAD] mark=0 use=2
conntrack v1.4.8 (conntrack-tools): 395 flow entries have been shown.
```

删除这条记录后，WireGuard就能成功连上了。但是一旦重建容器后就又会连不上，并且收到的MAC地址变成了上一次的MAC地址。到这里这个问题可以被稳定复现，触发的条件就是MAC地址发生了变化，因此可以推断出，主路由上一定有什么地方缓存了MAC地址。查看`ip neigh show`的结果一切正常，并没有已经失效的MAC地址。

在我的认知范围内，并没有什么东西会缓存MAC地址如此长的时间，只能在网上瞎搜。这时候前面conntrack日志里的`[OFFLOAD]`引起了我的注意，这个flag之前在Ubuntu里没有见过，然后就顺藤摸瓜找到了flowtable这个nftables引入的新功能。具体的介绍可以看这两篇文章：

- [Netfilter’s flowtable infrastructure](https://docs.kernel.org/networking/nf_flowtable.html)
- [Flowtables - Part 1: A Netfilter/Nftables Fastpath](https://thermalcircle.de/doku.php?id=blog:linux:flowtables_1_a_netfilter_nftables_fastpath)

在第一篇内核文档中，有这样一段话：

> The flowtable behaves like a cache. The flowtable entries might get stale if either the destination MAC address or the egress netdevice that is used for transmission changes.
> This might be a problem if:
> - You run the flowtable in software mode and you combine bridge and IP forwarding in your setup.
> - Hardware offload is enabled.

这倒是和我遇到的问题一致。ImmortalWrt（我的版本是24.10.4）的相关设置在防火墙中：

{{<image classes="center fig-50 clear" src="https://res.cloudinary.com/core2duoe6420/image/upload/v1763485317/posts/flowtable-mac-issue/firewall_fc42gf.png" alt="firewall">}}

这个流量卸载类型就是关键了，不知道为啥设置成了硬件流量卸载，我用的i350-T4网卡应该是不支持这个功能的。实验后得知，设置为“硬件流量卸载”后，nftables会出现：

```
flowtable ft {
    hook ingress priority filter
    devices = { eth0, eth1, eth2, eth3, eth4 }
    flags offload
    counter
}

chain forward {
    type filter hook forward priority filter; policy drop;
    meta l4proto { tcp, udp } flow add @ft
    ...
}
```

而当设置为“软件流量卸载”时，会出现：

```
flowtable ft {
    hook ingress priority filter
    devices = { br-lan, eth4 }
    counter
}
```

实际上这里设置成“硬件流量卸载”也应该是不生效的，根据文档，如果硬件流量卸载生效，conntrack中应该显示`HW_OFFLOAD`，而不是`OFFLOAD`。我这边不管设置成软件还是硬件，conntrack里现实的都是`OFFLOAD`。但是神奇的是，设置成硬件就会出现针对长期UDP连接（因为WireGuard每隔几秒就会尝试握手，所以这条UDP连接在conntrack里永远不会超时）的MAC地址被缓存的问题，但是设置成软件就不会有这个问题了。

这次排障也算是让我知道了flowtable这个新功能，因为之前已经习惯了iptables，一直都不想去再学习新的nftables，现在nftables已经是主流了，是该走出舒适区接触新东西了。