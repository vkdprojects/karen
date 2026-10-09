# Competitor research

Researched 2026-10-08. Sources linked inline. `[inference]` = not confirmed by a primary source.

## TL;DR

- **VirtFusion is the quality bar.** Hosts pick it for UX, reliability and support; the recurring complaint is price.
- **Virtualizor wins on price and breadth** (many hypervisors, VPC, native billing) but feels dated.
- **SolusVM 2** has a modern UI but launched missing core v1 features and dropped resellers.
- **Open-source options are Proxmox wrappers** (Convoy, Vormox, FeatherPanel), or heavy IaaS (OpenStack, CloudStack, OpenNebula). Nothing is a free, standalone, VirtFusion-grade panel. That's KAREN's gap.

---

## VirtFusion: why hosts love it

Sources: [docs](https://docs.virtfusion.com/), [v7 release notes](https://docs.virtfusion.com/releases/v7), [licensing](https://docs.virtfusion.com/licensing), [LET: SolusVM 2 vs VirtFusion](https://lowendtalk.com/discussion/186376/solusvm-v2-vs-virtfusion-rate-my-hypervisor), [LET: SolusVM vs Virtualizor 2025](https://lowendtalk.com/discussion/206316/solusvm-vs-virtualizor/p3).

### What it is
- B2B KVM panel for hosts/MSPs. Control server + hypervisors installed via a `curl | sh` script.
- PHP-based (has PHP upgrade/troubleshooting docs). Laravel `[inference]`.
- Current: v7.x (v7.0.0 on 2026-05-21, 7.0.6 on 2026-09-09). Seven major versions, fast and frequent releases.
- Control: Debian 12/13, Alma/Rocky 9/10, Ubuntu 22/24, x86_64 only. Hypervisor: same + **ARM64** (v7).

### Strengths
| Area | Detail |
|---|---|
| **End-user UX** | "Miles ahead… faster, more pleasant" (LET). "Putting price aside, VirtFusion's convenience is currently the best" (LET 2025). |
| **Support** | Repeatedly praised as outstanding, included in the license. |
| **Billing integrations** | WHMCS, Blesta, Clientexec, HostBill, BillingServ, Paymenter, Upmind + webhook proxy. |
| **Networking** | MacVTap (default, zero setup), bridged, routed, NAT (v4/v6), Open vSwitch with VLAN trunking, isolated. IPv6 route blocks. Provider guides for OVH and Hetzner. |
| **Storage** | Local (qcow2/raw), Ceph RBD, StorPool, Lightbits, remote filesystem. |
| **Backup Manager (v7)** | PBS native with dedup, S3/SFTP/Rclone/FTP. Incremental, differential and full. Encryption, GFS retention, cross-server restore, post-restore fsck, file-level mount CLI, resumable uploads. |
| **VNC console (v7)** | Auto-reconnect, quality presets, clipboard "type" mode, snippets, WebM session recording, PiP, mobile layout. |
| **Self-service** | Hourly billing with credit balance, resource packs, addons, usage reports. |
| **Ops** | Live migration, disaster recovery, server import, clone, event hooks + webhooks, task queue, rDNS, mail-outs, localization, custom ISO, Windows, cloud-init, CPU pinning. |
| **API** | Admin + end-user API. Per-token rate limits and expiry (7.0.6). Step-up auth for sensitive operations. |

### Weaknesses (KAREN's openings)
| Weakness | Evidence |
|---|---|
| **Price** | "The only reason I use Virtualizor is because VirtFusion is too expensive" (LET 2025). Per-hypervisor tiers (unlimited/35/5/1 VMs). Branding removal is a paid extra. |
| **License lock-in** | License locked to the control server, limited reissues. Licenses can be auto-cancelled on cross-account matches. Validation fails if the clock drifts more than 60s. A phone-home dependency. |
| **Closed source** | Can't audit it. 7.0.6 fixed several vulnerabilities including a "high-priority" one, with details withheld until a coordinated disclosure. |
| **Invasive install** | "Should be the only application on the server." No uninstaller: a mistake means reinstalling the OS. |
| **Default network has no filtering** | MacVTap and OVS modes lose anti-IP-hijack and firewall. Filtering needs bridged or routed. |
| **Self-service half-done** | Hourly billing has no billing-system integration. Credits are added manually. No action on negative balance. Tax display limited. |
| **Single control server** | No documented HA for the control plane. Moving it is a guide plus a license reissue. |
| **Control server is x86 only** | ARM is supported for hypervisors only. |

---

## Commercial alternatives

| Product | Strengths | Weaknesses | Source |
|---|---|---|---|
| **Virtualizor** (Softaculous) | Cheapest (~$9/license retail per LET), multi-hypervisor (KVM, Xen, OpenVZ, LXC, Proxmox), VPC, native billing, resellers | Dated UX. No cloud-init per a 2025 LET user `[unverified vs. current docs]`. Widespread pirated licenses (security risk) | [compare](https://www.virtualizor.com/compare), [LET](https://lowendtalk.com/discussion/206316/solusvm-vs-virtualizor/p3) |
| **SolusVM 2** (Plesk/WebPros) | Modern UI, cloud-init images, OVS with MAC-IP anti-spoof, hourly billing via WHMCS | Launched as rebranded SolusIO missing ISO mount, extra IPs, prepaid portal and LVM snapshots. **Resellers removed**. Price hikes | [LowEndBox critique](https://lowendbox.com/blog/solusvm-2-0s-troubled-start-industry-leader-delivers-detailed-devastating-critique) |
| **SolusVM 1** | Huge install base, resellers | Legacy, ebtables, templates via libguestfs | same |
| **Virtuozzo / VMware** | Enterprise features | Enterprise pricing, not hosting-panel focused | — |
| **Tenantos / SynergyCP** | Bare-metal: IPMI, PXE, OS install | Not VPS | — |
| **Vormox** | Proxmox + built-in billing, markets itself as a "VirtFusion alternative" | Proxmox-dependent, commercial | [vormox.com](https://vormox.com/virtfusion-alternative) |

## Open-source / source-available

| Product | Stack | Strengths | Gap KAREN fills | Source |
|---|---|---|---|---|
| **Proxmox VE** | Perl/Rust | Best free hypervisor platform: ZFS, Ceph, HA, PBS | No hosting portal, tenants, plans, per-customer IP pools or billing | — |
| **Convoy** | Laravel + React + Rust, on Proxmox | Modern UI | **Not free for commercial use ($6/node/month)**. Requires Proxmox | [GitHub](https://github.com/ConvoyPanel/panel) |
| **FeatherPanel** | — | Free, Proxmox VMs + game servers + web hosting in one panel | Proxmox-dependent, generalist | [featherpanel.com](https://featherpanel.com/virtualization) |
| **ProxPanel** | Go | Proxmox + libvirt backends | Early-stage | [GitHub](https://github.com/proxpanel/proxpanel) |
| **Incus / LXD** | Go | Solid container + VM engine, clustering | Infrastructure tool, not a hosting panel. Possible future backend for KAREN | — |
| **OpenStack / CloudStack / OpenNebula** | Python / Java / C++ | Complete IaaS | Heavy, needs a team; overkill for 1–50 nodes | — |
| **oVirt / Harvester** | Java / Go+K8s | Datacenter virt / HCI | Maintenance mode / requires Kubernetes | — |

---

## Feature matrix

`✅` built in · `🟡` partial/limited · `❌` no · `?` unverified

| Feature | VirtFusion | Virtualizor | SolusVM 2 | Proxmox | Convoy | KAREN target |
|---|---|---|---|---|---|---|
| Free for commercial use | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| Source available | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| KVM | ✅ | ✅ | ✅ | ✅ | ✅ | v0.1 |
| LXC / containers | ❌ | ✅ | 🟡 | ✅ | ❌ | v0.6 |
| ARM64 hypervisors | ✅ | ? | ? | 🟡 | via PVE | v0.6 |
| End-user portal | ✅ | ✅ | ✅ | ❌ | ✅ | v0.3 |
| Resellers | ? | ✅ | ❌ | ❌ | ❌ | v0.5 |
| Cloud-init | ✅ | ? | ✅ | ✅ | ✅ | v0.2 |
| Custom ISO / Windows | ✅ | ✅ | 🟡 | ✅ | 🟡 | v0.3 |
| Anti-spoof (MAC/IP) | 🟡 bridged/routed only | ✅ | ✅ OVS | 🟡 | ? | v0.2, **every mode** |
| NAT VPS (v4/v6) | ✅ | ✅ | ? | 🟡 manual | ❌ | v0.4 |
| VLAN / OVS | ✅ | ✅ | ✅ | ✅ | via PVE | v0.4 |
| rDNS | ✅ | ✅ | ✅ | ❌ | ❌ | v0.4 |
| Ceph RBD | ✅ | ✅ | ? | ✅ | via PVE | v0.4 |
| PBS / S3 incremental backups | ✅ | 🟡 | 🟡 | ✅ | via PVE | v0.4 |
| Live migration | ✅ | ✅ | ✅ | ✅ | ❌ | v0.5 |
| Disaster recovery | ✅ | ? | ? | ✅ | ❌ | v0.6 |
| Hourly billing / credits | 🟡 manual credits | ✅ | ✅ WHMCS | ❌ | ❌ | v0.5 |
| WHMCS / Blesta / Paymenter | ✅ | ✅ | ✅ | ❌ | ✅ | v0.4–0.5 |
| Webhooks / event hooks | ✅ | ? | ? | 🟡 | ? | v0.5 |
| Importer from competitors | ✅ | ✅ | ✅ | — | ❌ | v0.6 |
| HA control plane | ❌ documented | ❌ | ? | ✅ | ❌ | later |
| Uninstaller / clean removal | ❌ | ? | ? | — | ? | v0.6 |
| Prometheus metrics | ❌ | ❌ | ? | 🟡 | ❌ | v0.3 |

## What KAREN must copy from VirtFusion (table stakes)
1. A customer panel that's fast and pretty.
2. Zero-friction install: one command per node.
3. Billing integrations from day one of public release (WHMCS first).
4. Routed networking that just works on OVH and Hetzner.
5. Serious backups (PBS + S3, incremental, encryption, retention).
6. A console good enough that users never need SSH to rescue a box.

## Where KAREN beats it
1. **MIT, no license server, no phone-home.** Runs air-gapped; no reissue limits.
2. **Anti-spoofing in every network mode** (nftables on the bridge/tap), not just bridged/routed.
3. **Clean install/uninstall.** Two binaries + systemd units; `karen-agent uninstall`.
4. **HA control plane** on the roadmap; Postgres-backed.
5. **Finished self-service:** hourly billing with automatic suspend on negative balance and a billing API.
6. **Auditable security:** public advisories, reproducible builds.
7. **Observability first:** Prometheus, audit log, OpenTelemetry.
