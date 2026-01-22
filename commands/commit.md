## Context

- Current git status: !`git status`
- Current git diff (staged and unstaged changes): !`git diff HEAD`
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -10`

## Task

Create logically grouped, atomic commits based on the above context.
Use the git-commit skill for commit message formatting and best practices.
Use partial adds (`git add -p`) when a file contains multiple unrelated changes.

## Beads Sync

Before committing, check if `.beads/` exists and is tracked by git:

```bash
[ -d .beads ] && ! git check-ignore -q .beads/ 2>/dev/null && echo "beads tracked" || echo "beads ignored or missing"
```

If beads is tracked (not gitignored):
1. Run `br sync --flush-only` to export any pending changes to JSONL
2. Stage `.beads/issues.jsonl` along with other changes

$ARGUMENTS
