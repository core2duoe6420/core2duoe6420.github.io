---
title: "记一次Docker DNS排障"
date: 2020-02-04
slug: docker-dns-troubleshooting
tags:
- docker
---

今天下午同事来找我，说遇到了一个容器内无法解析域名的问题，我心想DNS问题不难解决，然后就开始了长达8小时的排障过程。

<!--more-->

## 第一阶段

查DNS问题第一步肯定是看`/etc/resolv.conf`，OK，瞅一瞅

```
search localdomain
nameserver 127.0.0.11
options ndots:0
```

咦，怎么DNS是奇怪的`127.0.0.11`？OK，这也不是什么大问题，看看有没有进程监听了53端口，因为容器里没有所需要的工具，DNS又坏了用不了包管理工具，只能另谋他法：

```bash
$ nsenter -t $(docker inspect -f {{.State.Pid}} b884630283ce) -n netstat -anu
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
udp        0      0 127.0.0.11:41287        0.0.0.0:*       
```

咦，没有进程在监听`53`端口，只有一个奇怪的`41287`，也不知道是谁监听的。

有点奇怪，顺手看一看iptables

```bash
$ nsenter -t $(docker inspect -f {{.State.Pid}} b884630283ce) -n iptables -t nat -L -n -v
(...省略无关内容）
Chain DOCKER_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            127.0.0.11           tcp dpt:53 to:127.0.0.11:37711
   29  1884 DNAT       udp  --  *      *       0.0.0.0/0            127.0.0.11           udp dpt:53 to:127.0.0.11:41287
```

出现了这个奇怪的`41287`端口，这样就能解释`netstat`的输出内容了，整个解析过程到此是：

1. 进程向`127.0.0.11:53`发出DNS请求
2. `iptables`将发送到`127.0.0.11:53`的数据包NAT到`127.0.0.11:41287`

那么问题来了，谁在监听`41287`端口？显然不可能是容器内的进程啊。祭出`lsof`大法：

```bash
$ nsenter -t $(docker inspect -f {{.State.Pid}} b884630283ce) -n lsof -i :41287
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
dockerd 7041 root   29u  IPv4 123505      0t0  UDP 127.0.0.11:41287 
```

出现了！居然是`dockerd`在监听这个端口。一个进程还能在监听在别的network namespace里？查资料后发现还真可以，通过`setns`系统调用可以进入其他`network namespace`，原来容器的DNS请求是这样的。

## 第二阶段

同事有点急，想赶紧先弄出个可以运行的容器让他测试，OK，那就先把`/etc/resolv.conf`改成和宿主机一样的DNS，结果发现还是不行，一番排查后发现是宿主机的iptables设置把包给REJECT了，改一改iptables，给UDP 53放行，问题解决。但是这是应急措施，下次重新启动容器难道再做一遍？况且公司的Linux机器装了Chef，iptables的设置过一段时间就被重置了。这样不解决根本问题。

这里先说明下，我们遇到问题的机器部署了Single node Openshift。按照第一阶段所描述的，我设想的整个DNS请求链路应该是：

1. 进程向`127.0.0.11:53`发出DNS请求
2. `iptables`将发送到`127.0.0.11:53`的数据包NAT到`127.0.0.11:41287`，也就是`dockerd`进程
3. （我YY的）`dockerd`将DNS请求转发到宿主机设置的DNS服务器上，并接受结果
4. （我YY的）`dockerd`再将结果返回到`127.0.0.11:41287`
5. （我YY的）`iptables`再将`127.0.0.11:41287`重写回`127.0.0.11:53`，进程接收到DNS响应，请求完毕

在这个链路里，1 2 4 5都不太会出问题，整个过程中也不会有防火墙介入，因为第4步就跟宿主机普通的DNS请求没什么两样，而宿主机的DNS显然是正常的。

那么就开始怀疑是不是Docker有bug？Google大法走起，找到了不少帖子：

- https://github.com/docker/for-linux/issues/179
- https://github.com/google/gvisor/issues/115

似乎遇到类似问题的人挺多的？有可能还不是Docker的bug是Docker compose的bug？

这时同事又跟我说他之前的一台机器是正常的，我一开始我还不相信，因为用的都是同一个版本的Docker。后来我登上去看了一下，发现还真的是好的。那么就不是Docker的bug咯？不过好的机器没有装Openshift，于是怀疑是不是Openshift改了Docker的网络配置，开始疯狂找各种
`/etc/sysconfig/docker`，`DOCKER_OPTS`，`/etc/docker/docker.json`各种配置，一无所获。

## 第三阶段

这时候就很绝望了，完全找不到问题的原因。我开始想有没有办法可以看到`dockerd`的全部流量，来看看它到底有没有收到DNS请求。问题是据我所知，Linux下也没有很好的工具可以看到一个进程的网络流量，唯一的办法大概是用`strace`看所有的系统调用，不过输出会包含很多无用信息，排查需要很大的耐心。死马当活马医，试试：

```
$ strace -p 7041 -f -s 10000
```

然后在容器里发起一个DNS请求，`strace`输出了一堆内容，不过还算能接受，仔细看了一下发现了非常关键的信息！

```
(...省略无关内容）
[pid  7055] setns(38, CLONE_NEWNET) = 0
[pid  7055] socket(AF_INET, SOCK_DGRAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_IP) = 39
[pid  7055] setsockopt(39, SOL_SOCKET, SO_BROADCAST, [1], 4) = 0
[pid  7055] connect(39, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.29.2")}, 16) = 0
[pid  7055] getsockname(39, {sa_family=AF_INET, sin_port=htons(50886), sin_addr=inet_addr("172.18.0.2")}, [112->16]) = 0
[pid  7055] getpeername(39, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.29.2")}, [112->16]) = 0
[pid  7055] setns(10, CLONE_NEWNET)     = 0
[pid  7055] write(39, "w\364\1\0\0\1\0\0\0\0\0\0\00278\003143\003241\003199\7in-addr\4arpa\0\0\f\0\1", 45) = 45
(...省略无关内容）
```

😱😱😱😱😱😱😱😱😱😱😱😱

这跟我之前想的第3步完全不一样！根据这段系统调用，真正的过程是这样的：

1. 进程向`127.0.0.11:53`发出DNS请求
1. `iptables`将发送到`127.0.0.11:53`的数据包NAT到`127.0.0.11:41287`，也就是`dockerd`进程
1. `dockerd`通过`setns`系统调用进入容器的network namespace，然后再向真正的DNS服务器发出请求，由于请求是从容器的network namespace发出的，请求包的IP地址是容器的IP地址
1. `dockerd`收到DNS响应，将结果写入`127.0.0.11:41287`
1. `iptables`再将`127.0.0.11:41287`重写回`127.0.0.11:53`，进程接收到DNS响应，请求完毕

为什么这里的第3步很关键呢，因为按照我之前想的，DNS请求由宿主机发出，不会有防火墙问题，但是通过容器的网络发出就不一样了，会被防火墙拦截，还记得第二阶段里我为了给同事一个快速的解决方案修改了iptables给UDP 53放行吗，一模一样的问题！

之后用Wireshark和tcpdump抓包证实了第3步中的DNS请求确实是从容器网卡发出的。

找到root cause后解决这个问题也不难，修改iptables就行，不过因为公司里Chef的关系，修改iptables无法持久化，为了根本性的解决问题，还是只能在Docker compose里指定一个别的DNS来解决了（恰好就是之前那两个github帖子里的解决方案）。