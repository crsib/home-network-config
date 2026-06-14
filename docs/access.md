# Access & Credentials

_Last verified: 2026-06-14._

## Principle: no secrets in this repo

Passwords, API tokens, PSKs, and private keys are **never** committed here. They
live on the hosts, in the device GUIs (UniFi / UISP / EdgeOS), or in a password
manager. The [.gitignore](../.gitignore) blocks the usual offenders
(`.env`, `*.key`, `*.pem`, `cloudflare.ini`, …). When a doc needs to reference a
secret, it names **where to find it**, not the value.

## SSH

Key-based auth is set up from the Windows workstation to the home hosts. Relevant
entries from `~/.ssh/config`:

| Alias | Host | User | Notes |
|-------|------|------|-------|
| `Router` | `192.168.1.1` | `ubnt` | EdgeRouter 4 (EdgeOS) — key auth works |
| — | `192.168.1.4` | — | EdgeSwitch — **web UI only** (`https://192.168.1.4`); SSH `:22` closed |
| — | `192.168.1.5` | — | EdgeSwitch — **web UI** (`https://192.168.1.5`); dropbear SSH up but **no key auth** (password-only) |
| — | `192.168.1.3` | — | TP-Link switch — web UI only (`http://192.168.1.3`), no SSH |
| `HomeController` / (bare IP) | `192.168.1.2` | `dvedenko` | services host |
| — | `192.168.1.13` | `dvedenko` | forti box; key auth, `BatchMode=yes` works |
| `MacBook` | `192.168.1.17` | `dvedenko` | _(unverified)_ |
| `MacBook-16` | `192.168.1.18` | `d.vedenko` | _(unverified)_ |
| `OrangePi` | `192.168.1.165` | `dvedenko` | _(unverified)_ |
| `UbuntuStudio` | `192.168.1.127` | `dvedenko` | _(unverified)_ |

The SSH config also holds **external** hosts (AWS Lightsail, VDS endpoints) that
are out of scope for the home network and not documented here.

### Quick connect

```powershell
ssh ubnt@192.168.1.1          # router (EdgeOS CLI)
ssh dvedenko@192.168.1.2      # home-controller
ssh dvedenko@192.168.1.13     # forti box
```

`192.168.1.13` host-key handling and the russh TOFU details used by the tray agent
are documented in [forti-tray/MEMORY.md](../../forti-tray/MEMORY.md).

## Web admin endpoints (LAN)

| Service | URL | Host |
|---------|-----|------|
| EdgeRouter UI | `https://192.168.1.1` | `.1` |
| UniFi Network | `https://192.168.1.2:8443` | `.2` |
| UISP / UNMS | `https://192.168.1.2:9443` (and `:443` via proxy) | `.2` |
| Nextcloud | `http://192.168.1.2:9876` (and via `local.crsib.me` proxy) | `.2` |
| Reverse-proxied apps | `https://<app>.local.crsib.me` | `.2` (nginx) |

Credentials for each live in their respective app / password manager — not here.

## sudo

- `192.168.1.13`: passwordless sudo via `/etc/sudoers.d/nopasswd` (see forti-tray notes).
- `192.168.1.2`: standard sudo (interactive password) unless noted.
- `192.168.1.1`: EdgeOS `ubnt` admin; full config (incl. secrets) via the EdgeOS
  CLI/GUI — intentionally not mirrored here.
