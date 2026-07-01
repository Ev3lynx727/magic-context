# Magic Context - Breakdown & Configuration

## Installation

### Proper Install - Global Scope (recommended)

```bash
opencode plugin @cortexkit/opencode-magic-context --global
```

This:
1. Runs `npm install --ignore-scripts` into the opencode cache
2. Adds the plugin to `opencode.jsonc` `plugin[]` list
3. Registers server and tui targets

### Package Cache Locations

| Location | Purpose |
|----------|---------|
| `~/.cache/opencode/packages/@cortexkit/` | Proper install by `opencode plugin` |
| `~/.cache/opencode/node_modules/@cortexkit/` | Manual `npm install` (not canonical) |
| `~/.npm/_npx/` | Cached from npx (not persistent) |

### Backend Data

| Location | Contents |
|----------|----------|
| `~/.local/share/cortexkit/magic-context/` | SQLite DB, model limits, ONNX model, RPC sockets |
| `~/.local/share/opencode/opencode.db` | OpenCode conversation database |

### Verify

```bash
grep 'magic-context' ~/.config/opencode/opencode.jsonc
ls ~/.cache/opencode/packages/@cortexkit/
ls ~/.local/share/cortexkit/magic-context/
```

---

## Plugin Schema

### opencode.jsonc Entry

```jsonc
{
  "plugin": [
    "@cortexkit/opencode-magic-context"
  ]
}
```

No `@latest` tag - `opencode plugin` resolves and strips it on install.

### Agent Model Binding

All agents reference `"model": "opencode/big-pickle"` (160K context).

Available `opencode/` model IDs:
- `big-pickle` - 160,000 context
- `deepseek-v4-flash` - 1,000,000 context
- `deepseek-v4-flash-free` - 200,000 context
- `deepseek-v4-pro` - 1,000,000 context
- `claude-sonnet-4` - 1,000,000 context
- `claude-opus-4-7` - 1,000,000 context
- `gpt-5.4` - 922,000 context
- `gemini-3.5-flash` - 1,048,576 context

### Compaction Settings

```jsonc
{
  "compaction": {
    "auto": false,
    "prune": false,
    "reserved": 20000
  }
}
```

When Magic Context is active, compaction is managed by the plugin, not OpenCode's native engine.

---

## Backend Architecture

### Data Directory: `~/.local/share/cortexkit/magic-context/` (~96 MB)

| File | Purpose |
|------|---------|
| `context.db` (+SHM, +WAL) | SQLite - session state, compacted history |
| `model-context-limits-opencode.json` | Per-model context limits (2,400+ entries) |
| `models/Xenova/all-MiniLM-L6-v2/` | ONNX embedding model |
| `rpc/` | RPC communication ports |
| `last_announced_version` | Version announcement tracker |

### Model Context Limits

The `model-context-limits-opencode.json` caches context window limits populated from OpenCode's `config.providers()` SDK endpoint. Magic Context reads the live resolved config (never reads `models.json` directly) to avoid torn-read issues.

```jsonc
{
  "opencode/big-pickle": 160000,
  "opencode/deepseek-v4-flash": 1000000,
  // ... 2400+ entries
}
```

### Context Resolution Flow

```
Plugin startup -> warm apiCache from last-known-good file
OpenCode SDK -> config.providers() -> merged models
  (live + compiled-in + auth-plugin caps)
Magic Context -> consumes merged result
Result bounded to [20k, 3M] range
```

### Process Architecture

```
OpenCode (main process)
  +-- MCP servers (git, mempalace, hf-mcp, etc.)
  +-- Magic Context (child process)
        +-- SQLite (context.db)
        +-- ONNX model (all-MiniLM-L6-v2)
        +-- RPC socket
```

---

## Troubleshooting

### "Model not found: opencode/small-pickle. Did you mean: big-pickle?"

**Cause:** OpenCode's runtime tries to resolve `opencode/small-pickle` which doesn't exist in the model catalog. Appears during context compaction cycles.

**Solution:** Harmless - no functional impact. `big-pickle` (160K context) is the correct model already configured for all agents. This is an upstream model alias issue - cannot fix via config. Ignore or report to OpenCode.

### Duplicate Plugin Entries

**Solution:** Check both config files:

```bash
grep 'magic-context' ~/.config/opencode/opencode.jsonc

# Check if local scope files exist and what's in them:
cat ~/.opencode/opencode.json 2>/dev/null
cat ~/.opencode/tui.json 2>/dev/null
```

**To remove only the plugin entry (not the whole file):**

```bash
# Targeted removal — removes just the magic-context entry from the plugin array,
# never deletes the entire file even if it contains other config.
if [ -f ~/.opencode/opencode.json ]; then
  # Always filter first, then del only if nothing remains.
jq '(.plugin |= map(select(. != "@cortexkit/opencode-magic-context"))) | if (.plugin | length) == 0 then del(.plugin) else . end' \
  ~/.opencode/opencode.json > /tmp/opencode_tmp.json && mv /tmp/opencode_tmp.json ~/.opencode/opencode.json
  echo "Removed magic-context entry from plugin array"
fi
```

Keep plugin in only one scope.

### Package Missing (Plugin Registered But Not Installed)

**Symptom:** Plugin listed in config but OpenCode shows loading errors.

**Solution:** Re-install with the proper command:

```bash
opencode plugin @cortexkit/opencode-magic-context --global
```

This installs to the canonical cache path (`~/.cache/opencode/packages/@cortexkit/`). Only use manual `npm install` inside `~/.cache/opencode` as a last resort — it installs to `node_modules/` which the package cache table above notes is **not canonical** and may not be discovered by OpenCode.

---

## FAQ

### Q: Why does Magic Context use its own SQLite database?

The plugin needs persistent session state for context management that survives OpenCode restarts. It cannot rely on OpenCode's in-memory state. The database at `context.db` stores compacted history, configuration overrides, and session bookkeeping.

### Q: Where does the ONNX model come from?

`all-MiniLM-L6-v2` is a sentence-transformer embedding model downloaded from Hugging Face (Xenova/onnx conversion). It's used by Magic Context's sidebar for semantic compression features.

### Q: Does Magic Context read `~/.cache/opencode/models.json`?

No. Magic Context explicitly does NOT read `models.json` due to torn-read issues. It consumes the merged, live-resolved model config from OpenCode's `config.providers()` API endpoint, which already combines the live cache, compiled-in snapshot, opencode.json overrides, and auth-plugin caps.

### Q: What happens if both local and global scope have the plugin?

OpenCode loads both entries, potentially causing duplicate initialization. Remove one scope (preferably the local `~/.opencode/` files) to keep only the global scope.

### Q: Is the opencode/small-pickle error breaking anything?

No. It's a console warning during compaction with zero functional impact. The model routing still works correctly with the configured `opencode/big-pickle`.
