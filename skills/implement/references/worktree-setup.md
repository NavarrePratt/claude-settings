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

## Step 6: Determine Implementation Mode

The lead agent decides between inline and subagent mode. See
[bead-implementation.md](bead-implementation.md) for the decision criteria.

For subagent mode, create the artifact directory using **REPO_NAME** (computed
in Step 1). Use `mktemp -d` or verify directory ownership before writing to
avoid symlink attacks on the predictable `/tmp` path:

```bash
mkdir -p /tmp/<REPO_NAME>-<EPIC_ID>
```
