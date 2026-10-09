# Testing strategy

KAREN is built mostly by AI agents. Tests are the contract that tells an agent its change is correct; a feature without its tests is not done ([../../AGENTS.md](../../AGENTS.md#definition-of-done)).

## Layers

| Layer | What | Where | Runs on |
|---|---|---|---|
| Unit | Pure logic: scheduler, IPAM, settings resolution, state machines, firewall compilation (policy → nft / OVN ACL), traffic delta math | `#[cfg(test)]` next to the code | Every PR |
| Property | Invariants over random inputs (`proptest`): state machines accept only listed transitions; IPAM never double-assigns; firewall compiler output is equivalent for `routed` and `ovn`; counter-reset handling never yields negative deltas | `tests/prop_*.rs` | Every PR |
| Contract | REST responses validate against `api/openapi.yaml`; route table = spec; `oasdiff breaking`; `buf breaking` for protos; generated client round-trip | `crates/karen-control/tests/contract` | Every PR |
| Component (fakes) | Control + in-process fake agents implementing the same traits (`FakeVmm`, `FakeStorage`, `FakeNetwork`), real DB (SQLite and Postgres in a container) | `tests/component` | Every PR |
| System (real KVM) | Real control + real agents in nested-KVM hosts (Ubuntu 26.04), real libvirt/QEMU/LVM/nftables/OVS | `tests/system`, driven by `xtask` | Nightly, and on PRs labelled `system-tests` |
| Chaos | Kill/restart/partition control, agents, DB during workflows | `tests/chaos` | Nightly |
| Load | Simulated agents at fleet scale | `tests/load` | Weekly, release gate |
| Upgrade | Control N with agent N-1; DB migrations from previous release with real data | `tests/upgrade` | Release gate |

## Required suites

### State machines
For each machine in [STATE_MACHINES.md](./STATE_MACHINES.md): a table-driven test listing every (state, event) pair, expecting either the documented next state or `invalid_state`. The table in the test is generated from the same Rust definition that the docs table describes, and a doc-sync test fails if the Markdown table and the code diverge.

### Task queue
- Every step is executed twice in a row in tests (idempotency harness): the second run must be a no-op with the same output.
- Lease expiry, stale epoch writes rejected, lock contention returns `resource_busy`, compensation order, `needs_attention` on failing compensation.
- Throughput target from [TASKS.md](./TASKS.md#throughput-target).

### Tenant isolation suite
Generated from the OpenAPI spec: for **every** endpoint with a path id, account B's token requests account A's resource and MUST get `404`; list endpoints MUST NOT include A's items; `Karen-Account` with a non-descendant account MUST get `404`. A new endpoint is automatically covered; an endpoint that can't be covered must be explicitly allow-listed with a reason.

Also: a lint test fails if repository code issues a query on a tenant table without the `AuthzContext` filter helper.

### Network (system tests)
- Anti-spoof: a VM sending packets with a foreign source IP or MAC is dropped (tested in both modes).
- Firewall: for a matrix of policies, real packets (scapy/hping from a peer VM) match the expected allow/deny; `routed` and `ovn` produce identical results (parity test from [ROADMAP.md](../../ROADMAP.md)).
- Metrics accuracy: transfer a known byte count with iperf; graph and billing counters within ±1 %.
- Counter reset: reboot / migrate a VM mid-transfer; no negative or double-counted deltas.
- SMTP policy `block` drops port 25; `allow` doesn't.

### CPU compatibility (system tests)
Nested hosts started with different emulated CPU models (`-cpu Broadwell-v4` and `-cpu Cascadelake-Server`) in one node group: baseline computed, VM live-migrates both ways under `baseline`, `host-passthrough` VMs are refused targets with a different CPU.

### Storage
- `jbod` with two pools: placement balances; pool full threshold stops placement.
- Thin pool at 85 % stops placement; alert at 80 %.
- Snapshot create/restore; disk grow online; discard frees allocation.
- Deleted disk space is not readable by the next VM (read new disk, expect zeros).

### Chaos
During `vm.create`, `vm.migrate` and `node.drain`: kill the agent, kill control, drop the network between them, restart the DB. Invariants checked afterwards: no duplicate domains, no leaked LVs or taps, no IP assigned twice, every task ends `succeeded` or `failed` (none stuck), compensation leaves no residue.

### Backup and restore
Weekly: restore the latest control backup into a fresh instance, start in recovery mode against the test fleet, verify audit hash chain and zero unexpected diffs.

### Load tests
- 50 simulated nodes / 5,000 VMs (fake agents speaking the real gRPC protocol).
- Task throughput: 50 tasks/s for 10 min, p99 dequeue < `500ms`.
- API: p99 < `200ms` for reads at 200 req/s.
- Metrics: agent collection for 200 real VMs < `200ms` and < 1 % of one core; VictoriaMetrics stable with 5,000 simulated VMs for 24 h ([METRICS.md](./METRICS.md#cost-budget)).

## CI tiers

| Tier | Trigger | Content | Budget |
|---|---|---|---|
| PR | Every push | fmt, clippy `-D warnings`, `cargo-deny`, typos, unit, property, contract, component (SQLite + Postgres) | < 15 min |
| Nightly | Schedule | System tests on nested KVM, chaos | < 2 h |
| Weekly | Schedule | Load, restore drill | < 4 h |
| Release | Tag | All of the above + upgrade tests | — |

A red nightly blocks the next release until fixed. Tests are never skipped or deleted to get green; a flaky test is fixed or quarantined with an issue and an owner within the same week.

## Fakes

Fakes implement the same traits as real backends (`Vmm`, `StorageBackend`, `NetworkBackend`, `Bmc`) and live in `crates/karen-testkit`. They simulate latency, failures (`fail_next(step, error)`), and crashes so component tests cover error paths without real hardware. A behaviour added to a real backend MUST be added to its fake in the same PR.
