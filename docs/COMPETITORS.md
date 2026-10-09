# Competitor map

Snapshot of the VPS panel / virtualization management space, used to define KAREN's scope. Pricing and features change — verify before quoting.

## Direct (commercial VPS hosting panels)

| Product | Stack | License | Hypervisors | Notes |
|---|---|---|---|---|
| **VirtFusion** | PHP (Laravel) | Proprietary, per-node | KVM | Modern UI, strong API, WHMCS/Blesta modules, the main reference |
| **Virtualizor** | PHP | Proprietary, per-node | KVM, Xen, OpenVZ, LXC, Proxmox | Old but widespread, broad hypervisor support |
| **SolusVM 2** | PHP / Go | Proprietary (Plesk) | KVM, VZ | Big install base, SolusVM 1 legacy |
| **SynergyCP / Tenantos** | — | Proprietary | Bare metal | Dedicated server focus, IPMI / PXE |

## Open source

| Product | Stack | Scope | Gap KAREN fills |
|---|---|---|---|
| **Proxmox VE** | Perl / Rust | Virtualization platform | No hosting portal, tenants, billing, IP pools per customer |
| **Convoy** | PHP (Laravel) on Proxmox | Hosting panel | Depends on Proxmox, PHP, smaller feature set |
| **Incus / LXD** | Go | Container + VM manager | Infra tool, not a hosting panel |
| **OpenNebula** | C++ / Ruby | Private cloud | Heavy, enterprise focus |
| **Apache CloudStack** | Java | IaaS | Heavy, complex for small hosts |
| **OpenStack** | Python | IaaS | Very heavy, team-sized ops |
| **oVirt** | Java | Datacenter virt | Effectively in maintenance |
| **Harvester** | Go / K8s | HCI on Kubernetes | Requires K8s, not a hosting panel |

## Feature matrix

`✅` yes · `🟡` partial/plugin · `❌` no · `🎯` KAREN target (milestone)

| Feature | VirtFusion | Virtualizor | SolusVM 2 | Proxmox | Convoy | KAREN |
|---|---|---|---|---|---|---|
| Open source | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| No per-node fee | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| KVM | ✅ | ✅ | ✅ | ✅ | ✅ | 🎯 v0.1 |
| LXC / containers | ❌ | ✅ | 🟡 | ✅ | ❌ | 🎯 v0.6 |
| End-user portal | ✅ | ✅ | ✅ | ❌ | ✅ | 🎯 v0.3 |
| Reseller role | ✅ | ✅ | ✅ | ❌ | ❌ | 🎯 v0.5 |
| Cloud-init templates | ✅ | ✅ | ✅ | ✅ | ✅ | 🎯 v0.2 |
| ISO / Windows | ✅ | ✅ | ✅ | ✅ | 🟡 | 🎯 v0.3 |
| IPv4/IPv6 pools | ✅ | ✅ | ✅ | ❌ | ✅ | 🎯 v0.2 |
| rDNS | ✅ | ✅ | ✅ | ❌ | ❌ | 🎯 v0.4 |
| VLAN / routed | ✅ | ✅ | ✅ | ✅ | 🟡 | 🎯 v0.4 |
| Anti-spoof filtering | ✅ | ✅ | ✅ | 🟡 | ❌ | 🎯 v0.2 |
| Bandwidth accounting | ✅ | ✅ | ✅ | ❌ | 🟡 | 🎯 v0.3 |
| Snapshots | ✅ | ✅ | ✅ | ✅ | ✅ | 🎯 v0.3 |
| Scheduled backups (S3/PBS) | ✅ | ✅ | ✅ | ✅ | 🟡 | 🎯 v0.4 |
| Live migration | ✅ | ✅ | ✅ | ✅ | ❌ | 🎯 v0.5 |
| ZFS / Ceph | 🟡 | ✅ | 🟡 | ✅ | via PVE | 🎯 v0.4 |
| Web console | ✅ | ✅ | ✅ | ✅ | ✅ | 🎯 v0.2 |
| REST API | ✅ | ✅ | ✅ | ✅ | ✅ | 🎯 v0.1 |
| WHMCS module | ✅ | ✅ | ✅ | ❌ | ✅ | 🎯 v0.4 |
| Blesta / Paymenter | ✅ | 🟡 | 🟡 | ❌ | 🟡 | 🎯 v0.5 |
| Prometheus metrics | ❌ | ❌ | 🟡 | 🟡 | ❌ | 🎯 v0.3 |
| HA control plane | 🟡 | ❌ | 🟡 | ✅ | ❌ | 🎯 v1.x |

## Differentiators to lean on

1. **Free + open** in a market where every hosting-grade panel charges per node.
2. **Single binaries** (control + agent), install in minutes, no PHP/ionCube.
3. **API-first + OpenAPI** — automation and billing integrations are first-class.
4. **Observability built in** — Prometheus, audit log, webhooks.
5. **Migration path** — importers from VirtFusion / Virtualizor / Proxmox (v0.6).
