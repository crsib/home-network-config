# Runbook: Deploy Pangolin (replace zrok)

Implements [ADR-0001](../docs/decisions/0001-replace-zrok-with-pangolin.md). Stands up
Pangolin on the VDSina VPS and exposes the three admin UIs behind SSO via a Newt
connector on `home-controller` (`.2`).

**Targets to publish**

| Resource | Internal target (reachable from `.2`) |
|----------|---------------------------------------|
| `router` | `https://192.168.1.1` (EdgeRouter UI) |
| `unifi` | `https://127.0.0.1:8443` (UniFi controller) |
| `uisp` | `https://127.0.0.1:9443` (UISP) |

**Hosts**

| Role | Host | Access |
|------|------|--------|
| Pangolin server | VDSina `89.110.79.146` | `ssh root@89.110.79.146` (alias `VDSinaWG`) — `:443` is free |
| Newt connector | `home-controller` `192.168.1.2` | `ssh dvedenko@192.168.1.2` (Docker) |

> ⚠️ Secrets (Cloudflare token, Pangolin admin creds, Newt id/secret) stay on the
> hosts — **never commit them**. This runbook references where they go, not values.

---

## 0. Decisions to confirm before starting

- **Public zone for resources** — recommend a dedicated wildcard, e.g.
  `*.ext.crsib.me` (keeps these off the `local.`/`headscale.` names that point home).
  Final names would be `router.ext.crsib.me`, `unifi.ext.crsib.me`, `uisp.ext.crsib.me`,
  plus `pangolin.ext.crsib.me` for the dashboard.
- **Cloudflare API token** — scoped token: *Zone → DNS → Edit* on `crsib.me` only.
- **First admin user** — email + strong password for Pangolin; enable TOTP after login.

## 1. Cloudflare DNS

In the Cloudflare dashboard for `crsib.me`:

1. `A  *.ext.crsib.me  → 89.110.79.146` (DNS-only / grey cloud, so Traefik terminates TLS).
2. `A  ext.crsib.me    → 89.110.79.146` (optional apex for the zone).
3. Create the scoped API token (above) — used by Traefik for DNS-01.

## 2. Pangolin on VDSina (`89.110.79.146`)

```bash
ssh root@89.110.79.146
# Docker + compose plugin if missing:
command -v docker || curl -fsSL https://get.docker.com | sh
```

Use Pangolin's installer (interactive) — it scaffolds the Traefik + Pangolin + Gerbil
compose stack and prompts for base domain, admin email, and Let's Encrypt:

```bash
mkdir -p /opt/pangolin && cd /opt/pangolin
curl -fsSL https://digpangolin.com/get-installer.sh | bash   # verify current install method on docs.fossorial.io first
./installer
```

During install set:
- **Base domain:** `ext.crsib.me`  · **Dashboard:** `pangolin.ext.crsib.me`
- **Let's Encrypt:** DNS-01, provider **Cloudflare**, paste the scoped API token
- **Admin:** your email + password

Confirm `:443` and `:51820/udp` (Gerbil/WireGuard) are open in the VDSina firewall.
Bring it up (`docker compose up -d`) and log into `https://pangolin.ext.crsib.me`.

## 3. Newt connector on `home-controller` (`.2`)

In the Pangolin UI: **create a Site** → it shows a `newt` run command with an
`id`/`secret` and the Gerbil endpoint. Run it as a container on `.2`:

```bash
ssh dvedenko@192.168.1.2
# from the values Pangolin shows (do not paste secrets into the repo):
docker run -d --name newt --restart unless-stopped \
  -e PANGOLIN_ENDPOINT=https://pangolin.ext.crsib.me \
  -e NEWT_ID=<id> -e NEWT_SECRET=<secret> \
  fosrl/newt
```

The site should show **connected** in the dashboard. Newt reaches the targets over the
LAN from `.2` (router by IP; UniFi/UISP on loopback).

## 4. Define resources (behind SSO)

For each, in **Resources → Add**:

| Resource name | Method | Host | Port | TLS to target |
|---------------|--------|------|------|---------------|
| `router` | site = the `.2` Newt site | `192.168.1.1` | 443 | yes (self-signed → enable "insecure"/skip-verify) |
| `unifi` | same site | `127.0.0.1` | 8443 | yes (self-signed → skip-verify) |
| `uisp` | same site | `127.0.0.1` | 9443 | yes (self-signed → skip-verify) |

Then attach **authentication** to each resource (Pangolin SSO; require login, enable
MFA). Optionally restrict to specific users. Test each URL in a fresh browser — you
should hit the Pangolin login first, then the admin UI.

> UniFi/UISP/EdgeOS present self-signed certs on their HTTPS ports, so the resource's
> upstream TLS must **skip verification** (Traefik `insecureSkipVerify`). The
> public-facing cert is still the valid Let's Encrypt one.

## 5. Cut over & decommission zrok

Once all three resources work with SSO:

```bash
ssh dvedenko@192.168.1.2
cd /home/dvedenko/zrok && docker compose -f unms/docker-compose*.yml down 2>/dev/null
# repeat for router/ and uisp/ ; then remove the dirs once satisfied
```

- Stop/remove the `*-zrok-*` containers and the `/home/dvedenko/zrok/{unms,router,uisp}` stacks.
- On VDSina, remove the old **zrok controller** service/containers (whatever held `:443`
  for `zrok.zrok.crsib.me`) so it doesn't fight Pangolin's Traefik for the port.
- Delete the `zrok.zrok.crsib.me` DNS record if no longer used.

## 6. Document

- Update [docs/services/overlay-and-remote-access.md](../docs/services/overlay-and-remote-access.md):
  replace the "zrok (DOWN)" section with the Pangolin design (server on VDSina, Newt on
  `.2`, resources + SSO).
- Update `MEMORY.md` and mark ADR-0001's action items done.
- Note in [access.md](../docs/access.md): admin UIs now at `https://{router,unifi,uisp}.ext.crsib.me` (SSO).

## Rollback

Pangolin runs entirely on VDSina + the Newt container on `.2`; it opens **no** home
ports. To roll back, stop the Newt container and the Pangolin stack — nothing on the
home LAN is changed. (Only revert zrok if you hadn't yet decommissioned it.)
