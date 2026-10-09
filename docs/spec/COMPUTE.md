# Compute policy: CPU models, overcommit, memory

## CPU models and migration across generations

Nodes in one fleet have **different CPUs** (e.g. HPE Gen9 Broadwell, Gen10 Cascade Lake, later EPYC). A live-migrated guest must see the same CPU features on the destination, or it crashes when it uses an instruction the new host lacks. KAREN handles this dynamically; operators never hand-pick CPU models.

### How it works

1. **Each node reports its CPU** at enrollment and on every inventory: vendor, model, flags, and libvirt's host CPU definition (`virConnectGetDomainCapabilities`).
2. **Migration domain = node group.** A node group contains nodes of one vendor (Intel and AMD can't live-migrate between each other; the agent refuses to join a group of the other vendor).
3. **Group baseline.** Control computes the common feature set of all `active` nodes in the group with libvirt `virConnectBaselineHypervisorCPU` and stores it in `node_group.cpu_baseline`. Recomputed when a node joins or leaves.
4. **A VM gets its CPU definition at creation**, according to `compute.cpu_mode`:

| `cpu_mode` | VM sees | Live migration | Use for |
|---|---|---|---|
| `baseline` (**default**) | The group baseline at creation time (named model + extra flags) | To any node whose CPU is a superset (`virConnectCompareHypervisorCPU`), also newer generations | Almost everything |
| `host-passthrough` | Exactly the host CPU (max performance, all flags such as AVX-512) | Only to nodes with identical CPU model and microcode | Plans that need every instruction and accept limited migration |

5. **The VM's CPU definition is frozen** in `vm.cpu_model` until a cold upgrade. Adding a newer node never changes running VMs; removing the oldest node raises the baseline for **new** VMs only.
6. **Upgrade path:** `POST /admin/vms/{id}/actions/cpu-model-upgrade` (or a fleet-wide rollout) sets the new baseline to apply at the next full stop/start. Customers may be notified (`notify.events.cpu_model_upgrade.customer`).

### Scheduler rules

- Placement and migration targets are filtered by `compare(vm.cpu_model, node.cpu) ∈ {identical, superset}`.
- Before a drain, control reports VMs with no valid target (e.g. `host-passthrough` on a unique CPU); the drain policy decides ([OPERATIONS.md](./OPERATIONS.md#drain)).
- `machine_type` (e.g. `pc-q35-10.2`) is pinned per VM the same way, so a newer QEMU on the destination keeps the guest's virtual hardware identical.

### Example

Group `sp1-intel` has Gen9 (Broadwell) and Gen10 (Cascade Lake). Baseline ≈ Broadwell. A VM created there runs on either and live-migrates Gen9 → Gen10 and back. When the last Gen9 is retired, the baseline becomes Cascade Lake for new VMs; old VMs keep Broadwell until a cold upgrade.

## Overcommit

| Setting | Default | Meaning |
|---|---|---|
| `compute.cpu_overcommit_ratio` | `1.0` | Sellable vCPUs = (host threads − `compute.host_reserved_cpus`) × ratio |
| `compute.ram_overcommit_ratio` | `1.0` | Sellable RAM = (host RAM − `compute.host_reserved_ram`) × ratio |

- Both are operator choices per node, node group, zone, region or global ([CONFIGURATION.md](./CONFIGURATION.md)).
- `cpu_class = dedicated` VMs are always pinned 1:1 and count against **physical** threads, never overcommitted. They are placed on cores outside the shared pool (cgroup v2 `cpuset` partitions).
- `hugepages = true` VMs reserve their RAM up front (hugepages can't be overcommitted).
- The scheduler refuses placement above the effective ratio (`no_capacity`).

## Memory: allocation, balloon, free page reporting

What the operator wants: "sold 20 GiB, guest uses 2 GiB → the host uses ~2 GiB; guest goes to 5 GiB and back to 2 GiB → the host gets 3 GiB back".

1. **Allocation on demand (always, built in):** QEMU guest RAM is not pre-allocated; the host only backs pages the guest touches. Exceptions: hugepages and `dedicated` plans with `compute.prealloc` (default `true` for dedicated, for predictable latency).
2. **Returning memory: virtio-balloon with free page reporting** (`compute.balloon.free_page_reporting`, default `true`). The guest kernel reports freed pages to the host, which discards them. This is what gives back the 3 GiB in the example, automatically, without any agent logic. Works with Linux guests ≥ 5.7 and Windows guests with the virtio balloon driver.
3. **Balloon stats** (`compute.balloon.stats_period`, default `10s`) feed the memory metrics ([METRICS.md](./METRICS.md#vm-metrics)).
4. **Active reclaim (optional):** `compute.balloon.auto_reclaim` (default `false`). When the host is under memory pressure (PSI `memory.some avg10 > compute.balloon.pressure_threshold`), the agent inflates balloons of overcommitted shared-plan VMs down to their observed working set + `compute.balloon.headroom` (default `20 %`), never below `compute.balloon.min_pct` (default `50 %`) of the plan. `deflate-on-oom` is always enabled so a guest under pressure gets memory back first.
5. Balloon is configurable per VM (`compute.balloon.enabled`, default `true`); dedicated/hugepages VMs ignore balloon reclaim.

## KSM

**Kernel Samepage Merging:** the host kernel scans VM memory and merges identical pages (e.g. 50 Ubuntu guests share one copy of the same kernel pages), saving RAM at the cost of CPU (`ksmd`) and a known **side-channel risk** (one VM can infer what another has in memory by timing writes to shared pages).

| Setting | Default |
|---|---|
| `compute.ksm.enabled` | `false` |
| `compute.ksm.pages_to_scan` | `100` |
| `compute.ksm.sleep_ms` | `20` |
| `compute.ksm.exclude_dedicated` | `true` |

When enabled, VMs of plans with `compute.ksm.opt_out = true` (default for `dedicated`) are marked unmergeable. The agent reports `ksm_pages_sharing` so the saving is visible ([METRICS.md](./METRICS.md#node-metrics)).

## Hot-add

- CPU and RAM hot-add when `compute.hotplug.enabled` (default `true` for Linux images that declare support, `false` otherwise). RAM via `virtio-mem`; maximum set at creation (`compute.hotplug.max_ram_factor`, default `2×` plan).
- Disk grow is always online (`blockresize` + guest notification); shrink is not supported.
- Hot-remove is not offered (unreliable on Windows).
