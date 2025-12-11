---
title: "Docker Bridging to Host Network"
date: 2025-11-11
slug: docker-host-bridge-network
hide_site_title: true
tags:
- docker
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

The subtitle of this post could be “My Hacky Docker Networking Setup”. It mainly describes the pitfalls I ran into when configuring Docker to bridge to the host network.

Docker’s bridge network is an internal network; all communication with the outside world has to go through L3 routing and NAT, which is inconvenient for some applications. If you want to bridge to the host network, the simplest approach is to use macvlan. However, macvlan has the drawback that it cannot communicate with the host, so it’s not a perfect solution.

<!--more-->

My host has multiple NICs, so a long time ago I put them all into a single bridge, similar to `br-lan` in OpenWRT. This setup gave me a way to work around macvlan’s limitation. The idea is simple: macvlan can’t talk directly to the host, so I create a veth pair, put one end into the host bridge, and use the other end as the parent for the macvlan. In that case, it doesn’t matter that the macvlan can’t talk to the veth. This setup works very well in most cases; both my HomeAssistant and UniFi-Controller use this scheme.

Readers of my previous DSM post will know that I also have a WebVirtCloud. I wanted the VMs inside that container to also be bridged directly onto the host network. That’s where macvlan fails, because interfaces created by macvlan cannot be added into a bridge, i.e., they cannot be bridged again. To solve this, my approach was to create another Docker network, then create a veth pair, adding one end to the main bridge and the other end to the new Docker bridge. After the container starts, I manually modify the IP address and default route. This solution is quite hacky: it requires the container to have the NET_ADMIN capability and a script to configure IP addresses, but it works.

In retrospect, when I first implemented this hacky solution, I already ran into the very issues described in this post. Somehow I came up with a workaround at the time; the problem was solved, but I never fully understood it, nor did I document it. As a result, today I stepped into the same trap again.

Here’s what happened. My previous hack existed because I hadn’t found a way to let Docker directly use an already-created bridge. Recently I decided to search again and found [this issue](https://github.com/moby/libnetwork/issues/2310). Docker has improved in the past couple of years and now seems to support using an existing bridge, so I tried it—and immediately hit some weird network problems again.

First, I defined the bridge and veth with netplan (my host is Ubuntu 24.04), roughly as follows:

```yaml
network:
  version: 2
  bridges:
    # This is my host bridge; unrelated content omitted
    br-mellanox:
      addresses:
        - 192.168.137.250/24
      interfaces:
        - eno1
        - veth-lan137-2
      nameservers:
        addresses:
          - 192.168.137.245
        search:
          - hljin.net
      routes:
        - to: default
          via: 192.168.137.245
    # This is for container bridging
    br-lan137:
      interfaces:
        - veth-lan137-1
  virtual-ethernets:
    veth-lan137-1:
      peer: veth-lan137-2
    veth-lan137-2:
      peer: veth-lan137-1
 ethernets:
    eno1:
      optional: true
      dhcp4: false
```

Then I created the Docker network with:

```bash
docker network create \
    --driver bridge \
    --subnet 192.168.137.0/24 \
    --gateway 192.168.137.245 \
    -o com.docker.network.bridge.enable_icc=true \
    -o com.docker.network.bridge.enable_ip_masquerade=false \
    -o com.docker.network.bridge.gateway_mode_ipv4=routed  \
    -o com.docker.network.driver.mtu=1500 \
    -o com.docker.network.bridge.name=br-lan137 \
    -o com.docker.network.bridge.inhibit_ipv4=true \
    lan137
```

Next, when creating containers, I just specify the `lan137` network and assign a fixed IP. With docker-compose it looks like:

```yaml
services:
  homeassistant:
    # other config
    networks:
      lan137:
        ipv4_address: 192.168.137.248
networks:
  lan137:
    external: true
    name: lan137
```

After starting, a problem surfaced: from the container, networking was mostly fine; it could talk to all devices on the LAN, except it couldn’t connect to services exposed on the host via Docker (i.e., containers on the default Docker bridge, with ports published using `--publish`, which is how most of my containers are set up). For example, `192.168.137.250:5000` was unreachable from the container, although ping worked.

With extensive help from ChatGPT, I spent an entire day investigating. That took so long because this failure was actually composed of four smaller issues, and I still don’t fully understand the last one.

## Issue 1: `bridge-nf-call-iptables`

In my mental model, the packet should go through `br-lan137`, then be forwarded via the veth to `br-mellanox`, then enter the host network stack and be DNATed by iptables to the container IP (in `172.18.0.0/16`). However, because `net.bridge.bridge-nf-call-iptables = 1`, the packet already enters the iptables pipeline when it hits `br-lan137`. This part is not hard to reason about, but I initially suspected that the connectivity issue was caused by firewall rules, so I kept searching for rules related to `br-lan137` and found nothing. Only later did I realize the key wasn’t filtering, but that the packet had already hit the `PREROUTING` `DNAT` on `br-lan137`, because that rule does not distinguish by input device:

```text
$ sudo iptables -t nat -L -n -v
Chain PREROUTING (policy ACCEPT 928K packets, 63M bytes)
 pkts bytes target     prot opt in     out     source               destination
 5376  320K DOCKER     0    --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     0    --  docker0 *       0.0.0.0/0            0.0.0.0/0
 1738  104K RETURN     0    --  br-da66fb6805df *       0.0.0.0/0            0.0.0.0/0
 1780  107K DNAT       6    --  !br-da66fb6805df *       0.0.0.0/0            0.0.0.0/0            tcp dpt:5000 to:172.18.0.4:5000
```

So the packet is already DNATed to `172.18.0.4` at `br-lan137`, and its outgoing device should be `br-da66fb6805df`. That’s why I couldn’t find any relevant firewall rule for `br-lan137`.

## Issue 2: `FORWARD` rules

Once Issue 1 was clear, locating the relevant firewall rules wasn’t hard. For clarity, here’s a condensed version showing only the important parts:

```text
$ sudo iptables -L -n -v
Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
7238K 3417M DOCKER-FORWARD  0    --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-FORWARD (1 references)
 pkts bytes target     prot opt in     out     source               destination
6716K 3069M DOCKER-ISOLATION-STAGE-1  0    --  *      *       0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
 pkts bytes target     prot opt in     out     source               destination
  227 13620 DOCKER-ISOLATION-STAGE-2  0    --  br-lan137 !br-lan137  0.0.0.0/0            0.0.0.0/0
 631K   64M DOCKER-ISOLATION-STAGE-2  0    --  br-da66fb6805df !br-da66fb6805df  0.0.0.0/0            0.0.0.0/0

Chain DOCKER-ISOLATION-STAGE-2 (3 references)
 pkts bytes target     prot opt in     out     source               destination
  218 13080 DROP       0    --  *      br-da66fb6805df  0.0.0.0/0            0.0.0.0/0
    0     0 DROP       0    --  *      br-lan137  0.0.0.0/0            0.0.0.0/0
```

The main purpose of these rules is to isolate different Docker networks (bridges) from one another. However, even after adding rules to bypass iptables, the network was still not working, which leads to the third issue.

## Issue 3: `rp_filter`

`rp_filter` ensures that the source IP of a packet is routable from this host, mainly to mitigate DDoS attacks. The default `rp_filter=2` is a relatively loose policy: the SRC IP only needs to be routable via any interface on the host. In my case, `192.168.137.0/24` is the host’s own network, so it is definitely routable. I therefore couldn’t see how `rp_filter` could be related to this problem. But after disabling `rp_filter`, TCP handshakes started to succeed, so I had to accept it:

```bash
sysctl -w net.ipv4.conf.all.rp_filter=0
sysctl -w net.ipv4.conf.br-lan137.rp_filter=0
```

Later, ChatGPT mentioned the `fib_validate_source()` function, which led me to [this article](https://github.com/centurycoder/martian_source). That’s where I learned that an interface without an IP can also trigger `rp_filter`, whereas in my mental model `br-lan137` was purely a L2 device and didn’t need an IP.

The verification method is simple: assign any random IP to `br-lan137`; the specific IP doesn’t matter.

After fixing the firewall and `rp_filter`, I ran into the most confusing part of this entire troubleshooting session. The network still didn’t work, but the symptoms changed: TCP could handshake successfully, but the connection was immediately reset:

```text
# telnet 192.168.137.250 5000
Connected to 192.168.137.250
^C

# curl http://192.168.137.250:5000
curl: (56) Recv failure: Connection reset by peer
```

## Issue 4: `conntrack`

As mentioned earlier, when `bridge-nf-call-iptables=1`, packets crossing a bridge go through the iptables hooks. When a packet is sent from `192.168.137.248` to `192.168.137.250`, it is DNATed on the first hop (`br-lan137`) to `172.18.0.4`. On the way back, the packet first passes `br-mellanox`, then `br-lan137` again. Both passes across a bridge trigger iptables hooks, causing conntrack conflicts. Here’s the conntrack log:

```text
$ sudo conntrack -E --output extended,id | grep 5000 
   [NEW] ipv4     2 tcp      6 120 SYN_SENT src=192.168.137.248 dst=192.168.137.250 sport=55094 dport=5000 [UNREPLIED] src=172.18.0.4 dst=192.168.137.248 sport=5000 dport=55094 id=1952199411
 [UPDATE] ipv4     2 tcp      6 60 SYN_RECV src=192.168.137.248 dst=192.168.137.250 sport=55094 dport=5000 src=172.18.0.4 dst=192.168.137.248 sport=5000 dport=55094 id=1952199411
 [UPDATE] ipv4     2 tcp      6 432000 ESTABLISHED src=192.168.137.248 dst=192.168.137.250 sport=55094 dport=5000 src=172.18.0.4 dst=192.168.137.248 sport=5000 dport=55094 [ASSURED] id=1952199411
    [NEW] ipv4     2 tcp      6 300 ESTABLISHED src=192.168.137.250 dst=192.168.137.248 sport=5000 dport=55094 [UNREPLIED] src=192.168.137.248 dst=192.168.137.250 sport=55094 dport=19460 id=1724107866
[DESTROY] ipv4     2 tcp      6 300 CLOSE src=192.168.137.250 dst=192.168.137.248 sport=5000 dport=55094 [UNREPLIED] src=192.168.137.248 dst=192.168.137.250 sport=55094 dport=19460 id=1724107866
 [UPDATE] ipv4     2 tcp      6 10 CLOSE src=192.168.137.248 dst=192.168.137.250 sport=55094 dport=5000 src=172.18.0.4 dst=192.168.137.248 sport=5000 dport=55094 [ASSURED] id=1952199411
```

Look at the first three lines: conntrack records a connection with `ORIGIN = 192.168.137.248:55094 => 192.168.137.250:5000`, `REPLY = 172.18.0.4:5000 => 192.168.137.248:55094`. When the packet exits the Docker bridge, is routed and comes out of `br-mellanox`, it is SNATed in POSTROUTING back to `192.168.137.250`. So far, everything is normal. But the fourth line is problematic: a new connection appears with `ORIGIN = 192.168.137.250:5000 => 192.168.137.248:55094`, `REPLY = 192.168.137.248:55094 => 192.168.137.250:19460`. Note that the source port has been rewritten to `19460`. Capturing packets inside the container confirms this:

![wireshark](https://res.cloudinary.com/core2duoe6420/image/upload/v1762948869/posts/docker-host-bridge-network/docker-bridge-wireshark_jfeay6.png)

Once you understand how conntrack works, it’s not hard to see why the port is rewritten. In conntrack, src and dst are not interchangeable, so `192.168.137.248:55094 => 192.168.137.250:5000` and `192.168.137.250:5000 => 192.168.137.248:55094` are two distinct tuples. When the packet passes `br-lan137`, conntrack sees a brand-new connection. When conntrack creates a new connection, it also needs to insert a REPLY tuple, but the REPLY tuple for this ORIGIN already exists in the hash table. That conflict leads to the port rewrite. Since the client cannot match this packet to its existing socket (due to the changed port), it responds with RST. When this RST reaches `br-lan137`, the port is rewritten back to the “normal” `5000`, so the server receives a RST and in turn sends another RST, closing the connection.

With help from Cursor, I located the line of code that rewrites the source port [here](https://github.com/torvalds/linux/blob/24172e0d79900908cf5ebf366600616d29c9b417/net/netfilter/nf_nat_core.c#L688). Since no iptables NAT rule matches this packet, the `nf_nat_alloc_null_binding` logic is used. While researching, I also found [this article](https://blog.csdn.net/dog250/article/details/112691374), which is quite interesting.

The part that really puzzled me—and took another full day of reading, going back and forth with Cursor, and even using `bpftrace`—was this: why does the TCP handshake succeed, but as soon as data is sent, a new connection is created and the source port gets rewritten? After all, during the handshake `br-lan137` also sees `192.168.137.250:5000 => 192.168.137.248:55094`. Why wasn’t the port rewritten right then?

After reading [this series on conntrack](https://thermalcircle.de/doku.php?id=blog:linux:connection_tracking_3_state_and_examples), I finally understood. If conntrack considers the return handshake packet (`SYN+ACK`) INVALID, it won’t mark the connection (set `skb->_nfct`). Without a conntrack mark, the packet won’t be NATed. Conntrack does not drop INVALID packets by itself, and I didn’t have iptables rules dropping them either, so the packet passes through normally. Logging INVALID packets confirms this:

```text
$ sudo iptables -I FORWARD -m conntrack --ctstate INVALID  -i br-lan137 -j LOG --log-prefix "CT INVALID OUT: " --log-level 4
$ journalctl -k | grep "CT INVALID" | grep 55094

Nov 12 19:46:36 nas kernel: CT INVALID OUT: IN=br-lan137 OUT=br-lan137 PHYSIN=veth-lan137-1 PHYSOUT=veth5a44ead MAC=72:ff:6f:af:95:56:98:03:9b:c4:d3:04:08:00 SRC=192.168.137.250 DST=192.168.137.248 LEN=60 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=5000 DPT=55094 WINDOW=65160 RES=0x00 ACK SYN URGP=0
Nov 12 19:46:36 nas kernel: CT INVALID OUT: IN=br-lan137 OUT=br-lan137 PHYSIN=veth-lan137-1 PHYSOUT=veth5a44ead MAC=72:ff:6f:af:95:56:98:03:9b:c4:d3:04:08:00 SRC=192.168.137.250 DST=192.168.137.248 LEN=40 TOS=0x00 PREC=0x00 TTL=63 ID=0 DF PROTO=TCP SPT=5000 DPT=55094 WINDOW=0 RES=0x00 RST URGP=0
```

So, is it INVALID for conntrack to see a `SYN+ACK` without having seen the initial `SYN`? The answer is in [`nf_conntrack_proto_tcp.c`](https://github.com/torvalds/linux/blob/24172e0d79900908cf5ebf366600616d29c9b417/net/netfilter/nf_conntrack_proto_tcp.c). Look at this state transition table:

```c
#define sNO TCP_CONNTRACK_NONE
#define sSS TCP_CONNTRACK_SYN_SENT
#define sSR TCP_CONNTRACK_SYN_RECV
#define sES TCP_CONNTRACK_ESTABLISHED
#define sFW TCP_CONNTRACK_FIN_WAIT
#define sCW TCP_CONNTRACK_CLOSE_WAIT
#define sLA TCP_CONNTRACK_LAST_ACK
#define sTW TCP_CONNTRACK_TIME_WAIT
#define sCL TCP_CONNTRACK_CLOSE
#define sS2 TCP_CONNTRACK_SYN_SENT2
#define sIV TCP_CONNTRACK_MAX
#define sIG TCP_CONNTRACK_IGNORE

/*
...
 * Packets marked as INVALID (sIV):
 *	if we regard them as truly invalid packets
 */
static const u8 tcp_conntracks[2][6][TCP_CONNTRACK_MAX] = {
	{
/* ORIGINAL */
/* 	     sNO, sSS, sSR, sES, sFW, sCW, sLA, sTW, sCL, sS2	*/
/*syn*/	   { sSS, sSS, sIG, sIG, sIG, sIG, sIG, sSS, sSS, sS2 },
...
/* 	     sNO, sSS, sSR, sES, sFW, sCW, sLA, sTW, sCL, sS2	*/
/*synack*/ { sIV, sIV, sSR, sIV, sIV, sIV, sIV, sIV, sIV, sSR },
/*
 *	sNO -> sIV	Too late and no reason to do anything
 *	sSS -> sIV	Client can't send SYN and then SYN/ACK
 *	sS2 -> sSR	SYN/ACK sent to SYN2 in simultaneous open
 *	sSR -> sSR	Late retransmitted SYN/ACK in simultaneous open
 *	sES -> sIV	Invalid SYN/ACK packets sent by the client
 *	sFW -> sIV
 *	sCW -> sIV
 *	sLA -> sIV
 *	sTW -> sIV
 *	sCL -> sIV
 */
...
/* 	     sNO, sSS, sSR, sES, sFW, sCW, sLA, sTW, sCL, sS2	*/
/*ack*/	   { sES, sIV, sES, sES, sCW, sCW, sTW, sTW, sCL, sIV },
/*
 *	sNO -> sES	Assumed.
 *	sSS -> sIV	ACK is invalid: we haven't seen a SYN/ACK yet.
 *	sS2 -> sIV
 *	sSR -> sES	Established state is reached.
 *	sES -> sES	:-)
 *	sFW -> sCW	Normal close request answered by ACK.
 *	sCW -> sCW
 *	sLA -> sTW	Last ACK detected (RFC5961 challenged)
 *	sTW -> sTW	Retransmitted last ACK. Remain in the same state.
 *	sCL -> sCL
 */
```

Now it’s clear: for this flow, the initial state is `sNO`, since conntrack hasn’t seen anything before. Upon receiving `SYN+ACK`, the state transitions to `sIV` (INVALID). Subsequent data packets are pure `ACK`s; from `sNO`, an `ACK` transitions to `sES` (ESTABLISHED), which matches the conntrack log and results in a new connection state, which in turn triggers NAT and port rewriting.

## Solution

All of the above was mainly to understand the true root cause and the packet path, and to deepen my understanding of networking. Fixing the issue is actually straightforward: just tell conntrack not to track traffic on `br-lan137`. So during startup, run:

```bash
iptables -t raw -I PREROUTING 1 -i br-lan137 -j NOTRACK
```

With this, there’s no need to tweak `bridge-nf-call-iptables`, no need to modify Docker’s firewall rules, and no need to assign an IP to `br-lan137`. The bridged networking then works perfectly.

## Conclusion

This issue troubled me for two full days. At one point I almost thought I wouldn’t find the answer anytime soon, but in the end I managed to figure it out. Of course, I still don’t fully understand all the details of conntrack, netfilter, or nftables; I usually only dig into them when I run into a problem. But with tools like ChatGPT and Cursor, common problems can be pinpointed directly, and even for uncommon ones they provide useful ideas. When source code is available, they can help analyze it too, which makes learning far more efficient than before. While debugging this conntrack problem, the AI also helped me write some `bpftrace` scripts. I’ve long heard of eBPF’s power but had never used it—this time I finally got to try it. Even though eBPF wasn’t the key to the final insight, it really is a powerful tool.