# Reverse Proxy & Certificates

_Last verified: 2026-06-14._

## Reverse proxy: native nginx on `home-controller` (`.2`)

A **native** nginx (systemd, not containerized) owns `:80` and `:443` (TCP **and**
UDP) on `192.168.1.2`.

- Config: `/etc/nginx/` (server blocks / site-confs).
- TLS material: `/etc/letsencrypt/`.
- Verified serving the `local.crsib.me` Let's Encrypt cert on `192.168.1.2:443`.

### Published vhosts (HTTP/TCP)

| `server_name` | Upstream | Purpose |
|---------------|----------|---------|
| `local.crsib.me` | `http://localhost:9876` + a WebSocket upstream | **Nextcloud** (`9876`); the WS upstream forwards to **sing-box** Trojan/VLESS inbounds (`10443`/`11443`) — see [sing-box](sing-box-proxy.md) |
| `headscale.crsib.me` | `http://127.0.0.1:8081` | **Headscale** HTTP listener (Tailscale control plane) — see [overlays](overlay-and-remote-access.md) |
| `_` (default) | — | Catch-all default server |

```bash
ssh dvedenko@192.168.1.2 'sudo nginx -T 2>/dev/null | grep -E "server_name|proxy_pass"'
```

### nginx `stream` — UDP relay on `:443`

Besides HTTP, nginx has a **`stream {}`** block that listens on **UDP `:443`** and
proxies it to an **external** host:

```
upstream udp_backend { server 104.238.29.139:55444; }
server { listen 443 udp; proxy_pass udp_backend; }
```

`104.238.29.139` is an external VPS (the `AezaNaive` host in the SSH config) — the
same box the **sing-box** `naive` outbound targets (`:443`). So inbound **UDP/443**
(port-forwarded by the router to `.2`) is relayed off-site as part of the proxy
datapath, distinct from the TCP/443 web stack. See [sing-box](sing-box-proxy.md).

## Certificates

| Cert | Where | Issuer | How |
|------|-------|--------|-----|
| `local.crsib.me` (+ SANs `headscale.crsib.me`, `derp.crsib.me`) | nginx on `.2` | Let's Encrypt | **certbot (snap)**, **`nginx` authenticator (HTTP-01)**. Renewal: `/etc/letsencrypt/renewal/local.crsib.me.conf`; auto-renew via `snap.certbot.renew.timer` |
| `UbiquitiRouterUI` | EdgeRouter `.1` | self-signed | Router UI only |
| `UniFi` | UniFi controller `.2:8443` | self-signed (Ubiquiti) | Controller GUI only |

> Correction (was previously assumed Cloudflare DNS-01 wildcard): the live cert is a
> **single SAN cert** covering `local.crsib.me`, `headscale.crsib.me`, and
> `derp.crsib.me` (the Headscale DERP relay), issued via the **nginx HTTP-01**
> plugin — not a Cloudflare DNS wildcard.

## Public exposure (router port-forwards → `.2`)

| External port | → Internal | Backed by |
|---------------|-----------|-----------|
| TCP/UDP `80`, `443` | `192.168.1.2:80/443` | nginx (web + UDP relay) |
| TCP `32400` | `192.168.1.2:32400` | **stale** — Plex not running (nothing listens) |
| TCP/UDP `51413` | `192.168.1.2:51413` | **stale** — Transmission not running |

Full DNAT table: [edgerouter-4.md](../hosts/edgerouter-4.md). Additional ad-hoc
public access via **zrok** tunnels — see [overlays](overlay-and-remote-access.md).

## TODO / to verify

- [ ] Confirm cert coverage for `headscale.crsib.me` (separate cert vs SAN).
- [ ] Confirm whether the UDP/443 → Aeza relay is QUIC/HTTP3 for the same proxy.
- [ ] Remove the stale `32400` / `51413` port-forwards on the router (or restore the services).
