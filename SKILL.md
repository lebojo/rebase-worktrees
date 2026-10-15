---
name: rebase-worktrees
description: >-
  Fan-out rebase of every Orca worktree of the current repo onto its base branch
  (main/master), one agent per worktree: launch, monitor, verify (clean tree,
  pushed, build), kill the agents, and report a summary table. Use when the user
  says "rebase tous les worktrees", "rebase all worktrees", "rebase tout",
  "rebase mes branches sur main", "lance les rebases", or asks to rebase several
  Orca worktrees at once with agents fixing the conflicts.
---

# Rebase All Worktrees

Rebase every non-main Orca worktree of the current repo onto the base branch, one
opencode agent per worktree, then verify, kill the agents, and summarize.

Requires Orca running and `opencode` at `/Users/jordan/.opencode/bin/opencode`.
Load the version-matched `orca-cli` guide first (`ORCA skills get orca-cli --json`),
where `ORCA` = `$ORCA_CLI_COMMAND` if set, else `orca` on macOS.

## Workflow

### 1. Discover

```sh
ORCA worktree list --json          # note repoId, every worktree id, branch, isMainWorktree
```

- Candidates = all worktrees of that repoId with `isMainWorktree: false`.
- Detect the base branch: `git symbolic-ref --short refs/remotes/origin/HEAD`
  (fallback: try `main`, then `master`). Fetch it once from any checkout.

### 2. Triage each candidate (git facts, not vibes)

For each worktree path `$d` on branch `$b`:

```sh
git -C "$d" status --porcelain | wc -l                     # dirty?
git -C "$d" rev-list --left-right --count "origin/$base...HEAD"
git -C "$d" rev-list --left-right --count "origin/$b...HEAD"  # pushed?
git -C "$d" status | grep -c "rebase in progress"          # stuck mid-rebase?
```

- `behind == 0` → already up to date: skip, report "déjà à jour".
- Rebase in progress or detached HEAD → skip and surface it; do not touch.
- Dirty → plan the WIP snapshot commit (step 3).

### 3. Build one tailored prompt per worktree (French)

Gather context first: `git log --oneline -12 "origin/$base"` and, per branch, the
base commits since merge-base — this is what makes conflict resolution good.

Prompt skeleton (one single line, no raw newlines, no shell-hostile quotes):

> Mission: rebase la branche `$b` sur `origin/$base` dans ce worktree et résoudre
> les conflits proprement. Contexte: <repo + langage + build system>. Récents
> changements sur `$base`: <log>. (Si dirty) Étape 0: git add -A && git commit
> -m wip-snapshot-avant-rebase. Ensuite: (1) git fetch origin && git rebase
> origin/$base. (2) Pour chaque conflit, comprendre les DEUX intentions avant de
> trancher: git log --oneline origin/$base -- fichier + git show côté base, git
> log origin/$base..HEAD côté branche; adapter le code de la branche à la
> NOUVELLE architecture; interdits: --ours/--theirs en bloc, perdre des commits
> de la branche, réintroduire du code supprimé par la base. (3) Chasse aux
> références mortes (grep symboles renommés/fichiers supprimés), regénérer le
> projet si des fichiers ont bougé, puis build de vérification au mieux; si le
> setup manque, l'installer sans committer d'artefacts; si build impossible,
> revue statique sérieuse et le signaler. (4) git push --force-with-lease origin
> $b. (5) orca worktree set --worktree <id> --comment "<résumé court>"
> --workspace-status in-review (ou in-progress si problème). (6) Rapport final:
> conflits rencontrés, décisions, résultat du build, confirmation du push.

Add branch-specific warnings when known (moved symbols, renamed views, WIP
renames to preserve, fix that must be re-localized…).

### 4. Launch (parallel)

```sh
ORCA terminal create --worktree id:<repoId>::<path> --title "rebase <name>" \
  --command "/Users/jordan/.opencode/bin/opencode" --json     # record handle!
ORCA terminal wait --terminal <handle> --for tui-idle --timeout-ms 90000 --json
ORCA terminal send --terminal <handle> --text "<prompt>" --enter --wait-submit 20 --json
```

- Create all terminals in parallel; keep a table handle → worktree.
- Send only after `wait.satisfied: true` (re-wait once with a bigger timeout on
  false). `accepted: true` is not proof of submission; if the provider is
  "unsupported", `terminal read` to confirm the prompt rendered, then a short
  `wait --for tui-idle --timeout-ms 5000` that is NOT satisfied = agent working.

### 5. Monitor

```sh
ORCA terminal wait --terminal <handle> --for tui-idle --timeout-ms 900000 --json
```

Loop until satisfied (builds can take 1h+; cap at ~2h then report status).
Do not kill a running agent. Prefer checking git state over spamming reads.

### 6. Verify each worktree

| Check | Command | Expect |
| --- | --- | --- |
| Clean | `git status --porcelain \| wc -l` | 0 |
| Rebased | `rev-list --left-right --count origin/$base...HEAD` | `0  N` |
| Pushed | `rev-list --left-right --count origin/$b...HEAD` | `0  0` |
| Report | `ORCA terminal read` tail | final summary present |

A dirty tree or non-zero "behind" = not done: nudge the agent or fix yourself
before killing anything.

### 7. Kill

Close ONLY the terminals you created (recorded handles), one by one:

```sh
ORCA terminal close --terminal <handle> --json
```

Never close pre-existing terminals, and never `--all`.

### 8. Summarize

Report a table: branch → rebase ✅/❌, ahead count, new tip (short sha), push,
build result, notable conflict decisions. Mention the WIP snapshot commits
created (they stay in history for the user to squash later).

## Hard rules

- Never delete worktrees or branches, and never push without `--force-with-lease`.
- Never kill/close a terminal you did not create in this run.
- Never send a prompt before `tui-idle` was satisfied; never resend on silence.
- One agent per worktree; if an agent session already lives there, surface it
  and ask before spawning a second one.
- If a worktree cannot be verified as good, say so in the summary instead of
  killing its agent mid-work.
