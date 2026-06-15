# Host: dvedenko-24.04-net (`192.168.1.13`)

The remote-access / VPN box. Headless (no monitor, no KVM), kept always-on, reached
over SSH, RustDesk, and the FortiClient corporate tunnel.

_Last verified: 2026-06-14._

## Identity

| Field | Value |
|-------|-------|
| Hostname | `dvedenko-24.04-net` |
| OS | Ubuntu 24.04.4 LTS, kernel `6.17.0-35-generic` |
| SSH | `ssh dvedenko@192.168.1.13` (key auth, `BatchMode=yes` works) |
| sudo | passwordless (`/etc/sudoers.d/nopasswd`) |
| Docker | none (services run as native systemd units) |

## Service map

| Service | Ports | Notes |
|---------|-------|-------|
| **FortiClient VPN** (`2GIS`) | tunnel `fctvpn*` | Corp VPN, user `d.vedenko`. CLI at `/usr/bin/forticlient`. Corp DNS `10.54.68.68` / `10.54.129.129` reachable only through the tunnel |
| **RustDesk** (self-hosted) | `21115`–`21119` (hbbs/hbbr) | Box ID **`255246947`**. Server + client both on this box; relay advertised as `local.crsib.me`. **LAN clients must connect via `192.168.1.13`** (UDP hairpin caveat) |
| **microsocks** (SOCKS5) | `1080` | Outbound proxy |
| **Tailscale** (Headscale mesh) | `tailscale0` | Node `dvedenko-24`, `100.64.0.1` / `fd7a:115c:a1e0::1`. Control plane is Headscale on [.2](home-controller.md) (`headscale.crsib.me`). Currently the only mesh node |
| **systemd-resolved** | `192.168.1.13:53`, `127.0.0.54:53` | DNS stub, LAN-visible |
| **iperf3** | `5201` | Throughput testing |
| **CUPS** | `631` | Printing stack |

## Operating notes (highlights)

- **Headless & no-sleep**: systemd sleep targets masked, logind idle ignored, XFCE
  power management disabled, X DPMS/screensaver off — so RustDesk always has a
  live `:0` session. A forced **virtual display** (GRUB `video=`/`drm.edid_firmware`
  with a self-generated 1080p EDID) replaces the removed KVM's HDMI/EDID.
- **UFW VPN-return fix**: `-A ufw-before-input -i fctvpn+ -j ACCEPT` in
  `/etc/ufw/before.rules` — fixes slow/dropped DNS caused by FortiClient's policy
  routing breaking conntrack on tunnel return traffic.
- **Sipeed NanoKVM**: physically removed 2026-06-03 (flaky USB); replaced by RustDesk.

## FortiClient hijacks inbound LAN DNS — `0x8` mark bypass

_Verified 2026-06-15. Distinct from the UFW VPN-return fix above._

FortiClient installs its own nftables (`fct_filter`, `fct_nat`, `fct_mangle`) at
higher-priority hooks than UFW. Its transparent DNS interception in `fct_mangle`
**PREROUTING** (priority `mangle` = −150) matches any UDP packet whose destination
is a **local listening socket** (`xt match "socket"`), sets `meta mark 0x1`, then
`ip rule 32765 (from all fwmark 0x1 lookup 100)` diverts it to FortiClient's own
DNS proxy — which **drops inbound LAN queries**.

- **Symptom:** `dig @192.168.1.13` works *locally* (arrives via `lo`/OUTPUT, skips
  PREROUTING) but **times out from any remote LAN host**. `tcpdump` shows the query
  arriving on `enp3s0` with **no reply and no UFW counter movement** — the packet is
  hijacked in PREROUTING before reaching the `ip filter` INPUT chain. Affects **any**
  daemon on `:53` (reproduced with both systemd-resolved's `DNSStubListenerExtra`
  and dnsmasq); it is **not** a resolver limitation and is unrelated to Tailscale.
- **Fix (persistent):** a separate nft table pre-marks inbound LAN DNS `0x8`
  *before* `fct_mangle` runs; FortiClient's `FCT-UDP-STAGE-1` has
  `mark and 0x8 == 0x8 accept`, so it leaves pre-marked packets alone.
  - `/etc/nftables-lan-dns-fix.conf` — table `lan_dns_fix`, chain PREROUTING
    `priority −200`: `iifname enp3s0 ip saddr 192.168.1.0/24 ip daddr 192.168.1.13
    udp/tcp dport 53 meta mark set 0x8`.
  - Loaded at boot by `lan-dns-fix.service` (oneshot, enabled). The table is
    independent of `fct_*`, so FortiClient reconnects don't remove it; only a full
    `nft flush ruleset` would — re-run the unit if that happens.
- **Generalizes:** the same `0x8`-bypass works for **any** future LAN-facing UDP
  service on `.13` that FortiClient's xt-socket match would otherwise hijack.

## Authoritative reference

This box is the target of the **[forti-tray](../../../forti-tray)** Windows tray
agent. The single source of truth for its FortiClient flow, the `forti-stats.sh`
metrics helper, the RustDesk setup, the headless/virtual-display config, and the
UFW fix is:

➡️ **[forti-tray/MEMORY.md](../../../forti-tray/MEMORY.md)**

Keep deep operational detail there; this page is the network-level summary and
should link, not duplicate.

## Monitoring

`forti-tray.exe` on the Windows workstation polls this box every 5 s over a
persistent SSH session (CPU / memory / tunnel & LAN throughput / VPN status),
shown in the system tray. Scheduled task: **"FortiClient VPN Monitor"** (at logon).
