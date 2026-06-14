# Overlay Networks & Remote Access

How the network is reached from outside, and the overlays layered on top of the
physical LAN.

_Last verified: 2026-06-14._

## Entry points from the internet

| Path | Terminates on | Use |
|------|---------------|-----|
| `local.crsib.me` (DDNS → `83.243.71.58`) + router port-forwards | nginx on `.2`, RustDesk on `.13` | Web apps (`https://local.crsib.me`) and RustDesk relay |
| **zrok** share tunnels | `home-controller` `.2` | On-demand public URLs for `unms`, `router`, `uisp` |
| **Headscale / Tailscale** | control plane on `.2` (`headscale.crsib.me`) | Self-hosted WireGuard mesh — reach member nodes by their `100.64.x` address |
| **ZeroTier** | `home-controller` `.2` | Second flat overlay to reach the box without port-forwards |
| **RustDesk** (by ID) | `dvedenko-net` `.13` | Remote desktop / headless control |
| **FortiClient VPN** | `dvedenko-net` `.13` | Outbound corp (`2GIS`) tunnel — not an inbound path |

## Headscale + Tailscale (self-hosted mesh)

- **Control plane**: `headscale` runs **natively** on `home-controller` (`.2`,
  `/usr/bin/headscale`, HTTP on `127.0.0.1:8081`, gRPC `127.0.0.1:50444`), published
  by nginx at **`headscale.crsib.me`**.
- **Embedded DERP relay**: enabled, advertised at **`derp.crsib.me`** with STUN on
  `:3478` (covered by the same TLS cert).
- **Mesh addressing**: IPv4 `100.64.0.0/10`, IPv6 `fd7a:115c:a1e0::/48` (standard
  Tailscale ULA).
- **Nodes** (2026-06-14): one — `dvedenko-24` (the `.13` box), user `dvedenko`,
  `100.64.0.1` / `fd7a:115c:a1e0::1`, online. The `.13` box runs the `tailscale`
  client (`tailscale0`).
- This is the preferred way to reach the forti box off-LAN at the network layer
  (RustDesk handles interactive desktop separately).
- **TBD:** enrol more nodes (workstation, phones); confirm whether subnet routes /
  exit node are advertised.

## ZeroTier

- `home-controller` is also a member of a **ZeroTier** network: node `09fe4d687b`,
  client v1.16.1, ONLINE — a second overlay alongside Headscale.
- _(ZeroTier network ID intentionally not investigated per request.)_

## zrok (OpenZiti) — currently **DOWN**

Three `openziti/zrok` reserved-share stacks on `.2` (compose dirs
`/home/dvedenko/zrok/{unms,router,uisp}`, entrypoint `zrok-share.bash`). They use a
**self-hosted zrok controller** at `zrok.zrok.crsib.me` → `89.110.79.146` (the
`VDSinaWG` VPS).

| Reserved share | Backend target | Effective service |
|----------------|----------------|-------------------|
| `unms` | `https://127.0.0.1:9443` | UISP / UNMS |
| `router` | `https://192.168.1.1` | EdgeRouter UI |
| `uisp` | `https://127.0.0.1:8443` | **UniFi** controller (note: name vs. target are swapped) |

> ⚠️ **Status 2026-06-14:** the controller `89.110.79.146:443` is **unreachable**
> (i/o timeout) from `.2`, so all three share containers **crash-loop**
> (`unable to create zrok client …`). The tunnels are not functional until the
> zrok controller VPS is back. Share tokens live in the per-dir compose `.env`
> (git-ignored) — not recorded here.

## RustDesk (self-hosted)

- Server (`hbbs`/`hbbr`) **and** client both on `.13`; box **ID `255246947`**.
- Ports `21115`–`21119`; relay advertised as `local.crsib.me` (router forwards
  `21115`–`21117`).
- **LAN clients must use `192.168.1.13`** (the UDP rendezvous reply returns from the
  LAN IP, which same-LAN clients reject when using the public name).
- Do **not** enable `direct-server` on `.13` (port `21118` collides with hbbs's
  websocket → crash loop).
- Full detail: [forti-tray/MEMORY.md](../../../forti-tray/MEMORY.md).

## SOCKS / HTTP proxies

| Host | Port | Service |
|------|------|---------|
| `.2` | `1080` | **sing-box** mixed inbound (SOCKS5 + HTTP) — rule-based egress via Aeza VPS. See [sing-box](sing-box-proxy.md) |
| `.13` | `1080` | microsocks (plain SOCKS5) |

## FortiClient VPN (`2GIS`)

Outbound corporate tunnel on `.13` (`fctvpn*`), driven/monitored by the
`forti-tray` agent. See [forti box](../hosts/dvedenko-net.md) and the forti-tray
project for the connect flow, credential quirks, and the UFW return-traffic fix.

## Security notes

- Every inbound path is a potential attack surface — keep router port-forwards
  minimal and prefer ZeroTier/zrok over opening ports where possible.
- RustDesk unattended password and zrok/ZeroTier tokens are **not** stored in this
  repo; rotate them in their own tools.
