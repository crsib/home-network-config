# Runbooks

Step-by-step operational procedures for this network. Each runbook is short,
copy-pasteable, and assumes the access in [docs/access.md](../docs/access.md).

## Index

- **[deploy-pangolin.md](deploy-pangolin.md)** — stand up Pangolin on the VDSina VPS
  + Newt connector on `.2` to publish the admin UIs behind SSO, replacing zrok
  (implements [ADR-0001](../docs/decisions/0001-replace-zrok-with-pangolin.md)).

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
