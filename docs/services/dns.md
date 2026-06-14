# DNS & Naming

_Last verified: 2026-06-14._

## Resolvers on the network

| Resolver | Host | Listen | Role |
|----------|------|--------|------|
| EdgeOS `dnsmasq` | EdgeRouter-4 `.1` | `192.168.1.1:53` | Primary LAN resolver + DHCP-advertised DNS |
| systemd-resolved | dvedenko-net `.13` | `192.168.1.13:53`, `127.0.0.54:53` | Stub resolver, LAN-visible; serves corp names over the FortiClient tunnel |

> The old **Pi-hole + Unbound** pair (DoT to Cloudflare/Quad9) from this repo's git
> history is **retired** — see the note in [services/overview.md](overview.md).
> If ad-blocking DNS is reintroduced, document the new design here.

## Naming: `crsib.me`

- **Public DDNS:** `local.crsib.me` → WAN IP `83.243.71.58` (kept current by the
  EdgeRouter). Used as the external entry point and the certificate name.
- **Internal hostnames:** `*.local.crsib.me` resolve to LAN hosts and are fronted
  by the nginx reverse proxy on `home-controller`
  (see [reverse proxy & certs](reverse-proxy-and-certs.md)).
- **TLS:** a Let's Encrypt wildcard for `local.crsib.me` (Cloudflare DNS-01) is
  served by nginx on `.2`. Cert observed: `CN=local.crsib.me`, issuer Let's Encrypt.

## Corporate DNS (VPN-only)

When the FortiClient `2GIS` tunnel is up on `.13`, corp resolvers
`10.54.68.68` and `10.54.129.129` are reachable **only through the tunnel**. The
UFW return-traffic fix on `.13` exists specifically to keep these fast/reliable —
see [forti box](../hosts/dvedenko-net.md).

## Verify

```powershell
# LAN resolver
nslookup local.crsib.me 192.168.1.1
# resolved stub on the forti box
nslookup local.crsib.me 192.168.1.13
```

## TODO / to verify

- [ ] Confirm what upstreams the EdgeRouter dnsmasq forwards to (ISP vs public).
- [ ] List internal `*.local.crsib.me` A-records actually configured today.
- [ ] Decide whether `.13`'s resolved should be advertised to LAN clients at all.
