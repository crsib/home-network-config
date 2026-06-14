# Home Network — Configuration & Knowledgebase

The single source of truth for how the home network at **`crsib.me`** is wired,
what runs where, and how to operate it. This repo is primarily a **knowledgebase**:
the actual live configuration lives on the devices themselves (EdgeOS config,
nginx, Docker Compose, MicroK8s, UISP). Docs here describe and index that reality —
they are kept in sync by reading the live hosts, **not** by inventing config.

> ⚠️ **No secrets in git.** Passwords, API tokens, PSKs, and private keys stay on
> the hosts (or in a password manager). See [docs/access.md](docs/access.md) and
> [.gitignore](.gitignore).

## At a glance

| Host | IP | What it is |
|------|----|------------|
| **EdgeRouter-4** | `192.168.1.1` | Ubiquiti EdgeRouter 4 — WAN gateway, routing, VLANs, DNS/DHCP, firewall |
| **home-controller** | `192.168.1.2` | Ubuntu services host — reverse proxy, UniFi & UISP controllers, Nextcloud, MicroK8s, overlays |
| **dvedenko-24.04-net** | `192.168.1.13` | Ubuntu remote-access box — FortiClient VPN, RustDesk, SOCKS, iperf |

- **WAN / public IP:** `83.243.71.58/28` (DDNS `local.crsib.me`)
- **Main LAN:** `192.168.1.0/24` &nbsp;·&nbsp; **VLANs:** `192.168.2.0/24`, `192.168.3.0/24`
- **Internal naming:** `*.local.crsib.me` (Let's Encrypt wildcard via Cloudflare DNS)

## Map of this repo

| Path | Contents |
|------|----------|
| [docs/topology.md](docs/topology.md) | Subnets, VLANs, WAN, full host inventory, network diagram |
| [docs/access.md](docs/access.md) | How to reach each host (SSH inventory), credentials policy |
| [docs/hosts/](docs/hosts/) | Per-host deep dives — [router](docs/hosts/edgerouter-4.md), [controller](docs/hosts/home-controller.md), [forti box](docs/hosts/dvedenko-net.md) |
| [docs/services/](docs/services/) | Cross-cutting service docs — [overview](docs/services/overview.md), [DNS](docs/services/dns.md), [reverse proxy & certs](docs/services/reverse-proxy-and-certs.md), [overlays & remote access](docs/services/overlay-and-remote-access.md) |
| [docs/decisions/](docs/decisions/) | Architecture Decision Records (ADRs) |
| [runbooks/](runbooks/) | Step-by-step operational procedures |
| [MEMORY.md](MEMORY.md) | Fast orientation for a returning operator (or Claude session) |

## Related projects

- **[forti-tray](../forti-tray)** — Windows tray agent that monitors/controls the
  FortiClient VPN on `192.168.1.13`. Its `MEMORY.md` is the authoritative deep
  reference for that box; this repo summarizes and links to it.

## Conventions

- Every doc states **what was verified and when**. Unverified or assumed facts are
  flagged `TBD` / `(unverified)`.
- Last full host sweep: **2026-06-14**.
