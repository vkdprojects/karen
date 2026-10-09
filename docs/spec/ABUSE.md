# Abuse handling and notifications

Every detector and every action is **off or neutral by default** and individually configurable ([CONFIGURATION.md](./CONFIGURATION.md)). An operator can, for example, enable mining detection with "notify admin only", leave DDoS detection off, and keep port 25 open.

## Port 25 (outbound SMTP)

`abuse.smtp.policy` (scopes down to `vm`):

| Value | Behaviour |
|---|---|
| `allow` (**default**) | No filtering |
| `block` | Drop new outbound TCP to ports 25 (and `abuse.smtp.extra_ports`, default `[]`) in the VM's egress chain / OVN ACL |
| `allow_on_request` | Blocked until an admin sets `allow` at the VM or account scope (customer requests it via the provider's own process) |

Enforced in the same per-VM chain/Port_Group as the firewall, but independent of whether the customer enabled their firewall.

## Detectors

Each detector has `abuse.detector.<name>.enabled` (default `false`), thresholds and an action ladder. Detectors use data the agent already collects ([METRICS.md](./METRICS.md)) unless noted.

| Detector | Signal | Default thresholds | Data source |
|---|---|---|---|
| `outbound_flood` | Sustained egress pps or bps per NIC | > `200 kpps` or > `2 Gbit/s` for `60s` | Tap counters (cheap, always available) |
| `outbound_ddos` | Many packets to few destinations, or SYN-heavy egress | Per FastNetMon rules | sFlow → FastNetMon ([ARCHITECTURE.md](../ARCHITECTURE.md#network-graphs--accounting)); detector unavailable without a flow source |
| `port_scan` | One VM contacting > N distinct destination IP:port pairs | > `1,000` distinct in `60s` | Flow source (sFlow) or sampled nflog (`abuse.port_scan.nflog_sample`, default 1/100) |
| `smtp_flood` | New TCP connections to port 25 | > `100` in `5m` | nft counter on the egress chain (only when `abuse.smtp.policy=allow`) |
| `crypto_mining` | All vCPUs > `95 %` for `6h` **and** (connections to known mining pool endpoints, if the list is configured) | `6h`, list `abuse.crypto_mining.pool_list_url` (default unset) | CPU metrics + flow/DNS data when available |
| `inbound_attack` | VM is the **target** of a DDoS (not abuse, but uses the same pipeline to notify and optionally null-route) | FastNetMon | Flow source |

Notes:
- `crypto_mining` on CPU alone has false positives (game servers, rendering, CI); its default ladder is `notify_admin` only.
- Detectors run in `karen-control` on aggregated data (agent counters, FastNetMon webhooks), not as extra work on the hypervisor, except counter reads the agent already does.

## Action ladder

Each detector has an ordered list of steps; a step runs when its condition holds, and the case escalates if the signal persists:

```json
"abuse.detector.outbound_flood.actions": [
  { "after": "0s",  "do": "notify_admin" },
  { "after": "0s",  "do": "rate_limit", "egress_mbps": 100 },
  { "after": "10m", "do": "notify_customer", "template": "abuse_outbound_flood" },
  { "after": "30m", "do": "suspend", "reason": "abuse", "mode": "isolate_network" }
]
```

| Action | Effect | Reversible |
|---|---|---|
| `notify_admin` | Event + admin channels | — |
| `notify_customer` | Event + customer channels (respecting `notify.customer.*`) | — |
| `rate_limit` | Temporary `tc`/OVN egress cap for `duration` (default until case resolved) | Yes |
| `block_port` | Egress drop for listed ports | Yes |
| `null_route` | (inbound attacks) RTBH for the VM's `/32` via the edge integration | Yes |
| `suspend` | Adds suspension reason `abuse` with `mode` `stop` or `isolate_network` ([STATE_MACHINES.md](./STATE_MACHINES.md#suspensions)) | Yes, by admin |

Default ladders ship as `notify_admin` only for every detector. Automatic customer-impacting actions exist only if the operator configures them.

## Data

| Table | Columns |
|---|---|
| `abuse_case` | `id`, `vm_id`, `account_id`, `detector` (or `manual` / `external_report`), `status` (`open`, `actioned`, `resolved`, `false_positive`), `opened_at`, `resolved_at`, `resolved_by`, `notes`, `actions_taken` (jsonb log) |
| `abuse_signal` | `case_id`, `at`, `metric`, `value`, `evidence` (jsonb: top destinations, ports, sample flows) |

- Repeated signals within `abuse.case_merge_window` (default `24h`) attach to the open case instead of opening a new one.
- External reports (abuse emails processed by the provider's own tooling) enter via `POST /admin/abuse-cases` with `detector=external_report`, evidence and the IP + timestamp; KAREN resolves the VM and customer through IP history ([DATA_MODEL.md](./DATA_MODEL.md#ip-history-legal-and-abuse-lookups)).

## Notifications

Every notification is an **event** first ([API.md](./API.md#8-events-and-webhooks)); channels decide where it goes:

| Channel | Delivery |
|---|---|
| `email` | KAREN sends through `notify.smtp` using templates (`notify.templates.<name>`, per reseller, per language) |
| `webhook` | Event POSTed to the account's webhook endpoints |
| `billing` | Event delivered to the billing module (WHMCS etc.), which emails the customer itself |
| none | `notify.customer.enabled=false`: events are still recorded and visible in the API, nothing is sent |

- `notify.customer.channels` and `notify.admin.channels` choose channels; `notify.events.<event>.customer` / `.admin` toggle individual events.
- A reseller can configure its own SMTP and templates, so customers see the reseller's brand.
- Templates: Markdown + variables (`{{vm.name}}`, `{{case.detector}}`…), rendered to text and HTML; missing variables fail template validation on save, not at send time.
