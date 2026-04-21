# Security Policy

## Reporting a vulnerability

Please report security issues privately by emailing **security@stevenjvik.tech**. Expect acknowledgement within 72 hours.

Do **not** open a public issue for security bugs.

## Scope

This repo contains study documentation plus a Rust reimplementation of a Claude Code-like agent harness. It is intended for learning and offline experimentation, not production use. Reportable issues include:

- Credential-handling bugs (API keys, OAuth tokens) that could leak secrets from a user's environment.
- Tool-dispatch bypasses that ignore the permission system.
- Path-traversal, command-injection, or similar issues in the Bash/Read/Write/Edit tool implementations.
- Unsafe deserialization or parsing bugs that allow arbitrary code execution via crafted input.

## Out of scope

- Behavioral differences from upstream Claude Code — this is an independent study, not a production replacement. File parity gaps as regular issues.
- Model-side content issues — those belong with the model provider.

## Supported versions

Only the latest commit on `main` is supported. The `archive/original` branch is preserved for provenance and receives no fixes.
