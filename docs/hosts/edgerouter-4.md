# Host: EdgeRouter-4 (`192.168.1.1`)

The network's edge — WAN gateway, router, firewall, VLAN trunk, and LAN DNS/DHCP.

_Last verified: 2026-06-14._

## Identity

| Field | Value |
|-------|-------|
| Hostname | `EdgeRouter-4` |
| Model | Ubiquiti **EdgeRouter 4** (`ER-e300`, board `cavium,ubnt_e300`) |
| Firmware | EdgeOS `v3.0.1.5862409.250924.1408` (built 2025-09-24) |
| Base OS | Debian 9 "stretch", kernel `4.9.79-UBNT`, MIPS64 (Cavium Octeon) |
| SSH | `ssh ubnt@192.168.1.1` (OpenSSH 8.4p1, UI build) |
| Web UI | `https://192.168.1.1` (cert CN `UbiquitiRouterUI`) |
| Managed by | **UISP/UNMS** running on [home-controller](home-controller.md) |

## Interfaces

| Interface | State | Address | Role |
|-----------|-------|---------|------|
| `eth0` | UP | `83.243.71.58/28` | **WAN 1** (primary uplink) |
| `eth1` | DOWN | — | **WAN 2** (secondary, dormant — NAT rules exist for `ADDRv4_eth1`) |
| `eth2` | UP | — | LAN (bonded) |
| `eth3` | UP | — | LAN (bonded) |
| `bond0` | UP | `192.168.1.1/24` | **Main LAN** (eth2+eth3 LAG) |
| `bond0.2` | UP | `192.168.2.1/24` | VLAN 2 — **"Devices"** (IoT / TVs / consoles) |
| `bond0.3` | UP | `192.168.3.1/24` | VLAN 3 — provisioned, **no active leases** (purpose TBD) |

Dual-WAN is configured (port-forward NAT has both `ADDRv4_eth0` and `ADDRv4_eth1`
variants) but `eth1` is currently down, so only WAN 1 is live.

## DHCP

Active pools (from `show dhcp leases`, 2026-06-14):

| Pool | Subnet | Used for |
|------|--------|----------|
| `LAN` | `192.168.1.0/24` | Main devices — workstations, phones, UniFi APs (U6-Lite `.219`, U6-Plus `.243`), Yandex stations, smart-home gear |
| `Devices` | `192.168.2.0/24` | VLAN 2 — TVs (LG webOS), Xbox, misc IoT |
| (VLAN 3) | `192.168.3.0/24` | No active leases observed |

Static / non-DHCP hosts on the main LAN include `.1` (router), `.2`, `.13`,
`.10` (Canon printer), and `.3` (unidentified — MAC `b0:95:75:…`, **not** a UniFi AP).

## Port-forwards (DNAT)

Captured from `iptables -t nat` (2026-06-14). Two mechanisms are in play — the UISP
"port-forward" feature (`UBNT_PFOR_*`) and manual NAT rules (`VYATTA_DNAT`, comments
`NAT-nn`):

| External port (proto) | → Destination | Service |
|-----------------------|---------------|---------|
| `80`, `443` (tcp+udp) | `192.168.1.2` | nginx reverse proxy (+ UDP/443 relay) |
| `21115`–`21117` (tcp+udp) | `192.168.1.13` | RustDesk (hbbs/hbbr) |
| `32400` (tcp) | `192.168.1.2` | **stale** — Plex not running |
| `51413` (tcp+udp) | `192.168.1.2` | **stale** — Transmission not running |

`NAT-21..24`, `NAT-41`, `NAT-61` are the WAN-2 (`eth1`) duplicates of the above,
inactive while `eth1` is down.

## Roles

- **Routing / NAT** between WAN and the three internal subnets.
- **Firewall** — WAN-in / port-forwards (e.g. the `21115–21117` RustDesk forwards
  noted in the forti-tray docs) live in the EdgeOS config.
- **DNS** — listens on `:53` (EdgeOS `dnsmasq`), forwarding/serving the LAN.
- **DHCP** — scopes per subnet (ranges not enumerated here).
- **DDNS** — keeps `local.crsib.me` pointed at the WAN IP `83.243.71.58`.

## Where the real config lives

The full running config is `/config/config.boot` on the device and is **not
mirrored into this repo** because it contains hashed credentials, PSKs, and API
keys. Inspect or change it through:

- the EdgeOS CLI (`configure` → `show` / `set` / `commit` / `save`), or
- the EdgeRouter web UI, or
- **UISP** on `home-controller` (which manages this device centrally).

To review a section safely from the CLI, e.g. port-forwards:

```bash
ssh ubnt@192.168.1.1
show configuration commands | grep port-forward     # inside the EdgeOS shell
```

> ⚠️ Avoid dumping the entire `config.boot` into logs or this repo — it carries
> secrets.

## Open ports (from the LAN)

`22` (SSH), `80`/`443` (router UI), `53` (DNS). Verified 2026-06-14.

## TODO / to verify

- [ ] Confirm VLAN 3's intended purpose (provisioned but idle) and which SSIDs /
      switch ports map to VLAN 2 ("Devices") vs the main LAN.
- [ ] Record DHCP ranges (start/stop) and reserved/static mappings.
- [ ] Decide whether to drop the stale `32400` / `51413` port-forwards.
- [ ] Identify the `.3` device (MAC `b0:95:75:b3:3f:00`).
