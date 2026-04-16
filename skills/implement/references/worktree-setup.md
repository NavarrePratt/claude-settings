# Worktree Setup

Create and configure an isolated worktree for implementation.

## Step 1: Compute Branch Name

If **BRANCH_NAME** was provided in `$ARGUMENTS`: use it as-is.

Otherwise, slugify **EPIC_TITLE**:
- Convert to lowercase
- Replace spaces with hyphens
- Strip characters that are not alphanumeric or hyphens
- Collapse consecutive hyphens
- Trim trailing hyphens

Record as **BRANCH_NAME**.

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

Verify the beads database is accessible:

```bash
br where
```

If it fails, the worktree may not share bead configuration. The `BR_HOME`
environment variable can point br at the main repo's `.beads/` directory.

Verify clean state:

```bash
git status --porcelain
```

## Step 6: Create Artifact Directory and Determine Implementation Mode

Create the artifact directory for state files, subagent summaries, and PR
descriptions. Use **REPO_NAME** (computed in Step 1).

Set `ARTIFACT_DIR=/tmp/<REPO_NAME>-<EPIC_ID>` and create it safely:

```bash
ARTIFACT_DIR="/tmp/<REPO_NAME>-<EPIC_ID>"

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
