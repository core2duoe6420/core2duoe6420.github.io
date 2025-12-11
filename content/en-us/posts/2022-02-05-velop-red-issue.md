---
title: "Fixing the Linksys Velop Red Status Light While Network Stays Fine"
date: 2022-02-05
slug: velop-red-issue
coverImage: https://res.cloudinary.com/core2duoe6420/image/upload/v1644055415/posts/velop-red-issue/velop_tztin8.jpg
coverMeta: out
thumbnailImagePosition: right
thumbnailImage: https://res.cloudinary.com/core2duoe6420/image/upload/v1644055415/posts/velop-red-issue/sf217443-002_en_v8_rsxhyt.png
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

During the earlier process of [tinkering with IPTV]({{< ref "2022-02-03-iptv-proxy" >}}), I noticed that the status light on the Linksys Velop used as an AP turned red, but the network on my computer was completely fine. At first I thought it was caused by `igmpproxy`, but later found that the issue remained even after stopping `igmpproxy`. This post documents how I solved this problem.

<!--more-->

After a series of power cycles and restarts, I found the issue followed this pattern:

- After bringing up the VLAN 85 interface `eth3.85`, the Velop’s status light would turn red in about 5 minutes
- After bringing down `eth3.85`, the Velop’s status light would return to normal blue within about 15 seconds

At this point, the only guess was that the Velop was sending out some kind of network probe, and enabling `eth3.85` caused that probe to fail. However, the Velop’s software is really terrible and is a complete black box. The default management UI exposes almost no information, so there was no way to know what that probe actually was. My initial suspicion was that the probe’s target server happened to fall within the IPTV private network `23.3.0.0/17`, causing the traffic to be pulled into the IPTV network and thus fail.

After some Googling, I found that there is very little information online about the Velop, and only located the following posts on reddit:

- [Red lights but all appears to be working](https://www.reddit.com/r/LinksysVelop/comments/iu31w3/red_lights_but_all_appears_to_be_working/)
- [Linksys Velop Red Light Despite Devices Connected Just Fine](https://www.reddit.com/r/pihole/comments/gn7rtf/linksys_velop_red_light_despite_devices_connected/)
- [linksys velop wired backhaul](https://www.reddit.com/r/LinksysVelop/comments/ip0h3k/linksys_velop_wired_backhaul/)

In fact, the first two threads already revealed that the cause of this problem is very likely related to the `heartbeat.belkin.com` domain, but I still took quite a few detours.

The third link above mentioned that `sysinfo.cgi` can be used to view some internal logs on the Velop. I tried it and it worked, but this CGI script simply runs a number of commands at access time and echoes their output, while the key log file `/var/log/message` only shows 200 lines. To track the logs, I wrote a Python script to poll them; the script is available [here](https://gist.github.com/core2duoe6420/3361c0ed7d9a654bd056f3c953d10767).

From the logs, the entries I cared about included:

- `service.devidentd Ethernet Agent: EthUtil returns multiple MACs` — this line appears repeatedly in large quantities, but Google turned up nothing
- `service.status Subscriber: Subject 'network/AC4C4927-CC4A-0BF1-B850-302303D68097/WLAN/neighbors' matches subscribed topic 'network/+/WLAN/neighbors'`
- `lldpd.status lldpd event lldp::update  received.`

The last two entries led me to a strange hypothesis: could there be some kind of WAN notification mechanism where the router tells the Velop “I just added a network,” such as LLDP like in the third log line? Of course, it turned out this was complete nonsense 🤣.

Since the logs didn’t yield anything useful, I could only rely on packet capture to see what communication actually occurred between OpenWrt and the Velop before and after toggling `eth3.85`. So I started tcpdump on `eth0`, the port connected to the Velop, and then analyzed the trace in Wireshark.

Initially, I focused on the `heartbeat.belkin.com` domain mentioned earlier, but found that when the Velop light turned red, both DNS and ping to this domain were normal.

![20220205165652_bdlds6](https://res.cloudinary.com/core2duoe6420/image/upload/v1644051527/posts/velop-red-issue/20220205165652_bdlds6.png)

So I started to suspect that this was not the probe I was looking for. I then spent over an hour trying to identify differences in traffic between the two situations, but got nowhere. Until I accidentally noticed that one particular AAAA DNS query to `heartbeat.belkin.com` actually received a response (while the others of the same query did not). This finally pointed my suspicion back to DNS. I tried unchecking “Use DNS servers advertised by peer” for the VLAN 85 interface in OpenWrt, then kept `eth3.85` up; after that, the Velop stayed normal and never turned red again.

![20220205170826_bvcqb6](https://res.cloudinary.com/core2duoe6420/image/upload/c_scale,w_500/v1644052201/posts/velop-red-issue/20220205170826_bvcqb6.png#center)

To confirm, I checked “Use DNS servers advertised by peer” again, and a few minutes later the Velop light turned red. In OpenWrt’s upstream DNS configuration file used by dnsmasq, `/tmp/resolv.conf.d/resolv.conf.auto`, I indeed found an extra DNS server obtained via DHCP from the IPTV network, and it was in a local 10.x.x.x range. Testing showed that this DNS server did respond to queries, but it was very slow, and the responses were mixed with IPv6 addresses. Because my computer uses a bypass router, many DNS queries were proxied, so I wasn’t affected much. My parents, however, had indeed complained that the network was unstable and slow to respond, I just hadn’t taken it seriously.

At this point, the problem was basically solved. To wrap up, the Velop’s software really is garbage 👎