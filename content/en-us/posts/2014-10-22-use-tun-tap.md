---
title:  "Importing Packets into the Protocol Stack with tun/tap"
date: 2014-10-22
slug: use-tun-tap
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

I originally thought the reason the dorm PC couldn’t ping the lab PC was that there was a layer of NAT in between. After talking with Wu Bo yesterday, I realized there was actually no NAT; it was a firewall issue. The firewall should be dropping ICMP packets and all inbound `TCP SYN` packets, so an external machine cannot establish a TCP connection directly to the lab machine. After some discussion with Wu Bo, I got the idea to try to break through the restriction that the dorm machine cannot establish TCP connections to the lab machine.

<!--more-->

The general idea is to send `TCP SYN` packets to the lab machine via some channel, and then inject those packets into the local protocol stack. This channel could be a third‑party server that connects to both the lab and dorm machines and acts as a relay. Our tests showed that the lab firewall does not block UDP packets, so we can simply use UDP to forward `TCP SYN` packets.

The key problem is injecting packets obtained at the application layer into the local protocol stack, so that the protocol stack on the lab machine will respond with a `TCP SYN ACK`. That packet will not be blocked by the firewall; moreover, once the firewall sees this packet it will create connection state, and subsequent packets will pass through the firewall. I had recently read about how OpenVPN implements `tun/tap`, which felt very similar, so I decided to try it.

For information on `tun/tap`, you can read this [article](http://www.ibm.com/developerworks/cn/linux/l-tuntap/) from IBM; it’s quite good. As I understand it, after a process creates a `tun/tap` device, if you route a certain routing entry to that virtual device, then whenever a packet matches that route, the kernel will write the packet into the `tun/tap` device. This is equivalent to the sending path of `tun/tap`. The process that reads from the `tun/tap` device (since `tun/tap` is also implemented as a character device driver) will then obtain the packet that is about to be sent, and can process it and forward it via some other channel. OpenVPN, for example, encrypts the packet and then sends it over UDP. When the process writes to the `tun/tap` device, that is the receive path of `tun/tap`: the data written will be handed by the virtual NIC driver of `tun/tap` to the protocol stack as a normal packet. This is exactly what I need.

## Environment

Let me first describe the environment of the whole setup. The client is the dorm PC, running `Windows 7 x64`, using `WinPcap` for packet capture, compiled with `Visual C++ 2013(VC130)`, IP address `10.100.248.83`. The server is the lab PC, running `Ubuntu 14.04` (fortunately Linux; with Windows it could certainly be done too, but I don’t know how), IP address `10.10.90.192`, listening on UDP port 10000.

## Getting the client done first

The client side is easier to implement, so I’ll start there. Its main task is to use `WinPcap` to capture `TCP SYN` packets destined for the server, then encapsulate them in UDP and send them out. The code for filtering packets is as follows:

```c
pcap_compile(fp, 
    &fcode, 
    "dst host 10.10.90.192 and tcp[tcpflags] & (tcp-syn) != 0",
    0,
    netmask);
```

The sending code after capturing a packet is shown below. Since this is an experimental program, let’s not worry about performance or code elegance.

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
	/* The +14 and -14 below are for the case where the server uses tun,
	 * because tun does not need the MAC layer, so we strip it.
	 * When using tap later, this can be removed.
	 */
	sendto(sockClient, (const char *)(data + 14), 
	        header->caplen - 14, 0, 
	        (SOCKADDR*)&addrServ, sizeof(SOCKADDR));
}
```

## TUN attempt on the server

The server side is a bit more complicated. My first thought was to use `tun`, because `tun` works at layer 3 and is conceptually simpler. Of course, much of the code is actually shared with `tap`. The code to open the `tun/tap` device was modeled after the IBM article above.

First is the code to open a `tun/tap` device. The type (`tap` vs `tun`) is distinguished by the `dev` name; the kernel uses the flags `IFF_TUN` and `IFF_TAP` to tell them apart:

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

After creating the `tun/tap` device, you need to bring it up, which is equivalent to running `ip link set xxx up` in the shell, implemented by setting flags via `ioctl`:

```c
int dev_bring_up(const char * dev)
{
    int sockfd;
    struct ifreq ifr;
    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) < 0)
        return -1;
        
    memset(&ifr, 0, sizeof ifr);
    strncpy(ifr.ifr_name, dev, IFNAMSIZ);
    // Get flags first, then set
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

Use these two functions to create and bring up a `tun` device:

```c
    int tun;
    char tun_name[IFNAMSIZ];
    tun = tun_creat("tap10", tun_name, sizeof(tun_name));
    dev_bring_up(tun_name);
```

Once the `tun` device is created and brought up, you can listen on the UDP port and write each received datagram entirely into the `tun` device:

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
        /* The following line needs to be uncommented later when using tap.
         * mac is a 6‑byte char array that should be declared earlier.
         */
        //memcpy(pkt, mac, 6);
        write(tun, pkt, n);
    }
```

Here, the `calc_checksum` function is used to compute the IP checksum and TCP checksum. The client machine is very likely to have various offload features enabled, delegating checksum computation to the NIC. Therefore, the packets captured by WinPcap may not have checksums calculated. We need to compute them ourselves to ensure the protocol stack will accept the packets.

At this point, the program is roughly in place. After running it and capturing with Wireshark, I found that the `tun` virtual device is indeed receiving packets, but the protocol stack is not responding with `TCP SYN ACK` packets as expected. I don’t fully understand the underlying reason, but my guess is that the IP address `10.10.90.192` is configured on the `eth0` interface, so although this IP address is indeed a local NIC address, the protocol stack will not accept packets destined for it that arrive on a different interface.

## Using tap

Since `tun` doesn’t work, we have to consider `tap`. The advantage of using `tap` is that it emulates a full Ethernet NIC. The Linux kernel supports virtual bridge devices, so if we attach both `eth0` and the virtual `tap` to a bridge interface `br0`, and assign the IP address `10.10.90.192` to the bridge `br0`, then packets received by `tap` can be forwarded to `br0`. This makes the destination IP match the actual IP, so the protocol stack should accept the packets.

First, create a bridge with the `brctl` command in the shell:

```shell
brctl addbr br0                 # create a bridge
brctl addif br0 eth0            # add eth0 to the bridge first
ifconfig br0 10.10.90.192 up    # bring up br0 and assign IP
ifconfig eth0 0.0.0.0           # eth0 no longer needs an IP address
```

Then in the code, after creating and bringing up `tap10`, add it to the bridge. I take a shortcut here and just use `system`:

```c
    system("brctl addif br0 tap10");
```

Adjust all IP header offsets in the code. The last issue to solve is that before writing the packet to `tap`, we need to modify the destination MAC address in the Ethernet header to the MAC address of the bridge. Otherwise, if the destination MAC does not match the bridge’s MAC, the bridge will not accept the packet. To obtain the MAC address of a network device, use `ioctl` with `SIOCGIFHWADDR`:

```c
/* mac should be a buffer of at least 6 bytes */
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

This time, after compiling and running, the result is exactly as I expected: the dorm PC and the server PC can successfully establish a TCP connection. The experiment is a success.

Finally, a few notes about using virtual bridges. After `br0` is created, many settings that were originally attached directly to `eth0` must be moved to `br0`. For example, the original default route was:

    default via 10.10.90.1 dev eth0
    
Now it needs to be changed to:

    default via 10.10.90.1 dev br0
    
Previously, when I used OpenVPN to do NAT, the setting was:

    iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
    
Now it has to be changed to:

    iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o br0 -j MASQUERADE
    
Because the original `eth0` no longer has an address, the result of `MASQUERADE` would otherwise be empty, and NAT would not be performed.