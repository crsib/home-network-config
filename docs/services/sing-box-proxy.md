# sing-box Proxy Gateway (`.2`)

A **rule-based proxy gateway** running natively on `home-controller`. It gives LAN
clients a smart SOCKS/HTTP proxy that sends *filtered* destinations through an
off-site relay and everything else out directly, and it also exposes encrypted
inbounds for remote clients.

_Last verified: 2026-06-14._

## Runtime

| Field | Value |
|-------|-------|
| Binary | `/usr/bin/sing-box` v1.13.11 (native, systemd `sing-box.service`, active) |
| Config | `/etc/sing-box/config.json` (**contains secrets** — UUIDs/passwords; not mirrored here) |
| Host | `home-controller` `192.168.1.2` |

## Inbounds

| Tag | Type | Listen | Exposure |
|-----|------|--------|----------|
| `mixed-in` | mixed (SOCKS5 + HTTP) | `0.0.0.0:1080` | LAN clients — the `:1080` proxy on `.2` |
| `trojan-in` | Trojan over WebSocket | `127.0.0.1:10443` | Remote — fronted by **nginx** (TLS + WS) on `local.crsib.me` |
| `vless-in` | VLESS over WebSocket | `127.0.0.1:11443` | Remote — fronted by **nginx** (TLS + WS) |

> The Trojan/VLESS inbounds bind to loopback; nginx terminates TLS and proxies the
> WebSocket path(s) to them — that's the `$ws_port` upstream seen in the nginx
> config. See [reverse proxy](reverse-proxy-and-certs.md).

## Outbounds & routing

| Tag | Type | Target |
|-----|------|--------|
| `proxy` | **naive** (NaïveProxy) | `104.238.29.139:443` — external **Aeza** VPS |
| `direct` | direct | local egress |

Routing is **rule-based**: domain/IP rule-sets (`refilter_domains`,
`refilter_ipsum`, `manual_domains_proxy` — remote-fetched lists) decide which
traffic takes the `proxy` outbound (→ Aeza) versus `direct`. This is a classic
anti-censorship split-routing setup.

## DNS (inside sing-box)

- `dns-secure` — DoT to `45.90.28.187` (**NextDNS**), used for proxied lookups.
- `dns-direct` — plain UDP to `127.0.0.1:5555` for direct lookups.

This is sing-box's *internal* resolver, independent of the LAN DNS on the
EdgeRouter (see [dns.md](dns.md)).

## Relationship to the nginx UDP/443 relay

nginx also `stream`-proxies inbound **UDP/443 → `104.238.29.139:55444`** (same Aeza
VPS, different port). Both the sing-box `naive` outbound and that UDP relay point at
the Aeza box — together they form the off-site datapath. **TBD:** confirm whether
the UDP/443 relay is a QUIC/HTTP3 path for the same proxy or a separate service.

## Related

- **`../sing-box-helper`** project on this workstation likely manages/generates this
  config — cross-check before hand-editing `/etc/sing-box/config.json`.

## Security note

`/etc/sing-box/config.json` holds inbound credentials (Trojan password, VLESS UUID)
and the NaïveProxy auth. Keep it on the host only; never commit it. Rotate via the
helper/config, not by pasting secrets anywhere.
