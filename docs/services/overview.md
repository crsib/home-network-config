# Services Overview

A single matrix of everything running, where, and on which port. Drill into the
per-host pages or the focused service docs for detail.

_Last verified: 2026-06-14._

## Service matrix

| Service | Host | Port(s) | Runtime | Doc |
|---------|------|---------|---------|-----|
| Routing / firewall / NAT | EdgeRouter-4 `.1` | — | EdgeOS | [router](../hosts/edgerouter-4.md) |
| LAN DNS | EdgeRouter-4 `.1` | `53` | dnsmasq | [DNS](dns.md) |
| DDNS (`local.crsib.me`) | EdgeRouter-4 `.1` | — | EdgeOS | [DNS](dns.md) |
| nginx reverse proxy | home-controller `.2` | `80`, `443` (TCP+UDP) | native | [reverse proxy](reverse-proxy-and-certs.md) |
| Headscale (Tailscale control) | home-controller `.2` | via `headscale.crsib.me` | native | [overlays](overlay-and-remote-access.md) |
| UniFi Network controller | home-controller `.2` | `8443`, `8080`, `8843`, `8880`, `6789`, `3478/udp` | native | [controller](../hosts/home-controller.md) |
| UISP / UNMS | home-controller `.2` | `9443`, `81`, `8089`, `9080`, `2055/udp` | Docker | [controller](../hosts/home-controller.md) |
| Nextcloud | home-controller `.2` | `9876` | Docker | [controller](../hosts/home-controller.md) |
| MicroK8s | home-controller `.2` | `16443`, `10250`, `25000` | snap | [controller](../hosts/home-controller.md) |
| ZeroTier | home-controller `.2` | `9993/udp` | native | [overlays](overlay-and-remote-access.md) |
| zrok shares | home-controller `.2` | — | Docker | [overlays](overlay-and-remote-access.md) |
| SOCKS5 proxy | home-controller `.2` | `1080` | native | [overlays](overlay-and-remote-access.md) |
| FortiClient VPN (`2GIS`) | dvedenko-net `.13` | `fctvpn*` | native | [forti box](../hosts/dvedenko-net.md) |
| RustDesk server | dvedenko-net `.13` | `21115`–`21119` | native | [overlays](overlay-and-remote-access.md) |
| microsocks (SOCKS5) | dvedenko-net `.13` | `1080` | native | [overlays](overlay-and-remote-access.md) |
| DNS stub (resolved) | dvedenko-net `.13` | `53` | native | [DNS](dns.md) |
| iperf3 | dvedenko-net `.13` | `5201` | native | [forti box](../hosts/dvedenko-net.md) |
| CUPS (printing) | dvedenko-net `.13` | `631` | native | [forti box](../hosts/dvedenko-net.md) |

## Two Ubiquiti management planes

The network runs **both** Ubiquiti controllers, on the same host:

- **UniFi Network** (`:8443`) — for UniFi gear (APs, switches).
- **UISP / UNMS** (`:9443`) — for the **EdgeRouter** (EdgeMAX line).

They are separate products with separate data stores; don't confuse them.

## Retired / historical

The earlier Docker stack (Pi-hole + Unbound DNS, Plex, Transmission, Heimdall,
Netdata, Portainer, Watchtower, SWAG, Authelia) recorded in this repo's **git
history** is **no longer the live setup** and should not be treated as current.
DNS today is served by the EdgeRouter (and resolved on `.13`); the reverse proxy
is native nginx on `.2`.

Leftovers from that era still visible on the router: port-forwards for **Plex**
(`32400`) and **Transmission** (`51413`) to `.2` remain configured but have **no
backing service** — candidates for cleanup.
