# Phase 5: Apply Cuts

Take the approved cuts from `$TMP_DIR/decisions.yaml` and apply them to the branch. Default strategy amends each cut into the commit that introduced the slop.

## Preconditions

- Working tree must be clean (pre-commit hook modifications re-staged).
- `decisions.yaml` exists and contains at least one finding with `disposition == "cut"`.
- If zero cuts, skip to Phase 6 with a noop.

## Step 1: Discover origin commits via git blame

For each finding marked `disposition: cut`:

```bash
git blame -L <line>,<line_end> --porcelain -- <file>
```

Extract the SHAs that introduced the cut lines. Populate `origin_commits`.

- Single SHA: set `primary_origin = <sha>` directly.
- Multiple SHAs: AskUserQuestion with options showing each commit's short sha + subject:

```
question: "Cut '<title>' at <file>:<line>-<line_end> spans multiple commits. Which commit should this cut amend into?"
header: "Ambiguous origin"
options:
  - { label: "<sha1> <subject1>", description: "Commit that introduced <N1> of the cut lines" }
  - { label: "<sha2> <subject2>", description: "Commit that introduced <N2> of the cut lines" }
  - { label: "Apply as new commit", description: "Create a fresh cleanup commit at HEAD instead of amending" }
```

If the user picks "Apply as new commit" for any finding, treat that specific cut as if `--single-commit` were set for it.

## Step 2: Group cuts by origin commit

Build a dictionary: `{origin_sha: [cut_1, cut_2, ...]}`. Order origin commits in reverse chronological order (newest first) to minimize conflict cascade during rebase.

## Step 3: Branch strategy selection

**Default (amend-per-origin):**

For the oldest origin commit in the group, start an interactive rebase:

```bash
OLDEST=<oldest origin sha>
git rebase -i "${OLDEST}^"
```

The rebase needs scripted editor input to mark each origin commit as `edit`:

```bash
# Use GIT_SEQUENCE_EDITOR to flip pick -> edit for target SHAs
export TARGET_SHAS="sha1 sha2 sha3"
GIT_SEQUENCE_EDITOR="python -c '
import sys, os
targets = os.environ[\"TARGET_SHAS\"].split()
path = sys.argv[1]
lines = open(path).readlines()
out = []
for line in lines:
    parts = line.split()
    if len(parts) >= 2 and parts[0] == \"pick\" and any(line.startswith(f\"pick {sha[:7]}\") for sha in targets):
        line = \"edit\" + line[4:]
    out.append(line)
open(path, \"w\").writelines(out)
'" git rebase -i "${OLDEST}^"
```

For each stop (one per origin commit):
1. Apply all cuts grouped under this origin using Edit tool.
2. Run pre-commit hooks if they exist (via a test `git commit --amend --dry-run`); re-stage any hook-modified files.
3. `git add <files touched>`.
4. `git commit --amend --no-edit`.
5. `git rebase --continue`.

Do NOT use `git rebase --continue` until all cuts for the current origin commit are applied and staged.

**`--single-commit` flag (or per-finding "Apply as new commit" selection):**

1. Ensure working tree is clean.
2. Apply all such cuts in the working tree with Edit tool.
3. `git add <files touched>`.
4. `git commit -m "grug: cut pass"` with body listing the cuts (`cut-classification` tags).

## Step 4: Handle rebase conflicts

If a `git rebase --continue` fails with a conflict:
1. Do NOT auto-resolve.
2. Surface conflicted files: `git status`.
3. AskUserQuestion:
   ```
   question: "Rebase conflict applying cut '<title>' to <origin_sha>. How to proceed?"
   header: "Rebase conflict"
   options:
     - { label: "Abort rebase", description: "git rebase --abort, restore pre-rebase state, drop this cut from the applied set" }
     - { label: "I will resolve manually", description: "Stop the skill; you resolve conflicts in your editor, then re-invoke /grug-cut-pass to resume" }
   ```
4. Record the outcome in the finding's `applied.status` and `applied.blocked_reason`.

## Step 5: Stacked-branch restack

If `SCOPE == "whole-stack"`, after finishing cuts on a branch:

```bash
gs stack restack
```

This propagates the amends through the rest of the stack locally. Do NOT run `gs stack submit` or any remote-facing gs command.

If restack fails, same handling as Step 4 (abort or user resolves).

## Step 6: Update findings state

For each cut, update `applied`:
- `status: applied` on success
- `status: blocked, blocked_reason: <reason>` on conflict
- `status: skipped` if user aborted this specific cut

Write back to `$TMP_DIR/decisions.yaml`.

## Verification before Phase 6

Sanity-check: after all cuts applied, `git diff <BASE_REF>..HEAD -- <touched files>` should show LESS content than before Phase 5. If the diff grew, something went wrong; abort with a recovery prompt.

## Outputs of Phase 5

- Branch with cuts amended into origin commits (or single cut-pass commit if flag set)
- Updated `$TMP_DIR/decisions.yaml` with applied statuses
- `$TMP_DIR/rebase-log.txt` capturing stdout/stderr of rebase operations for debugging
