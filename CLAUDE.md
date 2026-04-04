# CLAUDE.md

This repository is a study of Claude Code's agent harness internals, maintained as a reference for NOC operators.

## Stack

- Language: Rust (2021 edition)
- Binary: `claw` (built from `rust/crates/rusty-claude-cli/`)
- Workspace: `rust/` contains all active code

## Verification

From the `rust/` directory:
```bash
cargo fmt --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
```

## Repository structure

- `rust/crates/` — 6 Rust crates (api, commands, compat-harness, runtime, tools, rusty-claude-cli)
- `docs/` — Study guides (architecture, token budget, offline ops, internals guide)
- `PARITY.md` — Gap analysis between this Rust port and the real TypeScript CC
- `archive/original` branch — Preserved upstream fork state before this conversion

## Conventions

- Conventional commits (`feat:`, `fix:`, `chore:`, `docs:`)
- No force-push to main
- Unsafe code is forbidden (`#![forbid(unsafe_code)]` in workspace)
- Clippy pedantic warnings enabled
