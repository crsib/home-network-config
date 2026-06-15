# ADR-0002: Consolidate `home-controller` + EdgeRouter-4 onto one Topton box

**Status:** Accepted
**Date:** 2026-06-15
**Deciders:** Dmitry (operator)
**Supersedes:** parts of [ADR-0001](0001-replace-zrok-with-pangolin.md) (Pangolin no longer
runs on the VDSina VPS — see *Decision* below).

## Context

Two of the three core hosts are being collapsed onto a single new machine:

- **EdgeRouter-4** (`192.168.1.1`) — edge router/firewall/DNS/DHCP. Its **dual-WAN
  failover proved unreliable** (EdgeOS keys largely off link state / simple ping and
  flaps when the line is "up" but the upstream is dead), so the second WAN (`eth1`) was
  **disabled**. See [edgerouter-4.md](../hosts/edgerouter-4.md).
- **home-controller** (`192.168.1.2`) — nginx, sing-box, Headscale, UniFi, UISP,
  Nextcloud, MicroK8s, ZeroTier, zrok. Older/slower CPU. See
  [home-controller.md](../hosts/home-controller.md).

**WAN uplinks:**

| Uplink | Address family | Notes |
|--------|----------------|-------|
| **WAN1** | Public IPv4 (`/28`, stable) | Today's primary; natural inbound primary. |
| **WAN2** | Public IPv4 **+ IPv6** | Was disabled on the ER-4 (flaky failover). **Only WAN2 carries IPv6.** |

Because WAN2 is publicly routable, **inbound failover is viable**. But since **IPv6 exists
only on WAN2**, native v6 is only available while WAN2 is up — which shapes the dual-stack
design below.

**New hardware (in hand):**

| Component | Spec |
|-----------|------|
| Chassis | Topton soft-router |
| CPU | Intel **Core 3-100U** (6C/8T, Raptor Lake-U refresh) |
| NICs | **6× Intel i226-V** (2.5 GbE) |
| RAM | 2× DDR5 SODIMM slots — **1× 16 GB** installed now (room for a 2nd; DDR5 prices high) |
| Storage | **512 GB Samsung 980 Pro** NVMe (single disk) |

Forces at play:

- The headline requirement is **dual-WAN failover that actually works** — the ER-4's
  weakness and the reason this project started.
- **nginx must stay the high-performance edge.** It fronts the inbound **sing-box**
  connections (trojan/vless camouflage, with Nextcloud as the "face") — a latency- and
  throughput-sensitive path that must not sit behind a heavier L7 proxy.
- **Pangolin moves home.** The **VDSina VPS** (`89.110.79.146`) that ADR-0001 chose as
  Pangolin's host is **not being renewed** and proved unreliable. Pangolin's role is also
  **re-scoped**: not for exposing admin UIs (nginx already does that), but for occasional
  **zrok-style ad-hoc public shares**.
- CPU is a non-issue — the Core 3-100U is a generational leap over `.2`. The real limits
  on a 16 GB single-disk box are **RAM, IO, and blast radius**, not compute.
- This box becomes the **single point of failure for the entire network** (router *and*
  services), so isolation and recoverability matter more than before.

## Decision

Run **Proxmox VE** on the Topton box with **two VMs**:

```
Topton box  →  Proxmox VE
  ├─ VM: OPNsense (router)        2× i226-V passed through = WAN1 + WAN2
  │      dual-WAN failover · firewall · NAT · DHCP · VLAN routing
  │      local DNS (Unbound → NextDNS fallback) · DNAT :443 → services VM
  └─ VM: Ubuntu + Docker (services)
         nginx (edge) · sing-box · Headscale · Nextcloud · UniFi · Pangolin · RustDesk
```

**Router layer (OPNsense VM):**
- **Dual-WAN** via gateway groups with **per-WAN remote probe IPs** (e.g. WAN1→`1.1.1.1`,
  WAN2→`9.9.9.9`) and packet-loss/latency thresholds — detects a dead upstream behind a
  live link, the exact failure EdgeOS missed.
- **Local DNS resolver** (Unbound) with **NextDNS as upstream/fallback** (DoT). DHCP is
  changed to advertise **this resolver**, replacing today's public `1.1.1.1, 8.8.8.8`.
- **Router-on-a-stick unchanged** — VLAN 2/3 tagging stays at the EdgeSwitches; the new
  router just trunks to them. No re-cabling, low-risk cutover.
- **IPv6 (WAN2-only) — independent v4/v6 default routes.** Keep the IPv4 default via WAN1
  (failover WAN2), but run **IPv6 egress always via WAN2** regardless of the v4 failover
  tier, with **DHCPv6-PD** from the WAN2 ISP delegated to the LAN and each VLAN (a `/64`
  apiece) via RA/SLAAC + DHCPv6. Dual-stack clients then use native v6 over WAN2 *and* v4
  over WAN1; if WAN2 drops, v6 simply disappears and **Happy Eyeballs falls clients back to
  v4** — no outage. A **strict default-deny v6 inbound firewall** on WAN2 is mandatory
  (every GUA host is globally routable otherwise). **Inbound is IPv4-first** on WAN1's
  stable `/28`; publishing services over v6 (AAAA) is optional and deferred — it needs
  dynamic-prefix DDNS and is reachable only while WAN2 is up.

**Service layer (Ubuntu VM) — nginx is the edge, Traefik sits behind it:**

```
Internet :443 ─► OPNsense (DNAT 443 → nginx VM)
                    │
                 nginx  (SNI/host routing, lean & fast)
                  ├─ camouflage/fallback ─► sing-box  (trojan/vless inbound)
                  ├─ cloud.crsib.me       ─► Nextcloud
                  ├─ headscale.crsib.me   ─► Headscale
                  └─ share.crsib.me       ─► Traefik (Pangolin)   ◄── shares only
Internet :UDP(wg) ─► OPNsense (DNAT → Pangolin/Gerbil)   [optional: share-from-anywhere]
```

- **Pangolin** runs **behind nginx** (one upstream hostname), scoped to **ad-hoc public
  shares** — *not* admin-UI exposure. Keep **Gerbil/WireGuard** enabled only if shares may
  originate off-box (a Newt connector elsewhere dials in); that needs one **inbound UDP
  hole** DNAT'd straight to the Pangolin VM (cannot go behind nginx). Local-only shares
  need neither.
- **Certs:** Cloudflare **DNS-01 wildcard** (as today) → no inbound `:80` required.

**Drop (do not migrate):**

| Service | Why |
|---------|-----|
| **MicroK8s** | Idle — no user workloads, only `kube-system`. Pure 1–2 GB overhead. |
| **UISP** | Its managed device (the ER-4) is being deleted. Left only managing 2 EdgeSwitches, which have web UIs (`.4`/`.5`). Not worth a ~3–4 GB stack. |
| **ZeroTier** | Superseded by Headscale + Pangolin. |

**Inbound firewall footprint:** `:443/tcp` (already required for sing-box/nginx) + one
**optional** Gerbil UDP port. No new TCP port versus today.

## Options Considered

### Option A: Proxmox + OPNsense VM + Ubuntu services VM  ← chosen
| Dimension | Assessment |
|-----------|------------|
| Failover | **Best** — dedicated router OS, GUI gateway monitoring |
| Isolation | **Strong** — services VM can crash/reboot/upgrade without dropping WAN |
| Recoverability | `vzdump` snapshots of both VMs; restore to a fresh disk fast |
| Cost | Extra virtio hop in front of nginx (negligible at ≤2.5 G); ~4 GB RAM to host+router |

**Pros:** meets the dual-WAN requirement with least effort; contains blast radius on what
is now the only router; snapshot/backup answers the single-disk SPOF.
**Cons:** two OSes to maintain; NIC passthrough needs IOMMU; one virtio hop on the inbound
path (does not matter at these speeds with nginx kept lean).

**Variant A′:** swap OPNsense for **OpenWrt + mwan3** (~0.5 GB vs ~2.5 GB). `mwan3`'s
per-WAN tracking-IP failover is excellent (arguably finer-grained), at the cost of a more
hands-on, less-GUI config. Default to OPNsense; pick A′ if RAM is tight or more scripting
control is wanted.

### Option B: Bare-metal Linux (nftables router + native services)
| Dimension | Assessment |
|-----------|------------|
| nginx speed | **Best** — direct WAN, no virtio hop |
| Failover | **Weak/DIY** — keepalived/FRR/scripts; rebuilds the exact thing that burned us |
| Isolation | **None** — a Docker/Nextcloud/sing-box mishap can wedge routing |
| RAM | All to services (no hypervisor/router-VM overhead) |

**Pros:** maximum throughput, simplest data path, lowest RAM overhead.
**Cons:** makes the #1 requirement (reliable failover) a fragile custom build, and couples
the whole network's uptime to the services kernel. **Rejected** — the virtio hop it avoids
is a non-issue at 2.5 G, and the isolation it gives up is exactly what a single-box router
needs.

### Option C: Status quo — keep `.2` and ER-4 as separate boxes
| Dimension | Assessment |
|-----------|------------|
| Resilience | **Best** — router and services already decoupled |
| Goal fit | **Fails** — does not consolidate; keeps the unreliable ER-4 failover |

**Pros:** no migration risk; failure domains already separate.
**Cons:** doesn't deliver the consolidation goal, keeps the ER-4's failover problem, more
boxes/power. **Rejected on requirements** — but it is the de-facto **rollback** target
(keep the ER-4 as a cold spare).

## Trade-off Analysis

The core tension is **nginx raw speed vs. failover reliability + isolation**. Bare metal
(B) wins on speed but loses on the two things that matter most for a *single* router:
trustworthy WAN failover and a blast radius that can't take the house offline. Proxmox (A)
costs one virtio hop — immaterial at ≤2.5 G once nginx is kept lean (Traefik parked behind
it) — and in exchange gives a real router OS for failover plus VM isolation and snapshot
recovery. Since this box is now the only router *and* a single 980 Pro with no redundancy,
recoverability (vzdump + a cold-spare ER-4) outweighs shaving a hop. Bringing Pangolin home
trades ADR-0001's "no home inbound ports / outage-degrades-to-no-access" property for "no
VPS dependency, no recurring cost, simpler topology"; the exposure delta is small because
Pangolin reuses the already-open `:443` behind nginx and is scoped to occasional shares.

## Consequences

**Easier:** one box to power/manage; CPU headroom; working dual-WAN failover; one
authenticated reverse-proxy story (nginx edge → backends); snapshot/restore of the whole
stack; sing-box/Headscale/Nextcloud lift mostly as-is (Ubuntu → Ubuntu).

**Harder / new burden:**
- This box is now the **single point of failure for the whole network** (router + services
  + single NVMe). Mandatory: **`vzdump` backups** of both VMs off-box, periodic **OPNsense
  config XML** export, and **keep the ER-4 as a cold spare** for emergency internet.
- **Inbound now rides home WAN.** Publishing `:443` from home means public access depends
  on the active WAN. Need a **DDNS updater that follows the active gateway** (Cloudflare API
  driven by OPNsense gateway state) so a WAN1→WAN2 failover keeps inbound working.
- One more service patched at the edge (Traefik/Pangolin), now on the real edge box rather
  than a disposable VPS.

**To revisit:**
- **WAN2 v4 static vs. dynamic** — public confirmed (so inbound failover is viable); if the
  WAN2 v4 address is dynamic, the active-WAN DDNS updater must handle it.
- **IPv6 rollout** — deferred to a post-cutover phase: DHCPv6-PD on WAN2, per-VLAN `/64`s,
  strict v6 inbound firewall, then optionally AAAA-published services (dynamic-prefix DDNS,
  WAN2-up only).
- Second 16 GB SODIMM if UISP/k8s ever return, or for headroom.
- Whether to keep Gerbil (share-from-anywhere) or restrict Pangolin to local-only shares.
- Bulk-storage plan if Nextcloud data outgrows the 512 GB shared disk.

## Action Items

See the migration runbook: [runbooks/migrate-to-topton-box.md](../../runbooks/migrate-to-topton-box.md).

1. [ ] Confirm **IOMMU/VT-d** on the Topton (required for NIC passthrough).
2. [ ] Confirm whether **WAN2's IPv4 is static or dynamic** (WAN2 public + IPv6 confirmed).
3. [ ] Install Proxmox; create OPNsense VM (WAN1+WAN2 passthrough) and Ubuntu services VM.
4. [ ] OPNsense: dual-WAN gateway group + per-WAN probes; firewall/NAT; VLAN trunk;
       Unbound→NextDNS; DHCP advertising the local resolver.
4a. [ ] IPv6 (post-cutover): DHCPv6-PD on WAN2, per-VLAN `/64` via RA/SLAAC+DHCPv6, **v6
       inbound default-deny**; v6 default route always via WAN2.
5. [ ] Lift services to the Ubuntu VM (nginx, sing-box, Headscale, Nextcloud, UniFi,
       RustDesk); **drop** MicroK8s, UISP, ZeroTier.
6. [ ] Pangolin behind nginx (`share.crsib.me`); decide Gerbil on/off; DNAT `:443`
       (+ optional Gerbil UDP) on OPNsense.
7. [ ] DDNS-follows-active-WAN (Cloudflare API).
8. [ ] Cut over (DHCP/DNS/gateway), verify, then retire the ER-4 to cold-spare and power
       down `.2`.
9. [ ] Update [topology.md](../topology.md), [home-controller.md](../hosts/home-controller.md),
       [edgerouter-4.md](../hosts/edgerouter-4.md), overlays/DNS docs, and `MEMORY.md`.
