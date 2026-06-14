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
| `eth0` | UP | `83.243.71.58/28` | **WAN** (uplink to ISP) |
| `eth1` | DOWN | — | unused |
| `eth2` | UP | — | LAN (bonded) |
| `eth3` | UP | — | LAN (bonded) |
| `bond0` | UP | `192.168.1.1/24` | **Main LAN** (eth2+eth3 LAG) |
| `bond0.2` | UP | `192.168.2.1/24` | VLAN 2 |
| `bond0.3` | UP | `192.168.3.1/24` | VLAN 3 |

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

- [ ] Document VLAN 2 / VLAN 3 purposes and which switch ports / SSIDs map to them.
- [ ] Capture the port-forward list (service → external port → internal host).
- [ ] Record DHCP ranges and any static mappings.
