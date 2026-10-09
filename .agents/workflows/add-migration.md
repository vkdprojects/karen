# Add a database migration

1. Update `docs/spec/DATA_MODEL.md` first (table, columns, constraints, notes).
2. One migration file per change: `migrations/<timestamp>_<snake_name>.sql`, forward-only.
3. It must run on SQLite and PostgreSQL. Use portable SQL; where it can't be, provide both variants via the `sqlx` per-database migration directories.
4. Follow the conventions: UUIDv7 `id`, `created_at`/`updated_at`, `version` on mutable tables, `account_id` on tenant tables, `CHECK` constraints for enums, indexes for every foreign key and every documented filter.
5. Never rewrite a released migration. Data backfills that take long run as tasks, not inside the migration.
6. Append-only tables (`ip_assignment`, `audit_log`, `account_identity_history`, `task_event`) get no UPDATE/DELETE paths in code except the documented ones.
7. Tests: migration from an empty DB and from the previous release's fixture DB (upgrade test).
