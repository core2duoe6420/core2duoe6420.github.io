---
title: "部署Headscale组建私有Tailscale网络"
date: 2025-12-07
slug: deploy-headscale
hide_site_title: true
tags:
- headscale
- headplane
- tailscale
- openwrt
---

我的私人家庭网络最早用的是Zerotier，但是发现连接不稳定，即使我的两个节点都有公网IP有时也会连不上，后来换了WireGuard就非常稳定。但最近新买了几个VPS，又有朋友家加入了网络，节点变多，安全规则也更复杂，WireGuard维护起来有点力不从心了，于是下定决心部署Headscale切换到Tailscale网络。
<!--more-->

网上关于Headscale和Tailscale的资料不少，但很多都是重复的，而且不少还在用裸机部署Headscale，大段的介绍怎么用systemd配置自动启动，看得我无语。另外，有些资料可能过时或者没有说清楚，甚至Tailscale的官方文档都不一定完全正确。最后，Headscale的实现并没有包含Tailscale的所有功能，很容易踩坑。本文是半记录半教程性质，会介绍我的部署方式和配置，但细节需要读者自己理会并且随机应变。

## 环境和方案

本文介绍的部署环境和方案如下：

- Headscale版本`0.27.1`；选用headplane作为UI管理界面，版本`0.6.1`；Docker版本`29.1.1`。该版本的Headscale要求客户端tailscale至少是`1.64.0`，目前的tailscale最新版是`1.90.9`
- Headscale部署在海外服务器，使用内建的DERP服务；同时在境内再起一台DERP服务器加速连接。主要是因为我的域名没有备案，没法把Headscale直接部署在境内。我尝试过自签证书并且直接使用IP，Ubuntu上可以添加系统CA，没有问题，但我有客户端需要用OpenWrt，死活加不上证书，只能放弃
- 本文例子中会使用以下参数，请读者根据自己的情况自行调整：

    - Headscale+Headplane服务器
    
        - Headscale域名`hs.example.com`
        - IP地址`48.1.1.1`
        - HTTPS监听在`40000`
        - 内建DERP的STUN监听在UDP端口`41000`
        - 其它`metrics`，`grpc`端口都忽略
        - 使用Caddy反代Headscale和Headplane。Caddy通过ACME获取证书，使用DNS-01 challenge，域名放在Cloudflare，API key是`cloudflare-dns-api-key`
        - Headplane对外域名是`hp.example.com`，复用端口`40000`，同时我启用了mTLS保证安全，读者可以自行决定需不需要mTLS

    - 境内DERP服务器

        - IP地址`47.1.1.1`
        - HTTPS监听在`40000`
        - STUN监听在`41000`
        - 不需要HTTP端口
        - 不需要开放ICMP（官方文档说需要，但实测不需要也能跑）
        - 使用自签证书

## Headscale服务器配置

准备以下配置文件，本文给出的配置文件限于篇幅，会删除官方注释或省略部分内容，可以到官方仓库中找到example查看相应的注释。

#### `docker-compose.yaml`

```yaml
services:
  headscale:
    image: headscale/headscale:latest
    container_name: headscale
    restart: unless-stopped
    volumes:
      - ./headscale/config.yaml:/etc/headscale/config.yaml
      - ./headscale/derp.yaml:/etc/headscale/derp.yaml
      - ./headscale/data:/var/lib/headscale
      - ./headscale/run:/var/run/headscale
    ports:
      - "41000:41000/udp"
    command: serve
    networks:
      - headscale-net

  headplane:
    image: ghcr.io/tale/headplane:latest
    container_name: headplane
    restart: unless-stopped
    volumes:
      - ./headplane/config.yaml:/etc/headplane/config.yaml
      - ./headscale/config.yaml:/etc/headscale/config.yaml
      - ./headplane/data:/var/lib/headplane
      - /var/run/docker.sock:/var/run/docker.sock:ro  #见下文说明
    networks:
      - headscale-net

  caddy:
    build:
      context: caddy
      dockerfile: Dockerfile
    container_name: caddy
    restart: unless-stopped
    user: 1000:1000
    ports:
      - "40000:40000"
    environment:
      CF_API_TOKEN: cloudflare-dns-api-key
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy/certs:/certs:ro
      - ./caddy/data:/data
      - ./caddy/config:/config
    networks:
      - headscale-net

networks:
  headscale-net:

volumes:
  headscale-run:
```

#### `headscale/config.yaml`

Headscale配置文件，[官方示例](https://github.com/juanfont/headscale/blob/main/config-example.yaml)。如果不需要额外的DERP服务器，可以将`derp.paths`设为空数组。

```
server_url: https://hs.example.com:40000
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090
grpc_listen_addr: 127.0.0.1:50443
grpc_allow_insecure: false
noise:
  private_key_path: /var/lib/headscale/noise_private.key
prefixes:
  v4: 100.64.0.0/10
  allocation: sequential
derp:
  server:
    enabled: true
    region_id: 999
    region_code: "headscale"
    region_name: "Headscale Embedded DERP"
    verify_clients: true
    stun_listen_addr: "0.0.0.0:41000"
    private_key_path: /var/lib/headscale/derp_server_private.key
    automatically_add_embedded_derp_region: true
    ipv4: 48.1.1.1
  urls: []
  paths: ["/etc/headscale/derp.yaml"]
  auto_update_enabled: true
  update_frequency: 24h
disable_check_updates: false
ephemeral_node_inactivity_timeout: 30m
# database,log,unix_socket保持默认即可，省略
policy:
  mode: database
dns:
  magic_dns: true
  base_domain: ts.net
  override_local_dns: false
  nameservers:
    global: ["223.5.5.5"]
    split: {}
  search_domains: []
  extra_records: []
# 这个设置为true会使tailscale忽略启动时的--port参数使用随机端口
randomize_client_port: false
```

#### `headscale/derp.yaml`

配置额外的DERP服务器，如果不需要就可以不用，记得在`docker-compose.yaml`里删掉这个volume。

```
regions:
  900:
    regionid: 900
    regioncode: china
    regionname: China
    nodes:
      - name: node
        regionid: 900
        hostname: 47.1.1.1
        ipv4: 47.1.1.1
        stunport: 41000
        derpport: 40000
        insecurefortests: true
        stunonly: false
        canport80: false
```

#### `headplane/config.yaml`

headplane配置文件，[官方示例](https://github.com/tale/headplane/blob/main/config.example.yaml)。如果不需要Web UI，也可以不部署。

```yaml
server:
  host: "0.0.0.0"
  port: 3000
  cookie_secret: "长度为32的随机字符串请自行生成"
  cookie_secure: true
  cookie_max_age: 86400
  data_path: "/var/lib/headplane"
headscale:
  url: "http://headscale:8080"
  public_url: https://hs.example.com:40000
  config_path: "/etc/headscale/config.yaml"
  config_strict: true
integration:
  agent:
    enabled: false
  docker:
    # 见下文说明
    enabled: false
    container_name: "headscale"
    # container_label: "me.tale.headplane.target=headscale"
    # socket: "unix:///var/run/docker.sock"
```

#### `caddy/Caddyfile`

Caddy配置文件，[官方文档](https://caddyserver.com/docs/caddyfile)。

```text
{
    https_port 40000
    auto_https disable_redirects
    admin off
}

(cloudflare) {
    dns cloudflare {env.CF_API_TOKEN}
    resolvers 1.1.1.1
}

hs.example.com:40000 {
    tls {
        import cloudflare
    }
    reverse_proxy headscale:8080
}

hp.example.com:40000 {
    tls {
        import cloudflare

        # mTLS客户端验证，如果headplane不暴露在公网可以不用
        client_auth {
            trust_pool file /certs/my_CA.crt
            mode require_and_verify
        }
    }
    reverse_proxy headplane:3000
}
```

#### `caddy/Dockerfile`

这是用来创建caddy镜像的Dockerfile，caddy的dns都是以插件形式提供的，需要自己编译。如果读者使用别的DNS，请自行换成所使用的DNS，caddy官方支持的DNS可以在[这里](https://github.com/orgs/caddy-dns/repositories?type=all)找到。

```text
FROM caddy:builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:latest

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

### 部署说明

我这里容器的用户都是`1000:1000`，因此在启动之前要保证挂载进容器的目录都是`1000`用户可读写的，建议事先创建，如果文件夹不存在Docker自行创建的话owner是root，会有权限问题。

全部准备好后，运行`sudo docker-compose up -d`就搞定了。

通过`https://hp.example.com:40000/admin/`访问Headplane，第一次访问会要求输入Headscale Key，使用以下命令生成：

```
sudo docker exec headscale headscale apikeys create -e 999d
```

建议保留这个Key，因为每换一个浏览器都要重新输入，每次都重新生成的话麻烦。

然后在安装了tailscale的客户端上运行：

```
sudo tailscale up --login-server https://hs.example.com:40000
```

按步骤操作即可登录。

## DERP服务器

部署DERP服务器需要准备好自签证书，将私钥和证书分别命名为`47.1.1.1.key`和`47.1.1.1.crt`放在`certs`文件夹内。请自行替换成自己的IP或者域名（备案了的话），这个命名规范是DERP要求的。

然后准备`docker-compose.yaml`：

```yaml
services:
  derper:
    image: fredliang/derper:latest
    container_name: derper
    restart: unless-stopped
    ports:
      - 41000:41000/udp
      - 40000:40000
    environment:
      DERP_DOMAIN: "47.1.1.1"
      DERP_CERT_MODE: manual
      DERP_CERT_DIR: /certs
      DERP_ADDR: ":40000"
      DERP_STUN: "true"
      DERP_STUN_PORT: "41000"
      DERP_HTTP_PORT: "-1"
      # 见下文说明
      DERP_VERIFY_CLIENTS: "true"
      # DERP_VERIFY_CLIENT_URL: https://hs.example.com:40000
    volumes:
      - ./certs:/certs
      # 见下文说明
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock
```

使用`sudo docker-compose up -d`运行，需要注意的是，我这个服务器同时也是一个tailnet节点，所以可以用`/var/run/tailscale/tailscaled.sock`来验证已登录客户端。如果不是的话，考虑用`DERP_VERIFY_CLIENT_URL`代替，但可能有坑，见下文说明。

在节点上使用`tailscale netcheck`以及`tailscale debug derp-map`验证是否获取并且连接到DERP服务器。

## 踩坑记录

### Headplane和Docker兼容性问题

Docker 29版本提高了最低API版本要求，导致很多客户端不兼容，之前Portainer就遇到了这个问题，这次Headplane又遇到了，所以上面配置文件`integration.docker.enabled`设置为`true`那么无论如何都是连不上的。好在这个问题已经在[这个PR](https://github.com/tale/headplane/pull/370)里修复了，等新版本发布应该就能解决。

### 实现限制

部署过程中发现了一些Headscale没有实现的功能，越是偏门的功能越有可能没有被实现，我遇到的问题有：

- ACL不支持IP sets，见[这个issue](https://github.com/juanfont/headscale/issues/2409)，开发人员正在全力开发新的grants功能，估计没空管旧的ACL了，好在可以通过展开形式表达，就是会有重复
- DNS不支持wildcard，例如`*.example.com`，见[这个issue](https://github.com/juanfont/headscale/issues/2508)，不过这似乎是tailscale本身的限制

### DERP服务

部署DERP服务的过程中遇到了两个坑，一个是`InsecureForTests`的问题，这个flag对自签证书的环境至关重要，告知tailscale忽略证书错误。一开始看了[这个帖子](https://linux.do/t/topic/171651)里说headscale使用本地yaml文件读取DERP配置的的时候，不支持`InsecureForTests`，必须要用URL形式，完全被带歪了，[这篇文章](https://gist.github.com/zhangguanzhang/db9f6862eef4c82f6918852953a48458)说的才是对的，本地yaml文件完全也支持`InsecureForTests`。

还有就是derper的两个参数`--verify-clients`和`--verify-client-url`（在上面的docker-compose.yaml中它们通过`DERP_VERIFY_CLIENTS`和`DERP_VERIFY_CLIENT_URL`环境变量设置），这是为了防止DERP服务器被白嫖。我一开始以为要验证客户端必须访问Headscale，于是就想用`--verify-client-url`，并且想当然地以为`--verify-clients`是开关，设置为`true`时，`--verify-client-url`才会生效。试了半天才知道这两个参数是互斥值，`--verify-clients`的意思是从本地的`/var/run/tailscale/tailscaled.sock`获取用户信息，要想使用`--verify-client-url`必须先把`--verify-clients`设置成`false`。但是我使用`--verify-client-url`时DERP依旧无法验证客户端，可能是网络关系，毕竟我的Headscale服务器在境外。不过这时我突然意识到验证客户端其实只需要公钥信息，即使有人假冒了公钥，没有私钥的情况下他依旧无法和网络中的其它节点通信，于是用本地的tailscale节点信息验证客户端也就顺理成章了。

### 阿里云兼容问题

阿里云内网也会用到`100.64.0.0/10`这个CGNAT网段，例如阿里云的DNS服务器就是`100.100.2.136`，APT镜像服务器`mirrors.cloud.aliyuncs.com`也在这个网段，而tailscale为了安全默认会在iptables里插入一条规则，丢弃所有不是从tailscale网卡中进入本机的`100.64.0.0/10`数据包，规则如下所示：

```
Chain ts-input (1 references)
 pkts bytes target     prot opt in          out     source               destination
    0     0 ACCEPT     0    --  lo          *       100.64.0.4           0.0.0.0/0
    0     0 RETURN     0    --  !tailscale0 *       100.115.92.0/23      0.0.0.0/0
    0     0 DROP       0    --  !tailscale0 *       100.64.0.0/10        0.0.0.0/0
11231  587K ACCEPT     0    --  tailscale0  *       0.0.0.0/0            0.0.0.0/0
13256 1659K ACCEPT     17   --  *           *       0.0.0.0/0            0.0.0.0/0            udp dpt:41641
```

这就导致阿里云的各种内部服务全都不可用。网上有人干脆不用阿里云的内部服务，但我会用到阿里云的镜像服务，必须得用，于是干脆禁用tailscale的安全规则，反正看着问题不大：

```bash
sudo tailscale up \
    --accept-dns=false \
    --netfilter-mode=nodivert \
    --login-server https://hs.example.com:40000
```

关键参数是`--netfilter-mode=nodivert`，tailscale依旧会创建`ts-input`以及其它规则链，但不会把它插入到`INPUT`链里，所以它实际上没有生效。

## 后记

到这里整套服务已经可以正常运行了，只需要在每台设备上登录一下即可，我这里就不再详述了。此外还有配置ACL的过程，在ChatGPT的帮助下，除了踩了点小坑，还是很顺利的。tailscale整体的使用体验非常好，完全解决了WireGuard的痛点。如果DNS功能可以更强大的话，我甚至考虑把DNS也搬到tailnet上，其次还有SSH，service等一大堆高级功能，等headscale支持grants以后，也要再折腾一番。