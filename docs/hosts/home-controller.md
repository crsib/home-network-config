# Host: home-controller (`192.168.1.2`)

The workhorse — reverse proxy, network controllers, self-hosted apps, a
single-node Kubernetes, and overlay-network membership all live here.

_Last verified: 2026-06-14._

## Identity

| Field | Value |
|-------|-------|
| Hostname | `home-controller` |
| OS | Ubuntu 24.04.4 LTS, kernel `6.8.0-124-generic` |
| SSH | `ssh dvedenko@192.168.1.2` (alias `HomeController`) |
| Uptime at sweep | ~5 days |

## Service map

| Service | Runtime | Ports | Notes |
|---------|---------|-------|-------|
| **nginx** (reverse proxy) | native (systemd) | `80`, `443` (TCP+UDP) | Fronts `local.crsib.me` & `headscale.crsib.me`; LE cert via certbot/nginx. Also a `stream` block relaying **UDP/443 → external Aeza VPS** `104.238.29.139:55444`. See [reverse proxy doc](../services/reverse-proxy-and-certs.md) |
| **Headscale** | native (systemd) | (via nginx `headscale.crsib.me`) | Self-hosted Tailscale control plane. Mesh `100.64.0.0/10` / `fd7a:115c:a1e0::/48`. Current node: `dvedenko-24` = `.13`. See [overlays](../services/overlay-and-remote-access.md) |
| **UniFi Network** controller | native (Java + MongoDB) | `8443` GUI, `8080` inform, `8843`/`8880` portal, `6789` speedtest, `3478/udp` STUN, Mongo `27117` | Manages UniFi gear (e.g. ap-hallway) |
| **UISP / UNMS** | Docker | nginx `81`/`8089`/`9080`/`9443`, netflow `2055/udp` | Manages the [EdgeRouter](edgerouter-4.md). Stack: `unms`, `ucrm`, `unms-netflow`, `unms-nginx`, `unms-rabbitmq`, `unms-postgres`, `unms-siridb`, `unms-fluentd` (images `ubnt/unms:2.4.188`, `ubnt/unms-crm:4.4.30`) |
| **Nextcloud** | Docker Compose | web `9876` | Stack: `nextcloud-web-1` (nginx:alpine), `nextcloud-app-1` (nextcloud:fpm-alpine), `nextcloud-cron-1`, `nextcloud-redis-1`, `nextcloud-db-1` (mariadb:lts) |
| **MicroK8s** | snap `v1.32.13` | API `16443`, kubelet `10250`, cluster-agent `25000`, dqlite `19001`, … | Single-node Kubernetes — **idle**: only `kube-system` pods (calico, coredns), no user workloads as of 2026-06-14 |
| **ZeroTier** | native | `9993/udp` | Node `09fe4d687b`, v1.16.1, ONLINE |
| **zrok** (OpenZiti) | Docker | — | Public share tunnels: `unms-zrok-share`, `router-zrok-share`, `uisp-zrok-share` |
| **sing-box** | native | mixed `1080`, trojan `10443`, vless `11443` | Rule-based proxy gateway; egress via NaïveProxy → Aeza VPS. See [sing-box](../services/sing-box-proxy.md) |
| **systemd-resolved** | native | `127.0.0.53`/`127.0.0.54` | Local stub resolver |

> Container names/images above were captured live on 2026-06-14 (`docker ps`).
> The reverse proxy and UniFi controller are **native** (not containerized).

## Where the real config lives

| Thing | Location on host |
|-------|------------------|
| nginx reverse proxy | `/etc/nginx/` (sites + TLS); certs under `/etc/letsencrypt/` |
| UniFi Network | `/usr/lib/unifi/` + `/var/lib/unifi/`, Mongo on `27117` |
| Docker stacks (Nextcloud, UISP, zrok) | their respective Compose project directories under the user's home / `/opt` — enumerate with `docker inspect` labels |
| MicroK8s | `microk8s.config`, `microk8s kubectl …` |
| ZeroTier | `/var/lib/zerotier-one/` |

To list every container and its compose dir:

```bash
ssh dvedenko@192.168.1.2 \
  'sudo docker ps --format "{{.Names}}" | while read c; do \
     echo "$c -> $(sudo docker inspect -f "{{index .Config.Labels \"com.docker.compose.project.working_dir\"}}" "$c")"; \
   done'
```

## Reachable from off-LAN

Selected services are exposed externally via the nginx reverse proxy
(`local.crsib.me`, port-forwarded by the router), the **Headscale**/Tailscale mesh,
and **zrok** share tunnels. See
[overlay & remote access](../services/overlay-and-remote-access.md).

> The router also forwards `32400` (Plex) and `51413` (Transmission) to this host,
> but **nothing listens** on those ports — the forwards are stale leftovers.

## TODO / to verify

- [ ] Pin the exact Compose project directories for Nextcloud / UISP / zrok.
- [ ] Identify the app on `127.0.0.1:8081` fronted by nginx.
- [ ] Record the ZeroTier network ID(s) this node has joined.
