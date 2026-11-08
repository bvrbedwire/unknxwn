# unknxwn

Modern multi-user proxy server setup: **latest 3x-ui** (installed dynamically from the newest release, v3.11.0+ with Xray-core 26.x) behind **nginx**, with **WARP egress for all inbound connections**.

Everything is configured automatically by a single script — panel, SSL, nginx SNI-routing, inbounds, subscriptions, masquerade site, WARP.

## What this script sets up

- Auto Installation (lightweight), **always the latest 3x-ui release**
- Auto SSL via certbot / Daily reload of Nginx + X-UI
- Multi-user and config via port **443** (nginx SNI-routing, proxy_protocol)
- **WARP for all inbound connections** — a Cloudflare WARP device is registered
  automatically (`api.cloudflareclient.com`) and the `warp` wireguard outbound is
  installed as the **default route**, so every client's traffic egresses via a
  Cloudflare WARP IP instead of the server IP. Credentials are stored in the
  panel DB, so the panel's WARP UI (change IP, config) keeps working.

### Default inbounds (created automatically)

| Inbound | Protocol / Transport | Notes |
|---|---|---|
| REALITY | VLESS + xtls-rprx-vision, tcp | port 443 via nginx ssl_preread, `x25519mlkem768` post-quantum |
| WS | VLESS over WebSocket | legacy fallback, behind nginx on 443 |
| XHTTP | VLESS over **XHTTP** (`mode: auto`, `noGRPCHeader`, xPadding `100-1000`) | modern xray-core transport, plain HTTP/1.1 uplink via nginx |
| Trojan-XHTTP | Trojan over **XHTTP** | modern replacement for trojan-grpc (gRPC transport is deprecated) |
| **Hysteria2** | native 3x-ui hysteria inbound (version 2), UDP, ALPN h3 | **salamander UDP obfuscation** (finalmask), TLS cert of the panel domain, masquerade serves the fallback site over HTTP/3 |

All five land in one subscription link; the Hysteria2 link exports
`obfs=salamander&obfs-password=...`, the XHTTP links export the modern
`extra={"mode":"auto","noGRPCHeader":true,...}` parameters.

### Masquerade page

Sourced from the **`git_land/`** directory of this repo (`git-login-y2k.html` —
a GIT sign-in page). It is served on `/` by nginx **and** by the Hysteria2
inbound (HTTP/3 masquerade, `type: file`). For remote installs the file is
fetched from this repo via raw.githubusercontent.

### Subscriptions & client configs

- Auto enabled subscriptions via port **443** (raw base64 + JSON)
- **Custom Web Sub Page** with per-client configs
- **Local instance sub2sing-box**
- CLASH META configs (see the `-clash` option below)

## Requirements

- Linux Debian 12 / Ubuntu 22.04+ (verified on Ubuntu 24.04), root access
- **TWO domains or subdomains**, both resolving to the server IP:
  1. For panel, subscriptions, WS/XHTTP/Trojan-XHTTP/Hysteria2 (e.g. `git.example.com`)
  2. For REALITY destination (e.g. `set.git.example.com`)

## Install

```
bash <(wget -qO- https://raw.githubusercontent.com/bvrbedwire/unknxwn/master/unknow.sh) \
  -install yes -panel 1 -ONLY_CF_IP_ALLOW no \
  -subdomain <sub.domain.tld> -reality_domain <reality.domain.tld>
```

> Do not change the SubDomain for SSL renewal❗

**Backup panel and nginx configs**💾:

```
sudo su -c "bash <(wget -qO- https://raw.githubusercontent.com/bvrbedwire/unknxwn/master/backup.sh)"
```

**Uninstall**❌:

```
sudo su -c "bash <(wget -qO- https://raw.githubusercontent.com/bvrbedwire/unknxwn/master/unknow.sh) -Uninstall yes"
```

## Options

| Option | Values | Description |
|---|---|---|
| `-install` | yes/no | install packages (nginx, certbot, sqlite3, ufw...) |
| `-subdomain` | sub.domain.tld | panel + WS/XHTTP/Trojan/Hysteria2 domain |
| `-reality_domain` | sub.domain.tld | REALITY destination domain |
| `-panel` | 1 | panel preset |
| `-ONLY_CF_IP_ALLOW` | yes/no | restrict access to Cloudflare IPs |
| `-websub` | 0/1 | custom web sub page variant |
| `-clash` | 0-3 | clash meta config variant: 0 bundle, 1 skrepysh, 2 fullproxy, 3 re-filter ECH |
| `-uninstall` | yes | full uninstall |

## Firewall & tuning

ufw: 22, 80, 443/tcp, 443/udp + the random Hysteria2 UDP port. BBR congestion
control and sysctl socket-buffer tuning are applied automatically.
