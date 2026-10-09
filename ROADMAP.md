# Roadmap

Each milestone ends with something runnable. Order can change; scope only grows after the previous milestone ships.

Design: [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md). Behaviour: [docs/spec/](./docs/spec/). Rules for contributors and AI agents: [AGENTS.md](./AGENTS.md).

Each milestone has **Done when** criteria. A milestone is done only when every criterion passes in CI (or in the documented manual check), not when the checkboxes are ticked.

## v0.0 — Foundation
- [ ] Cargo workspace: `karen-core`, `karen-proto`, `karen-control`, `karen-agent`, `karen-metal`, `karen-edge`, `karen-cli`
- [ ] CI: fmt, clippy, test, `cargo-deny`, typos
- [ ] CODE_OF_CONDUCT, issue/PR templates
- [ ] Nested-KVM dev VM script (Ubuntu 26.04 host image, two emulated CPU models)
- [x] Specs: data model, state machines, task queue, API, security, configuration, metrics, abuse, operations, testing
- [x] `AGENTS.md` + `.agents/` playbooks
- [ ] `api/openapi.yaml` skeleton from [API.md](./docs/spec/API.md): conventions, error schema, all v0.1 endpoints
- [ ] `karen-proto` skeleton: `Enrollment`, `AgentChannel` and v0.1 step messages
- [ ] `karen-testkit` with fakes for `Vmm`, `StorageBackend`, `NetworkBackend`
- [ ] Settings registry + `settings::resolve` with catalog-sync test

**Done when:** `cargo test --workspace` and all PR-tier checks pass on an empty workspace with skeleton crates; `cargo xtask openapi-check` and `buf lint` pass; the nested-KVM script boots a host where `virsh capabilities` works.

## v0.1 — First VM
- [ ] Agent ↔ control gRPC over mTLS, one-time enrollment token
- [ ] QEMU via libvirt: create, start, stop, destroy
- [ ] `local` storage class: raw LVs on LVM-thin; `local_layout` detection (`jbod` default, `raid0/1/10/5`), one pool per NVMe in `jbod`
- [ ] Host baseline: Ubuntu 26.04 LTS; agent reports kernel/QEMU/libvirt/OVMF/OVS/OVN/FRR versions
- [ ] **Routed tap** networking with nftables `netdev` ingress anti-spoof (always on)
- [ ] SQLite, admin API token
- [ ] CLI: `karen vm create|list|start|stop|delete`
- [ ] Idempotent task queue
- [ ] Host hardening baseline applied by the agent (nested virt off, `/dev/kvm` 0660 check)
- [ ] Agent PKI: internal CA, CSR-based enrollment, 30-day certs with auto-renew, revocation on node retire
- [ ] Task queue per [TASKS.md](./docs/spec/TASKS.md): leases with epochs, steps, compensation, locks, agent journal, reconciler (power + anti-spoof)
- [ ] VM lifecycle and power state machines per [STATE_MACHINES.md](./docs/spec/STATE_MACHINES.md)
- [ ] Runtime settings store + `GET/PUT /admin/settings`
- [ ] Audit log (hash-chained) for every API write
- [ ] Control DB backup: SQLite online backup every 15 min, encrypted; `karen-control restore` with recovery mode; `karen backup export-keys`

**Done when:** on a nested-KVM host, `karen vm create` returns a task that ends `succeeded`, the VM boots and answers ping on its routed `/32`; spoofed packets from the VM are dropped; killing the agent or control mid-create leaves no orphan LV, tap or IP (chaos test); every step passes the double-run idempotency harness; a backup restored into a fresh control enters recovery mode and reports zero diffs.

## v0.2 — Usable by one admin
- [ ] cloud-init templates: Debian, Ubuntu, Rocky, Alma
- [ ] IPAM: IPv4 `/32` + IPv6 `/64` per VM
- [ ] VNC console proxy
- [ ] Web UI v1
- [ ] PostgreSQL
- [ ] rDNS (PowerDNS)
- [ ] **VM firewall in routed mode**: `FirewallPolicy` API → nftables `inet karen` per-VM chains, stateless default, opt-in stateful, deny logging via nflog
- [ ] Per-VM `tc` rate limits
- [ ] IP lifecycle (`reserved` → `assigned` → `cooldown`) and append-only `ip_assignment` history; `GET /admin/ip-history`; `account_identity_history`
- [ ] Console via single-use tokens over the agent stream (Unix-socket VNC/serial only)
- [ ] PostgreSQL control backup: WAL archiving + base backups (WAL-G or pgBackRest), point-in-time restore
- [ ] Postgres row-level security as second tenant filter
- [ ] SMTP policy `abuse.smtp.policy` (`allow` default, `block`, `allow_on_request`)

**Done when:** the system-test firewall matrix passes in routed mode; IP history returns the correct customer for an address at a past instant after release and reassignment; a console token can be used once and expires in 60 s; Postgres PITR restore to a timestamp passes; the tenant isolation suite passes for every endpoint.

## v0.3 — Hosting panel (VPS)
- [ ] Users/RBAC (admin, customer)
- [ ] Plans: storage class, per-disk QoS (`<iotune>` IOPS/bandwidth), backup schedule; monthly traffic quota and `overage_action: Throttle|Suspend` (default `Throttle` to 10 Mbit/s via `tc` / OVN qos until month rollover)
- [ ] Customer portal: power, reinstall, console, password reset, SSH keys, firewall editor
- [ ] ISO + Windows (virtio, OVMF, swtpm)
- [ ] Snapshots
- [ ] **Network graphs**: tap `IFLA_STATS64` every 10 s → VictoriaMetrics → tenant-scoped proxy; 1h/24h/7d/30d. Plus CPU/disk graphs
- [ ] **Traffic accounting**: 5-min deltas, monthly totals, 95th percentile
- [ ] Prometheus endpoint, audit log
- [ ] Benchmark qcow2 external data file (`data-file-raw=on`) vs plain raw; adopt if no latency cost (persistent bitmaps)
- [ ] Incremental backups to S3 (dirty bitmaps, chunked dedup)
- [ ] Admin UI: version drift per node; "no disk redundancy" flag; plan option `require_disk_redundancy`
- [ ] Full VM and node metrics catalog per [METRICS.md](./docs/spec/METRICS.md) (steal time, disk latency, balloon stats, PSI, NVMe SMART, thin pool, RAID); named-metric query API
- [ ] Memory: balloon with free page reporting on by default; optional auto-reclaim; CPU/RAM overcommit settings enforced by placement
- [ ] Abuse detectors (counter-based first: `outbound_flood`, `smtp_flood`, `crypto_mining`), action ladders, `abuse_case` API
- [ ] Notifications: event → channels (`email`, `webhook`, `billing`, none) with per-event toggles and reseller templates
- [ ] Webhooks with HMAC signatures, retries, auto-disable; `GET /events` catch-up

**Done when:** a customer can do everything in the portal through the public API alone (UI uses no private endpoint); iperf byte counts match graphs and billing within ±1 %; a guest that frees memory returns it to the host (RSS drops) with free page reporting; agent metrics collection for 200 VMs < 200 ms and < 1 % of one core; each abuse detector fires in a system test and runs only the configured ladder.

## v0.4 — Multi-node + OVN
- [ ] KAREN apt repository: CI builds and signs latest upstream OVS + OVN for Ubuntu 26.04
- [ ] Scheduler (capacity, groups, tags, storage class; stops placing at 85 % thin-pool data/metadata)
- [ ] OVN mode: `port_security`, **same `FirewallPolicy` compiled to OVN ACLs / Port_Groups / Address_Sets**, ACL logging with meter, VPC over Geneve, NAT, VLAN localnet, qos + pps meters
- [ ] Firewall and graph parity tests between `routed` and `ovn`
- [ ] FRR BGP unnumbered host ↔ leaf
- [ ] Live migration with local disks (NBD mirror) + `/32` move; firewall policy re-applied on the destination before cut-over
- [ ] Switch sFlow → FastNetMon
- [ ] WHMCS module
- [ ] CPU compatibility per [COMPUTE.md](./docs/spec/COMPUTE.md): node-group baseline, `baseline`/`host-passthrough`, migration target filtering, `cpu-model-upgrade`
- [ ] KSM setting (off by default) with metrics
- [ ] Node drain / maintenance / resume per [OPERATIONS.md](./docs/spec/OPERATIONS.md); rolling upgrades (`/admin/rollouts`)
- [ ] Flow-based abuse detectors (`outbound_ddos`, `port_scan`) from sFlow/FastNetMon

**Done when:** VMs live-migrate both ways between nested hosts with different CPU models; draining a node with 20 VMs moves all migratable VMs and reports the rest per policy; a rolling upgrade of 3 nodes completes with `max_unavailable=1` and pauses on an injected failure; firewall and graph parity tests pass between `routed` and `ovn`.

## v0.5 — Web hosting
- [ ] Account containers (cgroup v2 + userns + overlay + PHP-FPM)
- [ ] `karen-edge` (Pingora, on-demand ACME with ownership check)
- [ ] Domains / DNS / mail-relay integration
- [ ] One-click apps catalog (n8n, etc.) on VPS via cloud-init

**Done when:** two accounts on one host can't read each other's files or exhaust each other's CPU/RAM/IO/pids (isolation system test); a new domain gets a certificate only after the ownership check; `karen-edge` enforces per-site rate limits.

## v0.6 — Dedicated (metal)
- [ ] `karen-metal` power, console (SOL) and inventory via Redfish + IPMI
- [ ] HTTPS Boot → iPXE → ramdisk agent; reinstall
- [ ] NVMe crypto-erase, fail closed
- [ ] Switch-port VLAN automation

**Done when:** a server goes provision → customer → release → crypto-erase → quarantine → pool with no manual step; an injected erase failure keeps it out of the pool; the BMC network is unreachable from a customer VLAN.

## v0.7 — Business + scale
- [ ] Resellers with quotas
- [ ] Hourly billing + credit balance + auto-suspend
- [ ] Blesta / Paymenter modules, webhooks
- [ ] Scoped API tokens (rate limit, expiry); 2FA + step-up auth
- [ ] `replicated` storage class: Ceph RBD root disks + detachable volumes
- [ ] HA restart for `replicated` VMs, only after BMC fencing
- [ ] Live storage-class conversion (`blockdev-mirror` LVM-thin ↔ RBD)
- [ ] Cloud Hypervisor Linux tier
- [ ] Firecracker app tier (scale-to-zero)
- [ ] i18n: en, pt-BR

**Done when:** a reseller can't exceed its quota through any endpoint; hourly billing totals match `usage_hourly` exactly; killing a node with `replicated` VMs fences it via BMC before restart, and a failed fence never restarts VMs.

## v0.8 — Adoption
- [ ] Importers: VirtFusion, Virtualizor, Proxmox
- [ ] ARM64 hypervisors
- [ ] Disaster recovery
- [ ] `karen-agent uninstall`
- [ ] Packages: deb, rpm, static musl
- [ ] Docs site
- [ ] Edge kit docs (IX.br, RPKI Routinator, FastNetMon → RTBH/Flowspec)

**Done when:** a VirtFusion test export imports with VMs, IPs and customers intact; deb/rpm packages install and upgrade cleanly on supported OSes; a full control loss is recovered from backup on a new server following the docs only.

## v1.0 — Stable
- [ ] Semver API
- [ ] External security audit
- [ ] Load test: 50 nodes / 5,000 VMs

**Done when:** load tests in [TESTING.md](./docs/spec/TESTING.md#load-tests) pass; upgrade test from the previous release passes; no open security issues rated high or critical.

## Later
- HA control plane
- OVS TC offload / BlueField DPU
- Firmware automation
- EVPN for metal L2
- GPU (VFIO) SKUs
- Confidential VMs (SEV-SNP / TDX via QEMU)
- Anycast DNS
