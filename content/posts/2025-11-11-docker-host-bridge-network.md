---
title: "Docker桥接宿主机网络"
date: 2025-11-11
slug: docker-host-bridge-network
hide_site_title: true
tags:
- docker
---

本文副标题为我的野路子Docker网络环境，主要介绍我在配置Docker桥接宿主机网络过程中遇到的坑。

Docker的bridge网络是一个内部网络，与外界的通信都要通过三层路由和NAT转发，这在某些应用中不是很方便。而如果要桥接宿主机网络，最简单的方式是使用macvlan。但是macvlan有不能和宿主机通信的缺陷，并不是完美解。

<!--more-->

我的宿主机有多块网卡，所以很早就把这些网卡都配置进了一个bridge里，就像OpenWRT里的`br-lan`那样，这为我绕过macvlan的限制提供了条件。思路也很简单，macvlan不能直接和宿主机通信，那我就新建一对veth，一个加进宿主机bridge，另一个作为macvlan的parent，这样macvlan不能和veth通信也无所谓。这个方案在绝大多数情况下工作地很好，我的HomeAssistant和UniFi-Controller用的都是这个方案。

看过我之前DSM那篇文章的读者会知道，我还有一个WebVirtCloud，我希望在容器里的虚拟机也可以桥接到宿主网络中，这时候macvlan就傻眼了，因为macvlan虚拟出来的子接口不能再被放进bridge里，也就是不能桥接。为了解决问题，我的方案是再创建一个Docker网络，然后生成一对veth，分别加入到主bridge和新的Docker bridge，然后在容器启动之后，手动修改IP地址和默认路由。这个方案很野，需要容器有NET_ADMIN的CAP，而且要写脚本配置IP，但是能用。

回想起来，当时操作这个方案的时候就遇到了本文想要介绍的问题，但是当时不知怎么想到的解决方案，总之就是解决了，但是没有彻底搞明白，也没有记录下来，导致我今天又重新栽了一遍跟头。

事情是这样的，之前的野路子是因为我没有找到让Docker直接用上已经创建好的bridge的办法。这两天心血来潮又搜了一下，发现了[这个Issue](https://github.com/moby/libnetwork/issues/2310)，这两年Docker也有进步，好像可以支持使用已有的bridge了，于是就试了下，然后又遇到了奇怪的网络问题。

首先用netplan（我的宿主机是Ubuntu 24.04）定义bridge和veth，大致如下：

```yaml
network:
  version: 2
  bridges:
    # 这是我的宿主机bridge，省略了无关内容
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
    # 这是给容器桥接用的
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

然后用以下命令创建Docker网络：

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

接下来在创建容器的时候指定网络`lan137`并且定好IP就好了，用docker-compose的话类似于：

```
services:
  homeassistant:
    # 其它内容
    networks:
      lan137:
        ipv4_address: 192.168.137.248
networks:
  lan137:
    external: true
    name: lan137
```

启动之后就发现问题了，容器的网络基本通畅，和局域网内的设备都能通信，但唯独无法连接宿主机上通过Docker暴露的服务（就是普通Docker bridge的容器，通过`--publish`暴露的端口，大多数容器用的都是这个），例如`192.168.137.250:5000`，但是ping又是能ping通的。在ChatGPT的大力帮助下，花了一天时间研究这个问题。之所以会花这么长时间，是因为这个故障包含了四个小问题，甚至最后一个问题还没有完全弄明白。

## 问题1：`bridge-nf-call-iptables`

在我的设想中，数据包经过`br-lan137`后，直接通过veth转发到`br-mellanox`上，然后进入宿主机网络栈，通过iptables DNAT到容器IP（网段为`172.18.0.0/16`）。但是因为`net.bridge.bridge-nf-call-iptables = 1`，所以这个数据包在`br-lan137`的时候就已经进入了iptables的处理流程中。这一点不难想到，但是我怀疑网络不通是因为防火墙的关系，一直在找和`br-lan137`相关的防火墙规则，怎么都找不到。后来才意识到，关键不是防火墙，而是进入iptables栈后，在`br-lan137`这里就直接触发了`PREROUTING`的`DNAT`，因为这条规则并不区分来源设备：

```text
$ sudo iptables -t nat -L -n -v
Chain PREROUTING (policy ACCEPT 928K packets, 63M bytes)
 pkts bytes target     prot opt in     out     source               destination
 5376  320K DOCKER     0    --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 RETURN     0    --  docker0 *       0.0.0.0/0            0.0.0.0/0
 1738  104K RETURN     0    --  br-da66fb6805df *       0.0.0.0/0            0.0.0.0/0
 1780  107K DNAT       6    --  !br-da66fb6805df *       0.0.0.0/0            0.0.0.0/0            tcp dpt:5000 to:172.18.0.8:5000
```

因此数据包在`br-lan137`就已经被DNAT到了`172.18.0.8`，并且出口设备应该是`br-da66fb6805df`。这也是为什么我找防火墙规则一直找不到的原因。

## 问题2：`FORWARD`规则

搞清问题1后，找到对应的防火墙规则并不难。为了方便阅读，精简了输出，只包含了重要部分：

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

这些规则的主要目的就是Docker将不同的网络（也就是bridge）互相隔离。但是加入一条规则绕过iptables后，网络依旧不通，这就引出了第三个问题。

## 问题3：`rp_filter`

`rp_filter`是为了确保数据包的SRC IP是本机可路由的IP，主要为了防止DDOS。默认的`rp_filter=2`，也就是比较宽松的策略，只需要SRC IP能通过本机的任意网口路由即可，那在我这个案例中，`192.168.137.0/24`本身就是宿主机网络，肯定是可以路由的，所以我一开始始终无法理解这个问题怎么就和`rp_filter`扯上关系。但是跑以下命令禁用`rp_filter`后，TCP就能握手成功了，所以不得不信：

```bash
sysctl -w net.ipv4.conf.all.rp_filter=0
sysctl -w net.ipv4.conf.br-lan137.rp_filter=0
```

后来ChatGPT提到了`fib_validate_source()`这个函数，顺藤摸瓜找到了[这篇文章](https://github.com/centurycoder/martian_source)，这才明白不配IP的来源设备也会触发`rp_filter`，而在我的设想中`br-lan137`只是一个二层设备，不需要IP。

验证方法也很简单，随便给`br-lan137`加个IP即可，IP是啥不重要。

在解决防火墙和`rp_filter`之后，迎来了本次排障最难理解的地方，网络依旧不通，但症状变了，TCP可以成功握手，但是连接立刻就被reset掉：

```text
# telnet 192.168.137.250 5000
Connected to 192.168.137.250
^C

# curl http://192.168.137.250:5000
curl: (56) Recv failure: Connection reset by peer
```

## 问题4：`conntrack`

前面提到，`bridge-nf-call-iptables=1`会导致数据包在经过bridge的时候调用iptables hooks，当包从`192.168.137.248`发往`192.168.137.250`的时候，它在第一跳`br-lan137`就被DNAT到了`172.18.0.8`，但是返回的时候，还是会先经过`br-mellanox`，再到`br-lan137`，两次经过bridge都会调用iptables hook，然后就不知道在哪里造成了奇怪的conntrack冲突，以下是conntrack的日志：

```text
$ sudo conntrack -E --output extended,id | grep 5000 
[NEW] ipv4 2 tcp 6 120 SYN_SENT src=192.168.137.248 dst=192.168.137.250 sport=48076 dport=5000 [UNREPLIED] src=172.18.0.8 dst=192.168.137.248 sport=5000 dport=48076 id=1994087345 
[UPDATE] ipv4 2 tcp 6 60 SYN_RECV src=192.168.137.248 dst=192.168.137.250 sport=48076 dport=5000 src=172.18.0.8 dst=192.168.137.248 sport=5000 dport=48076 id=1994087345 
[UPDATE] ipv4 2 tcp 6 432000 ESTABLISHED src=192.168.137.248 dst=192.168.137.250 sport=48076 dport=5000 src=172.18.0.8 dst=192.168.137.248 sport=5000 dport=48076 [ASSURED] id=1994087345 
[NEW] ipv4 2 tcp 6 300 ESTABLISHED src=192.168.137.250 dst=192.168.137.248 sport=5000 dport=48076 [UNREPLIED] src=192.168.137.248 dst=192.168.137.250 sport=48076 dport=37926 id=1423083704 
[DESTROY] ipv4 2 tcp 6 300 CLOSE src=192.168.137.250 dst=192.168.137.248 sport=5000 dport=48076 [UNREPLIED] src=192.168.137.248 dst=192.168.137.250 sport=48076 dport=37926 id=1423083704 
[UPDATE] ipv4 2 tcp 6 10 CLOSE src=192.168.137.248 dst=192.168.137.250 sport=48076 dport=5000 src=172.18.0.8 dst=192.168.137.248 sport=5000 dport=48076 [ASSURED] id=1994087345
```

前3行输出都很正常，conntrack记录了`192.168.137.248:48076 => 192.168.137.250:5000, 172.18.0.8:5000 => 192.168.137.248:48076`的连接，到了第4条，src和dst反过来了，变成了`192.168.137.250:5000 => 192.168.137.248:48076`，实际上这应该正是`br-mellanox`和`br-lan137`看到的包，但不知道为什么conntrack没有把它和之前的连接关联到一起，而是新建了一条连接，又大概因为五元组相同造成了冲突，conntrack重写了端口`5000 => 37926`，直接导致`192.168.137.248`收到的包的src端口是37926而不是5000，于是回复了一个RST包，这个RST包在经过bridge后又变回了正常端口的包，于是双方都RST了。

但是为什么握手能成功，发送数据时就会失败呢，和ChatGPT聊了几个来回也没有答案，只说“在特定hash组合下被误判成新流”，真正的答案估计只有研究conntrack代码才能知道了。

## 解决方案

上面写了这么多，其实是为了搞清楚真正的故障原因还有数据包链路，加深对网络的理解。而要解决这个问题其实很简单，让conntrack不要追踪`br-lan137`即可，所以启动的时候跑一下这条命令问题就解决了：

```bash
iptables -t raw -I PREROUTING 1 -i br-lan137 -j NOTRACK
```

不需要配置`bridge-nf-call-iptables`，不需要修改Docker的防火墙规则，也不需要给`br-lan137`配个IP地址，桥接网络完美运行。

## 结束语

虽然没有完美能完美解答所有问题，但在排障的过程中还是学到了一些新东西，相信在不断积累后总会找到答案。