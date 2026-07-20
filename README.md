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
