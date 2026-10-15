# rebase-worktrees

An [agent skill](https://skills.sh/) that rebases **every Orca worktree of the current repo** onto its base branch (`main`/`master`) — one agent per worktree — then verifies the result, kills the agents, and hands you a summary table.

Built for the workflow: *"there was a big refactor on main, my 6 feature branches now conflict, fix them all for me."*

You say:

> rebase tous les worktrees

The skill does the rest.

## How it works

```
discover → triage → tailored prompts → parallel launch → monitor → verify → kill → summary
```

1. **Discover** — lists Orca worktrees of the current repo (`orca worktree list --json`), auto-detects the base branch.
2. **Triage** — checks dirty state, ahead/behind, rebase-in-progress per worktree; skips what is already up to date.
3. **Tailored prompts** — one prompt per branch, seeded with the base branch's recent log (the refactor context) and branch-specific warnings (WIP renames to preserve, fixes to re-localize…).
4. **Parallel launch** — one `opencode` agent per worktree in its own Orca terminal, prompt sent only after the TUI is ready.
5. **Monitor** — waits for each agent to go idle (builds can take 1h+), never killing a running agent.
6. **Verify** — clean tree, `0` commits behind base, force-with-lease push confirmed, agent's final report present.
7. **Kill** — closes only the terminals created for this run, never yours.
8. **Summary** — table: branch, rebase status, new tip, push, build, notable conflict decisions.

## Install

```sh
npx skills add lebojo/rebase-worktrees
```

or manually:

```sh
git clone https://github.com/lebojo/rebase-worktrees
cp -r rebase-worktrees ~/.agents/skills/    # or ~/.claude/skills/, ~/.config/opencode/skills/
```

## Requirements

- Orca running, with its `orca` CLI available (or `ORCA_CLI_COMMAND` set)
- `opencode` at `/Users/jordan/.opencode/bin/opencode` (configurable in the skill)
- Git, and a repo with Orca-managed worktrees

## Hard rules

The skill is deliberately conservative:

- Never deletes worktrees or branches
- Never pushes without `--force-with-lease`
- Never closes terminals it did not create
- Never sends a prompt before the agent TUI is ready
- One agent per worktree — an existing agent session is surfaced, not duplicated
- If a worktree can't be verified as good, it's reported instead of silently killed

The conflict-resolution method is baked into the prompts: understand **both** intentions (base refactor vs branch feature) before resolving, adapt the branch to the new architecture, and never resolve with blanket `--ours`/`--theirs`.

## License

[MIT](LICENSE)
