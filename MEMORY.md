# home-network-config — Orientation

Fast context for a returning operator or Claude session. Full detail is in `docs/`.

## What this is
Knowledgebase + config index for the home network at `crsib.me`. **Docs describe
the live hosts**, not git history. The old Docker stack in git history
(Pi-hole/Unbound/Plex/SWAG/Authelia/…) is **retired** — ignore it.

## The network (verified 2026-06-14)
- **WAN:** `83.243.71.58/28`, DDNS `local.crsib.me`.
- **LAN:** `192.168.1.0/24`; **VLANs** `192.168.2.0/24`, `192.168.3.0/24` (purposes TBD).
- **Router** `192.168.1.1` — Ubiquiti **EdgeRouter 4**, EdgeOS 3.0.1, `ssh ubnt@`.
  Routing/firewall/NAT/DNS(`:53`)/DHCP/DDNS; managed by UISP on `.2`. Dual-WAN
  (eth1 dormant). Port-forwards: `80/443`→.2, `21115-7`→.13, stale `32400`/`51413`→.2.
  DHCP pools: `LAN` (.1.x), `Devices` VLAN2 (.2.x = IoT/TVs), **VLAN3 = Guest** (.3.x). Printer `.10` (Canon).
- **home-controller** `192.168.1.2` — Ubuntu 24.04, `ssh dvedenko@`. Runs:
  - native **nginx** reverse proxy (`:80/:443`, LE cert `local.crsib.me`)
  - native **UniFi Network** controller (`:8443`, Mongo `:27117`)
  - **UISP/UNMS** (Docker) managing the EdgeRouter (`:9443`, netflow `2055/udp`)
  - **Nextcloud** (Docker, `:9876`)
  - **MicroK8s** v1.32.13 (`:16443`) — idle, only kube-system pods
  - **Headscale** (Tailscale control plane) — `headscale.crsib.me` → `127.0.0.1:8081`; mesh `100.64.0.0/10`
  - **sing-box** proxy gateway — mixed/SOCKS `:1080`, trojan `:10443`, vless `:11443` (last two fronted by nginx); egress NaïveProxy → Aeza `104.238.29.139:443`; DoT via NextDNS. nginx also relays UDP/443 → Aeza. See `docs/services/sing-box-proxy.md`
  - **ZeroTier** (node `09fe4d687b`), **zrok** shares (unms/router/uisp)
  - `local.crsib.me` root → **Nextcloud** (`:9876`)
- **dvedenko-24.04-net** `192.168.1.13` — Ubuntu 24.04, `ssh dvedenko@`. Runs:
  FortiClient VPN `2GIS`, **RustDesk** (ID `255246947`, `21115`–`21119`),
  **Tailscale** node `dvedenko-24`=`100.64.0.1`, microsocks `:1080`, resolved DNS
  `:53`, iperf3 `:5201`, CUPS `:631`. Headless. → deep ref: **`../forti-tray/MEMORY.md`**.

## Two Ubiquiti planes (don't confuse)
UniFi Network (`:8443`, UniFi gear) **and** UISP/UNMS (`:9443`, the EdgeRouter) —
both on `.2`.

## Rules
- No secrets in git. Read-only on hosts unless asked to change something.
- Don't dump EdgeRouter `config.boot` (holds secrets) into the repo/transcript.
- Re-verify against live hosts; bump "Last verified" dates. See
  `runbooks/README.md` → "Refresh the knowledgebase from live hosts".

## Open TODOs
- DHCP start/stop ranges; SSID/port → VLAN mapping.
- Protocol of UDP/443→Aeza relay (QUIC/HTTP3 for sing-box?); cert coverage for `headscale.crsib.me`.
- Pin `.3` guest device (MAC `b0:95:75:…`); zrok reserved URLs.
- Drop stale `32400`/`51413` port-forwards (Plex/Transmission gone).

_Resolved 2026-06-14: VLAN2="Devices", VLAN3=Guest; port-forwards mapped; cert =
certbot/nginx (not Cloudflare wildcard); vhosts enumerated (local→Nextcloud,
headscale.crsib.me→:8081); `:1080`/`:8081` identified; sing-box gateway documented;
MicroK8s idle; Headscale/Tailscale discovered; `.10`=Canon printer; APs `.219`/`.243`._
