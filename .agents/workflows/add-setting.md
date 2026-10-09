# Add a setting

Any threshold, interval, action, retention, limit or on/off policy is a setting, not a constant.

1. Pick a dotted key under an existing namespace (`compute.`, `storage.`, `network.`, `abuse.`, `notify.`, `metrics.`, `tasks.`, `security.`, `control_backup.`, `vm.`, `node.`, `suspend.`, `api.`, `webhooks.`, `audit.`). New namespaces need a reason in the PR.
2. Define it in `karen-core::settings` with: type/schema, built-in default, allowed scopes, whether it can be locked, whether it is pushed to agents.
3. Add a row to the catalog in `docs/spec/CONFIGURATION.md` (same PR).
4. Read it only through `settings::resolve(key, ctx)`; never read the table directly.
5. Defaults must be safe and neutral: no customer-impacting automatic action enabled by default.
6. Test: default value, override at the narrowest scope, lock at a broader scope blocks override, invalid value rejected with `setting_invalid`.
