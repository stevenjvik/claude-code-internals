# Claude Code Internals

A reference for Claude Code's agent harness architecture, based on a clean-room Rust reimplementation. Built for infrastructure operators who want to understand how Claude Code works under the hood, optimize token usage, and run offline workloads against local models.

> Original Rust port by [instructkr/claw-code](https://github.com/instructkr/claw-code). Upstream state preserved on the `archive/original` branch.

## Why this exists

Claude Code is a powerful agent harness, but it's a black box. Understanding its internals helps you:

- Know where your tokens are going (system prompt, tool definitions, context growth)
- Run a local CC-like agent against Ollama for low-stakes ops tasks
- Extend the tool system with MCP servers for your own infrastructure
- Make informed decisions about model routing, context management, and session persistence

## Architecture

The Rust implementation lives in `rust/crates/` as 6 focused crates:

```
rust/crates/
  api/              Anthropic HTTP client, SSE streaming, OAuth
  commands/         Slash command registry (/status, /compact, /model, etc.)
  compat-harness/   TS manifest extraction (for parity auditing)
  runtime/          The brain: conversation loop, config, sessions, MCP, prompts, hooks
  rusty-claude-cli/ The face: REPL, CLI, streaming display, tool call rendering
  tools/            The hands: 18 built-in tools (bash, read, write, edit, grep, glob, web, etc.)
```

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full crate map, conversation loop flow, and tool dispatch path.

## Study Guides

| Document | What it covers |
|----------|---------------|
| [docs/INTERNALS-GUIDE.md](docs/INTERNALS-GUIDE.md) | Annotated reading order through the Rust source. Start here. |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | 6-crate map, conversation loop, tool dispatch, session lifecycle |
| [docs/TOKEN-BUDGET.md](docs/TOKEN-BUDGET.md) | Where every token goes. System prompt breakdown, tool overhead, compaction |
| [docs/OFFLINE-OPS.md](docs/OFFLINE-OPS.md) | Running against Ollama for local/offline operation |
| [PARITY.md](PARITY.md) | Gap analysis: what this Rust port covers vs the real TypeScript CC |

## Token Budget (summary)

Biggest token drains in Claude Code, from the source:

| Component | Tokens per request | Source file |
|-----------|-------------------|-------------|
| System prompt (static sections) | ~300 | `runtime/prompt.rs` |
| Git diff injection | 500-1500 | `runtime/prompt.rs` (`read_git_diff`) |
| CLAUDE.md instruction chain | up to ~3000 | `runtime/prompt.rs` (4K/file, 12K total cap) |
| Tool definitions (all 18) | ~2000 | `tools/lib.rs` (`mvp_tool_specs`) |
| Config section | 200-500 | `runtime/prompt.rs` (`render_config_section`) |
| **Total baseline per request** | **~3000-7300** | |

Compaction triggers at 10K tokens, preserves last 4 messages. See [docs/TOKEN-BUDGET.md](docs/TOKEN-BUDGET.md) for optimization strategies.

## Build and Run

```bash
cd rust/
cargo build --release

# Interactive REPL
./target/release/claw

# One-shot prompt
./target/release/claw prompt "explain this codebase"

# With specific model
./target/release/claw --model sonnet prompt "fix the bug"
```

Set credentials:
```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# Or point to Ollama/OpenRouter:
export ANTHROPIC_BASE_URL="http://localhost:11434/v1"
```

## Parity Status

This is NOT a complete replacement for Claude Code. It covers the core architecture but has gaps. See [PARITY.md](PARITY.md) for the full breakdown. Key gaps: plugins (missing), hooks (parsed not executed), CLI breadth (18 vs 40+ commands), skills (local only).

## License

See repository root. This is a reference fork, not a production tool.
