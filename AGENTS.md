# AGENTS.md

Instructions for any AI coding agent (or human) working on KAREN. Tool-agnostic: tool-specific files (`CLAUDE.md`, `.cursor/rules`, `.github/copilot-instructions.md`…) MUST only point here, never duplicate or override it. Longer playbooks live in [`.agents/`](./.agents/).

## What this is

KAREN is a self-hosted VPS / bare-metal / web-hosting control panel in Rust: `karen-control` (API, scheduler, task queue, UI) and `karen-agent` (one per hypervisor). Status: pre-alpha, design phase.

## Sources of truth, in priority order

1. [`docs/spec/`](./docs/spec/) — how the system behaves (data model, state machines, tasks, API, security, config, metrics, abuse, operations, testing).
2. [`docs/ARCHITECTURE.md`](./docs/ARCHITECTURE.md) — technology decisions and invariants.
3. [`ROADMAP.md`](./ROADMAP.md) — what to build next and when a milestone is done.
4. [`docs/research/`](./docs/research/) — background only; never implement from research directly.

If the code you need conflicts with a spec, **change the spec in the same PR** and say why. Never silently diverge. If a spec is ambiguous, pick the safest reading, implement it, and record the clarification in the spec.

## Invariants (never break)

1. Control owns state; agents hold only rebuildable caches.
2. Infrastructure changes happen only in tasks; every step is idempotent ([TASKS.md](./docs/spec/TASKS.md)).
3. Anti-spoof is always on in every network mode.
4. Tenant isolation: queries go through `AuthzContext`; out-of-tree resources return `404`.
5. Policy is configuration ([CONFIGURATION.md](./docs/spec/CONFIGURATION.md)); never hard-code a threshold, action or retention.
6. No AGPL/GPL/proprietary code linked into KAREN crates; external tools only over a protocol.
7. The agent executes typed steps only; there is no generic "run command".
8. Secrets never reach logs, audit diffs, task payloads, errors or metric labels.

## Commands

```bash
cargo fmt --all --check
cargo clippy --all-targets --all-features -- -D warnings
cargo test --workspace                      # unit, property, contract, component
cargo deny check
cargo xtask system-test                     # needs /dev/kvm (nested-KVM host)
cargo xtask openapi-check                   # spec ↔ routes, oasdiff breaking
buf breaking crates/karen-proto --against '.git#branch=main'
```

## Rust rules

- Edition 2024, stable toolchain pinned in `rust-toolchain.toml`.
- No `unwrap()` / `expect()` / `panic!` outside tests and `main` startup.
- Errors: `thiserror` enums in library crates with stable codes mapping to API `code`s; `anyhow` only in binaries' top level.
- Async: `tokio`; no blocking calls in async context (`spawn_blocking` for libvirt/LVM calls that block).
- DB: `sqlx` with compile-time checked queries; one forward-only migration per schema change.
- Logging: `tracing` with `request_id`, `task_id`, `node_id` fields; never log secrets (`Secret<T>`).
- No shelling out where a library or netlink/ioctl API exists. When a CLI is unavoidable (e.g. `lvcreate`), arguments come only from validated typed values, never from user strings, and go through one wrapper module per tool.
- Public types serialized in the API derive their schema from the OpenAPI spec, not the other way round.

## Definition of done

A change is done when **all** hold:
- [ ] The behaviour matches the spec (spec updated in the same PR if it changed).
- [ ] Tests for the change exist at the right layers ([TESTING.md](./docs/spec/TESTING.md)), including error paths; fakes updated if a real backend changed.
- [ ] `fmt`, `clippy -D warnings`, tests, `cargo deny` pass locally.
- [ ] API changes: `api/openapi.yaml` updated, no breaking change in `/v1`, examples updated.
- [ ] Proto changes: no breaking change (`buf breaking`).
- [ ] New settings added to the catalog in [CONFIGURATION.md](./docs/spec/CONFIGURATION.md) with defaults.
- [ ] New metrics added to [METRICS.md](./docs/spec/METRICS.md).
- [ ] Roadmap checkbox ticked only when the milestone item's acceptance criteria pass.

## Never

- Skip, delete or weaken a test to make CI green.
- Change a stable API error `code`, field name or proto field number.
- Commit secrets, keys, tokens or real customer data (fixtures use RFC 5737 / RFC 3849 addresses and `example.com`).
- Add a dependency without checking its license (`cargo deny`) and maintenance status.
- Add a new external service dependency (broker, cache, DB) without an ARCHITECTURE.md decision.

## Workflow

See [`.agents/workflows/`](./.agents/workflows/): implementing a roadmap item, changing the API, adding a setting, adding a migration.

Commits: [Conventional Commits](https://www.conventionalcommits.org) (`feat:`, `fix:`, `docs:`, `test:`, `refactor:`). One concern per PR.
