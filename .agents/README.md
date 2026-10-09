# .agents

Tool-agnostic playbooks for AI coding agents. The entry point is [`../AGENTS.md`](../AGENTS.md); read it first.

| Path | Use when |
|---|---|
| [workflows/implement-roadmap-item.md](./workflows/implement-roadmap-item.md) | Picking up any item from `ROADMAP.md` |
| [workflows/change-api.md](./workflows/change-api.md) | Adding or changing a REST endpoint, field, error code, or a proto message |
| [workflows/add-setting.md](./workflows/add-setting.md) | Introducing any tunable value |
| [workflows/add-migration.md](./workflows/add-migration.md) | Changing the database schema |

Tool-specific config (e.g. `.claude/`, `.cursor/`) may exist for tool features, but rules live only here and in `AGENTS.md`.
