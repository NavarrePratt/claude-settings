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

## Check 8: Derive MAIN_REPO_BEADS_DB

Record the path to the main repo's beads database. Every `br` call made
after the Phase 2 `cd` into the worktree targets this path via the `--db`
flag, because `.beads/` is gitignored and does not exist in the worktree.

Derivation must run BEFORE the Phase 2 `cd`, so `git rev-parse --show-toplevel`
returns the main repo root. Ask `br` itself for the path first, and fall
back to the default location only if `br where` cannot report it:

```bash
MAIN_REPO_BEADS_DB="$(br where --json 2>/dev/null | jq -r '.database_path // empty' 2>/dev/null || true)"
if [ -z "$MAIN_REPO_BEADS_DB" ]; then
  MAIN_REPO_BEADS_DB="$(git rev-parse --show-toplevel)/.beads/beads.db"
fi
```

Reject paths containing shell metacharacters, since the value is rendered
verbatim into downstream Bash snippets:

```bash
case "$MAIN_REPO_BEADS_DB" in
  *[\$\`\"\'\\]*|*$'\n'*)
    echo "Error: repo path contains shell metacharacters; cannot safely derive MAIN_REPO_BEADS_DB"
    exit 1
    ;;
esac
```

Verify the database file exists:

```bash
[ -f "$MAIN_REPO_BEADS_DB" ]
```

If the file does not exist:
> "Beads database not found at `$MAIN_REPO_BEADS_DB`. Run `br where` from the repo root to find the actual path (if the workspace uses a non-default database name) or `br init` to create one."

Smoke-test the database with a real read-only `br --db` call so Phase 0
fails fast instead of failing later at the first bead operation:

```bash
br --db "$MAIN_REPO_BEADS_DB" list --limit 1 --json >/dev/null 2>&1 || { echo "Error: cannot open beads database at $MAIN_REPO_BEADS_DB"; exit 1; }
```

Record as **MAIN_REPO_BEADS_DB**. See
[shared/br-in-worktree.md](../../shared/br-in-worktree.md) for the canonical
pattern and rationale.
