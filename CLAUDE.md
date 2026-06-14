# CLAUDE.md — home-network-config

This repo is a **knowledgebase + configuration index** for the home network at
`crsib.me`. It is documentation-first: the authoritative configuration lives on the
devices (EdgeOS, nginx, Docker, MicroK8s, UISP/UniFi), and these docs describe and
index that live reality.

## Working principles

1. **Source of truth is the live host, not git history.** This repo's git history
   contains an old Docker stack (Pi-hole/Unbound/Plex/SWAG/Authelia/…) that is
   **retired** and must not be treated as current. When unsure, SSH in and check.
2. **Never commit secrets.** No passwords, API tokens, PSKs, or private keys.
   Reference *where* a secret lives, not its value. `.gitignore` enforces the
   common cases; keep it that way.
3. **Read-only by default on the hosts.** Inventory with `ssh`, `docker ps`,
   `ss -tlnp`, `ip -br addr`, `cat /etc/version`, etc. Do **not** change device
   config unless the user explicitly asks. Avoid dumping whole config files that
   carry secrets (e.g. EdgeRouter `/config/config.boot`) into the transcript or repo.
4. **Date your facts.** Each doc carries a "Last verified" line; update it when you
   re-confirm, and flag anything unverified as `TBD` / `(unverified)`.

## The three core hosts

| Host | IP | SSH | Role |
|------|----|-----|------|
| EdgeRouter-4 | `192.168.1.1` | `ubnt@` | Router / firewall / DNS / DHCP / VLANs |
| home-controller | `192.168.1.2` | `dvedenko@` | nginx proxy, UniFi + UISP, Nextcloud, MicroK8s, ZeroTier, zrok |
| dvedenko-24.04-net | `192.168.1.13` | `dvedenko@` | FortiClient VPN, RustDesk, SOCKS, iperf |

## Layout

- `README.md` — entry point and index.
- `docs/topology.md` — subnets, VLANs, WAN, host inventory, diagram.
- `docs/access.md` — how to reach hosts; credentials policy.
- `docs/hosts/` — per-host deep dives.
- `docs/services/` — cross-cutting service docs (DNS, proxy/certs, overlays).
- `runbooks/` — operational procedures (incl. how to refresh this KB from live hosts).
- `MEMORY.md` — fast orientation for a returning operator / Claude session.

## Related

- **`../forti-tray`** — Windows tray agent for the `.13` box. Its `MEMORY.md` is the
  authoritative deep reference for that host; link to it instead of duplicating.

## Environment notes

- Operator workstation is **Windows / PowerShell**. When piping native command
  output (e.g. `ssh ... 'docker ps --format ...'`), beware PowerShell stripping
  embedded double-quotes from Go templates — use separators without quotes/spaces
  (e.g. `--format={{.Names}}@@{{.Image}}`) or parse raw output locally.
