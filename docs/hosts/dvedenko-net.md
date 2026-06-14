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
