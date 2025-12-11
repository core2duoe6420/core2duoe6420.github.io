---
title: "Enable TLS Connection for Docker Daemon"
date: 2022-02-10
slug: docker-daemon-tls
tags:
- Docker
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

Background: The NAS uses a Xeon W-2140B CPU without an integrated GPU, and the onboard ASPEED graphics is essentially unusable for hardware acceleration, so Jellyfin cannot use hardware decoding. Although I tested that CPU software decoding can handle one 4K-to-4K transcoding stream smoothly, the CPU usage is already close to 100%. So I decided to move Jellyfin to an Ubuntu VM running on ESXi with a Core i3 8100T. To conveniently manage Docker on both machines, I want the Portainer instance running on the NAS to connect to the new Docker Daemon.

<!--more-->

In fact, Docker has supported SSH connections since version 18.09, but [Portainer explicitly states it will not support this](https://github.com/portainer/portainer/issues/431#issuecomment-820835339), so we still have to use TCP with TLS protection. The configuration itself is not complicated; this post is just a brief record. First, here are the references:

- Docker documentation: Protect the Docker daemon socket https://docs.docker.com/engine/security/protect-access/
- Enable Docker Remote API with TLS client verification: https://gist.github.com/kekru/974e40bb1cd4b947a53cca5ba4b0bbe5

## Generate certificates

Most other tutorials use OpenSSL for this step. I recommend a tool called [XCA](https://hohnstaedt.de/xca/), a GUI-based certificate management tool. It’s very convenient for managing keys and certificates used in a small scope, and I’ve been using it for years.

You need to generate the following keys and certificates:

- CA certificate, required by both the Docker Daemon and Portainer
- Server key and certificate, configured on the Docker Daemon
- Client authentication key and certificate, configured in Portainer

Both the server and client certificates must be signed by the CA certificate. The SAN of the server certificate must match the DNS name or IP address you will use.

## Configure Docker Daemon

Edit `/etc/docker/daemon.json`:

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

Here I ran into a pitfall: when using `systemd` to manage Docker, the default unit starts Docker with the `-H` parameter, and the [official documentation](https://docs.docker.com/config/daemon/#troubleshoot-conflicts-between-the-daemonjson-and-startup-scripts) clearly states:

> If you use a daemon.json file and also pass options to the dockerd command manually or using start-up scripts, and these options conflict, Docker fails to start with an error such as: ...

~~So you also need to edit `/lib/systemd/system/docker.service` to remove the `-H fd://` start-up parameter, then restart the service:~~

A better approach is to use `sudo systemctl edit docker.service` to override the configuration; otherwise, every Docker update will overwrite your changes. Enter:

```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
```

Ubuntu’s default editor is nano, which I’m not used to. To use vim instead:

```
sudo EDITOR=vim systemctl edit docker.service
```

After saving and exiting, the configuration will be written to `/etc/systemd/system/docker.service.d/override.conf`, then restart the service:

```bash
sudo systemctl daemon-reload
sudo systemctl start docker
```

The server-side configuration is done.

## Configure Portainer

Configuring Portainer is straightforward. Go to `Settings -> Environments -> Add Environment`, select Docker Directly connect to the Docker API. The rest is all GUI operations: upload the CA certificate, client key, and client certificate, and you’re done.