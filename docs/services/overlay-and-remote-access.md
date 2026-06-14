# Overlay Networks & Remote Access

How the network is reached from outside, and the overlays layered on top of the
physical LAN.

_Last verified: 2026-06-14._

## Entry points from the internet

| Path | Terminates on | Use |
|------|---------------|-----|
| `local.crsib.me` (DDNS → `83.243.71.58`) + router port-forwards | nginx on `.2`, RustDesk on `.13` | Web apps (`https://local.crsib.me`) and RustDesk relay |
| **zrok** share tunnels | `home-controller` `.2` | On-demand public URLs for `unms`, `router`, `uisp` |
| **ZeroTier** | `home-controller` `.2` | Flat overlay to reach the box (and routed LAN, if configured) without port-forwards |
| **RustDesk** (by ID) | `dvedenko-net` `.13` | Remote desktop / headless control |
| **FortiClient VPN** | `dvedenko-net` `.13` | Outbound corp (`2GIS`) tunnel — not an inbound path |

## ZeroTier

- `home-controller` is a member: node `09fe4d687b`, client v1.16.1, status ONLINE.
- Provides an off-LAN way to reach `.2` regardless of the home WAN IP.
- **TBD:** record the joined network ID(s) and whether managed routes expose the
  whole `192.168.1.0/24`.

## zrok (OpenZiti)

Three `openziti/zrok` share containers run on `.2`:

- `unms-zrok-share` — exposes the UISP/UNMS UI
- `router-zrok-share` — exposes the EdgeRouter UI
- `uisp-zrok-share` — exposes UISP

These create public zrok URLs without touching router port-forwards. **TBD:** pin
the reserved share names / URLs and where the zrok env/token is stored.

## RustDesk (self-hosted)

- Server (`hbbs`/`hbbr`) **and** client both on `.13`; box **ID `255246947`**.
- Ports `21115`–`21119`; relay advertised as `local.crsib.me` (router forwards
  `21115`–`21117`).
- **LAN clients must use `192.168.1.13`** (the UDP rendezvous reply returns from the
  LAN IP, which same-LAN clients reject when using the public name).
- Do **not** enable `direct-server` on `.13` (port `21118` collides with hbbs's
  websocket → crash loop).
- Full detail: [forti-tray/MEMORY.md](../../../forti-tray/MEMORY.md).

## SOCKS5 proxies

| Host | Port | Service |
|------|------|---------|
| `.2` | `1080` | SOCKS5 (outbound) |
| `.13` | `1080` | microsocks |

## FortiClient VPN (`2GIS`)

Outbound corporate tunnel on `.13` (`fctvpn*`), driven/monitored by the
`forti-tray` agent. See [forti box](../hosts/dvedenko-net.md) and the forti-tray
project for the connect flow, credential quirks, and the UFW return-traffic fix.

## Security notes

- Every inbound path is a potential attack surface — keep router port-forwards
  minimal and prefer ZeroTier/zrok over opening ports where possible.
- RustDesk unattended password and zrok/ZeroTier tokens are **not** stored in this
  repo; rotate them in their own tools.
