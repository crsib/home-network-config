# Runbook: Migrate `.2` + ER-4 onto the Topton box

Implements [ADR-0002](../docs/decisions/0002-consolidate-controller-and-router.md). Stands
up **Proxmox** on the new Topton box with an **OPNsense** router VM and an **Ubuntu**
services VM, migrates the keep-list off `home-controller` (`.2`), and retires the
**EdgeRouter-4** to a cold spare.

**Target end state**

```
Topton box  →  Proxmox VE
  ├─ VM: OPNsense   (WAN1+WAN2 passthrough; failover, FW/NAT, DHCP, VLANs, Unbound→NextDNS)
  └─ VM: Ubuntu     (nginx edge · sing-box · Headscale · Nextcloud · UniFi · Pangolin · RustDesk)
```

**Hosts**

| Role | Host | Access |
|------|------|--------|
| New box (Proxmox) | Topton — IP TBD | console first; then `https://<ip>:8006` |
| Old services host | `home-controller` `192.168.1.2` | `ssh dvedenko@192.168.1.2` |
| Old router | EdgeRouter-4 `192.168.1.1` | `ssh ubnt@192.168.1.1` |

> ⚠️ **Never commit secrets** (Cloudflare token, NextDNS profile, sing-box keys, UniFi/
> Nextcloud DB creds, Pangolin admin, Newt id/secret). This runbook references *where* they
> live, not values. Keep the network read-only until the explicit cutover step.

---

## 0. Pre-flight (before touching anything)

- [ ] **IOMMU/VT-d** present and enabled in Topton BIOS (needed for NIC passthrough).
- [ ] **WAN2:** public IPv4 **+ IPv6** (confirmed) — inbound failover is viable. Check
      whether WAN2's v4 is **static or dynamic** (affects the DDNS updater). **IPv6 is
      WAN2-only.**
- [ ] Map the **6× i226-V** ports to roles and label them physically:
      `WAN1`, `WAN2`, `LAN-trunk` (to the EdgeSwitches), spares.
- [ ] Inventory `.2` one more time (compose dirs, volumes, certs):
      ```bash
      ssh dvedenko@192.168.1.2 'sudo docker ps; sudo ss -tlnp; ls -d /opt/* ~/.* 2>/dev/null'
      ```
- [ ] Back up everything you'll migrate (see step 4) **before** the box is in the path.

## 1. Proxmox base

1. Install Proxmox VE to the 980 Pro; bridge `vmbr0` on the **LAN-trunk** NIC.
2. Updates, then create two VMs (don't start the router in-path yet):
   - **OPNsense VM** — 2–3 GB RAM, 2 vCPU. **PCI-passthrough** the `WAN1` + `WAN2` i226-V
     functions to it. Add a vNIC on `vmbr0` for LAN.
   - **Ubuntu VM** — 8 GB RAM, 4 vCPU, ~120 GB disk. One vNIC on `vmbr0`.
3. Configure **vzdump** backups for both VMs to an off-box target (`.13` / NAS). This is
   mandatory — the box is now a single point of failure (ADR-0002).

> Variant: for a leaner router VM use **OpenWrt + mwan3** instead of OPNsense (~0.5 GB).
> Same passthrough/bridge layout; failover via mwan3 tracking IPs.

## 2. OPNsense router VM (built offline, cut in at step 6)

Configure while the ER-4 is still live, using a temporary management address:

1. **Interfaces:** WAN1, WAN2 (the passed-through NICs), LAN on the `vmbr0` vNIC
   (`192.168.1.1/24` — same gateway IP as the ER-4, so clients don't change).
2. **VLANs (router-on-a-stick):** `VLAN 2 → 192.168.2.1/24`, `VLAN 3 → 192.168.3.1/24` on
   the LAN interface. Tagging stays at the EdgeSwitches — no switch changes.
3. **Dual-WAN:** a **gateway group** with per-WAN **monitor IPs** (WAN1→`1.1.1.1`,
   WAN2→`9.9.9.9`), packet-loss/latency thresholds, tier1=WAN1 / tier2=WAN2 for failover
   (or both tier1 for load-balance). This is the fix for the ER-4's flapping.
4. **DHCP:** recreate the three scopes from [edgerouter-4.md](../docs/hosts/edgerouter-4.md)
   (LAN/Devices/Guest), keeping the same ranges/static reservations. **Advertise the local
   resolver** (`192.168.x.1`), *not* the old public `1.1.1.1, 8.8.8.8`.
5. **DNS — Unbound:** local recursive resolver; set **NextDNS** as the upstream/forward
   (DoT, your NextDNS profile). Verify resolution + that NextDNS sees queries.
6. **NAT / port-forwards:** recreate the live ones, repointing at the **Ubuntu VM's LAN IP**:
   - `:443/tcp` → Ubuntu VM (nginx) — covers nginx + sing-box + Pangolin-behind-nginx.
   - RustDesk `21115–21117 tcp/udp` → wherever the relay ends up (Ubuntu VM, or leave on `.13`).
   - (Optional) Gerbil **UDP** → Ubuntu VM, only if Pangolin shares may originate off-box.
   - Drop the stale `32400`/`51413` forwards.
7. **IPv6 (WAN2-only) — defer to after a stable v4 cutover, then:** request **DHCPv6-PD**
   on WAN2; assign a `/64` to LAN and each VLAN via **RA/SLAAC + DHCPv6**; set the **IPv6
   default route always via WAN2** (independent of the v4 failover tier). Add a **v6 inbound
   firewall that defaults to deny** — GUA hosts are globally routable otherwise. Leave
   inbound services v4-only (WAN1 `/28`) for now; AAAA publishing is optional/later. With
   WAN2 down, clients lose v6 and fall back to v4 via Happy Eyeballs (no action needed).
8. Export the **config XML** and store it off-box (rollback + ADR-0002 backup requirement).

## 3. Ubuntu services VM — base

```bash
# Docker + compose plugin
curl -fsSL https://get.docker.com | sh
```

Recreate the directory layout you inventoried on `.2` (compose projects under `/opt` / home).
**Do not** install MicroK8s, UISP, or ZeroTier (dropped per ADR-0002).

## 4. Migrate services from `.2`

Lift each, keeping config/data. General pattern: stop on `.2` → copy state → start on the VM.

| Service | What to move |
|---------|--------------|
| **nginx** | `/etc/nginx/` (sites + `stream` block for sing-box) and `/etc/letsencrypt/`. Keep nginx the **edge** on `:443`. |
| **sing-box** | binary + config + keys (the inbound trojan/vless + egress rules). Re-point any `.2`-specific paths. |
| **Headscale** | `/var/lib/headscale` + config; DB. Nodes re-register against the same `headscale.crsib.me`. |
| **Nextcloud** | the Compose stack + the `mariadb`/`redis`/data volumes. It's the sing-box "face". |
| **UniFi** | controller data (`/var/lib/unifi`, Mongo) — APs re-adopt to the same controller. |
| **RustDesk** | hbbs/hbbr + their key, *if* relocating off `.13`. |

Certs: keep the **Cloudflare DNS-01** wildcard flow (no inbound `:80` needed). Copy or
re-issue with the scoped Cloudflare token.

## 5. Pangolin (home, behind nginx) — replaces the ADR-0001 VPS deployment

Pangolin now runs **on the Topton (Ubuntu VM)**, **behind nginx**, scoped to **ad-hoc
public shares** (not admin UIs — nginx still fronts those):

1. Deploy the Pangolin stack (Traefik + Pangolin + Gerbil) on the Ubuntu VM, bound to an
   **internal port** (e.g. `127.0.0.1:8443`), **not** `:443` — nginx owns `:443`.
2. nginx: add a vhost `share.crsib.me` → `proxy_pass` to Pangolin's internal port.
3. **Gerbil:** enable only if shares may originate off-box; then forward its **UDP** port at
   OPNsense (step 2.6). Local-only shares need no UDP hole.
4. Cloudflare: `CNAME share.crsib.me` (or the wildcard) → home (DDNS name). DNS-01 cert via
   nginx/Traefik as you prefer.
5. Verify a test share publishes and is reachable.

## 6. Cutover (the only disruptive step — schedule a window)

1. Lower DHCP lease times on the ER-4 beforehand (faster client migration).
2. **Move the LAN-trunk cable** from the ER-4 to the Topton `LAN-trunk` port; move both WAN
   uplinks to `WAN1`/`WAN2`. Start the OPNsense VM as `192.168.1.1`.
3. Clients renew DHCP → new gateway + local resolver. Verify:
   - LAN + each VLAN route to internet; inter-VLAN policy as before.
   - **Failover:** unplug WAN1, confirm WAN2 takes over within the threshold; re-plug.
   - DNS resolves via Unbound→NextDNS; NextDNS dashboard shows queries.
   - `:443` inbound → nginx → sing-box / Nextcloud / `share.crsib.me` all work.
   - RustDesk relay reachable.
4. **DDNS-follows-active-WAN:** configure the Cloudflare updater (driven by OPNsense gateway
   state) so the public record tracks the live WAN.

## 7. Decommission

- [ ] Power down `home-controller` (`.2`) — keep its disk/backup until you're confident.
- [ ] **Retire the ER-4 to a cold spare** — factory-reset *not* required; keep its config so
      it can be slotted back for emergency internet. Store it labelled.
- [ ] Remove the old VDSina Pangolin/Newt + dead zrok (ADR-0001 leftovers) and the
      `zrok.zrok.crsib.me` / VDSina DNS records, since VDSina is not being renewed.

## 8. Document

- [ ] [topology.md](../docs/topology.md) — new single-box edge; update diagram, WAN, host
      inventory (retire `.1`/`.2`, add the Topton).
- [ ] [hosts/](../docs/hosts/) — mark `edgerouter-4.md` and `home-controller.md` as retired;
      add a host doc for the Topton box (Proxmox + the two VMs).
- [ ] DNS / overlays / access docs — local resolver + NextDNS; Pangolin-at-home; admin-UI
      access via nginx.
- [ ] `MEMORY.md` and ADR-0002 action items.

## Rollback

Designed to be reversible at every stage until step 7:

- **Before cutover (steps 0–5):** the box is offline/out-of-path; nothing on the live
  network changed. Abort = walk away.
- **During/after cutover (step 6):** move the LAN-trunk + WAN cables **back to the ER-4** and
  re-enable WAN1 — internet restored on the proven router in minutes. The OPNsense XML
  (step 2.8) and the cold-spare ER-4 are the safety net.
- **Services:** `.2` stays powered-off-but-intact (and vzdump'd) until the new VM is trusted;
  re-power `.2` to revert any single service.
