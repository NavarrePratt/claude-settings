# Git-Spice (gs) - Stacked PR Workflow

Use git-spice (`gs`) to manage stacked branches and PRs for easier code review.

## CRITICAL: Never Submit Without Approval

**NEVER run `gs branch submit`, `gs stack submit`, or any submit command without explicit user approval.**

Submit commands push branches to remote and create/update PRs. Always:
1. Show the user what will be submitted (branches, commits)
2. Wait for explicit approval ("yes", "go ahead", "submit it")
3. Only then execute the submit command

This applies to all variations: `gs bs`, `gs ss`, `gs branch submit`, `gs stack submit`.

Local operations (branch create, navigate, restack, commit) are fine without approval.

## When to Use

- Large features that benefit from breaking into reviewable chunks
- Dependent changes that need sequential PRs
- When reviewers struggle with large diffs

## Core Concepts

- **Stack**: Linear chain of branches, each building on the previous
- **Trunk**: Target branch (main/master) - the base of all stacks
- **Upstack**: Branches above current branch
- **Downstack**: Branches below current branch (toward trunk)
- **Restacking**: Rebasing branches to maintain linear history after changes

## Setup

```bash
# Initialize git-spice in a repository
gs repo init
```

## Quick Reference

| Task | Command | Alias |
|------|---------|-------|
| Create stacked branch | `gs branch create NAME -m "msg"` | `gs bc` |
| Navigate up | `gs up` | |
| Navigate down | `gs down` | |
| Go to top of stack | `gs top` | |
| Go to bottom of stack | `gs bottom` | `gs b` |
| View stack tree | `gs log short` | `gs ls` |
| View with commits | `gs log long` | `gs ll` |
| Submit current branch PR | `gs branch submit` | `gs bs` |
| Submit entire stack | `gs stack submit` | `gs ss` |
| Restack after changes | `gs stack restack` | `gs sr` |
| Sync after PR merge | `gs repo sync` | `gs rs` |
| Move branch to new base | `gs branch onto TARGET` | `gs bon` |
| Track existing branch | `gs branch track` | `gs btr` |
| Delete branch | `gs branch delete` | `gs bd` |

## Common Workflows

### Creating a Stack

```bash
# Start from trunk (main)
git checkout main && git pull

# Create first feature branch
git add file1.txt
gs branch create feat-part1 -m "Add first part"

# Create second branch on top
git add file2.txt
gs branch create feat-part2 -m "Add second part"

# Submit all as PRs
gs stack submit --fill
```

### Making Mid-Stack Changes

```bash
# Navigate to branch needing changes
gs down  # or gs up, gs bottom, gs top

# Make changes and commit (auto-restacks upstack)
git add .
gs commit create -m "Fix issue"

# Update all PRs
gs stack submit
```

### After a PR is Merged

```bash
# Sync repo - deletes merged branches, rebases remaining
gs repo sync

# Update remaining PRs with new base
gs stack submit
```

### Amending Mid-Stack Commits

```bash
# Navigate to branch
gs down

# Make changes
git add .
gs commit amend  # Auto-restacks upstack branches

# Update PRs
gs stack submit
```

### Reorganizing a Stack

```bash
# Move current branch onto different base
gs branch onto new-base

# Insert new branch between existing ones
gs branch create new-branch --insert
```

## Best Practices

1. **Use `gs commit create` and `gs commit amend`** instead of plain git commands - they auto-restack upstack branches

2. **Keep stacks small** - 3-5 branches maximum for reviewability

3. **Run `gs repo sync` frequently** - keeps you current with trunk and cleans merged branches

4. **Use `--fill` with submit** - auto-populates PR titles/bodies from commits

5. **Check stack status** - `gs ls --cr-status` shows which PRs are merged/open

6. **Navigate with gs commands** - `gs up/down/top/bottom` are faster than git checkout

## Handling Conflicts

```bash
# If restack fails with conflicts
git status                    # See conflicting files
# ... resolve conflicts ...
git add .
gs rebase continue            # or: gs rc

# If stuck, abort and retry
gs rebase abort               # or: gs ra
```

## Integration with PR Workflow

- One branch = one PR
- PRs show dependency chain automatically
- Base branch updates propagate through `gs repo sync`
- Use GitHub/GitLab UI to merge, then sync locally

## State Storage

git-spice stores state in Git ref `refs/spice/data` - no external files needed. State travels with git operations.
