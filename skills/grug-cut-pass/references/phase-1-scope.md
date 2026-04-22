# Phase 1: Scope Detection

Determine which commits and files the cut pass will cover, and optionally build an upstack impact map for stacked branches.

## Default: single branch

Compute the file list and diff for the current branch vs BASE_REF:

```bash
git diff --name-only "$BASE_REF"...HEAD > "$TMP_DIR/file-list.txt"
git diff "$BASE_REF"...HEAD > "$TMP_DIR/diff.patch"
git log --oneline "$BASE_REF"..HEAD > "$TMP_DIR/commits.txt"
```

If `file-list.txt` is empty, report "no commits on this branch vs $BASE_REF, nothing to review" and exit cleanly (skip to Phase 8 with a no-op report).

## Stacked branches: ask for scope

If `STACKED=true` AND the current branch has upstack or downstack siblings in the gs stack, call AskUserQuestion:

```
question: "This branch is part of a git-spice stack. Scope the cut pass to which level?"
header: "Stack scope"
options:
  - { label: "Current branch only", description: "Review only this branch's commits. Simpler, faster. Cuts amend only into this branch's commits." }
  - { label: "Whole stack", description: "Review the entire stack from trunk up. Adds an upstack impact map so cuts can be routed to the right branch. Slower, more thorough." }
```

Detecting "has siblings":
```bash
# Number of branches in the stack minus 1 (the current branch)
siblings=$(gs ls --json 2>/dev/null | jq '[.branches[] | select(.name != env.CURRENT_BRANCH)] | length')
```

If `siblings == 0`, skip the question and use single-branch scope.

## Whole-stack: build the upstack impact map

When user selects whole-stack:

1. List all branches in the stack from trunk up:
   ```bash
   gs ll > "$TMP_DIR/stack-log.txt"
   ```

2. For each branch, capture its diff vs its parent:
   ```bash
   for branch in $(gs ls --json | jq -r '.branches[].name'); do
       parent=$(gs ls --json | jq -r ".branches[] | select(.name == \"$branch\") | .parent")
       git diff --name-only "$parent"..."$branch" > "$TMP_DIR/branch-$branch-files.txt"
   done
   ```

3. Build the cross-reference (which of *this branch's* files are also touched by other branches in the stack):
   ```
   files touched by current + touched by another branch = upstack impact
   ```

4. Write `$TMP_DIR/upstack-impact.md`:
   ```markdown
   # Upstack Impact Map

   Files in the current branch also touched by other stack branches. Cuts in
   these files risk conflicts with upstack work.

   | File | Also touched by |
   |------|-----------------|
   | pkg/auth/middleware.go | feat-audit-logs (parent), feat-auth-tokens (child) |
   ```

This file is passed into the Codex prompt as UPSTACK_IMPACT context so findings flagged as touching upstack files get an extra "note: this touches other branches" tag.

## FILE_LIST for Codex

The master FILE_LIST passed to Codex:
- Single-branch: just `$TMP_DIR/file-list.txt`
- Whole-stack: union of all branch file lists in the stack

Store as `$TMP_DIR/file-list.txt` (overwrite if whole-stack).

## Outputs of Phase 1

- `$TMP_DIR/file-list.txt` - files to review
- `$TMP_DIR/diff.patch` - full diff text for Codex
- `$TMP_DIR/commits.txt` - commits on scope
- `$TMP_DIR/upstack-impact.md` - only if whole-stack
- `SCOPE` state value: `single` or `whole-stack`
