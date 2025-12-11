---
title: "A Docker DNS Troubleshooting Story"
date: 2020-02-04
slug: docker-dns-troubleshooting
tags:
- docker
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

This afternoon a colleague came to me with a problem: a container couldn’t resolve domain names. I thought DNS issues wouldn’t be hard to fix… and then spent the next 8 hours troubleshooting.

<!--more-->

## First stage

The first step in checking DNS issues is of course to look at `/etc/resolv.conf`. OK, let’s take a look:

```
search localdomain
nameserver 127.0.0.11
options ndots:0
```

Huh, why is the DNS a strange `127.0.0.11`? OK, that’s not a huge problem either. Let’s see if any process is listening on port 53. Because the container didn’t have the tools we needed and DNS was broken so we couldn’t use the package manager, we had to find another way:

```bash
$ nsenter -t $(docker inspect -f {{.State.Pid}} b884630283ce) -n netstat -anu
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
udp        0      0 127.0.0.11:41287        0.0.0.0:*       
```

No process listening on port `53`, only a strange `41287`, and I had no idea who was listening on it.

That was odd, so I casually checked iptables:

```bash
$ nsenter -t $(docker inspect -f {{.State.Pid}} b884630283ce) -n iptables -t nat -L -n -v
(...省略无关内容）
Chain DOCKER_OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            127.0.0.11           tcp dpt:53 to:127.0.0.11:37711
   29  1884 DNAT       udp  --  *      *       0.0.0.0/0            127.0.0.11           udp dpt:53 to:127.0.0.11:41287
```

The strange `41287` port appeared, which explains the `netstat` output. The whole resolution process so far is:

1. The process sends a DNS request to `127.0.0.11:53`
2. `iptables` NATs packets sent to `127.0.0.11:53` to `127.0.0.11:41287`

So the question is: who is listening on port `41287`? It obviously can’t be a process inside the container. Time to bring out `lsof`:

```bash
$ nsenter -t $(docker inspect -f {{.State.Pid}} b884630283ce) -n lsof -i :41287
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
dockerd 7041 root   29u  IPv4 123505      0t0  UDP 127.0.0.11:41287 
```

There it is! It was actually `dockerd` listening on this port. Can a process listen in another network namespace? After checking the docs, it turns out it can: through the `setns` system call it can enter another `network namespace`. So it turns out container DNS requests work like this.

## Second stage

My colleague was getting anxious and wanted a working container ASAP to test with. So, we first changed `/etc/resolv.conf` to use the same DNS as the host, but it still didn’t work. After some digging we found that the host’s iptables rules were REJECTing the packets. We modified iptables to allow UDP 53, and the problem was solved. But that was only a temporary fix. Would we have to repeat this every time we restarted the container? Besides, the company’s Linux machines use Chef, and iptables rules get reset after a while. So this didn’t solve the root cause.

Let me first explain: the machine where we had the issue was running a single-node OpenShift deployment. Based on what I described in the first stage, this was the DNS request chain I *imagined*:

1. The process sends a DNS request to `127.0.0.11:53`
2. `iptables` NATs packets sent to `127.0.0.11:53` to `127.0.0.11:41287`, i.e., to the `dockerd` process
3. (My imagination) `dockerd` forwards the DNS request to the DNS server configured on the host, and receives the result
4. (My imagination) `dockerd` then returns the result back to `127.0.0.11:41287`
5. (My imagination) `iptables` rewrites `127.0.0.11:41287` back to `127.0.0.11:53`, and the process receives the DNS response, request complete

In this chain, steps 1, 2, 4, 5 are unlikely to fail, and the firewall wouldn’t interfere at all during the process, because step 4 is no different from a normal DNS request from the host, and the host’s DNS clearly works fine.

So I started to suspect whether Docker had a bug. I turned to Google and found quite a few issues:

- https://github.com/docker/for-linux/issues/179
- https://github.com/google/gvisor/issues/115

It seemed a lot of people had run into similar problems. Maybe it wasn’t even a Docker bug but a Docker Compose bug?

At this point, my colleague told me that one of his previous machines worked fine. At first I didn’t believe it, since they were all running the same Docker version. Later, I logged into that machine and saw that DNS was indeed working. So it wasn’t a Docker bug? But that working machine didn’t have OpenShift installed, so I suspected OpenShift might have changed Docker’s network configuration. I started frantically searching through various configurations like `/etc/sysconfig/docker`, `DOCKER_OPTS`, `/etc/docker/docker.json`, and found nothing.

## Third stage

By this time I was pretty desperate; I couldn’t find the cause at all. I wondered if there was a way to see all traffic from `dockerd` to check whether it actually received DNS requests. The problem is, as far as I know there isn’t a good tool on Linux to see a process’s network traffic. The only way is probably to use `strace` to look at all system calls, but the output would contain a lot of noise, and it would take a lot of patience to dig through it. Nothing ventured, nothing gained, so I tried:

```
$ strace -p 7041 -f -s 10000
```

Then I initiated a DNS request inside the container. `strace` printed out a bunch of stuff, but it was still manageable. After looking carefully, I found some crucial information!

```
(...unrelated content omitted)
[pid  7055] setns(38, CLONE_NEWNET) = 0
[pid  7055] socket(AF_INET, SOCK_DGRAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_IP) = 39
[pid  7055] setsockopt(39, SOL_SOCKET, SO_BROADCAST, [1], 4) = 0
[pid  7055] connect(39, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.29.2")}, 16) = 0
[pid  7055] getsockname(39, {sa_family=AF_INET, sin_port=htons(50886), sin_addr=inet_addr("172.18.0.2")}, [112->16]) = 0
[pid  7055] getpeername(39, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("192.168.29.2")}, [112->16]) = 0
[pid  7055] setns(10, CLONE_NEWNET)     = 0
[pid  7055] write(39, "w\364\1\0\0\1\0\0\0\0\0\0\00278\003143\003241\003199\7in-addr\4arpa\0\0\f\0\1", 45) = 45
(...unrelated content omitted)
```

😱😱😱😱😱😱😱😱😱😱😱😱

This is completely different from the step 3 I had imagined earlier! Based on these system calls, the real process is:

1. The process sends a DNS request to `127.0.0.11:53`
1. `iptables` NATs packets sent to `127.0.0.11:53` to `127.0.0.11:41287`, i.e., to the `dockerd` process
1. `dockerd` uses the `setns` system call to enter the container’s network namespace, and then sends the real DNS request to the DNS server. Since the request is sent from the container’s network namespace, the source IP address of the packet is the container’s IP
1. `dockerd` receives the DNS response and writes the result to `127.0.0.11:41287`
1. `iptables` rewrites `127.0.0.11:41287` back to `127.0.0.11:53`, and the process receives the DNS response, request complete

Why is this step 3 so crucial? Because in my earlier mental model, the DNS request was sent by the host, so there would be no firewall problems. But when it’s sent via the container’s network, it’s a different story—the firewall can block it. Remember in the second stage I modified iptables to allow UDP 53 as a quick fix for my colleague? It was exactly the same issue!

Later, using Wireshark and tcpdump, we confirmed that the DNS requests in step 3 were indeed sent from the container’s network interface.

Once we found the root cause, fixing it was not hard: just modify iptables. But because of Chef in our environment, iptables changes aren’t persistent. To fundamentally solve the problem, we had to specify an alternate DNS in Docker Compose instead—which happened to be exactly the solution mentioned in those two GitHub issues.