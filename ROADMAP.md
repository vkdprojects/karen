# Roadmap

Each milestone ends with something runnable. Order can change; scope only grows after the previous milestone ships.

## v0.0 — Foundation
- [ ] Cargo workspace: `karen-core`, `karen-proto`, `karen-control`, `karen-agent`, `karen-cli`
- [ ] CI: fmt, clippy, test, `cargo-deny`, typos
- [ ] License, CONTRIBUTING, SECURITY, CODE_OF_CONDUCT, issue/PR templates
- [ ] Architecture doc (`docs/ARCHITECTURE.md`) and data model draft
- [ ] Dev environment: nested-KVM test VM (Vagrant / cloud image + script)

## v0.1 — First VM (single node)
- [ ] Agent: libvirt connection, define/start/stop/destroy KVM domain
- [ ] Agent ↔ control: gRPC over mTLS, node enrollment with one-time token
- [ ] Control: REST API, SQLite, admin auth (API token)
- [ ] Local qcow2 storage, Linux bridge networking
- [ ] CLI: `karen vm create|list|start|stop|delete`
- [ ] Task queue with idempotent, retryable jobs

## v0.2 — Usable for one admin
- [ ] Cloud-init templates (Debian, Ubuntu, Rocky, Alma)
- [ ] IPAM: IPv4/IPv6 pools, auto-assign, static via cloud-init
- [ ] nftables anti-spoofing per VM (MAC/IP) in every network mode
- [ ] Web console (noVNC proxy through control plane)
- [ ] Web UI v1: nodes, VMs, IP pools
- [ ] PostgreSQL support

## v0.3 — Hosting panel
- [ ] Users, roles (admin / customer), RBAC
- [ ] Plans/packages (CPU, RAM, disk, bandwidth, IPs)
- [ ] Customer portal: power, reinstall, console, password reset, SSH keys
- [ ] ISO boot + Windows templates (virtio drivers)
- [ ] Snapshots
- [ ] Metrics: per-VM CPU/RAM/disk/net, bandwidth accounting, Prometheus endpoint
- [ ] Audit log

## v0.4 — Production networking & storage
- [ ] Multi-node scheduler (placement by capacity, groups, tags)
- [ ] VLAN, routed, NAT (v4/v6) and Open vSwitch modes; OVH/Hetzner routed guides
- [ ] rDNS (PowerDNS API plugin)
- [ ] Storage plugins: LVM-thin, ZFS, Ceph RBD
- [ ] Disk IOPS/throughput and network rate limits
- [ ] Backups: S3 and Proxmox Backup Server, incremental, encrypted, GFS retention
- [ ] WHMCS module

## v0.5 — Business features
- [ ] Reseller role with quotas
- [ ] Live migration (shared and local storage)
- [ ] Blesta and Paymenter modules
- [ ] Webhooks + event hooks, scoped API tokens (rate limit, expiry), OpenAPI published
- [ ] 2FA (TOTP, WebAuthn) + step-up auth for destructive actions
- [ ] Self-service: hourly billing, credit balance, auto-suspend on negative balance
- [ ] i18n (en, pt-BR)

## v0.6 — Adoption
- [ ] Importers: VirtFusion, Virtualizor, Proxmox
- [ ] LXC / Incus containers, ARM64 hypervisors
- [ ] Disaster recovery (cross-node restore), `karen-agent uninstall`
- [ ] Install script + packaged releases (deb, rpm, static musl)
- [ ] Documentation site

## v1.0 — Stable
- [ ] Stable API (semver), upgrade path guaranteed
- [ ] External security audit
- [ ] Load test: 50 nodes / 5,000 VMs on one control plane

## Later
- HA control plane (Postgres replication + leader election)
- Bare-metal provisioning (IPMI / PXE)
- Object storage / load balancer add-ons
- WASM plugin interface for non-Rust plugins
