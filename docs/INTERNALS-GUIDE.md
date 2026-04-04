# Internals Guide

A reading order for understanding Claude Code's architecture from the Rust source. Each entry explains what the file does, key functions to study, and what you'll learn.

All paths are relative to `rust/crates/`.

## Reading Order

### 1. runtime/src/conversation.rs (~300 LOC)

**What it does:** The agentic conversation loop. This is the beating heart of Claude Code.

**Key types:**
- `ConversationRuntime<C, T>` -- generic over API client and tool executor
- `ApiRequest` -- system prompt + message history sent to the model
- `AssistantEvent` -- stream events (TextDelta, ToolUse, Usage, MessageStop)
- `TurnSummary` -- what a turn produced (messages, tool results, iterations, usage)

**Key function: `run_turn()`** (line ~153)

This is the main loop. Read it carefully:
1. Pushes user input as a message
2. Loops: send to API, parse response, extract tool_uses
3. If no tools requested, break and return
4. For each tool: check permissions -> run pre-hook -> execute -> run post-hook
5. Append tool results, loop again

**What you'll learn:** How CC decides when to keep calling tools vs stop. How hooks intercept tool execution. How permission checks gate dangerous operations.

**Token implications:** The system prompt and entire message history are sent on every iteration of this loop. A 5-iteration turn (5 tool calls) means 5x the system prompt cost.

---

### 2. runtime/src/prompt.rs (~280 LOC of builder, plus helpers)

**What it does:** Builds the system prompt from static sections, project context, instruction files, and runtime config.

**Key types:**
- `SystemPromptBuilder` -- builder pattern for assembling prompt sections
- `ProjectContext` -- CWD, date, git status, git diff, instruction files
- `ContextFile` -- path + content of a discovered instruction file

**Key functions:**
- `SystemPromptBuilder::build()` (line ~134) -- assembles all sections into Vec<String>
- `discover_instruction_files()` (line ~192) -- walks directory tree for CLAUDE.md files
- `read_git_diff()` (line ~245) -- captures staged + unstaged diffs
- `render_instruction_files()` -- truncates at 4K/file, 12K total

**Constants to know:**
- `MAX_INSTRUCTION_FILE_CHARS = 4_000`
- `MAX_TOTAL_INSTRUCTION_CHARS = 12_000`
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` -- marks where static ends and dynamic begins

**What you'll learn:** Exactly what goes into every API request before your conversation even starts. Where tokens get burned. What you can trim for local models.

---

### 3. tools/src/lib.rs (~4200 LOC)

**What it does:** Defines all 18 built-in tools and their execution logic.

**Key types:**
- `ToolSpec` -- name, description, input_schema (JSON), required_permission
- `ToolRegistry` -- manifest of available tools
- `ToolSource` -- Base vs Conditional (for tool filtering)

**Key function: `mvp_tool_specs()`** (line ~60)

Returns the full tool list. Each tool is a struct with:
- A name the model sees
- A description the model reads to decide when to use it
- A JSON schema the model uses to format its input
- A permission level required to execute

**Tool execution** is dispatched by name matching in the CLI layer (`main.rs`), not in this file. This file defines what tools exist; execution happens elsewhere.

**What you'll learn:** How CC tells the model what tools are available. Why all tools are sent every request (no filtering by task type). Where to add new tools.

---

### 4. runtime/src/compact.rs (~180 LOC)

**What it does:** Context window management. Detects when the session is too large and compresses it.

**Key functions:**
- `should_compact()` (line ~32) -- checks message count > 4 AND tokens >= 10K
- `estimate_session_tokens()` (line ~27) -- sum of all message token estimates
- `compact_session()` (line ~75) -- removes old messages, generates summary
- `get_compact_continuation_message()` (line ~53) -- the message that replaces removed history

**Key defaults:**
- `preserve_recent_messages: 4`
- `max_estimated_tokens: 10_000`
- Token estimation: `text.len() / 4 + 1` (chars / 4 heuristic)

**What you'll learn:** How CC manages long conversations. Why context windows fill up. How to tune compaction for smaller local models.

---

### 5. runtime/src/config.rs (~1058 LOC)

**What it does:** Configuration loading and merging. Resolves settings from multiple sources into a single runtime config.

**Key types:**
- `ConfigLoader` -- loads and merges config from all sources
- `RuntimeConfig` -- the resolved configuration
- `RuntimeHookConfig` -- hook command definitions (pre/post tool use)
- `RuntimeFeatureConfig` -- feature flags and hook config combined

**Resolution order:**
1. Default values
2. `~/.claude/settings.json` (user global)
3. `.claude/settings.json` (project)
4. `.claude/settings.local.json` (local, gitignored)
5. Environment variables
6. CLI flags

**What you'll learn:** How CC's config hierarchy works. Where to put local overrides. How hook commands get from settings.json into the conversation loop.

---

### 6. runtime/src/hooks.rs (~170 LOC)

**What it does:** Executes pre/post tool use hooks as shell commands.

**Key types:**
- `HookEvent` -- PreToolUse or PostToolUse
- `HookRunner` -- configured with hook commands, executes them
- `HookRunResult` -- allowed/denied + captured messages

**Key function: `run_commands()`** (line ~95)

For each hook command:
1. Builds JSON payload with tool name, input, output, error status
2. Sets environment variables (HOOK_EVENT, HOOK_TOOL_NAME, etc.)
3. Pipes payload to command stdin
4. Interprets exit code: 0=allow, 2=deny, other=warn
5. Captures stdout as feedback messages

**What you'll learn:** How hookify and custom hooks work. How to build pre-flight checks that prevent wasteful tool calls. How to add audit logging without modifying the core loop.

---

### 7. runtime/src/mcp_stdio.rs (~1700 LOC)

**What it does:** JSON-RPC protocol implementation for MCP (Model Context Protocol) over stdio.

**Key concepts:**
- MCP servers are child processes that communicate via stdin/stdout
- Each server exposes tools via `tools/list` RPC call
- Tool execution uses `tools/call` RPC call
- Tools are namespaced: `mcp__{server_name}__{tool_name}`

**What you'll learn:** How CC extends its tool set with external servers. How to build an MCP server that provides custom tools. This is the extensibility layer.

**Note:** This is the most complex single file in the codebase. Skim on first read, return when you need to understand MCP integration.

---

### 8. runtime/src/session.rs (~430 LOC)

**What it does:** Session persistence -- saving and loading conversation state.

**Key types:**
- `Session` -- version + message list
- `ConversationMessage` -- role, blocks (text, tool_use, tool_result), metadata
- `ContentBlock` -- the individual pieces of a message

**Key functions:**
- `Session::save_to_path()` / `load_from_path()` -- JSON serialization
- `ConversationMessage::user_text()`, `tool_result()`, etc. -- constructors

**What you'll learn:** What gets persisted between sessions. What doesn't (system prompt, config). How to inspect session files for debugging.

---

### 9. api/src/client.rs + api/src/sse.rs (~1500 LOC combined)

**What it does:** HTTP client for the Anthropic Messages API. Handles request construction, SSE streaming, and OAuth.

**Key types:**
- `AnthropicClient` -- the HTTP client (reqwest-based)
- `MessageRequest` -- API request payload
- `StreamEvent` -- SSE events from the API
- `ToolDefinition` -- how tools are serialized for the API

**What you'll learn:** How CC communicates with Anthropic. The exact request format. How streaming works. How to point it at a different API endpoint (Ollama, OpenRouter).

---

### 10. rusty-claude-cli/src/main.rs (~3494 LOC)

**What it does:** Everything user-facing. REPL loop, streaming display, slash command dispatch, tool call rendering, session management, argument parsing.

**Warning:** This is a monolith. The TUI-ENHANCEMENT-PLAN.md proposes breaking it into smaller modules (Phase 0). Read it for understanding, not as a model of good architecture.

**Key sections:**
- Argument parsing and initialization (~lines 1-200)
- Tool execution dispatch (~lines 200-600) -- maps tool names to handlers
- REPL loop (~lines 600-1200) -- input, streaming, display
- Slash command handlers (~lines 1200-2000)
- Streaming display and formatting (~lines 2000-3000)
- Session management (~lines 3000+)

**What you'll learn:** How the high-level pieces connect. How streaming SSE events become terminal output. How slash commands work. Why this needs refactoring.

## What to Study After

Once you've read through the above, look at:

- `PARITY.md` -- what's missing vs the real TypeScript CC
- `rust/TUI-ENHANCEMENT-PLAN.md` -- the proposed refactoring roadmap
- `runtime/src/usage.rs` -- how costs are tracked and displayed
- `runtime/src/permissions.rs` -- the permission policy engine
- `rusty-claude-cli/src/render.rs` -- markdown-to-ANSI terminal rendering
