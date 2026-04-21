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
> "Usage: `/implement <epic-id> [branch-name] [--auto]`"

Record as **EPIC_ID**.

## Check 4: Epic exists

```bash
br show <EPIC_ID> --json
```

If the command fails:
> "Epic `<EPIC_ID>` not found. Verify with `br show <EPIC_ID>` or `br list --type epic`."

Record the title as **EPIC_TITLE**.

## Check 4b: Epic not already claimed

Read the `status` field from the `br show` output above.

If `status == "in_progress"`, another /implement session may be actively
working this epic. Surface the epic's notes (claim marker) and use
AskUserQuestion:

```
questions: [{
  question: "Epic <EPIC_ID> is already in_progress - another /implement session may be working it. What would you like to do?",
  header: "Epic claimed",
  options: [
    { label: "Abort", description: "Stop without making changes (recommended)" },
    { label: "Take over", description: "Proceed anyway - only safe if the other session is confirmed dead. Will overwrite claim notes." }
  ],
  multiSelect: false
}]
```

If "Abort": exit the skill.
If "Take over": proceed - the Phase 2 claim will rewrite the notes with the
current session's worktree path.

## Check 5: No dependency cycles

```bash
br dep cycles
```

If cycles exist, report them and stop.

## Check 6: Parse optional arguments

Parse remaining tokens from `$ARGUMENTS` (after the epic ID):
- `--auto` flag: if present anywhere after the epic ID, set **AUTO_MODE** = true;
  otherwise **AUTO_MODE** = false. Controls whether Phase 4 passes `--auto` to
  /team-branch-fix (autonomous fix selection) or invokes it interactively.
- Remaining positional token (not `--auto`): optional branch name.

The `--auto` flag can appear in any position after the epic ID. Strip it
before extracting the branch name.

Record **AUTO_MODE** (true/false) and **BRANCH_NAME** (or null if not provided).

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
