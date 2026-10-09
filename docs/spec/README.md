# Specifications

Behavioural specs for KAREN. [`../ARCHITECTURE.md`](../ARCHITECTURE.md) says **what** we use; these files say **how the system behaves**. Code implements these documents; when code and spec disagree, the spec wins until it is changed in the same PR as the code.

| File | Covers |
|---|---|
| [CONFIGURATION.md](./CONFIGURATION.md) | The "everything is configurable" rule: scopes, resolution, settings catalog |
| [DATA_MODEL.md](./DATA_MODEL.md) | Entities, fields, relations, ID and storage conventions, IP history |
| [STATE_MACHINES.md](./STATE_MACHINES.md) | VM, disk, IP, node and task lifecycles |
| [TASKS.md](./TASKS.md) | Durable task queue, workflows, retries, locks, reconciliation |
| [API.md](./API.md) | Public REST contract and the agent gRPC protocol |
| [SECURITY_MODEL.md](./SECURITY_MODEL.md) | PKI/mTLS, secrets, auth, console tokens, tenant isolation, audit |
| [COMPUTE.md](./COMPUTE.md) | CPU models and migration across CPU generations, overcommit, balloon, KSM |
| [METRICS.md](./METRICS.md) | Full metrics catalog, collection cost, retention |
| [ABUSE.md](./ABUSE.md) | Abuse detectors, action ladder, notifications |
| [OPERATIONS.md](./OPERATIONS.md) | Node maintenance, drain, rolling upgrades, control-plane database backup |
| [TESTING.md](./TESTING.md) | Test layers, CI tiers, required suites |

Conventions used in all specs:
- **MUST / MUST NOT / SHOULD / MAY** as in RFC 2119.
- Every tunable value is a setting in [CONFIGURATION.md](./CONFIGURATION.md); a spec that says "default 10 s" means "setting X, default 10 s".
- Durations are written `10s`, `5m`, `24h`, `30d`. Sizes in bytes unless suffixed (`GiB` = 2³⁰).
