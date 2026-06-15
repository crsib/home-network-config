# Runbooks

Step-by-step operational procedures for this network. Each runbook is short,
copy-pasteable, and assumes the access in [docs/access.md](../docs/access.md).

## Index

- **[migrate-to-topton-box.md](migrate-to-topton-box.md)** — consolidate `.2` + the ER-4
  onto the new Topton box (Proxmox + OPNsense VM + Ubuntu services VM); dual-WAN failover,
  local DNS→NextDNS, Pangolin-behind-nginx, retire the ER-4 to a cold spare
  (implements [ADR-0002](../docs/decisions/0002-consolidate-controller-and-router.md)).
- **[deploy-pangolin.md](deploy-pangolin.md)** — _(superseded by the migration above)_
  original VDSina-VPS + Newt deployment to publish admin UIs behind SSO, replacing zrok
  (implemented [ADR-0001](../docs/decisions/0001-replace-zrok-with-pangolin.md)).

Other good candidates:

- **Add an internal `*.local.crsib.me` app** — DNS record + nginx vhost on `.2`.
- **Renew / reissue the wildcard cert** — certbot + Cloudflare DNS-01 on `.2`.
- **Add a router port-forward** — EdgeOS / UISP, plus update
  [services/overview.md](../docs/services/overview.md).
- **Re-pair / reset RustDesk on `.13`** — see
  [forti-tray/MEMORY.md](../../forti-tray/MEMORY.md).
- **Refresh this knowledgebase from live hosts** — see below.

## Refresh the knowledgebase from live hosts

This repo documents reality; reality lives on the devices. To re-verify, sweep the
hosts read-only and update the docs (do **not** paste secrets):

```powershell
# Reachability
foreach ($ip in '192.168.1.1','192.168.1.2','192.168.1.13') {
  Test-Connection $ip -Count 1 -Quiet
}

# Router identity (non-secret)
ssh ubnt@192.168.1.1 'cat /etc/version; ip -br addr'

# Service hosts
ssh dvedenko@192.168.1.2  'sudo docker ps; sudo ss -tlnp; snap list'
ssh dvedenko@192.168.1.13 'systemctl list-units --type=service --state=running; sudo ss -tlnp'
```

Then bump the "Last verified" date in each doc you touched.
