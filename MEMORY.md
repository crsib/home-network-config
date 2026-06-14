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

## Key facts from deep dig (2026-06-14)
- **DHCP DNS = public** `1.1.1.1`/`8.8.8.8` for all subnets (no local/filtering resolver).
  Ranges: LAN `.38–164/166–206/208–243`, Devices `.2.20–200`, Guest `.3.10–255`.
- **Switches**: `.4` & `.5` = Ubiquiti **EdgeSwitch** (NOT in UniFi Network ctrlr —
  EdgeMAX gear, own web UI / UISP); `.3` = **TP-Link** managed switch. VLAN 2/3
  tagging is done at these switches' ports, NOT via Wi-Fi.
- **Wi-Fi**: UniFi controller (.2) manages only 2 APs (U6 Lite `.219`, U6+ `.243`);
  SSIDs `crsib-network`(+2.4) & `crsib-network-devices`(+2.4) all untagged→Default LAN.
  No guest SSID — guest is wired/port-based.
- **Cert** `local.crsib.me` SANs: `headscale.crsib.me`, `derp.crsib.me` (Headscale DERP).
- **zrok DOWN**: self-hosted controller `zrok.zrok.crsib.me` (`89.110.79.146`/VDSinaWG)
  unreachable → shares crash-loop. Targets: unms→`:9443`, router→`192.168.1.1`, uisp→`:8443`.
- **UDP/443→Aeza `104.238.29.139:55444`**: handled on the Aeza VPS (not sing-box,
  `quic:false`); external host, not probed.

## Open TODOs
- Capture per-port VLAN profiles + models from switches via **web UI** — none are
  SSH-key accessible (`.4` SSH off; `.5` dropbear password-only; `.3` no SSH).
- Confirm whether the EdgeSwitches are adopted in UISP on `.2`.
- Probe Aeza VPS for the UDP/443 datapath (needs explicit authorization — external host).
- Bring zrok controller (`89.110.79.146`) back, or retire the shares.
- Drop stale `32400`/`51413` port-forwards (Plex/Transmission gone).

_Earlier resolved: VLAN2="Devices"/VLAN3=Guest; port-forwards mapped; cert=certbot/nginx
(not Cloudflare wildcard); vhosts (local→Nextcloud, headscale→:8081); sing-box gateway;
MicroK8s idle; Headscale/Tailscale; `.10`=Canon; UniFi APs `.219`/`.243`._
