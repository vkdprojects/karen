# Configuration

**Rule: if an operator could reasonably want it different, it is a setting.** KAREN is open source and run by different providers with different policies. Code MUST NOT hard-code policy (abuse actions, notifications, overcommit, port blocks, retention, thresholds). Code MAY hard-code safety invariants (anti-spoof always on, tenant isolation, mTLS).

## Two kinds of configuration

| Kind | Where | Examples | Change |
|---|---|---|---|
| **Bootstrap** | TOML file + env overrides (`KAREN_<SECTION>_<KEY>`) | Listen address, DB URL, data dir, KEK path, log level | Restart |
| **Runtime settings** | `setting` table, edited via API/UI/CLI | Everything in the catalog below | Live (no restart), audited |

Bootstrap files: `/etc/karen/control.toml`, `/etc/karen/agent.toml`. Unknown keys are an error (fail fast on typos).

## Scopes and resolution

Settings are resolved from the most specific scope that sets them:

```
vm → plan → account → reseller → node → node_group → zone → region → global → built-in default
```

- Each setting declares which scopes it accepts (e.g. `compute.ram_overcommit_ratio`: `global`, `region`, `zone`, `node_group`, `node`; never `account`).
- A scope may **lock** a setting (`locked = true`): narrower scopes can't override it. Example: provider sets `abuse.smtp.policy=block` at `global` and locks it, so resellers can't open port 25.
- Resolution is computed by one function in `karen-core` (`settings::resolve(key, ctx)`), never ad hoc.
- The effective value and the scope it came from are visible in the API (`GET /v1/admin/settings/effective?...`), so "why is this VM throttled?" is answerable.

```rust
pub struct Setting {
    pub key: SettingKey,          // e.g. "abuse.smtp.policy"
    pub scope: Scope,             // Global | Region(Id) | Zone(Id) | NodeGroup(Id) | Node(Id)
                                  // | Reseller(Id) | Account(Id) | Plan(Id) | Vm(Id)
    pub value: serde_json::Value, // validated against the key's schema on write
    pub locked: bool,
    pub updated_by: ActorId,
    pub updated_at: Timestamp,
}
```

- Every key has a typed schema (type, range, enum) in `karen-core`; writes that don't validate are rejected with `422 setting_invalid`.
- Every write creates an `audit_log` row with old and new value.
- Agents receive the resolved settings that concern them in their task payloads or through `SettingsUpdate` messages ([API.md](./API.md#agent-protocol)); they never read the DB.

## Settings catalog

Defaults are built-in values. "Scopes" lists the narrowest scope allowed (all broader scopes are allowed too).

### Compute ([COMPUTE.md](./COMPUTE.md))
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `compute.cpu_overcommit_ratio` | float ≥ 1.0 | `1.0` | node |
| `compute.ram_overcommit_ratio` | float ≥ 1.0 | `1.0` | node |
| `compute.cpu_mode` | `baseline` \| `host-passthrough` | `baseline` | plan |
| `compute.balloon.enabled` | bool | `true` | vm |
| `compute.balloon.free_page_reporting` | bool | `true` | vm |
| `compute.balloon.stats_period` | duration | `10s` | node |
| `compute.balloon.auto_reclaim` | bool | `false` | node |
| `compute.ksm.enabled` | bool | `false` | node |
| `compute.ksm.pages_to_scan` | int | `100` | node |
| `compute.nested_virt` | bool | `false` | plan |
| `compute.host_reserved_ram` | bytes | `4 GiB` | node |
| `compute.host_reserved_cpus` | int | `2` | node |
| `compute.prealloc` | bool | `true` for `dedicated`, else `false` | plan |
| `compute.balloon.pressure_threshold` | PSI avg10 % | `10` | node |
| `compute.balloon.headroom` | percent | `20` | plan |
| `compute.balloon.min_pct` | percent of plan RAM | `50` | plan |
| `compute.ksm.sleep_ms` | int | `20` | node |
| `compute.ksm.opt_out` | bool | `true` for `dedicated`, else `false` | plan |
| `compute.hotplug.enabled` | bool | `true` if the image declares support | plan |
| `compute.hotplug.max_ram_factor` | float | `2.0` | plan |

### Storage
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `storage.local_layout` | `jbod`\|`raid0`\|`raid1`\|`raid10`\|`raid5` | detected | node |
| `storage.max_thin_overcommit` | float ≥ 1.0 | `1.0` | node |
| `storage.thin_alert_pct` | 1–99 | `80` | node |
| `storage.thin_stop_placement_pct` | 1–99 | `85` | node |
| `storage.require_disk_redundancy` | bool | `false` | plan |

### Network
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `network.ip_cooldown` | duration | `7d` | region |
| `network.ip_history_retention` | duration | `365d` | global |
| `network.firewall.max_rules` | int | `64` | plan |
| `network.default_rate_limit_mbps` | int \| null | `null` | plan |

### Abuse ([ABUSE.md](./ABUSE.md))
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `abuse.smtp.policy` | `allow`\|`block`\|`allow_on_request` | `allow` | vm |
| `abuse.detector.<name>.enabled` | bool | `false` | plan |
| `abuse.detector.<name>.thresholds` | object | per detector | plan |
| `abuse.detector.<name>.actions` | ladder | `notify_admin` only | plan |
| `abuse.smtp.extra_ports` | int[] | `[]` | vm |
| `abuse.case_merge_window` | duration | `24h` | global |
| `abuse.port_scan.nflog_sample` | 1/N | `100` | node |
| `abuse.crypto_mining.pool_list_url` | url \| null | `null` | global |

### Notifications
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `notify.customer.enabled` | bool | `true` | account |
| `notify.customer.channels` | set of `email`\|`webhook`\|`billing` | `["email"]` | account |
| `notify.admin.channels` | set of `email`\|`webhook` | `["email"]` | global |
| `notify.events.<event>.customer` | bool | per event | account |
| `notify.smtp` | object (host, port, user, secret ref, from) | unset (email disabled) | reseller |
| `notify.templates.<name>` | template per language | built-in | reseller |

When a billing system (WHMCS, Blesta…) owns customer communication, set `notify.customer.channels=["billing"]` or `["webhook"]`: KAREN emits the event and sends nothing itself.

### Metrics ([METRICS.md](./METRICS.md))
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `metrics.vm_interval` | duration ≥ 5s | `10s` | node |
| `metrics.node_interval` | duration ≥ 5s | `10s` | node |
| `metrics.smart_interval` | duration | `5m` | node |
| `metrics.guest_agent.enabled` | bool | `false` | plan |
| `metrics.retention` | duration | `90d` | global |
| `metrics.customer_visible` | set of metric groups | `["cpu","memory","disk","network"]` | plan |
| `metrics.remote_write_url` | url \| null | `null` | global |
| `metrics.per_vcpu` | bool | `false` | plan |
| `metrics.guest_agent.interval` | duration | `60s` | plan |
| `metrics.guest_agent.max_mounts` | int | `8` | plan |
| `metrics.bmc.enabled` | bool | `false` | node |
| `metrics.longterm.enabled` | bool | `false` | global |
| `metrics.longterm.retention` | duration | `2y` | global |

### Tasks ([TASKS.md](./TASKS.md))
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `tasks.max_attempts` | int | `5` | global |
| `tasks.node_concurrency.<kind>` | int | create `4`, migrate `2`, backup `2`, other `8` | node |
| `tasks.reconcile_interval` | duration | `60s` | global |
| `tasks.global_concurrency.migrate` | int | `10` | global |
| `tasks.account_concurrency` | int | `10` | account |
| `reconcile.destroy_orphans` | bool | `false` | node |

### Security ([SECURITY_MODEL.md](./SECURITY_MODEL.md))
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `security.agent_cert_ttl` | duration | `30d` | global |
| `security.enrollment_token_ttl` | duration | `1h` | global |
| `security.console_token_ttl` | duration | `60s` | global |
| `security.console_max_session` | duration | `8h` | plan |
| `security.require_2fa` | `off`\|`admins`\|`all` | `admins` | reseller |
| `security.session_idle_timeout` | duration | `12h` | reseller |
| `security.login.max_attempts` | int | `10` per `15m` | global |
| `security.login.lockout` | duration (doubles per lockout) | `5m` | global |

### Backups of the control database ([OPERATIONS.md](./OPERATIONS.md))
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `control_backup.enabled` | bool | `true` | global |
| `control_backup.target` | object (local path or S3 + secret ref) | local `/var/lib/karen/backups` | global |
| `control_backup.interval` | duration | `15m` (SQLite) / continuous WAL (Postgres) | global |
| `control_backup.retention` | duration | `30d` | global |

### VM lifecycle and nodes ([STATE_MACHINES.md](./STATE_MACHINES.md))
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `vm.stop_timeout` | duration | `120s` | vm |
| `vm.restart_on_crash` | bool | `true` | vm |
| `vm.expose_node_to_customer` | bool | `false` | reseller |
| `suspend.<reason>.action` | `stop`\|`isolate_network`\|`none` | `stop` | account |
| `suspend.allow_console` | bool | `true` | account |
| `node.offline_after` | duration | `30s` | node_group |
| `node.fence_after` | duration | `120s` | node_group |

### API, events, audit ([API.md](./API.md), [SECURITY_MODEL.md](./SECURITY_MODEL.md))
| Key | Type | Default | Narrowest scope |
|---|---|---|---|
| `api.rate_limit.per_token` | rate, burst | `20/s`, `60` | account |
| `api.rate_limit.actions` | rate | `5/s` | account |
| `webhooks.retry_for` | duration | `24h` | global |
| `webhooks.disable_after_failures` | int | `100` | global |
| `events.retention` | duration | `30d` | global |
| `audit.retention` | duration | `365d` | global |
| `audit.export` | object (syslog / S3) \| null | `null` | global |

New settings MUST be added to this catalog in the same PR that introduces them.
