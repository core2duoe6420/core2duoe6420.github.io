---
title: "Deploying Headscale to Build a Private Tailscale Network"
date: 2025-12-07
slug: deploy-headscale
hide_site_title: true
tags:
- headscale
- headplane
- tailscale
- openwrt
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

My private home network originally used Zerotier, but I found the connection to be unstable. Even when both of my nodes had public IPs, they sometimes failed to connect. Later I switched to WireGuard, which turned out to be very stable. Recently, however, I bought several new VPS instances and a friend’s home network also joined mine. The number of nodes increased, security rules became more complex, and maintaining WireGuard started to feel overwhelming. So I finally decided to deploy Headscale and migrate to a Tailscale-based network.
<!--more-->

There is no shortage of information online about Headscale and Tailscale, but much of it is repetitive. Many guides still deploy Headscale directly on bare metal and spend long sections explaining how to configure auto-start with systemd, which I found frustrating. Some resources are outdated or unclear, and even the official Tailscale documentation is not always entirely accurate. Furthermore, Headscale does not implement all of Tailscale’s features, which makes it easy to run into pitfalls. Half documentation and half tutorial, this article introduces my deployment approach and configuration. Readers are expected to understand the details and adapt them flexibly to their own environments.

## Environment and Architecture

The deployment environment and solution described in this article are as follows:

- Headscale version `0.27.1`; Headplane is used as the UI management interface, version `0.6.2-beta.2` (the reason for using the beta version is explained later); Docker version `29.1.1`. This version of Headscale requires Tailscale clients to be at least `1.64.0`. The latest Tailscale version at present is `1.90.9`.
- Headscale is deployed on an overseas server using the built-in DERP service. At the same time, an additional DERP server is deployed inside mainland China to accelerate connectivity. This is mainly because my domain is not ICP-registered, so I cannot deploy Headscale directly inside China. I tried using a self-signed certificate with a direct IP address. On Ubuntu, adding a system CA worked fine, but one of my clients runs OpenWrt, and I could not get the certificate imported no matter what I tried, so I had to give up.
- The examples in this article use the following parameters. Please adjust them according to your own situation:

    - Headscale + Headplane server
    
        - Headscale domain: `hs.example.com`
        - IP address: `48.1.1.1`
        - HTTPS listens on `40000`
        - Built-in DERP STUN listens on UDP port `41000`
        - Other ports such as `metrics` and `grpc` are ignored
        - Caddy is used to reverse proxy Headscale and Headplane. Caddy obtains certificates via ACME using DNS-01 challenges. The domain is hosted on Cloudflare, and the API key is `cloudflare-dns-api-key`
        - The public domain for Headplane is `hp.example.com`, sharing port `40000`. I also enabled mTLS for security; readers can decide for themselves whether mTLS is necessary

    - Mainland China DERP server

        - IP address: `47.1.1.1`
        - HTTPS listens on `40000`
        - STUN listens on `41000`
        - No HTTP port required
        - ICMP does not need to be opened (the official documentation says it is required, but in practice it works without it)
        - Uses a self-signed certificate

## Headscale Server Configuration

Prepare the following configuration files. Due to space limitations, the files provided here have official comments removed or some content omitted. You can find the full examples with detailed comments in the official repository.

#### `docker-compose.yaml`

```yaml
services:
  headscale:
    image: headscale/headscale:v0.27.1
    container_name: headscale
    restart: unless-stopped
    user: 1000:1000
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
    image: ghcr.io/tale/headplane:0.6.2-beta.2
    container_name: headplane
    restart: unless-stopped
    user: 1000:1000
    volumes:
      - ./headplane/config.yaml:/etc/headplane/config.yaml
      - ./headscale/config.yaml:/etc/headscale/config.yaml
      - ./headplane/data:/var/lib/headplane
      - /var/run/docker.sock:/var/run/docker.sock:ro  # See explanation below
    networks:
      - headscale-net
    group_add:
      - ${DOCKER_GROUP}

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
````

#### `headscale/config.yaml`

Headscale configuration file, [official example](https://github.com/juanfont/headscale/blob/main/config-example.yaml). If you do not need additional DERP servers, you can set `derp.paths` to an empty array.

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
  auto_update_enabled: false
  update_frequency: 24h
disable_check_updates: true
ephemeral_node_inactivity_timeout: 30m
# database, log, unix_socket can remain default and are omitted here
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
# Setting this to true makes Tailscale ignore the --port parameter at startup and use a random port
randomize_client_port: false
```

#### `headscale/derp.yaml`

Configuration for the additional DERP server. If you do not need it, you can omit this file and remove the corresponding volume from `docker-compose.yaml`.

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

Headplane configuration file, [official example](https://github.com/tale/headplane/blob/main/config.example.yaml). If you do not need the Web UI, you can skip deploying it.

```yaml
server:
  host: "0.0.0.0"
  port: 3000
  cookie_secret: "Please generate a random 32-character string yourself: tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 32"
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
    # See explanation below
    enabled: true
    container_name: "headscale"
    socket: "unix:///var/run/docker.sock"
```

#### `caddy/Caddyfile`

Caddy configuration file, [official documentation](https://caddyserver.com/docs/caddyfile).

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

        # mTLS client verification; not required if Headplane is not exposed to the public internet
        client_auth {
            trust_pool file /certs/my_CA.crt
            mode require_and_verify
        }
    }
    reverse_proxy headplane:3000
}
```

#### `caddy/Dockerfile`

This is the Dockerfile used to build the Caddy image. DNS support in Caddy is provided via plugins and must be compiled manually. If you use a different DNS provider, replace the plugin accordingly. All DNS plugins officially supported by Caddy can be found [here](https://github.com/orgs/caddy-dns/repositories?type=all).

```text
FROM caddy:builder AS builder

RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare

FROM caddy:latest

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

### Deployment Instructions

All containers here run as user `1000:1000`, so before starting, make sure that all mounted directories are readable and writable by user `1000`. If Docker creates the directories automatically, they will be owned by root, which will cause permission issues. Run the following commands to start the services:

```bash
mkdir -p headscale/{data,run} headplane/data caddy/{data,config,certs}
export DOCKER_GROUP=$(getent group docker | cut -d: -f3)
sudo -E docker compose up -d
```

The `DOCKER_GROUP` environment variable is used in `docker-compose.yaml` to allow Headplane to read `/var/run/docker.sock` and integrate with the Headscale container.

Access Headplane at `https://hp.example.com:40000/admin/`. On first access, you will be prompted to enter a Headscale API key, which can be generated using:

```
sudo docker exec headscale headscale apikeys create -e 999d
```

It is recommended to keep this key, because you will need to re-enter it every time you switch browsers. Constantly regenerating it is inconvenient.

Then, on a client with Tailscale installed, run:

```
sudo tailscale up --login-server https://hs.example.com:40000
```

Follow the prompts to complete the login process.

## DERP Server

To deploy the DERP server, prepare a self-signed certificate. Name the private key and certificate as `47.1.1.1.key` and `47.1.1.1.crt`, and place them in the `certs` directory. Replace the IP with your own IP or domain (if you have ICP registration). This naming convention is required by DERP.

Then prepare the `docker-compose.yaml`:

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
      # See explanation below
      DERP_VERIFY_CLIENTS: "true"
      # DERP_VERIFY_CLIENT_URL: https://hs.example.com:40000/verify
    volumes:
      - ./certs:/certs
      # See explanation below
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock
```

Start it with `sudo docker-compose up -d`. Note that this server is also a tailnet node in my case, so I can use `/var/run/tailscale/tailscaled.sock` to verify authenticated clients. If that is not the case for you, consider using `DERP_VERIFY_CLIENT_URL` instead, though there may be pitfalls (explained below).

On the node, use `tailscale netcheck` and `tailscale debug derp-map` to verify that the DERP server is being obtained and connected successfully.

## Pitfalls and Lessons Learned

### Headplane and Docker Compatibility Issues

Docker 29 raised the minimum API version requirement, causing compatibility issues with many clients. Portainer encountered this problem before, and now Headplane ran into it as well. Initially, I used the stable version `0.6.1`, but it simply could not connect to Docker. The logs were extremely misleading and contained no hints about API compatibility at all. Fortunately, this issue has already been fixed in [this PR](https://github.com/tale/headplane/pull/370). Using the `0.6.2-beta.2` image resolves the problem.

### Implementation Limitations

During deployment, I discovered that some Tailscale features are not implemented in Headscale. The more niche the feature, the more likely it is missing. The issues I encountered include:

* ACL does not support IP sets. See [this issue](https://github.com/juanfont/headscale/issues/2409). The developers are fully focused on implementing the new grants feature and probably have no time to maintain the old ACL system. Fortunately, the same effect can be achieved by expanding the rules manually, though it leads to duplication.
* DNS does not support wildcards such as `*.example.com`. See [this issue](https://github.com/juanfont/headscale/issues/2508). This seems to be a limitation of Tailscale itself.

### DERP Service Issues

Two major pitfalls arose during the DERP deployment. The first was the `InsecureForTests` flag. This flag is critical for self-signed certificate environments, as it tells Tailscale to ignore certificate errors. Initially, I was misled by [this post](https://linux.do/t/topic/171651), which claimed that when Headscale loads DERP configuration from a local YAML file, `InsecureForTests` is not supported and that only URL-based configuration works. This turned out to be incorrect. [This article](https://gist.github.com/zhangguanzhang/db9f6862eef4c82f6918852953a48458) is the accurate reference: local YAML files fully support `InsecureForTests`.

The second issue involved the `derper` parameters `--verify-clients` and `--verify-client-url` (configured as `DERP_VERIFY_CLIENTS` and `DERP_VERIFY_CLIENT_URL` in the `docker-compose.yaml` above). These are meant to prevent abuse of the DERP server. Initially, I assumed that client verification had to go through Headscale, so I tried using `--verify-client-url`. I also assumed that `--verify-clients` was simply a toggle, and that `--verify-client-url` would only take effect when `--verify-clients` was set to `true`. After struggling for quite some time, I finally realized that these two parameters are mutually exclusive. `--verify-clients` means retrieving user information from the local `/var/run/tailscale/tailscaled.sock`. To use `--verify-client-url`, `--verify-clients` must first be set to `false`. However, even with `--verify-client-url`, DERP still failed to verify clients in my setup. This might be due to network topology, since my Headscale server is overseas. At that point, I realized that client verification ultimately only requires public key information. Even if someone spoofs a public key, they still cannot communicate with other nodes without the corresponding private key. Therefore, verifying clients using local Tailscale node information is perfectly reasonable.

### Alibaba Cloud Compatibility Issue

Alibaba Cloud’s internal network also uses the `100.64.0.0/10` CGNAT range. For example, Alibaba Cloud’s DNS server is `100.100.2.136`, and the APT mirror `mirrors.cloud.aliyuncs.com` also falls within this range. For security reasons, Tailscale inserts an iptables rule by default to drop all `100.64.0.0/10` packets that do not originate from the Tailscale network interface. The rule looks like this:

```
Chain ts-input (1 references)
 pkts bytes target     prot opt in          out     source               destination
    0     0 ACCEPT     0    --  lo          *       100.64.0.4           0.0.0.0/0
    0     0 RETURN     0    --  !tailscale0 *       100.115.92.0/23      0.0.0.0/0
    0     0 DROP       0    --  !tailscale0 *       100.64.0.0/10        0.0.0.0/0
11231  587K ACCEPT     0    --  tailscale0  *       0.0.0.0/0            0.0.0.0/0
13256 1659K ACCEPT     17   --  *           *       0.0.0.0/0            0.0.0.0/0            udp dpt:41641
```

As a result, all Alibaba Cloud internal services became inaccessible. Some people simply avoid using Alibaba Cloud’s internal services altogether, but I rely on their mirror services, so that was not an option. Therefore, I disabled Tailscale’s netfilter rules entirely. The risk seemed minimal in my case:

```bash
sudo tailscale up \
    --accept-dns=false \
    --netfilter-mode=nodivert \
    --login-server https://hs.example.com:40000
```

The key parameter here is `--netfilter-mode=nodivert`. Tailscale will still create chains like `ts-input` and others, but it will not insert them into the `INPUT` chain, so they effectively do nothing.

## Epilogue

At this point, the entire setup is fully operational. All that remains is to log in on each device, which I will not go into here. There is also the process of configuring ACLs. With the help of ChatGPT, aside from a few minor pitfalls, that part went quite smoothly. Overall, the Tailscale experience has been excellent and completely solves the pain points I had with WireGuard. If the DNS feature becomes more powerful in the future, I would even consider moving DNS entirely onto the tailnet. On top of that, there are SSH, services, and many other advanced features. Once Headscale supports grants, I will definitely experiment with those as well.
