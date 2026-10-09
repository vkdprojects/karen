# Roadmap

Each milestone ends with something runnable. Order can change; scope only grows after the previous milestone ships.

Design: [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md).

## v0.0 — Foundation
- [ ] Cargo workspace: `karen-core`, `karen-proto`, `karen-control`, `karen-agent`, `karen-metal`, `karen-edge`, `karen-cli`
- [ ] CI: fmt, clippy, test, `cargo-deny`, typos
- [ ] CODE_OF_CONDUCT, issue/PR templates
- [ ] Nested-KVM dev VM script

## v0.1 — First VM
- [ ] Agent ↔ control gRPC over mTLS, one-time enrollment token
- [ ] QEMU via libvirt: create, start, stop, destroy
- [ ] LVM-thin raw disks
- [ ] **Routed tap** networking with nftables `netdev` ingress anti-spoof (always on)
- [ ] SQLite, admin API token
- [ ] CLI: `karen vm create|list|start|stop|delete`
- [ ] Idempotent task queue
- [ ] Host hardening baseline applied by the agent (nested virt off, `/dev/kvm` 0660 check)

## v0.2 — Usable by one admin
- [ ] cloud-init templates: Debian, Ubuntu, Rocky, Alma
- [ ] IPAM: IPv4 `/32` + IPv6 `/64` per VM
- [ ] VNC console proxy
- [ ] Web UI v1
- [ ] PostgreSQL
- [ ] rDNS (PowerDNS)
- [ ] **VM firewall in routed mode**: `FirewallPolicy` API → nftables `inet karen` per-VM chains, stateless default, opt-in stateful, deny logging via nflog
- [ ] Per-VM `tc` rate limits

## v0.3 — Hosting panel (VPS)
- [ ] Users/RBAC (admin, customer)
- [ ] Plans, incl. monthly traffic quota and `overage_action: Throttle|Suspend` (default `Throttle` to 10 Mbit/s via `tc` / OVN qos until month rollover)
- [ ] Customer portal: power, reinstall, console, password reset, SSH keys, firewall editor
- [ ] ISO + Windows (virtio, OVMF, swtpm)
- [ ] Snapshots
- [ ] **Network graphs**: tap `IFLA_STATS64` every 10 s → VictoriaMetrics → tenant-scoped proxy; 1h/24h/7d/30d. Plus CPU/disk graphs
- [ ] **Traffic accounting**: 5-min deltas, monthly totals, 95th percentile
- [ ] Prometheus endpoint, audit log
- [ ] Incremental backups to S3 (dirty bitmaps, chunked dedup)

## v0.4 — Multi-node + OVN
- [ ] Scheduler (capacity, groups, tags)
- [ ] OVN mode: `port_security`, **same `FirewallPolicy` compiled to OVN ACLs / Port_Groups / Address_Sets**, ACL logging with meter, VPC over Geneve, NAT, VLAN localnet, qos + pps meters
- [ ] Firewall and graph parity tests between `routed` and `ovn`
- [ ] FRR BGP unnumbered host ↔ leaf
- [ ] Live migration with local disks (NBD mirror) + `/32` move; firewall policy re-applied on the destination before cut-over
- [ ] Switch sFlow → FastNetMon
- [ ] WHMCS module

## v0.5 — Web hosting
- [ ] Account containers (cgroup v2 + userns + overlay + PHP-FPM)
- [ ] `karen-edge` (Pingora, on-demand ACME with ownership check)
- [ ] Domains / DNS / mail-relay integration
- [ ] One-click apps catalog (n8n, etc.) on VPS via cloud-init

## v0.6 — Dedicated (metal)
- [ ] `karen-metal` power, console (SOL) and inventory via Redfish + IPMI
- [ ] HTTPS Boot → iPXE → ramdisk agent; reinstall
- [ ] NVMe crypto-erase, fail closed
- [ ] Switch-port VLAN automation

## v0.7 — Business + scale
- [ ] Resellers with quotas
- [ ] Hourly billing + credit balance + auto-suspend
- [ ] Blesta / Paymenter modules, webhooks
- [ ] Scoped API tokens (rate limit, expiry); 2FA + step-up auth
- [ ] Ceph RBD volumes
- [ ] Cloud Hypervisor Linux tier
- [ ] Firecracker app tier (scale-to-zero)
- [ ] i18n: en, pt-BR

## v0.8 — Adoption
- [ ] Importers: VirtFusion, Virtualizor, Proxmox
- [ ] ARM64 hypervisors
- [ ] Disaster recovery
- [ ] `karen-agent uninstall`
- [ ] Packages: deb, rpm, static musl
- [ ] Docs site
- [ ] Edge kit docs (IX.br, RPKI Routinator, FastNetMon → RTBH/Flowspec)

## v1.0 — Stable
- [ ] Semver API
- [ ] External security audit
- [ ] Load test: 50 nodes / 5,000 VMs

## Later
- HA control plane
- OVS TC offload / BlueField DPU
- Firmware automation
- EVPN for metal L2
- GPU (VFIO) SKUs
- Confidential VMs (SEV-SNP / TDX via QEMU)
- Anycast DNS
