---
name: claude-code-hindsight-setup
category: software-development
tags: [claude-code, hindsight, memory, hooks, plugin, setup]
description: Install and configure the hindsight-memory plugin for Claude Code end-to-end — register SessionStart/UserPromptSubmit/Stop/SessionEnd hooks WITHOUT overwriting pre-existing hooks (the official `setup_hooks.py` script has a destructive merge bug for keys that already contain user-installed hooks like `herdr-agent-state.sh`), point the plugin at the correct Hindsight API host, validate round-trip before declaring success.
---

# Claude Code ↔ Hindsight Plugin Setup

## When to use

- User reports Claude Code "doesn't remember things across sessions" or "I configured Hindsight before but it's not working".
- User asks to set up Claude Code long-term memory with Hindsight.
- After `claude plugin install hindsight-memory` (the official installer does not register hooks automatically — confirmed June 2026).
- After `claude plugin update hindsight-memory` (re-runs the same broken install path).
- After any Hermes-side change that moves the Hindsight API host (e.g. local daemon started/stopped, LAN IP rotated, container rebuilt).

## Why this skill exists (root cause of the workflow)

The official `setup_hooks.py` shipped with the `hindsight-memory@hindsight` plugin v0.7.1 (cache path `~/.claude/plugins/cache/hindsight/hindsight-memory/0.7.1/scripts/setup_hooks.py`) has a **destructive merge bug**:

```python
# from setup_hooks.py, line 102
merged = {**existing_hooks, **new_hooks}
```

If the user already has a `SessionStart` hook (e.g. `herdr-agent-state.sh`, `Warp`, or any other agent-state injector), the official script silently replaces the entire `SessionStart` array with the plugin's own entry. Restoring from a backup is required.

The official script then exits with `Restart Claude Code for changes to take effect.` and never re-runs, so the data loss is permanent unless you backed up `~/.claude/settings.json` beforehand.

## Critical pitfalls

### 1. **Always backup `~/.claude/settings.json` before running `setup_hooks.py`**

```bash
cp ~/.claude/settings.json ~/.claude/settings.json.bak.$(date +%Y%m%d_%H%M%S)
```

If the official script already clobbered `SessionStart` (or any other event your custom hook used), restore first and re-merge manually.

### 2. **The official script also injects `CLAUDE_PLUGIN_ROOT` into `settings.json > env`**

This is a side-effect of running it. If you restored from backup, you may need to re-add `CLAUDE_PLUGIN_ROOT` env manually. Setting it lets the hook scripts resolve their plugin root regardless of how the user's shell wrapper sets `CLAUDE_PLUGIN_ROOT` for the MCP server.

### 3. **The official hook does NOT support multiple entries per event out-of-the-box**

The plugin's own docstring implies 1 hook per event, but Claude Code's `~/.claude/settings.json` schema supports an *array* of `{"matcher": "*", "hooks": [...]}` objects per event key. Multiple arrays are concatenated at runtime — confirmed by Hermes setup on 2026-07-20. So the safe pattern when concatenating is append-to-list, NOT replace-key.

### 4. **Smoke test hooks BEFORE restarting Claude Code**

Run each hook script directly with synthetic JSON stdin:

```bash
# SessionStart
echo '{"session_id":"smoke","transcript_path":"/dev/null","cwd":"/tmp","hook_event_name":"SessionStart"}' \
  | python3 ~/.claude/plugins/cache/hindsight/hindsight-memory/<VERSION>/scripts/session_start.py

# UserPromptSubmit (auto-recall)
echo '{"session_id":"smoke","transcript_path":"/dev/null","cwd":"/tmp","hook_event_name":"UserPromptSubmit","prompt":"smoke test query"}' \
  | python3 ~/.claude/plugins/cache/hindsight/hindsight-memory/<VERSION>/scripts/recall.py

# Stop + SessionEnd cannot be easily smoke-tested without an actual stop event — trust the static schema check.
```

`recall.py` should exit 0 and emit JSON to stdout of shape:
```json
{"hookSpecificOutput": {"hookEventName": "UserPromptSubmit", "additionalContext": "<hindsight_memories>...</hindsight_memories>"}}
```

If it returns empty stdout and exit 0, the call timed out, the bank is unreachable, or the recall API rejected the query.

### 5. **Always verify `~/.hindsight/claude-code.json` points at a live host**

The plugin has two connection modes:
1. **External API** — `hindsightApiUrl` set to a URL; the plugin does NOT auto-start any daemon. Requires the user to provide a working `hindsight-api`/`hindsight-embed` server elsewhere.
2. **Local daemon** — `hindsightApiUrl: ""` and `apiPort: 9077`; the plugin auto-starts `hindsight-embed` via `uvx`.

Default in `~/.hindsight/claude-code.json` (if missing) is **local daemon**. If the user has `hindsightApiUrl` set to `127.0.0.1:8888` but no local daemon is running, recall/retain will fail silently (timeout 10s per call). Always:

```bash
# Verify host is reachable
curl -fsS --max-time 5 "$HINDSIGHT_API/health"
curl -fsS --max-time 5 "$HINDSIGHT_API/version"

# If local daemon was expected, it lives at 127.0.0.1:<apiPort>
#   or you can run it explicitly:
uvx hindsight-embed@latest daemon --profile claude-code start
```

### 6. **The MCP server registers separately from the hooks**

The plugin v0.7.1 declares `mcpServers.hindsight` in its bundled `.mcp.json`. After install, `claude mcp list` should show `hindsight` or `hindsight-local` (alias name) connected. This is **independent** from the hooks — the MCP server requires a working `python venv` with `mcp` package, while the hooks only need `python3` + stdlib + `urllib`.

A common failure mode: MCP shows connected (so `agent_knowledge_*` tools work), but hooks are not registered (because installer doesn't auto-register hooks), so `auto-recall` does NOT happen on every prompt. Always register hooks explicitly with this skill's procedure.

### 7. **The bank ID is resolved from `bankId` in `~/.hindsight/claude-code.json`**

Default is `"claude_code"`. The Hermes user typically sets this to `"hermes"` so Claude Code shares the same bank as Hermes — see `~/.hindsight/claude-code.json`. If unset, Claude Code writes into a separate bank that doesn't talk to Hermes.

### 8. **The `retainEveryNTurns: 10` setting means recall fires on every prompt but retain fires only every 10 turns**

The `~/.claude/plugins/cache/hindsight/hindsight-memory/0.7.1/settings.json` ships with `retainEveryNTurns: 10`. Lower numbers mean more LLM extraction calls per session (cost). Higher numbers lose granularity. For most users, leave at 10. To override per-bank, add the key to `~/.hindsight/claude-code.json` (`retainEveryNTurns: 3`).

## End-to-end procedure (Hermes-tuned)

This is the recipe validated 2026-07-20 on /Users/alvarogonzalez.

### Step 1 — Pre-flight checks

```bash
# 1. Plugin installed?
grep "hindsight-memory" ~/.claude/settings.json   # expected: enabledPlugins.hindsight-memory@hindsight: true

# 2. Hindsight API host alive?
HOST=$(python3 -c "import json; print(json.load(open('/Users/alvarogonzalez/.hindsight/claude-code.json'))['hindsightApiUrl'])")
echo "Configured host: $HOST"
curl -fsS --max-time 5 "$HOST/health" || echo "DEAD — fix the api_url first"
curl -fsS --max-time 5 "$HOST/version"

# 3. Bank reachable?
curl -fsS --max-time 5 "$HOST/v1/default/banks/hermes/stats"

# 4. Existing hooks in settings.json (so we know what NOT to clobber)
python3 -c "import json; print(json.dumps(json.load(open('/Users/alvarogonzalez/.claude/settings.json'))['hooks'], indent=2))"
```

If the host is unreachable, STOP HERE and ask the user which Hindsight endpoint to use:
- Existing local daemon? Start it (Step 0).
- Remote LAN host? Update `api_url` and verify with `nc -zv <ip> <port>` first; agent-side `ssh` to LAN boxes is unreliable.
- New `hindsight-embed` local? Run `mkdir -p ~/.hindsight && echo '{"apiPort": 9077}' > ~/.hindsight/claude-code.json && export OPENAI_API_KEY=...`

### Step 2 — Backup both files

```bash
TS=$(date +%Y%m%d_%H%M%S)
cp ~/.claude/settings.json ~/.claude/settings.json.bak.$TS
cp ~/.hindsight/claude-code.json ~/.hindsight/claude-code.json.bak.$TS
echo "Backups at $TS"
```

### Step 3 — Edit `~/.hindsight/claude-code.json` to point at the live host

If `curl $HOST/health` failed at localhost, change `hindsightApiUrl` to the LAN host:

```bash
python3 - <<'PY'
import json, os
p = os.path.expanduser("~/.hindsight/claude-code.json")
c = json.load(open(p))
c["hindsightApiUrl"] = "http://192.168.1.110:8888"   # actual live host — verify first
c.setdefault("bankId", "hermes")
json.dump(c, open(p, "w"), indent=2)
PY
```

Common fields to ensure are set:
- `"bankId": "hermes"` (so it shares with Hermes; default is `"claude_code"`)
- `"autoRecall": true`, `"autoRetain": true`
- `"recallTypes": ["world","experience","observation"]` (wider than default `["observation"]` for richer context)
- `"recallBudget": "mid"` (default) or `"high"` for slower but deeper recall
- `"requestTimeoutSeconds": 30` if the LAN-hosted Hindsight takes >10s under contention

### Step 4 — Run the official hook installer (it WILL clobber SessionStart)

```bash
python3 ~/.claude/plugins/cache/hindsight/hindsight-memory/<VERSION>/scripts/setup_hooks.py
```

Where `<VERSION>` is the highest semver directory under `~/.claude/plugins/cache/hindsight/hindsight-memory/`.

### Step 5 — Restore from backup and re-merge correctly

The official script wrote a partial `settings.json` that **replaced** any pre-existing `SessionStart` hook. Restore from backup and re-merge with array concatenation:

```python
import json
settings = json.load(open("/Users/alvarogonzalez/.claude/settings.json.bak.<TS>"))
PLUGIN_ROOT = "/Users/alvarogonzalez/.claude/plugins/cache/hindsight/hindsight-memory/<VERSION>"

plugin_hooks = {
    "SessionStart":  [{"hooks":[{"type":"command","command":f'python3 "{PLUGIN_ROOT}/scripts/session_start.py"',"timeout":5}]}],
    "UserPromptSubmit":[{"hooks":[{"type":"command","command":f'python3 "{PLUGIN_ROOT}/scripts/recall.py"',"timeout":12}]}],
    "Stop":          [{"hooks":[{"type":"command","command":f'python3 "{PLUGIN_ROOT}/scripts/retain.py"',"timeout":15,"async":True}]}],
    "SessionEnd":    [{"hooks":[{"type":"command","command":f'python3 "{PLUGIN_ROOT}/scripts/session_end.py"',"timeout":10}]}],
}

existing = settings.setdefault("hooks", {})
for event, entries in plugin_hooks.items():
    lst = existing.setdefault(event, [])
    sig = json.dumps(entries)
    if not any(json.dumps(e) == sig for e in lst):
        lst.extend(entries)

settings.setdefault("env", {})["CLAUDE_PLUGIN_ROOT"] = PLUGIN_ROOT
json.dump(settings, open("/Users/alvarogonzalez/.claude/settings.json", "w"), indent=2)
```

Verify with:

```bash
python3 -c "
import json
s = json.load(open('/Users/alvarogonzalez/.claude/settings.json'))
# All 4 plugin events should now have entries
for e in ['SessionStart','UserPromptSubmit','Stop','SessionEnd']:
    n = len(s['hooks'].get(e, []))
    assert n >= 1, f'missing {e}'
    # Pre-existing hooks should still be there — verify by grep
    if e == 'SessionStart':
        assert any('herdr-agent-state' in json.dumps(h) for h in s['hooks'][e]), 'herdr-agent-state lost!'
print('OK — all 4 plugin hook events present, pre-existing hooks preserved')
print('CLAUDE_PLUGIN_ROOT:', s['env'].get('CLAUDE_PLUGIN_ROOT'))
"
```

### Step 6 — Smoke test the hooks

```bash
echo '{"session_id":"smoke","transcript_path":"/dev/null","cwd":"/tmp","hook_event_name":"UserPromptSubmit","prompt":"configurar Hindsight en Claude Code"}' \
  | python3 ~/.claude/plugins/cache/hindsight/hindsight-memory/<VERSION>/scripts/recall.py
```

Expected: JSON to stdout of shape `{"hookSpecificOutput":{"hookEventName":"UserPromptSubmit","additionalContext":"<hindsight_memories>\n- ...\n</hindsight_memories>"}}`.

If empty stdout, recall failed — check:
1. `curl $HINDSIGHT_API/health` is healthy
2. bank `hermes` exists
3. `~/.hindsight/claude-code.json` matches the live host

### Step 7 — Restart Claude Code and observe

```bash
# Kill any active claude code sessions, then start fresh
claude
```

Send a meaningful prompt (e.g. "configurar Hindsight en Claude Code"). In the response, expect Claude to reference facts from prior conversations, e.g. "Voy a apuntar a 192.168.1.110:8888" or "Como Spids / alvarogonzalez, ya tienes configurado X". If the response is generic and shows no awareness of prior context, hooks are not firing — re-check Step 5.

## Operational notes

- **Plugin cache version path is dynamic**: never hardcode `0.7.1`. Use `ls ~/.claude/plugins/cache/hindsight/hindsight-memory | sort -V | tail -1` to discover.
- **`scripts/setup_hooks.py` is the installer's responsibility to fix**. A community PR exists (2026-06) to make it append-to-list. Until that's merged, this skill exists.
- **Test before telling the user it's done**. The smoke test in Step 6 is the bare minimum. A user-facing smoke test is: restart Claude Code, send "Recuerdas algo sobre Hindsight?" and check if Claude talks about the swap to MiniMax-M3.
- **If `~/.hindsight/claude-code.json` is missing**, the plugin uses defaults — bank `claude_code` (NOT `hermes`), local daemon mode, recall type `observation` only. The plugin will auto-create this file on first hook fire, but you should pre-create it pointing at the right host for the user's setup.

## Cross-links

- Parent skill: `hermes-hindsight-memory-provider` — server-side Hindsight operations (LaunchAgents, API LLM swpas, debug).
- Companion plugin patterns: search for `claude plugin` and `claude mcp` in adjacent skills.
