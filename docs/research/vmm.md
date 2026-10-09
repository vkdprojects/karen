# KAREN VMM selection for general-purpose VPS (as of 2026-10-08)

Researched 2026-10-08. [unverified] = not confirmed by a primary source.

## 1. Current versions
| VMM | Latest | Lang / license | Source |
|---|---|---|---|
| QEMU | 11.1.2 / 11.0.5 (2026-09-28) | C, GPLv2 | [qemu.org/download](https://www.qemu.org/download) |
| libvirt | 12.6.0 (2026-08-03) | C, LGPL | [libvirt news](https://libvirt.org/news.html) |
| Cloud Hypervisor (CH) | v53.0 (2026-07-12) | Rust, Apache-2/BSD | [release](https://github.com/cloud-hypervisor/cloud-hypervisor/releases/tag/v53.0) |
| Firecracker | v1.17.0 (2026-09-10) | Rust, Apache-2 | [release](https://github.com/firecracker-microvm/firecracker/releases/tag/v1.17.0) |
| crosvm | rolling (no versioned releases) | Rust, BSD-3 | [repo](https://github.com/google/crosvm) |
| Kata | 3.27+; Dragonball is the default VMM | Rust/Go | [Kata virtualization doc](https://github.com/kata-containers/kata-containers/blob/main/docs/design/virtualization.md) |

## 2. Features for VPS sales
| Capability | QEMU+KVM | Cloud Hypervisor v53 | Firecracker 1.17 | crosvm | Kata |
|---|---|---|---|---|---|
| Windows guests | Full support (BIOS and UEFI) | UEFI only. Docs say Windows **install requires QEMU**, and "Windows 11 with TPM 2.0 was not proven successful" ([windows.md](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/windows.md)). v53 added "various TPM fixes incl. Windows" ([v53](https://www.cloudhypervisor.org/blog/cloud-hypervisor-v53.0-released)) | No | Not a target [unverified] | No (containers only) |
| Custom ISO install + graphical console | Yes. VNC/SPICE; libvirt 12.5 adds a standalone, sandboxed `qemu-vnc` ([news](https://libvirt.org/news.html)) | **No.** A display is an explicit anti-goal ([#3212](https://github.com/cloud-hypervisor/cloud-hypervisor/issues/3212)); serial console/SAC only | No (kernel + rootfs boot) | virtio-gpu, desktop-oriented | No |
| UEFI / Secure Boot | OVMF with a per-VM NVRAM; host-side `uefi-vars` device ([docs](https://www.qemu.org/docs/master/devel/uefi-vars.html)) | `CLOUDHV.fd` is opened **read-only** ([uefi.md](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/uefi.md)), so there is no documented persistent NVRAM or Secure Boot key enrollment [inference] | No | — | — |
| vTPM | swtpm, mature | swtpm, CRB interface only ([tpm.md](https://raw.githubusercontent.com/cloud-hypervisor/cloud-hypervisor/main/docs/tpm.md)) | No | Yes (README) | — |
| Live migration | Pre-copy, post-copy, multifd, TLS | Pre-copy; post-copy (new in v53); mTLS; up to 128 parallel connections; VFIO mig v2 ([live_migration.md](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/live_migration.md)). n-2 version compatibility starts at v54 | **None** | No [unverified] | No |
| Migration with **local** disks | Yes: `--copy-storage-all` via NBD mirror ([libvirt wiki](https://wiki.libvirt.org/NBD_storage_migration.html)) | Not built in. Migration docs cover memory/device state only; disk copy needs external block replication (KAREN or SPDK) [inference] | — | — | — |
| Snapshot / restore | Internal, external, and live block snapshots | Yes; offloaded daemon + userfaultfd lazy restore (v53) | Yes. Its core strength (diff snapshots, UFFD) | Yes (`snapshot/` crate) | Via VMM |
| Hotplug CPU / RAM / disk / NIC | All four; Windows supports CPU and RAM hot-add | All four. Windows cannot hot-remove CPU or RAM (windows.md) | virtio-mem RAM; PCI hotplug is **dev preview** | Limited | Yes |
| Networking | virtio-net, vhost-net, vhost-user, vDPA, macvtap | virtio-net (TAP), vhost-user-net, vDPA | virtio-net TAP only (MMIO; PCI opt-in) | virtio-net, vhost | — |
| VFIO / GPU | Best: arbitrary PCIe root-port topology, large-BAR fixes in 10.1 | Yes, but **flat PCI topology** | No | Yes | QEMU/CH/Dragonball |
| Nested virtualization | Yes | Yes; v53 adds nested Hyper-V/WSL2 for Windows | No | — | — |
| Boot time | ~1–3 s with OVMF [unverified] | ~100s of ms to boot a Linux kernel directly [unverified] | **≤125 ms** to `/sbin/init`; VMM start 8 CPU-ms ([SPEC](https://raw.githubusercontent.com/firecracker-microvm/firecracker/main/SPECIFICATION.md)) | Fast | ~100s ms |
| VMM memory overhead | ~20–60 MiB per VM [unverified] | Low, ~10s of MiB [unverified] | **≤5 MiB** (SPEC) | Low | Low |

**What disqualifies Cloud Hypervisor as the sole VPS engine:** it has no graphical console, ISO installs need QEMU, the Windows 11 vTPM path is unproven, Secure Boot NVRAM is not documented, and migration does not move disks. A real data point: Ubicloud (a CH shop) **had to switch to QEMU** for NVIDIA B200 passthrough. CH can't build PCIe root-port hierarchies, and CUDA failed with `cuInit -> 3` ([Ubicloud](https://www.ubicloud.com/blog/virtualizing-nvidia-hgx-b200-gpus-with-open-source)).

## 3. Security (2025–2026)
| Item | Layer | Facts | Lesson for KAREN |
|---|---|---|---|
| **Januscape** CVE-2026-53359 (CVSS 8.8) | In-kernel KVM/x86 shadow MMU, use-after-free | Affects Intel **and** AMD; bug from 2010 to fix `81ccda30b4e8` (2026-06-16). Used as a kvmCTF 0-day. Requires guest root **and nested virtualization exposed**. "Does not occur in QEMU", so it is independent of the VMM ([repo](https://github.com/V4bel/Januscape), [BleepingComputer](https://www.bleepingcomputer.com/news/linux/new-januscape-linux-kernel-flaw-allows-vm-escape-on-intel-amd-devices)) | Choosing a different VMM does **not** protect the host kernel |
| **Zapscape** CVE-2026-64561 | KVM/x86 shadow-MMU zap path, use-after-free | Bug from 2020 to fix `2abd5287f083` (2026-07-21). Needs nested virtualization; on Intel, only when both 4- and 5-level EPT are exposed to L1. Full PoC escape demonstrated ([repo](https://github.com/V4bel/Zapscape), [oss-sec](https://seclists.org/oss-sec/2026/q3/460)) | Ship with nested virtualization **off** by default |
| **ITScape** CVE-2026-46316 | KVM/arm64 vGIC-ITS race | Bug from 2024 to fix `13031fb6b835` (2026-06-05) ([repo](https://github.com/V4bel/ITScape)) | Same rule for ARM hosts |
| QEMU CVE-2026-48914 (CVSS 6.7) | QEMU virtio-blk SCSI request path | Heap out-of-bounds write by a privileged guest ([NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-48914)) | Disable `scsi=on` on virtio-blk; confine QEMU with sVirt/seccomp |
| Firecracker CVE-2026-5747 | virtio-pci transport (1.13–1.15.0) | Out-of-bounds write, possible host code execution; MMIO default not affected; reported by Anthropic ([AWS](https://aws.amazon.com/security/security-bulletins/2026-015-aws/)) | Rust device models still ship memory-corruption bugs (via `unsafe`/logic paths) |
| Kata CVE-2026-24834 (CVSS 9.3) | Kata + CH guest rootfs writable | Root inside the guest VM; host not affected ([NVD](https://nvd.nist.gov/vuln/detail/cve-2026-24834)) | Integration glue is part of the attack surface |

**Sandboxing per VMM:**
- **QEMU:** libvirt sVirt (SELinux/AppArmor labels), `-sandbox` seccomp, an unprivileged user per VM.
- **Cloud Hypervisor:** per-thread seccomp filters; v53 adds `--seccomp=errno`. Accepts pre-opened VFIO/iommufd FDs, so the VMM can run unprivileged.
- **Firecracker:** jailer (chroot, namespaces, cgroups, seccomp).
- **crosvm:** a minijailed process per device. Strongest design, but built for ChromeOS and Android.

**Host hardening that matters more than the VMM choice:**
- Disable nested virtualization: `kvm_intel nested=0` / `kvm_amd nested=0`.
- Make `/dev/kvm` mode 0660. RHEL ships 0666, which turns these bugs into local privilege escalation.
- Run a kernel livepatch pipeline. The researcher's warning: "*Winter is coming*."

## 4. Confidential computing
| | AMD SEV-SNP | Intel TDX |
|---|---|---|
| QEMU | Upstream since 9.1 [unverified]; 11.0 adds SNP/TDX reset ([11.0](https://www.qemu.org/2026/04/22/qemu-11-0-0)) | Upstream since QEMU 10.1 + libvirt 11.6 + Linux 6.16 ([Nova docs](https://docs.openstack.org/nova/latest/admin/tdx.html)) |
| CH | KVM + IGVM build features; **no hotplug, no hugepages, no vIOMMU** under SNP ([doc](https://raw.githubusercontent.com/cloud-hypervisor/cloud-hypervisor/main/docs/amd_sev_snp.md)); libvirt `ch` can start SNP guests | `--features tdx`; docs still point to Intel out-of-tree KVM trees ([doc](https://github.com/cloud-hypervisor/cloud-hypervisor/blob/main/docs/intel_tdx.md)), so experimental |
| Firecracker / crosvm | No [unverified]; crosvm targets Android pKVM | No |

## 5. Rust-friendliness and production users
| | Control API | Rust integration | Production users |
|---|---|---|---|
| QEMU | QMP (JSON over Unix socket); libvirt XML/RPC | Drive QMP from Rust (`serde`), or libvirt bindings (`virt` crate) [unverified] | Red Hat/OpenStack/Proxmox; most VPS providers [unverified] |
| CH | REST/OpenAPI on a Unix socket; event monitor | Native Rust, rust-vmm crates (`kvm-ioctls`, `vm-memory`, `virtio-queue`, `vhost-user-backend`). libvirt `ch` driver is "early stage" ([drvch](https://libvirt.org/drvch.html)) | Ubicloud (CH + SPDK vhost-user-blk, [blog](https://www.ubicloud.com/blog/cloud-virtualization-red-hat-aws-firecracker-and-ubicloud-internals)); contributors from Microsoft, Meta, Crusoe, Cyberus (v53 credits) |
| Firecracker | REST/OpenAPI | Native Rust, rust-vmm | AWS Lambda/Fargate; E2B ([fc-versions](https://github.com/e2b-dev/fc-versions/releases)) |
| crosvm | Control socket | Rust, but its own crates (`base`, `cros_async`) instead of rust-vmm | ChromeOS, Android AVF, Cuttlefish |

## 6. Host tuning for latency
| Knob | Recommendation |
|---|---|
| Hugepages | 1 GiB pages reserved at boot (`default_hugepagesz=1G hugepages=N`) for dedicated-CPU plans; THP for burstable plans. Firecracker 1.17 added a THP option. Note: CH's SNP mode disables hugetlb |
| CPU pinning / isolation | Pin vCPUs 1:1 to sibling-aware pCPUs on dedicated plans; keep housekeeping cores (≥2 per socket) for QEMU I/O threads, vhost, and OVS/SPDK. Prefer cgroup v2 `cpuset` partitions (`cpuset.cpus.partition=isolated`) over `isolcpus` (runtime-changeable) [unverified detail]; `nohz_full`/`rcu_nocbs` only on dedicated cores |
| NUMA | Keep guest RAM and vCPUs in a single NUMA node (`numatune strict`); also place NIC/NVMe IRQs and iothreads on that node. CH supports remapping memory zones to a NUMA node during migration (`zone_updates`) |
| Halt-polling | Host `halt_poll_ns` defaults to ~200 µs [unverified]: raise it for latency SKUs, lower it or set 0 for overcommitted shared plans (burns CPU). Linux guests can use the `cpuidle-haltpoll` driver ([kernel doc](https://docs.kernel.org/virt/kvm/halt-polling.html)) |
| Network | Use virtio-net multiqueue (`queues = vCPUs`) + vhost-net as the default. Move to vhost-user (OVS-DPDK) only when PPS-bound. Firecracker's own spec adds ~60 µs of latency and is limited to 14.5–25 Gbps per core ([SPEC](https://raw.githubusercontent.com/firecracker-microvm/firecracker/main/SPECIFICATION.md)) |
| Disk (QEMU) | virtio-blk with `io_uring` or `native`, `cache=none`, plus **iothread-vq-mapping** with 4–8 iothreads pinned away from vCPUs ([Red Hat](https://developers.redhat.com/articles/2024/09/05/scaling-virtio-blk-disk-io-iothread-virtqueue-mapping)) |
| Disk (high end) | vhost-user-blk with SPDK gives the lowest latency (the Ubicloud model) and works with **both** QEMU and CH. It costs a dedicated polling core per host |

## Recommendation for KAREN

**Primary VMM: QEMU 11.x on KVM, managed through libvirt 12.x, behind a Rust `Vmm` trait in KAREN.**
Why:
- It is the only option that covers the whole VPS product checklist: Windows, any ISO, a VNC console (sandboxed `qemu-vnc`), OVMF Secure Boot with NVRAM, swtpm vTPM, VFIO/GPU with real PCIe topology, SEV-SNP and TDX upstream.
- It does **live migration with local NVMe** (NBD mirror), which matters for cheap São Paulo/US/EU nodes without SAN.
- Using libvirt gives sVirt, cgroups, and migration orchestration for free.
- Keep the dependency thin: KAREN's Rust agent emits domain XML and talks QMP only for what libvirt lacks.

**Fallback / second engine: Cloud Hypervisor v53+ (direct REST, not libvirt `ch`)** for a Linux-only "cloud-image" VPS tier and the internal/managed-service fleet.
- Benefits: Rust, a smaller attack surface, per-thread seccomp, fast hotplug, mTLS migration and post-copy, fast snapshot restore.
- Constraints: build images and Windows installs with QEMU, and pair it with SPDK vhost-user-blk so disk replication or migration happens in the storage layer.
- Re-evaluate it for Windows after Windows 11 + TPM is proven upstream.

**Firecracker: yes, but only for the website/app-hosting tier.**
- Use: one microVM per site/tenant (PHP/Node/static) with ≤5 MiB VMM overhead and ≤125 ms boot. Snapshot/restore enables scale-to-zero, and the jailer gives strong isolation for shared hosting where you control the guest kernel.
- Not for VPS: it has no Windows, no UEFI or ISO install, no live migration, and hotplug is only a dev preview.
- Keep it on the default MMIO transport unless you track PCI CVEs closely (CVE-2026-5747).

**Avoid:**
1. crosvm as the VPS VMM (consumer/Android focus, Gerrit-only workflow, no cloud migration story).
2. Kata as a VPS layer (it is a container runtime; consider it only for a future "containers-as-a-service" product).
3. libvirt's `ch` driver in production (self-described early stage).
4. **Exposing nested virtualization by default.** Januscape and Zapscape both require it; sell it as an opt-in on dedicated-host SKUs only.
5. Shared plans without a kernel livepatch/rolling-reboot pipeline. The 2026 escapes are in KVM itself, so no VMM choice removes that risk.
6. virtio-blk `scsi=on` and unneeded emulated devices in QEMU; keep machine type q35 with virtio-only devices to shrink the attack surface.
