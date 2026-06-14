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
| `bond0.3` | UP | `192.168.3.1/24` | VLAN 3 — **Guest network** (no active leases at sweep) |

Dual-WAN is configured (port-forward NAT has both `ADDRv4_eth0` and `ADDRv4_eth1`
variants) but `eth1` is currently down, so only WAN 1 is live.

## DHCP

Pools from the generated `dhcpd.conf` (2026-06-14). **All pools advertise public
DNS `1.1.1.1, 8.8.8.8`** (see [dns.md](../services/dns.md)).

| Pool | Subnet | Gateway | Range | Used for |
|------|--------|---------|-------|----------|
| `LAN` | `192.168.1.0/24` | `.1.1` | `.1.38–.164`, `.166–.206`, `.208–.243` | Main devices — workstations, phones, UniFi APs (U6-Lite `.219`, U6-Plus `.243`), Yandex stations, smart-home gear |
| `Devices` | `192.168.2.0/24` | `.2.1` | `.2.20–.2.200` | VLAN 2 — TVs (LG webOS), Xbox, misc IoT |
| `Guest` | `192.168.3.0/24` | `.3.1` | `.3.10–.3.255` | VLAN 3 — guest network |

The gaps in the LAN range (`.165`, `.207`, `.244`+) leave room for static hosts:
`.1` (router), `.2`, `.13`, `.10` (Canon printer), `.165` (OrangePi), and `.3`
(**TP-Link** device, MAC `b0:95:75:…` — associated with the guest network).

## Wi-Fi / SSIDs (via UniFi controller on `.2`)

UniFi APs (U6-Lite, U6-Plus) broadcast these SSIDs, all mapped to the **untagged
`Default` network** (`192.168.1.0/24`) in the controller — **none are VLAN-tagged**:

| SSID | Band | Network |
|------|------|---------|
| `crsib-network` / `crsib-network-2.4` | 5 GHz / 2.4 GHz | Default (LAN) |
| `crsib-network-devices` / `crsib-network-devices-2.4` | 5 GHz / 2.4 GHz | Default (LAN) |

> **How VLAN 2 / VLAN 3 are actually assigned:** the UniFi SSIDs are untagged, so
> Wi-Fi does **not** place clients on the Devices/Guest VLANs. The router does
> router-on-a-stick (`bond0.2` / `bond0.3`); the **VLAN tagging happens at the
> managed switches** (the EdgeSwitches `.4`/`.5` and/or the TP-Link switch `.3`) via
> per-port VLAN profiles. No guest SSID exists in the UniFi controller, so guest
> access is wired/port-based. **TBD:** capture the per-port VLAN assignments from the
> switches (they aren't in the UniFi Network controller — see below).

## Switches (Layer 2)

The router trunks the LAN to managed switches. The two EdgeSwitches **are adopted
in UISP** on `.2` (they're EdgeMAX gear — that's why the UniFi *Network* controller
doesn't list them); the TP-Link is unmanaged (a "blackBox" node in UISP). Models /
names confirmed via the UISP API 2026-06-14:

| IP | Name (UISP) | Model | Ports | FW | MAC | Mgmt |
|----|-------------|-------|-------|----|----|------|
| `.4` | **MainSwitch** | Ubiquiti **EdgeSwitch 10X** | 10 (+8 LAGs) | 1.3.1 | `18:e8:29:47:21:4d` | UISP + `https://192.168.1.4` |
| `.5` | **BedroomSwitch** | Ubiquiti **EdgeSwitch 5XP** (PoE) | 5 | 2.1.0 | `e0:63:da:e5:3b:3f` | UISP + `https://192.168.1.5` |
| `.3` | — | **TP-Link** managed switch | — | — | `b0:95:75:b3:3f:00` | `http://192.168.1.3` only |

> **Per-port VLAN map still pending.** UISP reports the switches but **not** their
> per-port VLAN assignments (the API returns `vlan=None` for every port). None of the
> switches are SSH-key accessible (`.4` `:22` closed; `.5` dropbear is password-only;
> `.3` no SSH), so the port→VLAN map must be read from the **web UIs** or a config
> backup. This is the one remaining manual step for the L2 picture.

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

- [ ] Resolve the Wi-Fi VLAN gap above (how clients reach VLAN 2 / VLAN 3).
- [ ] Confirm the `.3` TP-Link device's exact model/role (guest AP?).
- [ ] Decide whether to drop the stale `32400` / `51413` port-forwards.
