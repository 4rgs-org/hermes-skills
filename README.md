# Hermes Skills Plugin for Claude Code

A curated subset of the Hermes skill library, auto-discoverable by Claude Code as a plugin,
automatically synced with `~/.hermes/skills/` via symlinks.

## Auto-sync

Each `skills/<name>/SKILL.md` here is a symlink to `/Users/alvarogonzalez/.hermes/skills/<name>/SKILL.md`,
so any change you make in either direction is reflected immediately — no need to re-publish.

Hermes runs `~/.local/bin/hermes-skills-resync` (or the helper at the bottom of this README)
when new skills are added/removed in Hermes.

## Updating for everyone (push to a shared remote)

Once you set up the remote on GitHub as `Spids/hermes-skills`, you can run:

```bash
cd ~/integ/hermes-skills
git add -A && git commit -m "sync: <message>"
git push origin main
```

Other developers then:

```bash
claude plugin marketplace add Spids/hermes-skills
claude plugin install hermes-skills
```

## Skills included

This plugin exposes a subset of Hermes skills filtered for general Claude Code use (no
Odizey-specific or Hermes-internals skills). Edit `scripts/sync.py` in Hermes to change
the include list. Today (2026-07-20) the list is:

- `claude-code-hindsight-setup`
- `hermes-hindsight-memory-provider`
- `hindsight-monitor-deploy`
- `mmx-cli`
- `cloudflare`
- `cloudflare-email-service`
- `cloudflare-one`
- `cloudflare-one-migrations`
- `cloud-run-basics`
- `cloud-sql-basics`
- `workers-best-practices`
- `wrangler`
- `durable-objects`
- `turnstile-spin`
- `sandbox-sdk`
- `firebase-basics`
- `firebase-auth-basics`
- `agents-sdk`
- `3d-web-game-threejs-stack`
- `opencode-plugin-config`
- `find-skills`
- `web-perf`
- `commit-style-compact`
- `vitest-mock-dual-import-symmetry`
- `drawio-skill`
- `macos-ssh-from-hermes-session`
- `dogfood`
- `yuanbao`

## Hooks

None — this plugin provides skills only. Hindsight auto-recall/retain hooks come from the
`hindsight-memory@hindsight` plugin (configured separately).


## Use from Claude Code

Once installed via this marketplace, the skills appear in Claude Code under
the `hermes-skills` plugin. Invoke them with slash commands:

```
/hermes-skills:claude-code-hindsight-setup      # install/configure Hindsight
/hermes-skills:cloudflare                       # Cloudflare platform help
/hermes-skills:wrangler                         # Cloudflare Workers CLI
/hermes-skills:mmx-cli                          # MiniMax media generation
... (etc, 28 skills)
/hermes-skills:list                             # (no built-in — use claude plugin list)
```

## Use from Hermes

The symlinks under `skills/` always reflect `~/.hermes/skills/<name>/SKILL.md`.
To add a new skill from Hermes to this plugin:

```bash
# 1. Add to the canonical include list (creates the symlink everywhere, dry-run safe)
~/.local/bin/hermes-skills-resync --add my-new-skill
~/.local/bin/hermes-skills-resync --apply

# 2. Push to GitHub (resolves symlinks to real files, commits, pushes)
~/.local/bin/hermes-skills-publish
```

To remove:

```bash
~/.local/bin/hermes-skills-resync --remove my-old-skill
~/.local/bin/hermes-skills-resync --apply
~/.local/bin/hermes-skills-publish
```

## Bidirectional sync notes

- **Hermes → Claude Code**: edit any `SKILL.md` under `~/.hermes/skills/`. Since
  Claude Code's plugin cache points at the same path (via symlink in the marketplace
  directory), changes are visible instantly to running Claude Code sessions.
- **Claude Code → Hermes**: write a new file into the plugin's `skills/<name>/SKILL.md`
  in your local plugin cache; the symlink means it lands in `~/.hermes/skills/`. Then
  run `hermes-skills-publish` to push to GitHub so the team sees it.
- **GitHub → local**: `claude plugin marketplace update hermes-skills && claude plugin install hermes-skills`.

## Why this is public

GitHub blocks `raw.githubusercontent.com` for private repos unless the request is
authenticated, and Claude Code's marketplace fetcher uses anonymous HTTPS. A private
version is mirrored via the git remote in `~/integ/hermes-skills` if needed; just
delete the `gh-pages` (or this `main`) branch on `4rgs-org/hermes-skills` and
re-publish to a private repo to take it back down.
