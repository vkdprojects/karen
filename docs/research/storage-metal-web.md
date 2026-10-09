# KAREN — Storage, bare-metal, and web-hosting isolation (2026-10-08)

Researched 2026-10-08. [unverified] = not confirmed by a primary source.

## A. VPS storage

### Options
| Option | QD1 4k latency (typical) | Failure domain | Live migration | Notes |
|---|---|---|---|---|
| Local NVMe, raw on LVM-thin | ~50 µs; the network alone costs about as much ([yourcmc](https://yourcmc.ru/wiki/Ceph_performance)) | One host | Copy memory and disk together (QEMU `blockdev-mirror` over NBD) | Thin snapshots; lowest CPU per IO |
| Local ZFS zvol | Local NVMe plus CoW/ARC overhead [unverified magnitude] | One host | Same, or `zfs send` | Checksums, send/recv replication; ARC competes with guest RAM |
| qcow2 on ext4/xfs | Local NVMe plus L2 metadata cost [unverified] | One host | Same | Persistent dirty bitmaps live in qcow2 ([QEMU](https://www.qemu.org/docs/master/interop/bitmaps.html)); good for templates and backing chains |
| Ceph RBD (Squid 19.2.x / Tentacle 20.2.x) | Hard to get under ~0.5 ms read / ~1 ms write; ~0.37/0.72 ms in a vendor test ([yourcmc](https://yourcmc.ru/wiki/Ceph_performance)) | CRUSH-defined (host/rack) | Instant: shared storage | Tentacle 20.2.0 (Nov 2025) added FastEC for RBD ([ceph.io](https://ceph.com/en/news/blog/2025/v20-2-0-tentacle-released)); latest is 20.2.4, a CVE hotfix (Aug 2026) ([ceph.io](https://ceph.io/en/news/blog/2026/v20-2-4-v19-2-6-combo-released)) |
| Crimson / SeaStore | n/a | n/a | n/a | Tentacle ships SeaStore as a **tech preview** only ([ceph.io](https://ceph.io/en/news/blog/2025/crimson-T-release)). Not production |
| LINSTOR / DRBD 9 | Local reads; writes pay one network RTT to the peer ([LINBIT](https://linbit.com/blog/the-impact-of-network-latency-on-write-performance-when-using-drbd)) | 2–3 replicas | Primary/secondary roles swap when the VM moves ([NSRC](https://nsrc.org/workshops/2026/nsrc-ku-virt/cloud-virt/en/presentations/linstor.pdf)) | Diskless clients beat iSCSI on random IO ([LINBIT](https://linbit.com/blog/drbd-client-diskless-mode-performance)) |
| SPDK vhost-user-blk | Lowest software overhead, but polling burns pinned cores ([SPDK reports](https://spdk.io/news/2023/11/22/Performance_Report_Update)) | Whatever backs it | Needs vhost-user-aware migration | io_uring gets close with less complexity ([yourcmc](https://yourcmc.ru/wiki/Ceph_performance)) |
| NVMe-oF/TCP | ~146 µs remote access demonstrated by Intel/Lightbits ([B&F](https://www.blocksandfiles.com/composable/2020/09/30/intel-tech-and-lightbits-labs-make-nvme/tcp-faster/1595871)) | The target | Shared storage | Kernel initiator and target are in mainline; profiling study: [NSDI'25](https://www.usenix.org/system/files/nsdi25-kang.pdf) |
| Longhorn v2 (SPDK) | Better than v1 | Replicas | k8s-only | V2 engine GA in v1.12 (May 2026) ([release](https://newreleases.io/project/artifacthub/helm/longhorn/longhorn/release/1.12.0)). Tied to Kubernetes |
| StorPool (commercial) | 0.06 ms random write with 3× replication (vendor lab, 2018) ([SNL](https://www.storagenewsletter.com/2018/08/03/storpool-0-06ms-latency-on-nvme-powered-shared-storage-system/)) | Replicas | Shared | Proves a fast SDS is possible; Vitastor got 0.14 ms vs Ceph's 1 ms on the same hardware ([yourcmc](https://yourcmc.ru/wiki/Ceph_performance)) |
| Lightbits (commercial) | Sub-ms NVMe/TCP | Replicas | Shared | Used on OCI/AWS/Azure ([B&F](https://www.blocksandfiles.com/block/2024/11/05/lightbits-brings-high-performance-block-storage-to-oracle-cloud/1599610)) |

### What hosts actually run
| Provider | Storage |
|---|---|
| DigitalOcean | Droplet disks on local SSD. Volumes and Spaces on Ceph: 47 production clusters, 200+ PB, 28,500+ OSDs. New capacity goes into new clusters rather than growing old ones ([Ceph Day NYC 2024](https://ceph.io/assets/pdfs/events/2024/ceph-days-nyc/2024%20Ceph%20Day%20NYC%20How%20we%20Operate%20Ceph%20at%20Scale.pdf)) |
| Hetzner Cloud | Local NVMe for server disks. Dropped its Ceph server-disk option in 2022, citing better performance on local storage ([forum](https://forum.proxmox.com/threads/hetzner-is-dropping-ceph-support.110289)). Volumes are triple-replicated ([FAQ](https://docs.hetzner.com/cloud/volumes/faq)). Live migration copies local NVMe and RAM with **<1 s blackout** ([docs](https://docs.hetzner.com/cloud/servers/technical-concepts/architecture/)) |
| Fly.io | A volume is a slice of local NVMe on the Machine's host ([docs](https://fly.io/docs/volumes/overview)) |

### Backup
- **QEMU dirty bitmaps** track changed blocks for incremental backup ([QEMU](https://www.qemu.org/docs/master/interop/bitmaps.html)).
- **Proxmox VE + PBS:** the backup stack creates the bitmap automatically. It survives migration but is **dropped on VM shutdown or disk resize**; persistent bitmaps are an open feature request ([Proxmox staff](https://forum.proxmox.com/threads/using-pbs-with-qemu-dirty-bitmaps.176970)).
- **PBS format:** content-addressed chunks. Each snapshot uploads only new chunks but still references every chunk, so it is a full backup ([PBS docs](https://pbs.proxmox.com/docs/technical-overview.html)). PBS is written in Rust. Its license is AGPL [unverified], so talk to it over its protocol; don't link it into an MIT codebase.
- **MIT-friendly alternative:** rustic/restic-style repositories on S3 [unverified license detail]. Ceph users can also use `rbd export-diff`.

## B. Bare-metal / dedicated
| Tool | State in 2026 | Fit for KAREN |
|---|---|---|
| Equinix Metal | **Shut down 30 Jun 2026** ([Equinix](https://docs.equinix.com/goodbye-metal/)) | Its tooling legacy is Tinkerbell |
| Tinkerbell | Still CNCF **Sandbox** (since 2020); stars −14% year over year ([CNCF](https://www.cncf.io/projects/tinkerbell/)) | Go, k8s CRDs; momentum risk |
| Metal3 | CNCF Incubating since Aug 2025 ([CNCF](https://www.cncf.io/blog/2025/08/27/metal3-io-becomes-a-cncf-incubating-project)) | Kubernetes plus Ironic underneath; heavy |
| OpenStack Ironic | Most mature. Cleaning steps `erase_devices`, `erase_devices_metadata`, `erase_devices_express`; NVMe secure erase on by default; hardware erase can fall back to `shred` ([docs](https://docs.openstack.org/ironic/latest/admin/cleaning.html)). Switch VLAN automation via networking-generic-switch (Netmiko) ([NGS](https://docs.openstack.org/networking-generic-switch/latest/netmiko-device-commands.html)); VXLAN support ([docs](https://docs.openstack.org/ironic/2026.1/admin/vxlan.html)) | Python; usable standalone via Bifrost |
| Canonical MAAS | 3.7 current ([notes](https://canonical.com/maas/docs/latest/reference/release-notes)) | Ubuntu/snap-centric |
| NVIDIA Infra Controller (ex-Carbide) | **Rust**, Apache-2.0. Zero-touch lifecycle with DPU-enforced isolation; repo moved to `dsx-ai-factory` on 4 Sep 2026 ([GitHub](https://github.com/NVIDIA/infra-controller)) | Best Rust reference design. Assumes k8s, Vault, Temporal, and BlueField DPUs |
| Latitude.sh | São Paulo-founded bare-metal cloud. Advertises "5-second deployments" ([features](http://latitude.sh/features)); cut deploy time 50% in Sep 2026 ([changelog](https://www.latitude.sh/changelog/bare-metal-deployments-are-now-50-faster-090126)) | Commercial benchmark for BR |

### Building blocks
- **Redfish in Rust:**
  - `nv-redfish` 0.7.x (Apache-2.0): generated from schemas with only the features you enable, ETag cache, SSE events, UpdateService (firmware), SecureBoot, BIOS, and OEM features for Dell, HPE, Lenovo, Supermicro, AMI and NVIDIA ([repo](https://github.com/NVIDIA/nv-redfish)).
  - `libredfish` is the older NVIDIA crate ([docs.rs](https://docs.rs/libredfish)).
- **IPMI:** keep it as a fallback for power control and Serial-over-LAN on old BMCs. The SOL console can be bridged to a web terminal.
- **OpenBMC:** where available, its BMC web server is Redfish-native (NVIDIA ships a bmcweb fork).
- **Boot:**
  - UEFI HTTP(S) Boot removes TFTP; it can chainload iPXE ([iPXE](https://ipxe.org/appnote/uefihttp), [UEFI](https://uefi.org/sites/default/files/resources/FINAL%20Pres4%20UEFI%20HTTP%20Boot.pdf)).
  - Redfish VirtualMedia works when the provisioning network is untrusted.
  - Ironic supports both ([boot interfaces](https://docs.openstack.org/ironic/latest/admin/interfaces/boot.html)).
- **Out-of-band network:** a separate BMC VLAN/VRF, reachable only from controllers. Never route it to tenants.
- **Secure erase between customers:** NVMe Format with crypto erase (SES=2) or Sanitize. Verify with a read-back sample, and fail closed the way Ironic does by default.
- **Switch automation:** keep a per-port VLAN/VRF state machine (provisioning VLAN → tenant VLAN) and push it via gNMI/NETCONF/EOS API. Netmiko CLI is the lowest common denominator.

## C. Web hosting isolation
| Approach | Isolation | Density / latency | Notes |
|---|---|---|---|
| CloudLinux LVE + CageFS | Kernel-level per-user limits on CPU, IO, IOPS, NPROC, entry processes, inodes ([docs](https://docs.cloudlinux.com/cloudlinuxos/limits)) | Highest (shared PHP) | Proprietary per-server license; ties you to cPanel-era tooling |
| cgroup v2 + namespaces (systemd-nspawn, crun/youki, Incus) | Shared kernel | Thousands per host; native syscalls | Reproduces LVE with `cpu.max`, `memory.max`, `io.max`, `pids.max`. Incus 7.0 LTS (May 2026) also runs OCI images ([Incus](https://linuxcontainers.org/incus/news/2026_05_05_16_29.html)) |
| gVisor | User-space kernel (Sentry); systrap platform | Syscall/IO overhead; own netstack ([docs](https://gvisor.dev/docs/architecture_guide/platforms/), [USENIX](https://www.usenix.org/system/files/hotcloud19-paper-young.pdf)) | Poor fit for syscall-heavy PHP |
| Kata (Cloud Hypervisor) | VM per pod | Per-sandbox overhead its own docs call non-negligible ([Kata](https://kata-containers.github.io/kata-containers/design/host-cgroups)) | k8s-oriented |
| Firecracker microVM per app | Hardware (KVM); 6 emulated devices | <5 MiB VMM overhead, ~125 ms boot, up to 150 microVMs/s per host ([spec](https://github.com/firecracker-microvm/firecracker/blob/main/SPECIFICATION.md), [paper](https://css.csail.mit.edu/6.5660/2026/readings/firecracker.pdf)) | Fly.io model: microVMs plus local NVMe volumes, ~300 ms Machine boot via API ([Fly](https://fly.io/docs/reference/architecture)) |

### Edge / ACME
| Edge | Facts | Fit |
|---|---|---|
| LiteSpeed Enterprise | Proprietary | Avoid in an MIT stack |
| nginx | Proven | C, reload-based config |
| Caddy | On-Demand TLS issues certificates at the first handshake, gated by an `ask` endpoint ([Caddy](https://caddyserver.com/docs/automatic-https)). Abuse risk from CT-log scanners ([#6188](https://github.com/caddyserver/caddy/issues/6188)) | Go |
| Pingora | Rust, Apache-2.0. At Cloudflare: 70% less CPU and 67% less memory than the old NGINX-based proxy, >1T requests/day ([blog](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/)) | Library, not a server: KAREN writes routing, ACME, and cache |

## Recommendation for KAREN

### A. Storage
**Phase 1 — local NVMe:**
- `raw` volumes on **LVM-thin**, virtio-blk with iothreads and io_uring. This is the Hetzner/DO-droplet model: lowest latency, no shared failure domain.
- Live migration: block mirror over NBD plus memory pre-copy. Hetzner shows a <1 s blackout is achievable.
- Backups: KAREN-managed dirty bitmaps → a chunked, deduplicated repository on S3 (rustic-style, MIT-compatible). Also write a PBS-protocol target for users who already run PBS.
- Raw volumes lose their bitmap on shutdown. Fall back to a chunk-hash full read, which is still deduplicated.

**Phase 2 — replicated tier:**
- **Ceph Tentacle 20.2.x RBD** with classic OSD/BlueStore for detachable Volumes and "HA VPS" plans. Follow DO's lesson: many medium-sized clusters, not one giant one.
- Optional **LINSTOR/DRBD** for a low-latency 2-replica HA tier.

**Avoid:** Crimson/SeaStore (tech preview), Longhorn (needs k8s), SPDK vhost on day one (core-pinning cost), qcow2 for hot data. Treat StorPool and Lightbits as benchmarks, not dependencies.

### B. Bare metal
Build a **native Rust metal service**:
- **BMC control:** `nv-redfish` for Redfish, plus an IPMI/SOL fallback.
- **Boot:** UEFI HTTPS Boot → iPXE → a Rust ramdisk agent. The agent handles inventory, image streaming, NVMe crypto-erase/sanitize, and Redfish UpdateService firmware updates.
- **Network:** a switch-port VLAN state machine.
- **References:** NICo (Rust) for architecture; Ironic's cleaning and networking-generic-switch semantics for behavior. Run Bifrost/Ironic standalone only as a stopgap driver for exotic hardware.
- **Phase order:** power, console and inventory → reinstall with erase → VLAN automation → firmware.

**Avoid:** MAAS, Metal3 (needs k8s and Ironic), Tinkerbell (Sandbox, waning momentum), and building on anything Equinix-hosted.

### C. Web hosting
**Phase 1:** one hardened container per account.
- cgroup v2 limits that mirror LVE, user namespaces, a read-only base image plus a per-user overlay (CageFS-equivalent), and per-account PHP-FPM.
- A **Pingora-based KAREN edge** with on-demand ACME gated by an ownership check, rate limits and a pre-issued certificate store.

**Phase 2:** "App hosting" (Docker/Node/Python) as **Firecracker microVMs** on local NVMe volumes, with snapshot restore for scale-to-zero.

**Avoid:** CloudLinux and LiteSpeed (proprietary licensing), gVisor for PHP, Kata outside k8s.

### Phase ordering
1. Local-NVMe VPS plus S3 backups
2. Shared hosting containers plus Pingora edge
3. Bare-metal power/console/reinstall/erase
4. Ceph volumes plus HA tier
5. Firecracker app platform
6. Bare-metal VLAN and firmware automation
