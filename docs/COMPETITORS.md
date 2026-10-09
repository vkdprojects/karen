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
| **cPanel/WHM** (WebPros) | Industry default for shared hosting; Apache/LiteSpeed; CloudLinux LVE + CageFS add-on | Solo $29.99 → Premier $69.99/mo + **$0.49/account over 100**; up to ~20% more in 2027. Exploited bug in Apr 2026. WHMCS lock-in (same owner) | [pricing](https://support.cpanel.net/hc/en-us/articles/30117774089879-2026-cPanel-Store-License-Pricing), [research](./research/providers.md) |
| **Plesk** (WebPros) | Apache + nginx, per-subscription users | $18–$50/mo; same owner as cPanel | [CostBench](https://costbench.com/software/cloud-infrastructure/plesk) |
| **Enhance** | Each site in its own container; cluster of 1–10,000 servers; zero-downtime site moves | Per-site billing; complaints about container RAM use | [features](https://enhance.com/product/features) |
| **DirectAdmin** | Flat tiers, unlimited accounts on top tier; cheapest credible cPanel replacement | Unix user + CloudLinux isolation | [pricing](https://www.directadmin.com/pricing.php) |

## Open-source / source-available

| Product | Stack | Strengths | Gap KAREN fills | Source |
|---|---|---|---|---|
| **Proxmox VE** | Perl/Rust | Best free hypervisor platform: ZFS, Ceph, HA, PBS | No hosting portal, tenants, plans, per-customer IP pools or billing | — |
| **Convoy** | Laravel + React + Rust, on Proxmox | Modern UI | **Not free for commercial use ($6/node/month)**. Requires Proxmox | [GitHub](https://github.com/ConvoyPanel/panel) |
| **FeatherPanel** | — | Free, Proxmox VMs + game servers + web hosting in one panel | Proxmox-dependent, generalist | [featherpanel.com](https://featherpanel.com/virtualization) |
| **ProxPanel** | Go | Proxmox + libvirt backends | Early-stage | [GitHub](https://github.com/proxpanel/proxpanel) |
| **Incus / LXD** | Go | Solid container + VM engine, clustering | Infrastructure tool, not a hosting panel. Possible future backend for KAREN | — |
| **OpenStack / CloudStack / OpenNebula** | Python / Java / C++ | Complete IaaS | Heavy, needs a team; overkill for 1–50 nodes | — |
| **OpenPanel** | — | Each user in their own container (web server, DB, network) | Web hosting only | [GitHub](https://github.com/stefanpejcic/openpanel) |
| **CloudPanel** | — | Free, nginx + PHP-FPM | Single admin, not built for multi-tenant resale | — |
| **CyberPanel** | — | Free, OpenLiteSpeed | **PSAUX ransomware hit ~22k instances** via CVE-2024-51567; more auth-bypass CVEs in 2026 | [SOCRadar](https://socradar.io/blog/over-22000-cyberpanel-servers-at-risk-from-critical-vulnerabilities-exploitation-by-psaux-ransomware) |
| **oVirt / Harvester** | Java / Go+K8s | Datacenter virt / HCI | Maintenance mode / requires Kubernetes | — |

---

## Hosting providers (who sells on these stacks)

Details and sources: [research/providers.md](./research/providers.md).

| Provider | Stack highlights |
|---|---|
| DigitalOcean | KVM, L3 Clos with GoBGP on each hypervisor, OVS, Ceph (250+ PB) |
| Hetzner | KVM, local NVMe live migration (<1 s blackout), triple-replicated volumes, Arbor DDoS |
| OVHcloud | OpenStack KVM, vRack L2, in-house VAC DDoS (FPGA + x86) |
| Vultr | KVM, own attack mitigation farm |
| Fly.io | Firecracker, Rust proxy + gossip state, Anycast |
| Hostinger | KVM VPS, in-house hPanel, CephFS, LiteSpeed + CloudLinux |
| StayCloud | cPanel + LiteSpeed + WHMCS + Cloudflare; hypervisor not disclosed |

## Feature matrix

`✅` built in · `🟡` partial/limited · `❌` no · `?` unverified

| Feature | VirtFusion | Virtualizor | SolusVM 2 | Proxmox | Convoy | KAREN target |
|---|---|---|---|---|---|---|
| Free for commercial use | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ |
| Source available | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| KVM | ✅ | ✅ | ✅ | ✅ | ✅ | v0.1 |
| LXC / containers | ❌ | ✅ | 🟡 | ✅ | ❌ | web containers v0.5 |
| ARM64 hypervisors | ✅ | ? | ? | 🟡 | via PVE | v0.8 |
| End-user portal | ✅ | ✅ | ✅ | ❌ | ✅ | v0.3 |
| Resellers | ? | ✅ | ❌ | ❌ | ❌ | v0.7 |
| Cloud-init | ✅ | ? | ✅ | ✅ | ✅ | v0.2 |
| Custom ISO / Windows | ✅ | ✅ | 🟡 | ✅ | 🟡 | v0.3 |
| Anti-spoof (MAC/IP) | 🟡 bridged/routed only | ✅ | ✅ OVS | 🟡 | ? | v0.1, **every mode** |
| VM firewall (per-VM rules) | 🟡 bridged/routed only | ✅ | ✅ | ✅ | ? | v0.2 routed / v0.4 OVN |
| Per-VM network graphs + 95th billing | ✅ | ✅ | ✅ | 🟡 | 🟡 | v0.3 |
| NAT VPS (v4/v6) | ✅ | ✅ | ? | 🟡 manual | ❌ | v0.4 |
| VLAN / OVS | ✅ | ✅ | ✅ | ✅ | via PVE | v0.4 |
| rDNS | ✅ | ✅ | ✅ | ❌ | ❌ | v0.2 |
| Ceph RBD | ✅ | ✅ | ? | ✅ | via PVE | v0.7 |
| PBS / S3 incremental backups | ✅ | 🟡 | 🟡 | ✅ | via PVE | v0.3 |
| Live migration | ✅ | ✅ | ✅ | ✅ | ❌ | v0.4 |
| Disaster recovery | ✅ | ? | ? | ✅ | ❌ | v0.8 |
| Hourly billing / credits | 🟡 manual credits | ✅ | ✅ WHMCS | ❌ | ❌ | v0.7 |
| WHMCS / Blesta / Paymenter | ✅ | ✅ | ✅ | ❌ | ✅ | v0.4 / v0.7 |
| Webhooks / event hooks | ✅ | ? | ? | 🟡 | ? | v0.7 |
| Importer from competitors | ✅ | ✅ | ✅ | — | ❌ | v0.8 |
| HA control plane | ❌ documented | ❌ | ? | ✅ | ❌ | later |
| Uninstaller / clean removal | ❌ | ? | ? | — | ? | v0.8 |
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
2. **Anti-spoofing in every network mode** (nftables on the tap in routed mode, OVN `port_security` in OVN mode).
3. **Clean install/uninstall.** Two binaries + systemd units; `karen-agent uninstall`.
4. **HA control plane** on the roadmap; Postgres-backed.
5. **Finished self-service:** hourly billing with automatic suspend on negative balance and a billing API.
6. **Auditable security:** public advisories, reproducible builds.
7. **Observability first:** Prometheus, audit log, OpenTelemetry.
