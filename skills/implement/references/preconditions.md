# Precondition Checks

Phase 0 checks run in order. Stop at the first failure.

## Check 1: br CLI available

```bash
which br
```

If missing:
> "The `br` CLI is required but not found in PATH. Install it or ensure it is on your PATH."

## Check 2: Not already in a worktree

```bash
test -f "$(git rev-parse --show-toplevel)/.git" && echo "worktree" || echo "main-repo"
```

If "worktree":
> "You are already inside a git worktree. Exit the current worktree first (`/exit-worktree` or `ExitWorktree`), then re-run `/implement`."

## Check 3: Epic ID provided

Parse `$ARGUMENTS` to extract the epic ID (first whitespace-delimited token).

If empty or not a bead ID:
> "Usage: `/implement <epic-id> [branch-name]`"

Record as **EPIC_ID**.

## Check 4: Epic exists

```bash
br show <EPIC_ID> --json
```

If the command fails:
> "Epic `<EPIC_ID>` not found. Verify with `br show <EPIC_ID>` or `br list --type epic`."

Record the title as **EPIC_TITLE**.

## Check 5: No dependency cycles

```bash
br dep cycles
```

If cycles exist, report them and stop.

## Check 6: Parse optional branch name

Parse the second whitespace-delimited token from `$ARGUMENTS`. Record as
**BRANCH_NAME** (or null if not provided).

## Check 7: Discover BASE_REF

Detect the repository's default branch for use throughout the skill.

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'
```

If that returns nothing (no remote configured), fall back:

```bash
for branch in main master trunk; do
  git show-ref --verify --quiet "refs/heads/$branch" && echo "$branch" && break
done
```

Record as **BASE_REF**. All subsequent references to the base branch use this
variable instead of hard-coding a branch name.

If neither method produces a result:
> "Could not detect the default branch. Ensure `refs/remotes/origin/HEAD` is set (run `git remote set-head origin --auto`), or create a local branch named `main`, `master`, or `trunk`."
