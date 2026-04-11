# Commit Strategy

Wave-boundary commit heuristic for the /implement skill.

## When to Commit

Commit at **wave boundaries**, not after each individual bead. A wave is all
beads at the same depth in the dependency tree.

Exception: if a wave contains only one bead, commit immediately after that bead.

## Grouping Heuristic

After all beads in a wave complete (or are skipped), review accumulated changes
and group them into logical commits:

### Step 1: Survey changes

```bash
git diff --stat
git diff --name-only
```

### Step 2: Group by functional area

Apply these rules in order:

1. **Same-concern files go together**: source code and its configuration,
   middleware and its registration, a handler and its route definition.

2. **Test files commit with the code they test**: `foo.go` and `foo_test.go`
   belong in the same commit, not separate "add code" and "add tests" commits.

3. **Unrelated changes get separate commits**: if bead A modified auth code
   and bead B modified logging code, those are two commits even though they
   were in the same wave.

4. **Never group more than 5 beads worth of changes into a single commit**:
   if a wave has 6+ beads that all touch the same functional area, split into
   multiple commits by sub-concern.

### Step 3: Create commits

Use the `/commit` slash command (git-commit skill) for each logical group:

```bash
# Stage files for one logical group
git add path/to/related/files...

# Use /commit to create the commit with proper formatting
```

The git-commit skill analyzes git history to match the project's commit style,
creates atomic commits, and follows best practices (imperative mood, 50-char
subject, explanatory body).

## Commit Message Content

- Explain why the change was made, not what files were changed
- Reference the epic being implemented for context
- Do not include bead IDs in commit messages (internal tracking detail)
- Do not include specific counts of items (files, tests, etc.)

## Wave-Boundary Flow

```
Wave 1: [bead-a, bead-b, bead-c]
  -> implement bead-a (changes in working tree)
  -> implement bead-b (more changes)
  -> implement bead-c (more changes)
  -> wave complete: survey changes, group, commit

Wave 2: [bead-d]
  -> implement bead-d
  -> single bead wave: commit immediately

Wave 3: [bead-e, bead-f]
  -> implement bead-e
  -> implement bead-f
  -> wave complete: survey changes, group, commit
```

## Skipped Beads

Skipped beads may have left partial changes in the working tree. Before
committing at a wave boundary:

1. Check if any beads were skipped in this wave
2. If so, review `git diff` carefully for incomplete changes
3. Either: include the partial work if it is self-contained, or revert it
   with `git checkout -- <files>` if it would break the build

## ANSI Prevention

The `/commit` skill handles ANSI prevention automatically. Do not copy-paste
colored terminal output into commit messages. Write commit message text from
scratch.
