# Implement a roadmap item

1. **Find the item** in `ROADMAP.md` and its milestone's "Done when" criteria.
2. **Read the specs it touches** in `docs/spec/` (at least DATA_MODEL, STATE_MACHINES, TASKS, API for anything user-visible) and the relevant ARCHITECTURE section.
3. **List gaps.** If the spec doesn't answer something you need (a field, a state, an error code, a default), write the answer into the spec first, in the same PR, and keep it consistent with the rest of `docs/spec/`.
4. **Write tests first** at the layers TESTING.md requires: unit/property for logic, contract for API, component with fakes for workflows, system test if it touches real infrastructure.
5. **Implement** in the crate that owns the concern (`karen-core` for domain logic and traits, `karen-control` for API and orchestration, `karen-agent` for node steps, `plugins/*` for backends).
6. **Update** the OpenAPI spec, protos, settings catalog and metrics catalog as needed.
7. **Run** all commands in AGENTS.md → Commands.
8. **Check the Definition of done** in AGENTS.md; tick the roadmap box only if the item's acceptance criteria pass.
9. **PR description:** what changed, which spec sections changed and why, how it was tested.
