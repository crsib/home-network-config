# DNS & Naming

_Last verified: 2026-06-14._

## What clients actually use

**DHCP hands every subnet public resolvers — `1.1.1.1` (Cloudflare) and `8.8.8.8`
(Google) — not a local resolver.** Verified in the router's generated
`dhcpd.conf` (2026-06-14): the `LAN`, `Devices`, and `Guest` scopes all advertise
`domain-name-servers 1.1.1.1, 8.8.8.8`. So normal LAN clients resolve straight to
the internet; there is **no network-wide ad-blocking or split-horizon DNS** today.

## Resolvers present on the network

| Resolver | Host | Listen | Role |
|----------|------|--------|------|
| EdgeOS `dnsmasq` | EdgeRouter-4 `.1` | `192.168.1.1:53` | Listening, but **not** advertised via DHCP (clients use 1.1.1.1/8.8.8.8) |
| systemd-resolved | dvedenko-net `.13` | `192.168.1.13:53`, `127.0.0.54:53` | Stub resolver, LAN-visible; serves corp names over the FortiClient tunnel. ⚠️ LAN-facing `:53` only works because of the FortiClient **`0x8` nftables bypass** — see [forti box](../hosts/dvedenko-net.md#forticlient-hijacks-inbound-lan-dns--0x8-mark-bypass) |
| sing-box DNS | home-controller `.2` | internal | DoT→NextDNS / direct, for proxied traffic only — see [sing-box](sing-box-proxy.md) |

> The old **Pi-hole + Unbound** pair (DoT to Cloudflare/Quad9) from this repo's git
> history is **retired** — see the note in [services/overview.md](overview.md).
> If ad-blocking DNS is reintroduced, point DHCP at it (currently public DNS).

## Naming: `crsib.me`

- **Public DDNS:** `local.crsib.me` → WAN IP `83.243.71.58` (kept current by the
  EdgeRouter). Used as the external entry point and the certificate name.
- **Internal hostnames:** `*.local.crsib.me` resolve to LAN hosts and are fronted
  by the nginx reverse proxy on `home-controller`
  (see [reverse proxy & certs](reverse-proxy-and-certs.md)).
- **TLS:** a Let's Encrypt wildcard for `local.crsib.me` (Cloudflare DNS-01) is
  served by nginx on `.2`. Cert observed: `CN=local.crsib.me`, issuer Let's Encrypt.

## mDNS / service discovery across VLANs

There is **no mDNS reflection today** — `.local` discovery (Bonjour, Chromecast,
AirPlay) is confined to each VLAN's broadcast domain (TTL=1 link-local multicast).
Post-migration, OPNsense can bridge it with the **`os-mdns-repeater`** (Avahi
reflector) plugin, **scoped to LAN ↔ VLAN 2 only** (Guest excluded), with scoped
inter-VLAN firewall rules for the follow-up unicast. See the optional step
[2a in the migration runbook](../../runbooks/migrate-to-topton-box.md#2a-cross-vlan-mdns-optional--bonjourchromecastairplay-across-vlans).

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

## Names published under `crsib.me` (TLS SANs on the `.2` cert)

`local.crsib.me`, `headscale.crsib.me`, `derp.crsib.me` — all resolve/route to
`home-controller` via the nginx proxy. `derp.crsib.me` is the Headscale embedded
**DERP** relay.

## TODO / to verify

- [ ] Decide whether to reintroduce a filtering resolver and point DHCP at it
      (today DHCP = public `1.1.1.1` / `8.8.8.8`).
- [ ] List internal `*.local.crsib.me` records actually configured today.
- [ ] Confirm the live `.13:53` front-end. A 2026-06-15 debug session found
      **dnsmasq** (`dnsmasq-lan.service`, `cache-size=0`) bound to `192.168.1.13:53`
      forwarding to `127.0.0.53` (resolved); the table above lists only resolved.
      dnsmasq is now redundant given the `0x8` bypass — resolved's
      `DNSStubListenerExtra=192.168.1.13` (commented out in
      `/etc/systemd/resolved.conf.d/external.conf`) would serve LAN equally well.
