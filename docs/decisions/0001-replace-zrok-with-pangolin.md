# ADR-0001: Replace zrok with Pangolin for public admin-UI access

**Status:** Accepted
**Date:** 2026-06-14
**Deciders:** Dmitry (operator)

## Context

Three internal **network admin UIs** were published to the internet via self-hosted
**zrok** reserved shares running on `home-controller` (`.2`):

| Share | Internal target | Service |
|-------|-----------------|---------|
| `router` | `https://192.168.1.1` | EdgeRouter UI |
| `unms` | `https://127.0.0.1:9443` | UISP / UNMS |
| `uisp` | `https://127.0.0.1:8443` | UniFi Network controller |

The zrok deployment is **broken**: it depends on a self-hosted zrok controller at
`zrok.zrok.crsib.me` → `89.110.79.146` (the *VDSina* VPS), whose `:443` service is
down, so the three share containers crash-loop (`unable to create zrok client …`).
The VPS itself is alive (`:22` and `:80` are up).

Forces at play:
- These are **high-value admin panels** — exposing them publicly demands real auth in
  front, not just a tunnel. zrok's reserved shares had **no authentication layer**.
- `crsib.me` is on **Cloudflare** (DNS-01 wildcard certs are easy).
- The home WAN is `83.243.71.58` (EdgeRouter); `.2` already runs nginx, Headscale,
  and Docker.
- The operator wants **public, browser-reachable, authenticated** access (decided
  2026-06-14) — not merely private remote access.

## Decision

Stand up **Pangolin** (self-hosted tunneled reverse proxy: Traefik + Gerbil
WireGuard server + built-in identity/SSO) on the **VDSina VPS `89.110.79.146`**
(repurposing the now-free `:443`), with a **Newt** connector on `home-controller`
(`.2`). Each admin UI becomes a Pangolin **resource** behind Pangolin's SSO, reached
at `https://<name>.<zone>` with certs via **Cloudflare DNS-01**. Retire the three
zrok share stacks and the dead zrok controller.

```
Browser ──TLS──> Pangolin/Traefik (VDSina :443, SSO gate)
                      │  WireGuard tunnel (Newt dials out)
                      ▼
                 Newt on .2 ──LAN──> 192.168.1.1 (router) / 127.0.0.1:8443 (UniFi) / :9443 (UISP)
```

## Options Considered

### Option A: Pangolin on VDSina + Newt on .2  ← chosen
| Dimension | Assessment |
|-----------|------------|
| Complexity | Med — Docker stack on VPS + one connector |
| Cost | Reuses an existing VPS; no new spend |
| Security | **Strong** — built-in SSO/MFA in front of every resource; no home ports opened |
| Maintenance | Single dashboard; actively-developed project |

**Pros:** Auth is built in (the whole point for admin panels); outbound-only tunnel
from home (no inbound firewall holes); Cloudflare DNS-01 wildcard; clean per-resource
access control; nice UI.
**Cons:** New component to learn/operate; depends on the VDSina VPS staying up (single
point of failure for public access — but failure just means "no public access", not a
breach).

### Option B: Rebuild the zrok controller on VDSina
| Dimension | Assessment |
|-----------|------------|
| Complexity | Med–High — self-hosting ziti/zrok controller is fiddly |
| Cost | Reuses VDSina |
| Security | **Weak by default** — shares have no auth; would still need a separate auth layer |
| Maintenance | Smaller ecosystem for this use case |

**Pros:** Closest to the existing setup; shares/targets already defined.
**Cons:** Reproduces the original sin (no auth on admin panels); more operational
overhead than Pangolin for the same outcome.

### Option C: Keep private — Headscale mesh + existing nginx, no public exposure
| Dimension | Assessment |
|-----------|------------|
| Complexity | **Low** — reuse what's already running |
| Cost | Zero |
| Security | **Strongest** — nothing public at all |
| Maintenance | Minimal |

**Pros:** No public attack surface; no VPS dependency; uses Headscale (already on `.2`).
**Cons:** Requires every client to be a mesh node — fails the operator's explicit
requirement for browser access from arbitrary/shared devices. **Rejected on
requirements**, but remains the recommended posture if that requirement ever relaxes.

## Trade-off Analysis

The core trade-off is **public reach vs. attack surface**. Option C is safest but was
ruled out by the need for arbitrary-device browser access. Given that constraint, the
deciding factor between A and B is **authentication**: Pangolin bundles SSO/MFA so the
admin panels are never exposed raw, whereas a zrok rebuild would re-expose them and
*then* need a bolted-on auth layer. Pangolin therefore achieves the requirement with
**less** total moving parts than B once auth is counted, and with a better operating
experience. Both A and B share VDSina as a single point of failure, which is
acceptable because an outage degrades to "no remote admin access," never to a breach.

## Consequences

**Easier:** authenticated public access to all three admin UIs from any browser; add
future resources (Nextcloud, etc.) as Pangolin resources in minutes; no home inbound
ports for these.
**Harder / new burden:** one more service to patch (Pangolin/Traefik/Newt); a
Cloudflare API token for DNS-01 must be stored on the VPS (secret); VDSina becomes
infrastructure that must stay patched and monitored.
**To revisit:** MFA enforcement policy; whether to front Nextcloud/`local.crsib.me`
through Pangolin too or keep them on home nginx; backup of Pangolin's config/DB;
whether to later swap built-in SSO for an external OIDC IdP.

## Action Items

See the implementation runbook: [runbooks/deploy-pangolin.md](../../runbooks/deploy-pangolin.md).

1. [ ] Pick the public zone for resources (e.g. `*.ext.crsib.me`) and add Cloudflare records → VDSina.
2. [ ] Create a scoped Cloudflare API token (DNS edit for `crsib.me`) for DNS-01.
3. [ ] Deploy the Pangolin stack on VDSina (`89.110.79.146`).
4. [ ] Install the Newt connector on `home-controller` (`.2`).
5. [ ] Define resources: `router` → `192.168.1.1`, `unifi` → `127.0.0.1:8443`, `uisp` → `127.0.0.1:9443`; put each behind SSO.
6. [ ] Verify end-to-end with MFA, then **decommission** the three zrok stacks (`/home/dvedenko/zrok/*`) and remove the dead `zrok.zrok.crsib.me` controller.
7. [ ] Update [overlay-and-remote-access.md](../services/overlay-and-remote-access.md) and `MEMORY.md`.
