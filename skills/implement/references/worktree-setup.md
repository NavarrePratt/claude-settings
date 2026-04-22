# Worktree Setup

Create and configure an isolated worktree for implementation.

## Step 1: Compute Branch Name

Determine the raw input: if **BRANCH_NAME** was provided in `$ARGUMENTS`,
use that value; otherwise use **EPIC_TITLE**.

Always slugify the raw input - user-provided BRANCH_NAME is NOT accepted
as-is. The same slugification rules apply to both the user-provided and
epic-title-derived cases:

- Convert to lowercase
- Replace spaces with hyphens
- Strip characters that are not alphanumeric or hyphens
- Collapse consecutive hyphens
- Trim trailing hyphens

Record the slugified result as **BRANCH_NAME**. This value flows into
shell commands (`git branch --list`, `git worktree add`) and heredoc
bodies below, so slugification is a required input-validation step -
never pass raw `$ARGUMENTS` through verbatim.

Compute the repository directory name for use in state file paths:

```bash
basename "$(git rev-parse --show-toplevel)"
```

Record as **REPO_NAME** (e.g., `claude-settings`).

## Step 2: Check for Existing Worktree

```bash
git worktree list
```

Check if `.claude/worktrees/implement-<BRANCH_NAME>` already exists.

If it exists, use AskUserQuestion:

```
questions: [{
  question: "Worktree implement-<BRANCH_NAME> already exists. What would you like to do?",
  header: "Worktree",
  options: [
    { label: "Reuse", description: "Continue working in the existing worktree" },
    { label: "Abort", description: "Stop without making changes" }
  ],
  multiSelect: false
}]
```

If "Reuse": check for an existing state file at
`/tmp/<REPO_NAME>-<EPIC_ID>/state.json`. If it exists:
- Verify `epic_id` matches the current EPIC_ID
- Verify `branch_name` matches
- If mismatch: warn and discard stale state
- If match: load state for context (bead statuses, last completed phase)

Then change working directory to the existing worktree and skip to Step 5.

If "Abort": exit the skill.

Also check if the branch name already exists without a worktree:

```bash
git branch --list "feat/<BRANCH_NAME>"
```

If the branch exists but no worktree uses it, warn and offer to use a
different name or reuse the branch.

## Step 3: Create the Worktree

```bash
git worktree add .claude/worktrees/implement-<BRANCH_NAME> -b feat/<BRANCH_NAME> <BASE_REF>
```

Record the full worktree path as **WORKTREE_PATH**.

## Step 4: Change Working Directory

Change to WORKTREE_PATH. All subsequent commands execute inside the worktree.

## Step 5: Verify Environment

All `br` calls from this point forward must use the `--db` flag to target
the main repo's database. See [../../shared/br-in-worktree.md](../../shared/br-in-worktree.md)
for the canonical pattern. Phase 0 Check 8 already verified the database
exists - no re-verification is needed here.

Verify clean state:

```bash
git status --porcelain
```

## Step 5b: Claim the Epic

Mark the epic in_progress so other sessions (and `br ready` scans) see it
as taken. Write a claim note recording the worktree path so a human can
locate the active session.

```bash
br --db "$MAIN_REPO_BEADS_DB" update <EPIC_ID> --status in_progress --notes "$(cat <<'EOF'
CLAIMED: /implement session on branch feat/<BRANCH_NAME>
Worktree: <WORKTREE_PATH>
EOF
)"
```

Do this AFTER worktree creation (not in Phase 0). If Phase 1 ends in
"Cancel" or worktree creation fails, we never claim the epic.

If the claim fails (br error, permissions), warn but do not abort -
implementation can still proceed, the epic just will not be visible as
in_progress to other sessions.

## Step 6: Create Artifact Directory and Determine Implementation Mode

Create the artifact directory for state files, subagent summaries, and PR
descriptions. Use **REPO_NAME** (computed in Step 1).

Set `ARTIFACT_DIR=/tmp/<REPO_NAME>-<EPIC_ID>` and create it safely.
Also record **ARTIFACT_BASENAME** - the shared base name for PR
artifacts (title and description). Including the epic ID and branch
slug in the filenames makes concurrent `/implement` sessions
distinguishable in editor tabs when reviewing drafts side by side.

```bash
ARTIFACT_DIR="/tmp/<REPO_NAME>-<EPIC_ID>"
ARTIFACT_BASENAME="<EPIC_ID>-<BRANCH_NAME>"

if [ -e "$ARTIFACT_DIR" ] || [ -L "$ARTIFACT_DIR" ]; then
  # Path already exists - verify it is safe before proceeding
  if [ -L "$ARTIFACT_DIR" ]; then
    echo "Error: $ARTIFACT_DIR is a symlink" >&2
    exit 1
  fi
  if [ ! -d "$ARTIFACT_DIR" ]; then
    echo "Error: $ARTIFACT_DIR exists but is not a directory" >&2
    exit 1
  fi
  if [ ! -O "$ARTIFACT_DIR" ]; then
    echo "Error: $ARTIFACT_DIR is not owned by current user" >&2
    exit 1
  fi
  chmod 700 "$ARTIFACT_DIR"
else
  # Path does not exist - create with restricted permissions in one step
  mkdir -m 700 "$ARTIFACT_DIR"
fi

# Final verification
if [ -L "$ARTIFACT_DIR" ] || [ ! -d "$ARTIFACT_DIR" ] || [ ! -O "$ARTIFACT_DIR" ]; then
  echo "Error: $ARTIFACT_DIR failed post-creation verification" >&2
  exit 1
fi
```

When the path already exists, all safety checks (not a symlink, is a
directory, owned by current user) run before any mutation. When the
path does not exist, `mkdir -m 700` creates it with restricted
permissions in a single invocation. A final verification catches
anything unexpected. If any check fails, stop and report the error.

The lead agent decides between inline and subagent mode. See
[bead-implementation.md](bead-implementation.md) for the decision criteria.
