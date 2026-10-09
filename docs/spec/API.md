# API contracts

Two contracts:
1. **Public REST API** (`karen-control`): used by the web UI, the CLI, billing modules and customers' scripts. The UI has no private API: anything the UI does, a script can do.
2. **Agent protocol** (gRPC, `karen-proto`): control ↔ `karen-agent` (and later `karen-metal`, `karen-edge`).

**Contract first.** The source of truth is `api/openapi.yaml` (OpenAPI 3.1) and `crates/karen-proto/proto/**/*.proto`. They are written from this document before handlers exist. CI fails if:
- a route exists in code but not in the spec, or vice versa (route-table test);
- a response doesn't validate against its schema (contract tests, [TESTING.md](./TESTING.md#layers));
- the spec has a breaking change inside `/v1` (`oasdiff breaking`) or the protos have one inside a package version (`buf breaking`).

---

## 1. REST basics

| Topic | Rule |
|---|---|
| Base URL | `https://<panel>/api/v1` |
| Format | JSON, UTF-8. `Content-Type: application/json`. Errors: `application/problem+json` |
| Naming | Paths plural nouns, `snake_case` fields, `kebab-case` multiword paths (`/ip-pools`) |
| IDs | Prefixed public ids (`vm_01J9…`), see [DATA_MODEL.md](./DATA_MODEL.md#conventions). Opaque to clients |
| Timestamps | RFC 3339 UTC with `Z` (`2026-10-09T14:03:00Z`) |
| Sizes / rates | Integers: bytes, bits/s, IOPS. Field names carry the unit: `ram_bytes`, `rate_bps`, `read_iops` |
| Null vs absent | Response: every documented field is present; unknown value = `null`. Request (PATCH): absent = unchanged, `null` = clear |
| Unknown fields | In requests: rejected with `422 unknown_field` (catches typos). In responses: clients MUST ignore fields they don't know (we add fields without a version bump) |
| Enums | Clients MUST tolerate new enum values (treat as "unknown"). Adding a value is not breaking |
| Booleans | Never tri-state; use an enum when a third state exists |
| Request id | Every response has `Karen-Request-Id`; clients may send their own (UUID) and it is reused in logs and tasks |

## 2. Authentication and authorization

| Method | Who | How |
|---|---|---|
| API token | Scripts, billing modules, CLI | `Authorization: Bearer krn_<prefix>_<secret>` |
| Session cookie | Web UI | `karen_session` (HttpOnly, Secure, SameSite=Strict) + `X-CSRF-Token` on non-GET |

**Scopes** (tokens carry a subset; sessions get the user's role permissions):

| Scope | Allows |
|---|---|
| `vm:read`, `vm:write`, `vm:power`, `vm:console`, `vm:delete` | VMs |
| `network:read`, `network:write` | IPs, firewall, rDNS |
| `storage:read`, `storage:write` | Disks, snapshots, backups |
| `account:read`, `account:write` | Account, users, keys, tokens |
| `billing:read`, `billing:write` | Usage, plans assignment (billing modules) |
| `admin:*` (`admin:nodes`, `admin:ip-pools`, `admin:settings`, `admin:abuse`, …) | Provider/reseller admin endpoints |

**Roles → permissions:** `owner` (all in account), `admin` (all but billing/ownership), `operator` (vm:*, network:*, storage:* without delete), `member` (read + power + console), `read_only`. Provider and reseller accounts also get `admin:*` over their sub-tree.

**Acting on sub-accounts:** a provider/reseller token may send `Karen-Account: acct_…` to act inside a descendant account (billing modules create VMs for customers this way). Non-descendant ⇒ `404`.

**Not found vs forbidden:** a resource outside the caller's tenant tree is always `404 not_found`, never `403`, so IDs can't be probed. `403 forbidden` is only for resources the caller can see but lacks the scope or role for.

**Step-up:** destructive or sensitive actions (`vm:delete`, reinstall, token creation, 2FA changes, `admin:settings`) require a session with `step_up_until > now` (re-auth or 2FA within `10m`). Tokens are exempt but need the explicit scope.

## 3. Lists: pagination, filtering, sorting

```
GET /api/v1/vms?limit=50&cursor=eyJ...&status=active&label.env=prod&sort=-created_at
```

- Cursor pagination only. `limit` 1–200, default 50. Response:
  ```json
  { "data": [ ... ], "next_cursor": "eyJ..." | null }
  ```
- Cursors are opaque, expire after `24h`, and are stable under inserts.
- Filters are documented per endpoint; equality by default, `field.gte` / `field.lte` for ranges, `label.<key>=<value>` for labels.
- `sort` is a documented whitelist per endpoint; `-` prefix = descending. Default `created_at` descending.
- No total counts by default (expensive); `?count=true` adds `"total"` where documented.

## 4. Writes: idempotency and concurrency

- **Idempotency:** every `POST` that creates or triggers something accepts `Idempotency-Key: <uuid>`. The first response is stored for `24h` per account+key; a retry with the same key and same body returns the same response (status included). Same key with a different body ⇒ `422 idempotency_key_reused`. Billing modules MUST send it.
- **Optimistic concurrency:** single-resource `GET` returns `ETag: "<version>"`. `PATCH`/`PUT`/`DELETE` accept `If-Match`; on mismatch ⇒ `412 precondition_failed`. Clients that omit `If-Match` get last-writer-wins.
- `PATCH` uses JSON Merge Patch (RFC 7396).

## 5. Long-running operations

Anything that touches infrastructure returns `202 Accepted` with the task, and the resource reflects intent immediately:

```http
POST /api/v1/vms/vm_01J9.../actions/reboot
Idempotency-Key: 4b0c...

202 Accepted
Location: /api/v1/tasks/task_01J9...
{
  "task": { "id": "task_01J9...", "kind": "vm.reboot", "status": "queued", "target": "vm_01J9...",
            "created_at": "2026-10-09T14:03:00Z", "progress_pct": 0 },
  "vm":   { "id": "vm_01J9...", "operation": "reboot", "power_observed": "running", ... }
}
```

- `GET /api/v1/tasks/{id}` — status, steps (names + status), `error` when failed.
- `GET /api/v1/tasks/{id}/events` — progress log; `Accept: text/event-stream` streams it (SSE).
- `POST /api/v1/tasks/{id}/cancel` — best effort; returns the task.
- Actions are `POST /<resource>/{id}/actions/<verb>`; they are not modelled as PATCHes of state fields.

## 6. Errors

[RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) problem details plus a stable machine `code`:

```json
{
  "type": "https://karen.dev/errors/resource_busy",
  "title": "Resource is busy",
  "status": 409,
  "code": "resource_busy",
  "detail": "vm_01J9... is running task task_01J8... (vm.resize)",
  "request_id": "0f6c...",
  "errors": [ { "field": "ram_bytes", "code": "out_of_range", "message": "max 68719476736" } ],
  "meta": { "blocking_task": "task_01J8..." }
}
```

| HTTP | `code` (non-exhaustive; full list in the OpenAPI `components`) |
|---|---|
| 400 | `bad_request`, `malformed_json` |
| 401 | `unauthenticated`, `token_expired` |
| 403 | `forbidden`, `missing_scope`, `step_up_required`, `account_suspended` |
| 404 | `not_found` |
| 409 | `invalid_state`, `resource_busy`, `already_exists` |
| 412 | `precondition_failed` |
| 422 | `validation_failed`, `unknown_field`, `setting_invalid`, `idempotency_key_reused`, `quota_exceeded`, `no_capacity`, `image_incompatible` |
| 429 | `rate_limited` |
| 500 | `internal` (never leaks internals; `request_id` for support) |
| 503 | `unavailable`, `maintenance` |

`code` values are part of the contract: never renamed, only added.

## 7. Rate limits

Token bucket per token / session and per account, settings `api.rate_limit.*` (default 20 req/s burst 60 per token; 5 req/s for `actions/*`). Headers: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset` (IETF draft), and `Retry-After` on `429`.

## 8. Events and webhooks

- Events come from the outbox table ([DATA_MODEL.md](./DATA_MODEL.md#operations)), so every committed change produces its event exactly once (delivery at least once).
- Envelope:
  ```json
  { "id": "evt_01J9...", "type": "vm.power_changed", "created_at": "...",
    "account_id": "acct_...", "data": { "vm": { ... full resource ... }, "previous": { "power_observed": "running" } } }
  ```
- Delivery: `POST` to the endpoint with headers `Karen-Event-Id`, `Karen-Timestamp`, `Karen-Signature: v1=<hex HMAC-SHA256(secret, timestamp + "." + body)>`. Receivers MUST reject timestamps older than `5m`.
- Retries: exponential backoff for up to `24h` (`webhooks.retry_for`); endpoint auto-disabled after `webhooks.disable_after_failures` (default 100) consecutive failures, with an admin notification.
- Ordering is not guaranteed; use `created_at` and resource `version`.
- `GET /api/v1/events?type=vm.*&after=evt_...` lists events (retention `events.retention`, default `30d`) so integrations can catch up without webhooks.

Event catalog (v1): `vm.created`, `vm.updated`, `vm.deleted`, `vm.power_changed`, `vm.suspended`, `vm.unsuspended`, `vm.reinstalled`, `vm.resized`, `vm.migrated`, `vm.failed`, `disk.*`, `snapshot.*`, `backup.completed`, `backup.failed`, `ip.assigned`, `ip.released`, `task.succeeded`, `task.failed`, `abuse.case_opened`, `abuse.action_taken`, `traffic.quota_reached`, `node.offline`, `node.online`, `account.suspended`.

## 9. Resource catalog (v1)

Customer surface (scoped to the caller's account tree):

| Resource | Endpoints |
|---|---|
| VMs | `GET/POST /vms`, `GET/PATCH/DELETE /vms/{id}`, `POST /vms/{id}/actions/{start,stop,force-stop,reboot,reset,reinstall,resize,rescue-enter,rescue-exit,reset-password}` |
| Console | `POST /vms/{id}/console` → `{ "url": "wss://…/console/<token>", "kind": "vnc", "expires_at": … }` |
| Disks | `GET /vms/{id}/disks`, `GET/POST /disks` (volumes, `replicated`), `POST /disks/{id}/actions/{attach,detach,resize}`, `DELETE /disks/{id}` |
| Snapshots | `GET/POST /disks/{id}/snapshots`, `POST /snapshots/{id}/actions/restore`, `DELETE /snapshots/{id}` |
| Backups | `GET /vms/{id}/backups`, `POST /vms/{id}/backups`, `POST /backups/{id}/actions/restore`, `PUT /vms/{id}/backup-policy` |
| Network | `GET /vms/{id}/ips`, `POST /vms/{id}/ips` (extra IP), `DELETE /vms/{id}/ips/{ip}`, `PUT /ips/{ip}/rdns` |
| Firewall | `GET/PUT /vms/{id}/firewall` (whole policy, atomic, `If-Match`) |
| Metrics | `GET /vms/{id}/metrics?metric=cpu_usage&range=24h&step=60s` (server injects `vm_id`; [METRICS.md](./METRICS.md#query-api)) |
| Traffic | `GET /vms/{id}/traffic?month=2026-10` |
| Images, plans | `GET /images`, `GET /plans` (what the caller may use) |
| Account | `GET/PATCH /account`, `GET/POST/DELETE /account/users`, `GET/POST/DELETE /ssh-keys`, `GET/POST/DELETE /api-tokens` |
| Tasks, events | `GET /tasks`, `GET /tasks/{id}`, `GET /tasks/{id}/events`, `POST /tasks/{id}/cancel`, `GET /events` |
| Webhooks | `GET/POST /webhooks`, `GET/PATCH/DELETE /webhooks/{id}`, `POST /webhooks/{id}/actions/test` |

Admin surface (`/admin/*`, provider and resellers within their tree; `admin:*` scopes):

| Resource | Endpoints |
|---|---|
| Accounts | `GET/POST /admin/accounts`, `GET/PATCH /admin/accounts/{id}`, `POST /admin/accounts/{id}/actions/{suspend,unsuspend,close}` |
| Topology | `GET/POST /admin/regions`, `/admin/zones`, `/admin/node-groups` |
| Nodes | `GET /admin/nodes`, `GET/PATCH /admin/nodes/{id}`, `POST /admin/nodes/enrollment-tokens`, `POST /admin/nodes/{id}/actions/{drain,maintenance,resume,fence,retire,recheck}` |
| Storage pools | `GET /admin/storage-pools`, `PATCH /admin/storage-pools/{id}` |
| IP pools | `GET/POST /admin/ip-pools`, `GET/PATCH/DELETE /admin/ip-pools/{id}`, `GET /admin/ip-pools/{id}/addresses`, `POST /admin/ips/{ip}/actions/{block,unblock}` |
| IP history | `GET /admin/ip-history?address=&at=` ([DATA_MODEL.md](./DATA_MODEL.md#ip-history-legal-and-abuse-lookups)) |
| Plans, images | `GET/POST /admin/plans`, `GET/PATCH/DELETE /admin/plans/{id}`, `GET/POST /admin/images`, `PATCH/DELETE /admin/images/{id}` |
| VMs (all tenants) | `GET /admin/vms`, `POST /admin/vms/{id}/actions/{migrate,suspend,unsuspend,cpu-model-upgrade}` |
| Settings | `GET /admin/settings?scope=`, `PUT /admin/settings/{key}?scope=`, `DELETE /admin/settings/{key}?scope=`, `GET /admin/settings/effective?key=&vm=` ([CONFIGURATION.md](./CONFIGURATION.md)) |
| Abuse | `GET/POST /admin/abuse-cases`, `GET/PATCH /admin/abuse-cases/{id}`, `POST /admin/abuse-cases/{id}/actions/{apply,resolve}` |
| Audit | `GET /admin/audit-log?actor=&target=&action=&from=&to=` |
| Maintenance | `POST /admin/rollouts` (rolling upgrade), `GET /admin/rollouts/{id}`, `POST /admin/rollouts/{id}/actions/{pause,resume,abort}` |
| System | `GET /admin/system/health`, `GET /admin/system/backups`, `POST /admin/system/backups` |

Unauthenticated: `GET /healthz` (liveness), `GET /readyz` (DB + queue reachable), `GET /api/v1/openapi.json`.

## 10. Example: create a VM

```http
POST /api/v1/vms
Authorization: Bearer krn_ab12cd34_...
Karen-Account: acct_01J8...        (optional, billing modules)
Idempotency-Key: 2f1e7c0a-...

{
  "name": "game-01",
  "hostname": "game-01.example.com",
  "plan_id": "plan_01J7...",
  "image_id": "img_01J6...",
  "region": "br-sp",
  "ssh_key_ids": ["key_01J5..."],
  "password": null,
  "user_data": "#cloud-config\n...",
  "labels": { "env": "prod" },
  "placement": { "node_group_id": null, "anti_affinity_label": "game-cluster" },
  "firewall": null,
  "start": true
}
```

```http
202 Accepted
Location: /api/v1/vms/vm_01J9...
{
  "vm": {
    "id": "vm_01J9...", "account_id": "acct_01J8...", "name": "game-01", "hostname": "game-01.example.com",
    "lifecycle": "creating", "power_desired": "running", "power_observed": "unknown", "operation": "create",
    "suspensions": [], "rescue": false,
    "plan_id": "plan_01J7...", "vcpus": 4, "cpu_class": "dedicated", "ram_bytes": 17179869184,
    "storage_class": "local", "region": "br-sp", "node": null,
    "ips": [ { "address": "203.0.113.7/32", "family": "v4", "status": "reserved", "rdns": null },
             { "address": "2001:db8:10:7::/64", "family": "v6", "status": "reserved", "rdns": null } ],
    "image_id": "img_01J6...", "boot_mode": "uefi", "labels": { "env": "prod" },
    "version": 1, "created_at": "2026-10-09T14:03:00Z", "updated_at": "2026-10-09T14:03:00Z"
  },
  "task": { "id": "task_01J9...", "kind": "vm.create", "status": "queued", "progress_pct": 0 }
}
```

Validation (all checked before enqueuing, each with its own `code`): plan visible to the account; image compatible with plan (`min_disk_bytes`, boot mode); quota (`quota_exceeded`); capacity in region with the plan's storage class and redundancy requirement (`no_capacity`); hostname format; user_data ≤ `64 KiB`; either SSH keys or password or neither (then a random password is returned once in the task output).

`node` is `null` for customers unless `vm.expose_node_to_customer` (default `false`); admins always see it.

## 11. Versioning and deprecation

- Within `/v1`: only additive changes (new endpoints, fields, enum values, error codes).
- Breaking changes go to `/v2`; `/v1` stays at least `12` months after `/v2` GA.
- Deprecated endpoints/fields send `Deprecation` and `Sunset` headers (RFC 8594) and are marked `deprecated: true` in the spec.
- Pre-1.0 releases may break `/v1` but MUST list breaks in the changelog.

## 12. CLI

`karen` CLI is a thin client of this API: every command maps to one endpoint (`karen vm create` → `POST /vms`), supports `--output json`, and waits for tasks with `--wait`. No CLI-only behaviour.

---

## Agent protocol

gRPC over HTTP/2, TLS 1.3 with mutual auth ([SECURITY_MODEL.md](./SECURITY_MODEL.md#agent-pki)). The **agent dials control**; control never connects to nodes.

```proto
package karen.agent.v1;

service Enrollment {
  // Only RPC allowed without a client certificate (token-authenticated).
  rpc Enroll(EnrollRequest) returns (EnrollResponse); // token, CSR, host facts → signed cert, CA chain, node_id
  rpc RenewCertificate(RenewRequest) returns (RenewResponse); // mTLS, new CSR
}

service AgentChannel {
  // One long-lived bidirectional stream per agent.
  rpc Connect(stream AgentMessage) returns (stream ControlMessage);
}

message AgentMessage {
  oneof msg {
    Hello hello = 1;              // agent_version, proto_version, node_id, in_flight steps, component versions
    Heartbeat heartbeat = 2;      // every 5s: load, free RAM, pool usage summary
    Inventory inventory = 3;      // every reconcile interval and on request: domains, power, LVs, taps, chains
    StepAck step_ack = 4;         // persisted to journal
    StepProgress step_progress = 5;
    StepResult step_result = 6;   // ok + output | error{code, retryable, message}
    TrafficDeltas traffic = 7;    // 5-min per-NIC deltas, sequence-numbered, replayed after reconnect
    NodeEvent event = 8;          // vm crashed, guest shutdown, disk error, thin pool threshold
    ConsoleData console = 9;      // multiplexed console bytes
  }
}

message ControlMessage {
  oneof msg {
    Welcome welcome = 1;          // accepted proto version, settings snapshot
    StepAssign step_assign = 2;   // task_id, step, epoch, idempotency_key, input (typed oneof per step), deadline
    StepCancel step_cancel = 3;
    InventoryRequest inventory_request = 4;
    SettingsUpdate settings = 5;  // resolved settings for this node
    ConsoleOpen console_open = 6; // session id, vm id, kind
    ConsoleData console = 7;
    ConsoleClose console_close = 8;
    TrafficAck traffic_ack = 9;   // highest sequence stored
  }
}
```

Rules:
- Step inputs are **typed messages** (`CreateDisk`, `WriteImage`, `ConfigureNetwork`, `DefineDomain`, …), never shell commands or free-form strings. The agent has no "run command" capability.
- Compatibility: control supports agents one minor version behind (`N-1`); `Hello.proto_version` is negotiated; unknown fields are ignored (proto3 rules). Removing or renumbering fields is forbidden within `v1` (`buf breaking`).
- Backpressure: the stream uses gRPC flow control; console data has its own per-session window so a busy console can't delay step messages.
- Reconnect: exponential backoff `1s → 30s` with jitter; on reconnect `Hello` lists in-flight steps and the last traffic sequence acknowledged.
- Heartbeat loss for `node.offline_after` marks the node `offline` ([STATE_MACHINES.md](./STATE_MACHINES.md#node)).
