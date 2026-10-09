# Change the API (REST or agent protocol)

REST (`api/openapi.yaml`):
1. Edit `docs/spec/API.md` if the change affects a convention or the resource catalog.
2. Edit `api/openapi.yaml` **before** code: path, parameters, request/response schemas, every error `code` the endpoint can return, an example.
3. Inside `/v1` only additive changes are allowed: new endpoints, optional request fields, new response fields, new enum values, new error codes. Renames, removals, type changes, new required request fields → `/v2` (discuss first).
4. Implement the handler: validate → authorize (`authz::check`) → write intent + enqueue task in one transaction → `202` with task for infrastructure changes.
5. Add contract tests and make sure the tenant isolation suite covers the new path (it is generated; add an allow-list entry with a reason only if it truly can't apply).
6. Run `cargo xtask openapi-check`.

Agent protocol (`crates/karen-proto`):
1. New step = new typed message in the `StepAssign.input` oneof; never a generic string/command field.
2. Never renumber or remove fields; deprecate instead. Run `buf breaking`.
3. Control must keep working with agents one minor version older; gate new steps on `Hello.proto_version`.
4. Implement the step in the agent idempotently (check real state first) and in the matching fake in `karen-testkit`.
