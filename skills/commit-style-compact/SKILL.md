---
name: commit-style-compact
description: Format git commits in compact style for reviewable PRs — short subject line, 2-4 line body, one logical change per commit. Load when authoring commits in any project where the user values PR review speed over commit-by-commit context. Apply on first commit after the user states a preference about commit length, verbosity, or "make it shorter". Use the user's own phrasing in subject lines where possible.
---

# Compact commit style

> **⚠️ Read this section BEFORE any `git commit` action.** See
> [`references/dev-authorship-rule-2026-07-09.md`](references/dev-authorship-rule-2026-07-09.md)
> for full rationale, defense layers, and audit commands.

## HARD INVARIANTS — non-negotiable (governed by the developer)

### H1. No profile signs commits. Ever.
`commit.gpgsign=false` and `tag.gpgsign=false` are pinned in
`~/.gitconfig` (host-global). Effective in every repo unless
overridden locally — verified on jul-09-2026 across all 6 Odizey +
4world repos. **If you ever find yourself running `git commit -S`,
`--gpg-sign`, or any code path that invokes gpg/ssh signing → STOP.
The rule is non-negotiable; the override is an ALERT, not a courtesy.**

Defense layers (in priority order):
1. Host `~/.gitconfig`: `commit.gpgsign = false`, `tag.gpgsign = false`,
   `push.gpgsign = false`, `user.signingkey = ""`, `gpg.format = ssh`.
2. Transversal rule in `~/.hermes/AGENTS.md` § "AUTORÍA DE COMMITS"
   (read by all profiles at startup, including subagents).
3. This skill's H1–H5 below.
4. Operational audit command (when sanity-checking):
   `git -C <repo> config --list --show-origin | grep -i gpg`.

### H2. Author and committer are always the developer.
- Author: `Alvaro Gonzalez <alvaro.gonzalez.dev@gmail.com>`
- Committer: same identity.
- Profiles MUST NOT touch `user.name`, `user.email`,
  `GIT_AUTHOR_NAME`, `GIT_AUTHOR_EMAIL`, `GIT_COMMITTER_NAME`,
  `GIT_COMMITTER_EMAIL`. If a subagent reports wanting to "match the
  committer to the author they wrote" — refuse; it is not your commit.
- If you need a one-off override (e.g. signing a tagged release under
  the maintainer's key), ASK FIRST. The default is the developer.

### H3. The developer is responsible for `push`, `--amend`, and any rewrite of published history.
Profile roles:
- MAY: stage files, write the working tree, suggest commit message,
  run `git diff --staged` for review.
- MAY NOT: `git push`, `git commit --amend` on a published commit,
  `git reset --hard` on a published branch, force-push of any kind,
  rebase onto `main`/`develop` without explicit approval.

The flow is: profile prepares → developer commits → developer pushes.
This is by design — published history is the developer's responsibility
and theirs alone.

### H4. `Co-authored-by:` lines from LLMs only with explicit approval.
Default: NO `Co-authored-by: Hermes` / `Claude` / `GPT` / `Codex` /
`MiniMax` etc. in commit trailers. Add only when the user says in
that session "añade Co-authored-by" or equivalent explicit ask.
This is a per-commit decision; default silence is the safe state.

### H5. Override detection — what to do if any of H1–H4 trip
1. Halt the action immediately.
2. Surface to the user with: which rule, which command attempted,
   which repo (if local override).
3. Audit with: `git -C <repo> config --list --show-origin | grep -i gpg`
   to find the override source.
4. Do not proceed until the user confirms.

## When to use

- The user has expressed a preference about commit length or format. Triggers:
  - "los commits se estan volviendo muy extensos" / "very verbose commits"
  - "sintetizar los cambios mas importantes" / "synthesize the main changes"
  - "no quiero X en los commits" / explicit format preferences
  - "esto no es para que lo apliques ahora es para que lo apliques a futuro" / "aplica a futuro" — explicit forward-looking scope (only NEW commits)
  - Any request that implies "make the diff easier to skim"
- A feature has accumulated 5+ commits in the same branch and the user wants to clean up before opening the PR.
- Default for any new feature branch in projects the user reviews regularly.

## Core rules

### Subject line (line 1)
- **Max 70 characters.**
- Format: `<type>(<scope>): <imperative-summary>`
- `<type>` is one of: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `style`, `perf`, `build`, `ci`.
- `<scope>` is the module, file group, or feature touched. Prefer 2 segments max (e.g. `rds/streak`, `dashboard`, `quiz-tower`).
- Imperative mood, lowercase first letter, no trailing period.
- **NO** commit body in the subject. The subject names the change; the body explains the why only if non-obvious.

```bash
# good
fix(rds/streak): render v2 reward shapes in dashboard cards

# bad — too long, restates the change
fix(rds/streak): render dailyRewards[i] as amount+currency in DashboardRachaCard and update types

# bad — what does it do?
refactor(stuff): clean up
```

### Body (optional)
- **Max 2-4 lines** (in addition to a brief explanation of why the change was needed).
- One paragraph max. **NO** historical context ("earlier we had X, then we changed to Y, now we...").
- **NO** bullet-list commits that re-summarize the diff.
- **NO** "this commit fixes...", "this adds...", "we now..." — the verb is implicit in `<type>`. Subject + 2-4 lines of "why" only.

```bash
# good
fix(rds/streak): render v2 reward shapes in dashboard cards

The v2 rewards migration (bo_process commit f65efa5) made
schema.daily_rewards and schema.monthly_big_prize return
{day,amount,currency} objects instead of flat numbers /
amount_odicoins. Three cards rendered those values as raw
fields and crashed on mount with React "Objects are not valid
as a React child".

# bad — context overload
fix(rds/streak): render v2 reward shapes in dashboard cards

The v2 rewards migration (bo_process commit f65efa5) made
schema.daily_rewards and schema.monthly_big_prize return
{day,amount,currency} objects instead of flat numbers /
amount_odicoins. Three cards rendered those values as raw
fields and crashed on mount with React "Objects are not valid
as a React child".

Fixed by extracting .amount/.currency from each object and
falling back to the legacy v1 fields (amount_odicoins, flat
number) when the new shape is missing. RdsSchema union now
exposes both shapes for the consumers that still see v1.

Files:
  - DashboardRachaCard.tsx:    render + fallback to v2 shape
  - DashboardCalendarioCard.tsx: monthly_big_prize.amount
  - StreakCalendarioTab.tsx:   monthly_big_prize.amount
  - streak.interface.ts:        daily_rewards + monthly_big_prize
                                  now union v1 | v2

tsc --noEmit 0 errors.
```

### One commit = one logical change
- A single commit CAN touch multiple files if they are part of the same logical fix (e.g. v2 shape evolution across 4 components).
- A single commit CANNOT bundle unrelated changes (e.g. a refactor + a bug fix + an i18n update).
- **Rule**: if the diff has more than ~200 lines, split. If files are in unrelated folders (different features / different repos), split.

### History rewriting
- If you accumulated 5+ verbose commits in a single session, **squash them with `git reset --soft <earlier> && git commit`** before opening the PR. The user's exact phrase: "los commits se estan volviendo muy extensos necesito que de ahora en adelante los commits que se generen sintetisen los cambios mas importantes de los archivos para mejor lectura en los PR".
- Do NOT rewrite commits the user did not ask to rewrite. The history-rewrite recipe applies only to NEW work in the current session.
- **Default for "make commits shorter" requests: forward-looking only.** When the user says "make future commits more compact" / "aplica esto a futuro" / "a partir de ahora", apply the compact style to all NEW commits from that point on. Do NOT rewrite commits that already landed unless the user explicitly asks (e.g. "haz squash de los últimos commits", "reescribe el historial"). The user may want a clean history on `develop`, but the working branch's commit log is a record of what happened — overwriting it without permission erases that record. The user's exact phrasing: "esto no es para que lo apliques ahora es para que lo apliques a futuro" — that distinction is binding. **Operational sequence that bit me in June 2026 RDS:** the user asked "los commits se estan volviendo muy extensos". I interpreted it as "rewrite history" and proactively squashed the 3 most recent verbose commits with `git reset --soft`. The user's next message was "esto no es para que lo apliques ahora es para que lo apliques a futuro" — explicitly rejecting the proactive rewrite. The lesson: when the user asks for a format change without specifying scope, **ask before doing any `git reset --soft`** ("¿aplicas la regla nueva a los commits que ya están, o solo a los de aquí en adelante?"). Default to forward-looking only. Don't assume the user wants history rewritten just because they asked for "more compact commits".

## Recipe: squash a verbose commit chain into 1-2 commits

```bash
# Step 1: identify the range to squash (e.g. last 5 commits on the branch)
git log --oneline -5

# Step 2: soft-reset to the commit BEFORE the chain (keep all changes staged)
git reset --soft <earlier-sha>

# Step 3: stage the files into 2 groups if needed
# - Group A: the main change (all files that implement the feature)
git add src/components/Streak/StreakMisionesTab.tsx \
        src/components/Streak/StreakCalendarioTab.tsx \
        src/components/Streak/DashboardRachaCard.tsx \
        src/interfaces/streak.interface.ts
git -c user.name="..." -c user.email="..." \
  commit -m "fix(rds/streak): render v2 reward shapes in dashboard cards

The v2 rewards migration (bo_process f65efa5) made
schema.daily_rewards and schema.monthly_big_prize return
{day,amount,currency} objects. Three cards rendered raw
fields and crashed with React 'Objects are not valid as
a React child'. Extracting .amount/.currency fixes them.

tsc --noEmit 0 errors."

# - Group B: a logically separate change (e.g. a test mock)
git add src/pages/Dashboard/DashboardPage.pendingInvitationRedirect.test.tsx
git -c user.name="..." -c user.email="..." \
  commit -m "test(dashboard): mock streak hooks in pendingInvitationRedirect test

DashboardPage mounts StreakModalTrigger which calls
useStreakSchema() and friends. Test rendered without a
QueryClientProvider, crashed the render, husky pre-push
aborted. Mock the barrel like useWallet / useSocialMedia /
useLogoutAndBack already do.

tsc --noEmit 0 errors."
```

## What this skill is NOT

- It is **not** a "drop every detail" recipe. Two pieces of context belong in the body:
  - The "why" of an unusual choice (e.g. "we use the partial UNIQUE INDEX here because…").
  - The "what was tried first that didn't work" if it's not obvious from the diff (e.g. "ON CONFLICT ON CONSTRAINT does not work for partial UNIQUE INDEXes — see references/postgres-partial-unique-index-on-conflict-inference.md").
- It is **not** an excuse to omit file lists when the scope is non-obvious. A commit that touches 6 files across 3 unrelated features still needs a file list. But a commit that touches 3 files in the same component doesn't.

## References

- `references/long-vs-short-commit-examples.md` — a side-by-side comparison of verbose vs compact commit messages for the same hypothetical change.
