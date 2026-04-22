# Phase 0: Preconditions

Run these checks before any other phase. If any fails, stop with an actionable message. Do not attempt workarounds.

## Step 1: Git repository

```bash
git rev-parse --git-dir 2>/dev/null
```

If this fails: "Not a git repository. Run /grug-cut-pass from the root of a git repo." Stop.

## Step 2: Working tree clean

```bash
git status --porcelain
```

If non-empty, the tree is dirty. AskUserQuestion:

```
question: "Working tree has uncommitted changes. The cut pass will rebase and amend commits, which requires a clean tree. How do you want to proceed?"
header: "Dirty tree"
options:
  - { label: "Stash and continue", description: "git stash push, run the pass, then restore with git stash pop after" }
  - { label: "Abort", description: "Stop so you can commit or discard changes manually" }
```

If Stash: `git stash push -u -m "grug-cut-pass: pre-run stash"`. Record that a stash was created so Phase 8 can restore it.
If Abort: stop.

## Step 3: Codex MCP tool available

Check the tool list for `mcp__codex__codex`. If absent:

"Codex MCP is required for the grug pass (Phase 3). Either enable the Codex MCP server or run /grug-review for a checklist-only review without Codex validation."

Stop. Do not fall through silently.

## Step 4: BASE_REF discovery

Detect the base branch using the same logic as other skills:

```bash
# Prefer origin/HEAD if set
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|^refs/remotes/origin/||'
# Fallback: main, then master
git show-ref --verify --quiet refs/remotes/origin/main && echo main
git show-ref --verify --quiet refs/remotes/origin/master && echo master
```

If none of these resolve, AskUserQuestion for the base branch name. Abort if user cannot supply one.

Set `BASE_REF=origin/<detected_branch>`.

## Step 5: Git-spice detection

```bash
git show-ref --verify --quiet refs/spice/data && echo STACKED || echo SINGLE
```

Set `STACKED=true` if the ref exists. This is used in Phase 1 to offer stack-aware scoping.

If STACKED=true, also capture the current branch's stack neighbors:

```bash
gs ls 2>/dev/null
```

If `gs` is not on PATH but `refs/spice/data` exists, fall back to `SINGLE` mode and note the warning in the final report. Do not abort - just degrade gracefully.

## Step 6: Branch slug for temp directory

```bash
BRANCH_SLUG=$(git branch --show-current | sed 's|[/:]|-|g')
TMP_DIR="/tmp/grug-cut-${BRANCH_SLUG}"
mkdir -p "$TMP_DIR"
```

Subsequent phases write their artifacts to `$TMP_DIR`. Phase 8 cleans this up.

## Outputs of Phase 0

State values to carry into later phases:
- `BASE_REF` - e.g., `origin/main`
- `STACKED` - `true` or `false`
- `STASHED` - `true` if a stash was created, `false` otherwise
- `TMP_DIR` - e.g., `/tmp/grug-cut-feat-audit-logs`
- `BRANCH_SLUG` - sanitized branch name
