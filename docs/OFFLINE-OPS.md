# Offline Operations

Running the claw binary against local models for offline and low-cost operation.

## Prerequisites

- Rust toolchain (build from `rust/`)
- Ollama installed and running (or any OpenAI-compatible API)
- A local model pulled (e.g., `ollama pull qwen3.5:32b`)

## Build

```bash
cd rust/
cargo build --release
# Binary at: rust/target/release/claw
```

## Configuration

### Point to Ollama

```bash
export ANTHROPIC_BASE_URL="http://localhost:11434/v1"
export ANTHROPIC_API_KEY="ollama"  # Ollama doesn't check this, but claw requires it
```

### Point to a remote Ollama instance

If you run Ollama on a different node (e.g., <OLLAMA_NODE> at <OLLAMA_NODE_IP>):

```bash
export ANTHROPIC_BASE_URL="http://<OLLAMA_NODE_IP>:11434/v1"
export ANTHROPIC_API_KEY="ollama"
```

### Point to OpenRouter

```bash
export ANTHROPIC_BASE_URL="https://openrouter.ai/api/v1"
export ANTHROPIC_API_KEY="sk-or-v1-your-key-here"
```

## Model Selection

Use the `--model` flag to specify the model:

```bash
# Ollama model names
./target/release/claw --model qwen3.5:32b
./target/release/claw --model gemma4:27b
./target/release/claw --model deepseek-coder-v2:16b

# OpenRouter model names
./target/release/claw --model qwen/qwen-3.5-coder:free
```

## Context Window Considerations

Local models typically have smaller context windows than Opus (200K). Adjust accordingly:

### For 8K context models

Strip everything non-essential:
- Use a minimal or no CLAUDE.md
- Disable git context (don't run from a dirty git repo, or set a clean working dir)
- Filter tools to essentials only (bash, read_file, write_file, grep_search)
- Set aggressive compaction (4K threshold, preserve 2 messages)

### For 32K context models

Standard operation is viable with some trimming:
- Keep a short CLAUDE.md (~500 chars max)
- Git status is fine, but git diff may be too large
- Full tool set works but consider filtering for simple tasks
- Compaction at 8K threshold

### For 128K+ context models (Qwen 3 coder, etc.)

Mostly standard operation. The main limit is inference speed, not context.

## Ollama Context Configuration

Ollama defaults may not give you the full advertised context window. To increase:

```bash
# Create a custom model with larger context
ollama create qwen3.5-64k -f - <<'EOF'
FROM qwen3.5:32b
PARAMETER num_ctx 65536
EOF

# Run with the custom model
./target/release/claw --model qwen3.5-64k
```

VRAM requirements scale with context size. Monitor with:
```bash
nvidia-smi  # GPU
ollama ps    # Active models and memory usage
```

## NOC Use Cases for Local Models

### Low-stakes, high-volume tasks

- Reading and summarizing log files
- Grepping through configs to find relevant settings
- Generating repetitive scaffolding (systemd units, nginx configs)
- Organizing files and writing manifest docs
- Parsing structured data (JSON, YAML, CSV)

### Fallback when Claude is down

If Anthropic has an outage (check status.anthropic.com), switch to local:
```bash
export ANTHROPIC_BASE_URL="http://localhost:11434/v1"
./target/release/claw --model qwen3.5:32b
```

### Private/air-gapped work

For sensitive infrastructure work where you don't want data leaving the network:
```bash
# Everything stays on your LAN
export ANTHROPIC_BASE_URL="http://<OLLAMA_NODE_IP>:11434/v1"
```

### Session limit workaround

Hit your API rate limit? Switch to local for the next few hours:
```bash
# Continue working with a local model while limits reset
./target/release/claw --model gemma4:27b
```

## Hybrid Routing Strategy

For a NOC with multiple compute tiers:

| Task type | Route to | Why |
|-----------|----------|-----|
| File reads, searches, scaffolding | Ollama CPU (<OLLAMA_NODE>) | Free, good enough |
| Code review, config generation | Ollama GPU (workstation) | Fast local, no cost |
| Complex reasoning, multi-file edits | Opus via API | Worth the tokens |
| Agentic loops with many tool calls | Sonnet via API | Good balance of cost/capability |

The claw binary doesn't support mid-session model switching. Use separate sessions for different tiers, or modify `ConversationRuntime::run_turn()` to add routing logic.

## MCP Servers for Local Tool Providers

You can extend claw's capabilities with local MCP servers. In `.claude/settings.json`:

```json
{
  "mcp": {
    "servers": {
      "local_tools": {
        "type": "stdio",
        "command": "/path/to/your/mcp-server",
        "args": ["--config", "/path/to/config.json"],
        "env": {}
      }
    }
  }
}
```

Tools from MCP servers appear as `mcp__local_tools__tool_name` and are sent to the model alongside built-in tools.

Potential NOC MCP servers:
- Log query server (reads from journald, syslog)
- Infrastructure state server (reads from Proxmox API)
- Monitoring server (reads from Grafana/Prometheus)
- Vault server (reads from Obsidian via SSH)

These would let a local model query your infrastructure directly without burning API tokens.

## Monitoring Token Usage

Claw tracks usage in `runtime/src/usage.rs`. Use the `/cost` and `/status` slash commands in the REPL to see:
- Input/output token counts
- Cache read/write tokens
- Estimated cost (based on model pricing)
- Session duration

For local models, costs will show as $0 but token counts are still tracked for context window awareness.
