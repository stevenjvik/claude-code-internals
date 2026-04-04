# Architecture

How the Rust reimplementation of Claude Code is structured, and how the pieces connect.

## Crate Map

```
rust/crates/
  api/              HTTP client layer
  commands/         Slash command definitions
  compat-harness/   TS manifest extraction (parity tooling)
  runtime/          Core engine (conversation loop, config, sessions, MCP, prompts, hooks)
  rusty-claude-cli/ User-facing CLI binary
  tools/            Built-in tool implementations
```

### Dependency flow

```
rusty-claude-cli
  depends on: api, runtime, tools, commands

runtime
  depends on: (no crate deps, uses std + serde)

api
  depends on: reqwest, serde

tools
  depends on: api, runtime, reqwest, serde

commands
  depends on: (minimal, mostly definitions)
```

## The Conversation Loop

This is the core of how Claude Code works. Located in `runtime/src/conversation.rs`.

```
User types input
       |
       v
ConversationRuntime::run_turn()
       |
       v
  Append user message to session
       |
       v
  +---> Build ApiRequest (system_prompt + message history)
  |         |
  |         v
  |    api_client.stream(request) --> Anthropic API (or Ollama)
  |         |
  |         v
  |    Parse response into AssistantEvent stream
  |    (TextDelta, ToolUse, Usage, MessageStop)
  |         |
  |         v
  |    Build assistant message from events
  |         |
  |         v
  |    Extract pending tool_uses from response
  |         |
  |    No tools? --> Break loop, return TurnSummary
  |         |
  |    Has tools:
  |         |
  |         v
  |    For each tool_use:
  |      1. permission_policy.authorize(tool_name, input)
  |      2. hook_runner.run_pre_tool_use(tool_name, input)
  |         - Exit code 0: Allow (capture stdout as feedback)
  |         - Exit code 2: Deny (block execution)
  |      3. tool_executor.execute(tool_name, input)
  |      4. hook_runner.run_post_tool_use(tool_name, input, output)
  |      5. Append tool_result message to session
  |         |
  +--------+ (loop back for next iteration)
```

Key design points:
- `max_iterations` defaults to `usize::MAX` (effectively unlimited)
- The loop continues as long as the model returns tool_use blocks
- Permission checks happen per-tool, before execution
- Hooks wrap tool execution with pre/post shell commands
- Usage is tracked cumulatively across all iterations

## System Prompt Assembly

Located in `runtime/src/prompt.rs`. The system prompt is a `Vec<String>` where each section is a separate string, joined with `"\n\n"` at request time.

```
Section order:
  1. Introduction (static, ~100 tokens)
  2. Output style (if configured)
  3. System section (tool execution guidance, ~100 tokens)
  4. Doing tasks section (code quality guidance, ~100 tokens)
  5. Actions section (reversibility warnings)
  6. __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__  <-- marks static/dynamic split
  7. Environment context (model, CWD, date, platform)
  8. Project context (git status + git diff)
  9. Instruction files (CLAUDE.md chain, 4K/file cap, 12K total cap)
  10. Runtime config (settings.json dump)
  11. Custom appended sections
```

The `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` marker separates cacheable static content from per-request dynamic content. This is where prompt caching could split.

### Instruction file discovery

`discover_instruction_files()` walks from the working directory up to the filesystem root, checking each directory for:
- `CLAUDE.md`
- `CLAUDE.local.md`
- `.claude/CLAUDE.md`
- `.claude/instructions.md`

Files are deduplicated by content hash, truncated at 4K chars each, 12K total.

### Git context injection

`read_git_diff()` captures both staged and unstaged diffs. `read_git_status()` captures branch status. Both are included in the project context section of every request.

## Tool System

Located in `tools/src/lib.rs`.

### Tool definitions

Each tool is a `ToolSpec`:
```rust
pub struct ToolSpec {
    pub name: &'static str,
    pub description: &'static str,
    pub input_schema: Value,          // JSON Schema
    pub required_permission: PermissionMode,
}
```

`mvp_tool_specs()` returns all 18 built-in tools. These are serialized to JSON and sent with every API request.

### Current tool inventory

| Tool | Permission | Purpose |
|------|-----------|---------|
| bash | DangerFullAccess | Execute shell commands |
| read_file | ReadOnly | Read file contents (with offset/limit) |
| write_file | WorkspaceWrite | Create or overwrite files |
| edit_file | WorkspaceWrite | String replacement edits |
| glob_search | ReadOnly | Find files by pattern |
| grep_search | ReadOnly | Search file contents (regex) |
| web_search | ReadOnly | Web search |
| web_fetch | ReadOnly | Fetch URL content |
| agent | ReadOnly | Spawn sub-agent |
| todo_write | ReadOnly | Track task progress |
| notebook_edit | WorkspaceWrite | Edit Jupyter notebooks |
| skill | ReadOnly | Load and execute skills |
| tool_search | ReadOnly | Search available tools |
| config | ReadOnly | Show configuration |
| structured_output | ReadOnly | Return structured data |
| repl | DangerFullAccess | Run REPL session |
| powershell | DangerFullAccess | PowerShell execution |
| sleep | ReadOnly | Pause execution |

### Tool execution flow

```
Model returns ToolUse block
       |
       v
ConversationRuntime extracts (id, name, input)
       |
       v
PermissionPolicy.authorize(name, input)
  - ReadOnly tools: always allowed
  - WorkspaceWrite: allowed in write modes
  - DangerFullAccess: only in danger mode
       |
       v
HookRunner.run_pre_tool_use(name, input)
  - Runs shell commands from settings.json hooks
  - Exit 0 = allow, Exit 2 = deny
       |
       v
ToolExecutor.execute(name, input)
  - Dispatches to tool-specific handler
  - Returns output string or ToolError
       |
       v
HookRunner.run_post_tool_use(name, input, output, is_error)
  - Can flip is_error to true (deny after the fact)
       |
       v
Tool result appended to session as ConversationMessage
```

## Session Persistence

Located in `runtime/src/session.rs`.

```rust
pub struct Session {
    pub version: u32,
    pub messages: Vec<ConversationMessage>,
}
```

Sessions serialize to JSON. Resume loads the file and continues the conversation. The system prompt is NOT saved in the session -- it's rebuilt from disk on resume (which means re-reading CLAUDE.md, git status, git diff).

## Context Compaction

Located in `runtime/src/compact.rs`.

Triggers when:
- Message count > `preserve_recent_messages` (default: 4)
- AND estimated tokens >= `max_estimated_tokens` (default: 10,000)

Token estimation uses a 4-chars-per-token heuristic:
```rust
fn estimate_message_tokens(message: &ConversationMessage) -> usize {
    // Approximates as text_length / 4 + 1 per block
}
```

Compaction preserves the last N messages verbatim and summarizes the rest into a system message that includes: scope, tools used, recent requests, pending work, key files, and timeline.

## Config Hierarchy

Located in `runtime/src/config.rs`.

Resolution order (later overrides earlier):
1. Default values
2. `~/.claude/settings.json` (user global)
3. `.claude/settings.json` (project)
4. `.claude/settings.local.json` (local, gitignored)
5. Environment variables
6. CLI flags

Config includes: model, permission mode, allowed tools, hook commands, MCP server definitions, and feature flags.

## MCP Integration

Located in `runtime/src/mcp_stdio.rs` (~1700 LOC) and `runtime/src/mcp_client.rs`.

Supports stdio-based MCP servers (JSON-RPC over stdin/stdout). Server lifecycle:
1. Config defines servers with `command`, `args`, `env`
2. Runtime spawns server as child process
3. Tools from MCP servers are namespaced: `mcp__{server_name}__{tool_name}`
4. Tool calls are marshaled as JSON-RPC requests
5. Responses are parsed and returned to the conversation loop

MCP servers can provide additional tools beyond the built-in set, which is key for extensibility.

## Hook System

Located in `runtime/src/hooks.rs`.

Hooks are shell commands that run before/after tool execution:

```
PreToolUse hook:
  - Receives env vars: HOOK_EVENT, HOOK_TOOL_NAME, HOOK_TOOL_INPUT
  - Receives JSON payload on stdin
  - Exit 0 = Allow (stdout captured as feedback)
  - Exit 2 = Deny (block tool execution)
  - Other = Warn (allow with warning)

PostToolUse hook:
  - Same env vars + HOOK_TOOL_OUTPUT, HOOK_TOOL_IS_ERROR
  - Exit 2 = flip is_error to true
```

Note: In this Rust port, hooks are fully implemented at the runner level but the conversation loop wiring is present. The original TypeScript CC uses this for things like permission prompts, audit logging, and tool filtering.
