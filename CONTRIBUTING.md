# Contributing

## Before you start
- Read [AGENTS.md](./AGENTS.md) (rules for humans and AI agents alike) and the specs in [docs/spec/](./docs/spec/).
- Check [ROADMAP.md](./ROADMAP.md) and open issues. Big changes: open an issue first.
- One PR = one concern.

## Dev setup
```bash
rustup toolchain install stable
cargo build
cargo test
```
VM-level tests need Linux with KVM (`/dev/kvm`) and libvirt. On macOS, use a nested-KVM Linux VM.

## Checks (CI runs these)
```bash
cargo fmt --all --check
cargo clippy --all-targets -- -D warnings
cargo test --all
```

## Conventions
- Commits: [Conventional Commits](https://www.conventionalcommits.org) (`feat:`, `fix:`, `docs:` …).
- Agent tasks must be idempotent: running the same task twice yields the same state.
- New storage/network/billing backends go in `plugins/`, behind the trait in `karen-core`.
- Public API changes update the OpenAPI spec in the same PR.

## License
By contributing you agree your work is licensed under MIT.
