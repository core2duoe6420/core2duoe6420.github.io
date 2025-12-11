---
title:  "使用tun/tap将数据包导入协议栈"
date: 2014-10-22
slug: use-tun-tap
---

我原本以为寝室的电脑ping不通实验室的电脑是因为之间隔了一层NAT的关系，昨天听吴博说了才知道原来没有NAT，而是防火墙的关系。防火墙应该是丢弃了ICMP包和所有入站的`TCP SYN`包，所以外面的电脑无法通过TCP直接连接实验室电脑。跟吴博一番讨论后，萌发了尝试突破寝室电脑无法TCP连接到实验室电脑的限制。

<!--more-->

大致上的思路是通过某种渠道将`TCP SYN`数据包发到实验室电脑上，然后将数据包导入本机协议栈。这个渠道可以是一个第三方服务器，分别和实验室电脑和寝室电脑连接，作为一个中转站。因为我们测试后发现实验室防火墙没有封锁UDP数据包，所以直接用UDP来转发`TCP SYN`数据包。

问题的关键在于将应用层获得的数据包导入到本机协议栈，让实验室电脑的协议栈响应`TCP SYN ACK`握手包，这个包不会被防火墙拦截，而且防火墙在看到这个包之后会建立连接信息，之后的数据包就能顺利通过防火墙了。之前正好了解了一下OpenVPN关于`tun/tap`的实现原理，觉得非常相似，所以决定尝试一下。

有关`tun/tap`的资料，可以阅读IBM的这篇[文章](http://www.ibm.com/developerworks/cn/linux/l-tuntap/)，讲的还是挺不错的。就我个人的理解，在进程创建了一个`tun/tap`设备后，如果将路由表中的某一项路由到该虚拟设备后，当有数据包满足该路由条目后，内核就会将该数据包写入到`tun/tap`设备，相当于`tun/tap`的发送过程，而此时读取该`tun/tap`设备（`tun/tap`也实现了字节设备驱动）的进程就能获得这个要发送的数据包，此时可以对数据包进行处理后再通过其他渠道发送出去，OpenVPN就是对数据包进行加密后，再用UDP发送出去。而当进程向`tun/tap`设备写入时，就是`tun/tap`设备的接收过程，写入的数据会被`tun/tap`设备的虚拟网卡驱动作为正常的数据包放入协议栈处理。这正是我需要的功能。

## 环境
还是先介绍一下整个应用的环境，客户机是位于寝室的电脑，`Windows 7 x64`，使用`WinPcap`抓包，编译器是`Visual C++ 2013(VC130)`，IP地址`10.100.248.83`。服务器是位于实验室的电脑，`Ubuntu 14.04`（幸好是Linux，Windows的话虽然肯定也能做但我是不知道了），IP地址`10.10.90.192`，监听UDP端口10000。

## 先搞定客户机

客户机这边比较好搞定，所以先来介绍客户机的工作。主要的任务是用`WinPcap`抓取发往服务器的`TCP SYN`数据包，然后用UDP封装后发出去。过滤数据包的代码如下：

```c
pcap_compile(fp, 
    &fcode, 
    "dst host 10.10.90.192 and tcp[tcpflags] & (tcp-syn) != 0",
    0,
    netmask);
```

抓到数据包后的发送代码如下，因为做的是实验性的程序，所以就不要纠结性能和代码是不是优美了。

```c
void packet_handler(u_char * user, 
                    const struct pcap_pkthdr * header,
                    const u_char * data)
{
	SOCKET sockClient = socket(AF_INET, SOCK_DGRAM, 0);
	SOCKADDR_IN addrServ;
	addrServ.sin_addr.S_un.S_addr = inet_addr("10.10.90.192");
	addrServ.sin_family = AF_INET;
	addrServ.sin_port = htons(10000);
	/* 这里+14和-14是当服务器使用tun时的数据，因为tun不需要mac层
	 * 所以去掉，后面用tap的时候就可以去掉了
	 */
	sendto(sockClient, (const char *)(data + 14), 
	        header->caplen - 14, 0, 
	        (SOCKADDR*)&addrServ, sizeof(SOCKADDR));
}
```

## 服务器的TUN尝试

服务器这边有点复杂，我首先想到的是用`tun`做，因为`tun`工作在第三层，逻辑上简单一些。当然好多代码其实是与`tap`通用的，打开`tun/tap`设备的代码是从上面那篇IBM的文档中模仿的。

首先是打开`tun/tap`设备的代码，主要是通过`dev`名字区分是`tap`还是`tun`，内核通过`IFF_TUN`和`IFF_TAP`标志来区分：

```c
int tun_creat(const char * dev, char * actual, int size)
{
    struct ifreq ifr;
    int fd, err;
    if ((fd = open("/dev/net/tun", O_RDWR)) < 0)
        return -1;
    memset(&ifr, 0, sizeof(ifr));
    ifr.ifr_flags = IFF_NO_PI;
    if (!strncmp(dev, "tun", 3)) {
        ifr.ifr_flags |= IFF_TUN;
    } else if (!strncmp(dev, "tap", 3)) {
        ifr.ifr_flags |= IFF_TAP;
    } else {
        fprintf(stderr, "Device %s illegal\n", dev);
        close(fd);
        return -1;
    }
    if (strlen(dev) > 3)
        strncpy(ifr.ifr_name, dev, IFNAMSIZ);

    if ((err = ioctl(fd, TUNSETIFF, (void *)&ifr)) < 0) {
        fprintf(stderr, "Cannot ioctl TUNSETIFF %s\n", dev);
        close(fd);
        return err;
    }
    strncpy(actual, ifr.ifr_name, size);
    return fd;
}
```

创建完成后要将设备启动起来，相当于在shell中执行`ip link set xxx up`，通过`ioctl`设置标志来实现：

```c
int dev_bring_up(const char * dev)
{
    int sockfd;
    struct ifreq ifr;
    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
        return -1;
        
    memset(&ifr, 0, sizeof ifr);
    strncpy(ifr.ifr_name, dev, IFNAMSIZ);
    //先获取再设置
    if (ioctl(sockfd, SIOCGIFFLAGS, &ifr) < 0) {
        fprintf(stderr, "Error up %s\n", dev);
        return -1;
    }
    ifr.ifr_ifru.ifru_flags |= IFF_UP;
    if (ioctl(sockfd, SIOCSIFFLAGS, &ifr) < 0) {
        fprintf(stderr, "Error up %s\n", dev);
        return -1;
    }
    close(sockfd);
    return 0;
}
```

通过调用这两个函数来启动一个`tun`设备：

```c
    int tun;
    char tun_name[IFNAMSIZ];
    tun = tun_creat("tap10", tun_name, sizeof(tun_name));
    dev_bring_up(tun_name);
```

创建后启动`tun`设备后就可以监听UDP端口，每次读到一个数据报就将数据全部写入到`tun`设备中：

```c
    int sockfd;
    struct sockaddr_in servaddr, cliaddr;
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(10000);
    bind(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr));
    while (1) {
        char pkt[1500];
        int clilen = sizeof(cliaddr);
        int n = recvfrom(sockfd, pkt, sizeof(pkt), 0, 
                (struct sockaddr *)&cliaddr, &clilen);
        calc_checksum(pkt, n);
        /* 下面这行在后面用tap的时候需要去掉注释。
         * mac是一个6字节的char数组，应该在之前申明。
         */
        //memcpy(pkt, mac, 6);
        write(tun, pkt, n);
    }
```

这里的`calc_checksum`函数是用来计算IP校验和和TCP校验和的，因为客户机很有可能开启了各种`offload`功能，将校验和的计算工作交给了网卡做处理，所以WinPcap抓到的数据包是没有经过校验和计算的，我们需要自己计算也确保协议栈接收数据包。

到此程序大致已经成型，运行后，通过Wireshark抓包发现`tun`虚拟设备已经有接收到的数据了，但是协议栈并没有按照预期的响应`TCP SYN ACK`数据包。其中的原理我也不是很明白，不过估计是因为`10.10.90.192`这个IP是设置在`eth0`网卡上的，所以虽然这个IP地址确实是本机网卡地址，但因为与接收的网卡不同所以协议栈不接收。

## 使用tap

既然`tun`不行，那只能考虑`tap`，使用`tap`的好处是`tap`会虚拟出一块完整的以太网卡，而Linux内核支持虚拟网桥设备，如果把`eth0`和虚拟出来的`tap`都挂接到虚拟出的网桥`br0`上，把`10.10.90.192`这个IP分配给网桥`br0`，就可以让`tap`接收的数据包转发到`br0`上从而实现了目的IP与实际IP相同，这样协议栈就应该会接收这个数据包了。

首先要创建网桥，在shell中使用`brctl`命令：

```shell
brctl addbr br0                 #创建网桥
brctl addif br0 eth0            #将eth0先加入网桥
ifconfig br0 10.10.90.192 up    #启动网桥并分配IP
ifconfig eth0 0.0.0.0           #eth0现在不需要IP地址了
```

然后在代码中，创建并启动完`tap10`后，将它加入到网桥中，这里偷个懒直接用`system`函数了：

```c
    system("brctl addif br0 tap10");
```

调整所有代码中的IP首部的偏移量。最后一个要解决的问题是要在把数据包写入到`tap`之前，修改以太网首部的目的MAC地址为网桥的MAC地址，不然目的MAC地址与网桥MAC地址不一致，网桥仍然不会接受这个包。获取网络设备MAC地址的方法是用`SIOCGIFHWADDR`调用`ioctl`：

```c
/* mac应该是一个至少6字节的buffer */
int dev_get_mac(const char * dev, char * mac)
{
    int sockfd;
    struct ifreq ifr;
    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
        return -1;

    memset(&ifr, 0, sizeof ifr);
    strncpy(ifr.ifr_name, dev, IFNAMSIZ);
    if (ioctl(sockfd, SIOCGIFHWADDR, &ifr) < 0) {
        fprintf(stderr, "Error get mac %s\n", dev);
        return -1;
    }
    memcpy(mac, &ifr.ifr_hwaddr.sa_data, 6);
    close(sockfd);
    return 0;
}
```

这次编译运行后，结果就如同我预期的那样，寝室电脑与服务器电脑已经可以顺利建立起TCP连接了，整个实验宣告成功。

最后说一下几个使用虚拟网桥的问题，在创建了`br0`之后，好多直接挂在`eth0`上的设置要变为`br0`，例如原来的默认路由表项是：

    default via 10.10.90.1 dev eth0
    
现在要修改为：
    
    default via 10.10.90.1 dev br0
    
原本我使用OpenVPN做nat时的设置是：

    iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
    
现在要修改为：

    iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o br0 -j MASQUERADE
    
因为原本的`eth0`已经没有地址了，`MASQUERADE`的结果就变成空的了，也就不会做nat了。