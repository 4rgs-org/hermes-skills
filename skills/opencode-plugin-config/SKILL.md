---
name: opencode-plugin-config
description: Configure plugins for the OpenCode CLI (`@vectorize-io/opencode-*`, `@another/opencode-*`, etc.) including the `plugin` array in `~/.config/opencode/opencode.json`, plugin-specific persistent config files (`~/.<plugin-name>/<name>.json`), the inline-options vs separate-config-file precedence, the empirical verification recipe (init count + plugin's own "initialized" log line), and the load-order gotcha between `opencode.json` and `opencode.jsonc`. Use when the user says "configurar plugin X en opencode", "install <plugin> in opencode", "add <plugin> to opencode", hands you a `@vendor-io/opencode-*` package name, or when a plugin was declared in `opencode.jsonc` and silently disappeared after the `.jsonc` was deleted.
---

# Configure a plugin for OpenCode CLI

OpenCode loads plugins via two complementary surfaces: a top-level `plugin` array in its JSON config, and an optional plugin-specific persistent config file at `~/.<plugin-name>/<name>.json`. Both surfaces matter, and they conflict in non-obvious ways when the user has been editing config ad-hoc.

This skill is the class-level recipe. Specific plugins (Hindsight, telemetry, etc.) live in `references/` so the umbrella stays focused on the *mechanism*, not the per-plugin wiring.

---

## 1. Discover the operative config file

OpenCode v1.17.x reads config files in this load order, merging:

1. **Remote** (`.well-known/opencode`) — organizational defaults
2. **Global** (`~/.config/opencode/opencode.json`) — user preferences
3. **Custom config** (`OPENCODE_CONFIG` env var)
4. **Project** (`opencode.json` in cwd)
5. **`.opencode` directories**
6. **Inline config** (`OPENCODE_CONFIG_CONTENT` env var)
7. **Managed** (`/Library/Application Support/opencode/` on macOS)

The Hindsight session (2026-07-13) revealed a critical detail the official docs underplay: **when both `opencode.json` AND `opencode.jsonc` exist in `~/.config/opencode/`, OpenCode loads BOTH in order.** The log shows:

```
INFO loading path=/Users/alvarogonzalez/.config/opencode/config.json
INFO loading path=/Users/alvarogonzalez/.config/opencode/opencode.json
INFO loading path=/Users/alvarogonzalez/.config/opencode/opencode.jsonc   ← also loads
INFO init count=42                                                       ← plugin present
```

Implications:

- **Don't assume `.jsonc` is inert.** If you remove a `plugin` declaration from `.json` thinking it lives in `.jsonc`, you'll silently lose the plugin (`init count` drops from 42 to 19 with no error).
- **Corruption in `.jsonc` crashes the server.** A malformed `mcp` entry or stray `disabled_providers` key in `.jsonc` produces a generic `Error: Unexpected error / ServeError` whose root cause is buried three log lines above (you have to look for `loading path=...opencode.jsonc` immediately before the crash).
- **The plugin MUST be declared in whichever file is actually operative.** Read both, identify which file carries the live `model: "..."` and live `plugin: [...]` blocks, and patch that one. Leave the other untouched unless the user confirms it's stale.

```bash
ls -la ~/.config/opencode/opencode.json*
```

## 2. Declare the plugin in `opencode.json`

The `plugin` key is a top-level array. The simplest entry is a string (npm package name); OpenCode auto-installs the plugin at startup:

```json
{
  "plugin": ["@vectorize-io/opencode-hindsight"]
}
```

For plugin-specific options, pass an array form: `[<package-name>, <options-object>]`. The options object's keys are plugin-specific (the Hindsight plugin accepts `hindsightApiUrl`, `hindsightApiToken`, `bankId`, `autoRecall`, `autoRetain`, `recallBudget`, `retainEveryNTurns`, `debug`, etc.). Two examples:

```json
{
  "plugin": [
    "@vectorize-io/opencode-hindsight"
  ]
}
```

```json
{
  "plugin": [
    ["@vectorize-io/opencode-hindsight", {
      "hindsightApiUrl": "https://api.hindsight.vectorize.io",
      "hindsightApiToken": "your-api-key",
      "bankId": "my-project",
      "autoRecall": true,
      "autoRetain": true,
      "recallBudget": "mid"
    }]
  ]
}
```

`OpenCode auto-installs plugins from the array on startup — no separate `npm install` step.` If the package isn't installed, OpenCode fetches and caches it under `~/.cache/opencode/packages/@<vendor>/<plugin>@<version>/`.

### Surgical edit pattern

1. Read the current file: `read_file ~/.config/opencode/opencode.json`.
2. Add or update the top-level `"plugin": [...]` array. Preserve existing `provider`, `model`, `mcp` blocks — don't nest plugin entries inside them.
3. Verify JSON parses: `python3 -c "import json; json.load(open('<path>'))"`.
4. Verify the file actually loads (see §5) — `ls -la` is not verification.

## 3. The plugin-specific persistent config file (the recommended path)

Many plugins (Hindsight, observability tools, etc.) ship a default config location of the form `~/.<plugin-name>/<name>.json`. This is the **recommended place for settings you want to persist across all OpenCode projects**:

```
~/.hindsight/opencode.json         ← Hindsight plugin reads this
~/.telemetry/opencode.json        ← hypothetical
~/.myplugin/opencode.json         ← hypothetical
```

Why use it instead of inline plugin options:

- Survives between projects (no per-project override pollution).
- Doesn't bloat `opencode.json` with non-shared config.
- The plugin reads it via its own bootstrap path — independent of OpenCode's config merge.

### Precedence (later wins)

For the Hindsight plugin (representative), the documented precedence is:

```
defaults  <  ~/.<plugin>/<name>.json  <  inline plugin options in opencode.json  <  env vars
```

So an inline `"hindsightApiUrl"` in `opencode.json` overrides the value in `~/.hindsight/opencode.json`, and `HINDSIGHT_API_URL` env var overrides both. **Use env vars for secrets** (API tokens); **use the persistent config file** for non-secret defaults that apply everywhere.

### When the plugin needs env vars

Set env vars in your shell profile, in a project-local `.env`, or via `OPENCODE_CONFIG_DIR`-style mechanisms. Env vars are the standard mechanism and survive reloads. The plugin reads them at startup.

## 4. When to use inline plugin options vs the persistent config file

| Use case | Where it goes |
|---|---|
| Bank/project ID, default URL, debug flag | `~/.<plugin>/<name>.json` (persistent) |
| API token, per-environment endpoint | env var (`<PLUGIN>_API_TOKEN`, `<PLUGIN>_API_URL`) |
| Per-project override | inline plugin options in project's `opencode.json` |
| Plugin enabled/disabled (boolean) | top-level `"plugin": [...]` array presence |

Inline options in `opencode.json` are per-OpenCode-instance and visible to anyone reading the file. The persistent config file at `~/.<plugin>/<name>.json` is private to your user, but the plugin reads it from a fixed path (not configurable), so it's a single global config per plugin. Env vars are the most portable and CI-friendly.

## 5. Empirical verification (the recipe that catches silent breakage)

Don't trust `ls` or `opencode.json` looking right. Three independent checks:

### 5a. Confirm the file is actually loaded

Start `opencode --print-logs --log-level INFO run "say ok"` (any trivial prompt). In `~/.local/share/opencode/log/opencode.log`, grep for:

```bash
grep -E "loading path=.*opencode.json" ~/.local/share/opencode/log/opencode.log | tail -10
```

You should see one `INFO loading path=...` line per file that exists in `~/.config/opencode/`. **If you see a `loading path=...opencode.jsonc` line, that file is being read** — do not assume it's inert.

### 5b. Confirm the plugin initialized

The Hindsight plugin (and most others) logs a single line at startup with the resolved config. Grep for it:

```bash
grep -E "(<plugin-name> plugin initialized|Recall failed)" ~/.local/share/opencode/log/opencode.log | tail -10
```

The Hindsight success line looks like:

```
INFO message="Hindsight plugin initialized" api=http://127.0.0.1:8888 bank=hermes
  authenticated=false autoRecall=true autoRetain=true
```

If you see this, the plugin is loaded AND reading its config correctly. The `api=` and `bank=` fields confirm the persistent config file was honored.

If the plugin failed to load, you'll see one of:
- `ERROR message="Hindsight plugin initialized"` with `authenticated=false` and `api=https://api.hindsight.vectorize.io` — the persistent config file wasn't read; the defaults won. Common cause: wrong filename (`opencode.json` vs `config.json`), wrong directory (`~/.hindsight/` vs `~/.config/hindsight/`).
- No `Hindsight plugin initialized` line at all — the plugin isn't loaded. Check the `"plugin": [...]` array in the operative `opencode.json`.
- `init count` dropped compared to a known-good baseline — plugin presence changed.

### 5c. Confirm the plugin actually talks to the backend

If the plugin made it past init, the next failure mode is "initialized but can't reach the backend". For Hindsight, this manifests as `Recall failed: "Unable to connect..."`. That's **expected** if the backend is down — but a `401 Authentication failed` is a config-level problem (token mismatch, wrong endpoint). Differentiate by message text:

| Message | Root cause | Fix |
|---|---|---|
| `Unable to connect` / `connection refused` | Backend down or wrong port | Verify with `curl -s http://<api>:<port>/health` |
| `Authentication failed: API key required` | Plugin didn't read the token | Check `~/.<plugin>/<name>.json` and env vars |
| `Authentication failed: ...` with token set | Token wrong for this endpoint | Re-check token vs URL pairing |

### 5d. The `init count` signal

`init count=N` in the log is a rough proxy for "how many components loaded". Before any plugin: `init count=19` (the OpenCode core). After Hindsight loads: `init count=42` (typical; the delta is plugin-specific). A sudden drop in `init count` after a config edit is a strong signal something was removed.

## 6. Common mistakes

- **Declaring the plugin in `.jsonc` and then deleting `.jsonc`.** Plugin silently disappears (`init count` drops, no error). Always re-declare in the operative file.
- **Editing `~/.hindsight/opencode.json` (or similar) and not restarting OpenCode.** Same trap as the Hermes `~/.hermes/hindsight/config.json` pitfall: the plugin's client was constructed at startup with the OLD config. Restart `opencode` to pick up changes.
- **Putting `apiKey` / `apiToken` in `opencode.json` inline.** Visible in the file, ships with backups. Use env vars for secrets.
- **Assuming the persistent config file's path is documented.** Most plugins pick a path like `~/.<plugin-name>/<name>.json` but it's NOT standardized. Read the plugin's docs OR grep the plugin's source for `readFileSync|fs.readFile` calls that hardcode a path.
- **Forgetting the inline `plugin: [...]` entry in `opencode.json`.** The persistent config file alone doesn't load the plugin — you need BOTH the declaration (in `opencode.json`) AND the config file (at the plugin's hardcoded path). This is a common first-time setup mistake.
- **Mixing MCP + plugin for the same service.** If a service exposes both a plugin and an MCP server (Hindsight does), pick one path. Running both produces duplicate retain/recall traffic and possible locks against the same bank. Disable the MCP block when using the plugin.

## 7. Verification checklist (after any plugin config change)

- [ ] `python3 -c "import json; json.load(open('<path>'))"` exits 0
- [ ] Operative config file confirmed (check log `loading path=...` lines)
- [ ] `"plugin": [...]` array contains the package name
- [ ] Persistent config file at `~/.<plugin>/<name>.json` if applicable
- [ ] `opencode --print-logs` startup shows the plugin's `initialized` line with the right `api=`/`bank=`/etc.
- [ ] `init count` did not drop compared to baseline
- [ ] Backend reachable: `curl <plugin-api-url>/health` returns 200 (if applicable)
- [ ] End-to-end: invoke one of the plugin's tools (`hindsight_retain`, etc.) and verify the operation succeeded

## Support files

- `references/hindsight-plugin-example.md` — worked example for the `@vectorize-io/opencode-hindsight` plugin (v0.2.5), with the exact files written, the verification commands, and a list of the three problems the original session hit.
- `references/plugin-state-verification.md` — the empirical recipe for diagnosing "plugin was working yesterday, isn't today". Three layers (config loaded, plugin initialized, backend reachable), with the `init count` heuristic and the round-trip validation test.