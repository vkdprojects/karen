# Operations: maintenance, upgrades, control-plane backup

## Node maintenance

Node states are in [STATE_MACHINES.md](./STATE_MACHINES.md#node).

### Drain

`POST /admin/nodes/{id}/actions/drain` with:

```json
{
  "mode": "live",                  // live | cold | none
  "unmigratable": "notify_and_wait", // notify_and_wait | cold_migrate | stop | skip
  "deadline": "2026-10-12T03:00:00Z",
  "max_parallel": 2
}
```

1. Node → `draining`. The scheduler stops placing VMs on it.
2. Control plans a target per VM (capacity, storage class, CPU compatibility per [COMPUTE.md](./COMPUTE.md#scheduler-rules)) and returns the plan in the drain task output **before** moving anything, including VMs with no valid target.
3. Child `vm.migrate` tasks run with `max_parallel` (capped by `tasks.node_concurrency.migrate` and the global limit). `local` disks are copied (NBD mirror); `replicated` move RAM only.
4. VMs with no live target follow `unmigratable`:
   - `notify_and_wait`: customers notified (`notify.events.maintenance_scheduled`), VM moved cold at `deadline`.
   - `cold_migrate`: stop → copy → start now.
   - `stop`: stop and leave on the node.
   - `skip`: leave running; drain ends with those VMs listed.
5. When the node has no running VMs (or only skipped ones) → `maintenance`.

### Maintenance and resume

- In `maintenance` the operator updates firmware, kernel, packages, hardware.
- `POST /admin/nodes/{id}/actions/resume` runs the same checks as enrollment (hardening, versions vs fleet target, pools, RAID status, network). Pass → `active`. Fail → stays in `maintenance` with the failed checks listed.

## Rolling upgrades

`POST /admin/rollouts` upgrades a set of nodes one batch at a time:

```json
{
  "selector": { "node_group_id": "ngrp_...", "tags": ["sp1"] },
  "target_versions": { "qemu": "11.1.2", "libvirt": "12.6.0", "kernel": "7.2.x", "ovs": "4.0.0" },
  "max_unavailable": 1,
  "drain": { "mode": "live", "unmigratable": "notify_and_wait" },
  "soak": "30m",
  "pause_on_failure": true
}
```

For each batch: drain → agent applies the packages (apt, pinned versions from the fleet target) and reboots if needed → resume checks → `soak` period watching VM health (crashes, PSI, task failures) → next batch. Any failure pauses the rollout. Rollouts follow the update policy in [ARCHITECTURE.md](../ARCHITECTURE.md#update-policy): only version sets that passed staging.

`karen-agent` itself upgrades the same way; control supports agents `N-1` ([API.md](./API.md#agent-protocol)), so control is upgraded first.

## Control database backup

Starts in **v0.1**, not later. Losing the control DB doesn't stop running VMs (agents keep them), but it loses customers, IP history, billing and configuration.

| DB | Method | Default schedule |
|---|---|---|
| SQLite (v0.1) | Online backup API (consistent snapshot while running), compressed + encrypted, to `control_backup.target` | Every `15m`, plus one at shutdown |
| PostgreSQL (v0.2+) | Continuous WAL archiving + daily base backup via WAL-G or pgBackRest (external tool, configured by the installer), to S3 or local | Continuous (point-in-time recovery) |

- Encryption: backups are encrypted with a key derived from the KEK ([SECURITY_MODEL.md](./SECURITY_MODEL.md#secrets)). The KEK and the CA keys are **not** stored with the backups; `karen backup export-keys` writes them to a separate, operator-chosen location (printed once, offline storage recommended).
- Retention: `control_backup.retention` (default `30d`).
- Monitoring: `karen_control_backup_last_success_timestamp`; alert if older than `2 × interval` (SQLite) or WAL lag > `5m` (Postgres).
- Restore: `karen-control restore --from <backup> [--at <timestamp>]` restores into a new data dir, verifies schema version and audit hash chain, then starts.
- **After restore, reconcile with reality:** on startup control enters `recovery` mode — no new tasks dequeued, agents connect and send inventory, control lists differences (VMs created or deleted after the backup point, IPs in use) for admin review before resuming. Nothing is destroyed automatically.
- Restore drill: CI restores the latest test backup weekly ([TESTING.md](./TESTING.md#ci-tiers)); operators can run `karen backup verify` to test-restore into a temporary dir.

## Disaster scenarios

| Scenario | What happens | Recovery |
|---|---|---|
| Control down | VMs keep running; no API, no new tasks; agents buffer traffic deltas and events | Restart; tasks resume from the DB |
| Control DB lost | As above | Restore + recovery mode |
| Node down (`local`) | Its VMs are down | Bring node back, or restore VMs from backups onto other nodes (`POST /admin/vms/{id}/actions/restore-elsewhere`) |
| Node down (`replicated`) | Fence → HA restart elsewhere | Automatic ([STATE_MACHINES.md](./STATE_MACHINES.md#node)) |
| Disk failure, `jbod`/`raid0` | VMs on that pool lost | Replace disk, restore from backups |
| Disk failure, RAID 1/10/5 | Degraded, alert | Replace disk, rebuild |
| CA intermediate compromised | — | `karen ca rotate-intermediate`; agents re-enroll via renew with the old valid cert, or with new tokens |
