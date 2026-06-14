# Reverse Proxy & Certificates

_Last verified: 2026-06-14._

## Reverse proxy: native nginx on `home-controller` (`.2`)

A **native** nginx (systemd, not containerized) owns `:80` and `:443` on
`192.168.1.2` and publishes internal apps under `*.local.crsib.me`.

- Config: `/etc/nginx/` (server blocks / site-confs).
- TLS material: `/etc/letsencrypt/`.
- Active and serving the `local.crsib.me` Let's Encrypt certificate (verified by
  TLS handshake on `192.168.1.2:443`).

```bash
ssh dvedenko@192.168.1.2 'sudo nginx -T 2>/dev/null | grep -E "server_name|proxy_pass|listen"'
```

## Certificates

| Cert | Where | Issuer | Notes |
|------|-------|--------|-------|
| `local.crsib.me` (wildcard) | nginx on `.2` | Let's Encrypt | Cloudflare **DNS-01** validation (managed by the certbot/Cloudflare flow on `.2`) |
| `UbiquitiRouterUI` | EdgeRouter `.1` | self-signed | Router UI only |
| `UniFi` | UniFi controller `.2:8443` | self-signed (Ubiquiti) | Controller GUI only |

The Cloudflare API credentials used for DNS-01 live on `.2` (e.g. a
`cloudflare.ini`) and are **git-ignored** — never commit them.

## Public exposure

- The router port-forwards `80`/`443` (and selected others) to `.2`, so
  `https://local.crsib.me` reaches the nginx proxy from the internet.
- Additional ad-hoc public access is provided by **zrok** tunnels — see
  [overlays & remote access](overlay-and-remote-access.md).

## TODO / to verify

- [ ] Enumerate the published vhosts (`server_name` → upstream) from `nginx -T`.
- [ ] Confirm the renewal mechanism (certbot timer / deploy hook) for the wildcard.
- [ ] Confirm exactly which external ports the router forwards to `.2`.
