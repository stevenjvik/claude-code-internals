# Token Budget Analysis

Where every token goes in Claude Code, and how to optimize for local models and cost savings.

All numbers derived from reading the Rust source. The real TypeScript CC will be similar but not identical.

## Per-Request Overhead

Every API call sends the system prompt + full message history + all tool definitions. Here's the breakdown:

### System Prompt (~1000-3000 tokens baseline)

| Section | Source | Est. Tokens | Can trim? |
|---------|--------|-------------|-----------|
| Introduction + output style | `prompt.rs` static text | ~100 | Yes -- minimal for local models |
| System section (tool guidance) | `prompt.rs` `get_simple_system_section()` | ~100 | Yes -- local models may not need it |
| Doing tasks section | `prompt.rs` `get_simple_doing_tasks_section()` | ~80 | Yes |
| Actions section (reversibility) | `prompt.rs` `get_actions_section()` | ~50 | Yes |
| Environment context | `prompt.rs` `environment_section()` | ~30 | No -- model needs this |
| Git status | `prompt.rs` `read_git_status()` | 20-100 | Yes -- skip for non-git work |
| Git diff (staged + unstaged) | `prompt.rs` `read_git_diff()` | **500-1500+** | **Yes -- biggest win** |
| CLAUDE.md instruction chain | `prompt.rs` `discover_instruction_files()` | 200-3000 | Partially -- trim verbose files |
| Runtime config dump | `prompt.rs` `render_config_section()` | 200-500 | Yes -- skip for local |

**Key insight:** Git diff injection is the single largest variable cost. In an active repo with staged changes, this can be 1500+ tokens on every request. For local model sessions, disable it.

### Tool Definitions (~2000 tokens)

`mvp_tool_specs()` in `tools/lib.rs` returns 18 tools. Each tool sends:
- Name (string)
- Description (50-200 chars)
- Input schema (JSON, 200-2000 chars depending on tool)

All 18 are sent on every request regardless of whether they're needed.

| Tool | Schema size (approx chars) |
|------|---------------------------|
| bash | 250 |
| read_file | 200 |
| write_file | 200 |
| edit_file | 300 |
| glob_search | 200 |
| grep_search | 800 (many flags) |
| web_search | 200 |
| web_fetch | 300 |
| agent | 250 |
| todo_write | 200 |
| notebook_edit | 300 |
| skill | 200 |
| Others (6) | ~1000 combined |

**Total tool definition overhead:** ~4400 chars, or approximately 1100-2000 tokens.

### Message History (grows unbounded)

Each conversation turn adds:
- User message (your input)
- Assistant message (model response, often 200-2000 tokens)
- Tool results (can be large -- file contents, grep results, etc.)

A typical 10-turn session easily reaches 10K-30K tokens in history alone.

## Compaction

Defined in `compact.rs`:

```
Trigger: message count > 4 AND estimated tokens >= 10,000
Preserves: last 4 messages verbatim
Removes: everything before that
Replaces with: a summary system message
```

Token estimation: `text.len() / 4 + 1` per content block (rough heuristic).

The summary includes: scope, tools used, recent requests, pending work, key files, timeline.

## Optimization Strategies

### For local models (Ollama on your NOC)

**1. Disable git diff injection**

The biggest per-request savings. If you're not doing git-aware coding, skip it entirely. In the Rust source, this is controlled by whether `ProjectContext::discover_with_git()` or `ProjectContext::discover()` is called.

Estimated savings: **500-1500 tokens per request**

**2. Filter tool set to essentials**

For file reading and search tasks, you only need: `bash`, `read_file`, `write_file`, `edit_file`, `glob_search`, `grep_search`.

Drop: `web_search`, `web_fetch`, `agent`, `notebook_edit`, `skill`, `tool_search`, `config`, `structured_output`, `repl`, `powershell`, `sleep`, `todo_write`.

Estimated savings: **800-1000 tokens per request** (removing 12 tool definitions)

**3. Aggressive compaction**

Default compaction at 10K tokens is too late for small-context local models (8K-32K). Reduce to:
- `max_estimated_tokens: 4000` for 8K context models
- `max_estimated_tokens: 8000` for 32K context models
- `preserve_recent_messages: 2` instead of 4

**4. Trim instruction files**

If your CLAUDE.md is 3000 tokens, that's 3000 tokens on every request. For local model sessions, use a minimal CLAUDE.md or none at all.

**5. Skip config section**

The runtime config dump in the system prompt adds 200-500 tokens. Not needed for local operation.

### For API cost reduction (Anthropic billing)

**1. Model routing**

Route tasks by complexity:
- Grunt work (file reads, searches, scaffolding): Use Haiku ($1/$5 per M tokens)
- Standard coding: Use Sonnet ($3/$15 per M tokens)
- Complex reasoning: Use Opus ($15/$75 per M tokens)

The Rust source sets model once per session (`--model` flag). No mid-session routing exists. Adding a routing layer in `ConversationRuntime::run_turn()` would require classifying each turn.

**2. Prompt caching**

The `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker in `prompt.rs` separates static from dynamic content. Everything above the boundary is identical across requests and could be cached.

Anthropic's cache pricing:
- Cache write: 1.25x input price (one-time)
- Cache read: 0.1x input price (subsequent)

For a 2000-token static prefix used 20 times: saves ~90% on that portion.

**3. Session resume efficiency**

Current behavior: resume rebuilds the system prompt from scratch and replays full message history. No prompt caching between sessions. Implementing a session prompt hash to detect unchanged prompts would reduce re-send costs.

## Cost Reference

From `runtime/src/usage.rs`:

| Model | Input | Output | Cache Write | Cache Read |
|-------|-------|--------|-------------|------------|
| Haiku | $1/M | $5/M | $1.25/M | $0.10/M |
| Sonnet | $3/M | $15/M | $3.75/M | $0.30/M |
| Opus | $15/M | $75/M | $18.75/M | $1.50/M |

## Example: Minimal Local Session Budget

For an Ollama session with a 32K context Qwen model:

| Component | Tokens | Status |
|-----------|--------|--------|
| System prompt (trimmed) | ~400 | Static intro + env only |
| Tool definitions (6 tools) | ~600 | Essential set only |
| CLAUDE.md | 0 | Skipped |
| Git context | 0 | Disabled |
| Config section | 0 | Skipped |
| **Available for conversation** | **~31,000** | |

Compare to a default Claude Code session on Opus:

| Component | Tokens | Status |
|-----------|--------|--------|
| System prompt (full) | ~2500 | All sections |
| Tool definitions (18 tools) | ~2000 | Full set |
| CLAUDE.md chain | ~1500 | Multiple files |
| Git context | ~800 | Status + diff |
| Config section | ~300 | Full dump |
| **Overhead before you say anything** | **~7100** | |

That's 7K tokens of overhead on a 200K context model (3.5%). On a 32K local model, that's 22% of your context gone before the conversation starts.
