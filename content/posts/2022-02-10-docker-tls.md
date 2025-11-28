---
title: "启用Docker Daemon的TLS连接"
date: 2022-02-10
slug: docker-daemon-tls
tags:
- Docker
---

背景介绍：因为NAS使用的CPU Xeon W-2140B没带集显，主板集成的显卡ASPEED性能可以忽略，因此没法在jellyfin里使用硬解。虽然测试了一下，靠CPU软解可以支持一路4K到4K重编码流畅观看，但此时CPU使用率已经接近100%。于是还是决定把jellykin放到装有Core i3 8100T的ESXi虚拟出的Ubuntu上。为了方便管理两台机器上的Docker，想让运行在NAS上的Portainer连接到新的Docker Daemon上。

<!--more-->

实际上，Docker于18.09版本开始已经支持SSH连接，奈何[Portainer明确表示不会支持](https://github.com/portainer/portainer/issues/431#issuecomment-820835339)，还是只能用TCP+TLS保护的连接方式。配置本身不算复杂，本文就是做个简单的记录。先放出参考资料：

- Docker文档：Protect the Docker daemon socket https://docs.docker.com/engine/security/protect-access/
- Enable Docker Remote API with TLS client verification: https://gist.github.com/kekru/974e40bb1cd4b947a53cca5ba4b0bbe5

## 生成证书

通常别的教程这一步会用OpenSSL，我这里推荐一个工具[XCA](https://hohnstaedt.de/xca/)，这是一个图形化界面的证书管理工具，用来管理小范围使用的秘钥和证书很方便，我已经用了几年了。

需要生成的秘钥和证书有：

- CA证书，Docker Daemon和Portainer都需要
- 服务器使用的密钥和证书，配置在Docker Daemon上
- 客户端认证使用的密钥和证书，配置在Portainer上

服务器证书和客户端证书均需使用CA证书签名。服务器证书的SAN需要和使用到的DNS或者IP匹配。

## 配置Docker Daemon

编辑`/etc/docker/daemon.json`：

```json
{
    "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
    "tls": true,
    "tlscacert": "/opt/container-data/docker-certs/ca.crt",
    "tlscert": "/opt/container-data/docker-certs/server-cert.crt",
    "tlskey": "/opt/container-data/docker-certs/server-key.pem",
    "tlsverify": true
}
```

这里踩了个坑，当使用`systemd`管理Docker时，默认的Unit启动是带有`-H`参数的，而[官方文档](https://docs.docker.com/config/daemon/#troubleshoot-conflicts-between-the-daemonjson-and-startup-scripts)里明确写了：

> If you use a daemon.json file and also pass options to the dockerd command manually or using start-up scripts, and these options conflict, Docker fails to start with an error such as: ...

~~因此还需要编辑`/lib/systemd/system/docker.service`，将其中的启动参数`-H fd://`删去，然后重启服务：~~

更好的办法是使用`sudo systemctl edit docker.service`覆盖配置，否则每次docker更新都会把更新覆盖，输入：

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
```

Ubuntu默认的编辑器是nano，很不习惯，要换用vim的话：

```
sudo EDITOR=vim systemctl edit docker.service
```

保存后退出，会把配置写入`/etc/systemd/system/docker.service.d/override.conf`，然后重启服务


```bash
sudo systemctl daemon-reload
sudo systemctl start docker
```

服务器端配置完成。

## 配置Portainer

Portainer的配置就很简单了，`Settings -> Environments -> Add Environment`，选择Docker Directly connect to the Docker API。后面都是图形操作，上传CA证书，客户端秘钥和证书，就好了。