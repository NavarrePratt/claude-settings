---
name: postmerge
description: >
  Reconcile local git state after a PR has merged upstream. Use when the user
  says "the PR merged", "PR is merged, clean up", "pull down main", "reconcile
  after merge", "clean up the worktree", "wrap up this branch", or any
  variation of "post-merge cleanup". Use this skill proactively whenever a
  user mentions that a PR from an /implement worktree has landed and local
  state needs to catch up - even if they only ask for part of it (e.g. just
  "pull main"). This skill exits any active worktree, removes it, deletes
  the merged feature branch, fast-forwards the default branch from the
  remote, and sweeps /implement artifact directories from /tmp.
argument-hint: "[pr-number-or-url] [--auto]"
---

# Post-Merge Cleanup

Reconcile local state after a PR merges. This skill is the inverse of
`/implement` Phase 2: where `/implement` creates a worktree, branch, and
`/tmp/<repo>-<epic>` artifact directory, `/postmerge` tears them down once
the PR has landed.

## When to use

Trigger on any variation of:

- "PR merged, clean up"
- "pull down main"
- "clean up the worktree"
- "wrap up this branch"
- "reconcile after merge"
- "postmerge"

If the user only asks for part of the workflow (e.g. "pull main"), still
invoke the skill but skip phases that are not needed - see Phase 0 for the
self-assessment.

## Arguments

- **Optional positional**: a PR number (`14`) or URL
  (`https://github.com/org/repo/pull/14`). If provided, the skill uses it
  to verify the merge rather than resolving from the current branch.
- **`--auto`**: skip prompts on the safe path - PR verified merged, branch
  merged to the remote default branch, artifact directory unambiguous.
  `--auto` still refuses risky operations (`git branch -D`, deleting
  unowned `/tmp` dirs, discarding uncommitted changes).

Parse `--auto` anywhere in arguments; treat the first non-flag token as
the PR identifier.

## Why this exists

Manual post-merge cleanup is repetitive and has sharp edges:

- `ExitWorktree` refuses to remove worktrees it did not itself create.
- `git pull --ff-only` silently no-ops if `git fetch` has not run.
- `/tmp` artifact paths can live on for days, accumulating state.
- Forgetting to delete the local branch leaves `git branch` cluttered with
  dead feature branches.

Each step individually is trivial; chaining them correctly each time is
the value this skill adds.

---

## Phase 0: Sanity checks and state survey

Before any mutation, survey the current state. Everything destructive in
later phases needs this context.

### Commands to run in parallel

```bash
git rev-parse --show-toplevel 2>/dev/null
git rev-parse --is-inside-work-tree 2>/dev/null
git worktree list
git branch --show-current
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'
which gh
```

Record:

- **MAIN_REPO**: the main repo root. If cwd is inside a worktree, this is
  the parent repo, not the worktree. Use `git worktree list` - the first
  line is the main working tree.
- **CURRENT_WORKTREE**: if cwd is inside a worktree (not the main repo),
  the worktree path. Otherwise `null`.
- **DEFAULT_BRANCH**: from `git symbolic-ref`. Fall back to the first of
  `main`, `master`, `trunk` that exists locally if the symbolic ref is
  missing. Match `/implement`'s BASE_REF detection for consistency.
- **GH_AVAILABLE**: whether `gh` is on PATH. If not, PR-merge verification
  falls back to comparing commits between the branch and origin/default.

### Failure conditions

Stop with a clear error if:

- Not inside a git repo (`git rev-parse --show-toplevel` failed).
- No remote named `origin` (we cannot fetch or verify).

Continue with a warning if `gh` is missing - the skill can still work, it
just cannot query the PR API.

---

## Phase 1: Identify target branch and PR

### If the user supplied a PR identifier

Use it directly with `gh pr view <id> --json headRefName,state,mergedAt,url`.

### If the session is inside a worktree

The branch is `git -C <worktree> branch --show-current`. Cross-check with
`git worktree list` to confirm the worktree-to-branch mapping.

### If the session is in the main repo

List worktrees. If exactly one non-main worktree exists and its branch
has no unpushed commits (i.e. the branch has been pushed and likely
merged), propose that worktree. If multiple worktrees exist, prompt via
AskUserQuestion:

```
questions: [{
  question: "Multiple worktrees present. Which one is the post-merge cleanup target?",
  header: "Worktree",
  options: [
    // one option per worktree, label = branch name, description = path
    { label: "feat/<name>", description: "<path>" },
    ...
  ],
  multiSelect: false
}]
```

With `--auto`: if exactly one non-main worktree exists, pick it silently.
If multiple exist, still prompt - `--auto` does not guess between real
options.

Record:

- **TARGET_WORKTREE**: the worktree path (or `null` if the user is just
  asking for a "pull main" with no worktree involved).
- **TARGET_BRANCH**: the branch to delete. Must not equal DEFAULT_BRANCH.

### Safety gates

Refuse to proceed if `TARGET_BRANCH == DEFAULT_BRANCH`. That would mean
deleting `main`, which is never what the user wants.

---

## Phase 2: Verify the PR is merged

Cleanup is destructive. Verify the PR actually merged before touching
local state.

### Preferred: gh

```bash
gh pr list --head "$TARGET_BRANCH" --state merged --json number,url,mergedAt,mergeCommit --limit 1
```

A non-empty result with `mergedAt` set confirms the merge. If the result
is empty, try `--state all` to see if the PR exists at all (open, closed
without merge).

### Fallback when gh is unavailable

After `git fetch`, check whether the branch's tip commit is reachable
from `origin/<DEFAULT_BRANCH>`:

```bash
git fetch origin --quiet
git merge-base --is-ancestor "$(git rev-parse "$TARGET_BRANCH")" "origin/$DEFAULT_BRANCH" && echo merged || echo not-merged
```

If `merged`, the branch's commits are part of the default branch. This
covers both merge-commit and squash-merge cases where the exact commit
still appears.

**Squash-merge edge case**: GitHub's squash strategy produces a new
commit hash; `--is-ancestor` will return false. If `gh` is available,
prefer the API check; if not and the fallback fails, prompt the user:

```
questions: [{
  question: "Could not verify PR #<N> is merged via git. Proceed with cleanup anyway?",
  header: "Unverified",
  options: [
    { label: "Cancel", description: "Abort - verify manually, re-run with --auto once confirmed" },
    { label: "Proceed", description: "I have confirmed the merge out-of-band" }
  ],
  multiSelect: false
}]
```

With `--auto`: refuse to proceed. This is exactly the kind of risky
guess `--auto` must not make.

Record:

- **PR_URL** and **PR_NUMBER** if gh produced them. Used in the final
  report.

---

## Phase 3: Exit the worktree if active

If CURRENT_WORKTREE matches TARGET_WORKTREE, we are about to delete the
directory the session is running inside. Leave it first.

### If the session entered via EnterWorktree

```
ExitWorktree(action: "keep")
```

Do not pass `action: "remove"` - ExitWorktree refuses to remove worktrees
it did not itself create (which is the common case for `/implement`
worktrees entered via the `path=` argument). `action: "keep"` always
works and leaves the on-disk worktree for Phase 4 to delete explicitly.

After ExitWorktree, the session cwd is the main repo root.

### If the session was started inside the worktree (no EnterWorktree call)

Change the working directory to MAIN_REPO via Bash. Subsequent tool calls
will operate on the main repo.

### If CURRENT_WORKTREE does not match TARGET_WORKTREE

Nothing to exit. Skip to Phase 4.

---

## Phase 4: Remove the worktree

```bash
git -C "$MAIN_REPO" worktree remove "$TARGET_WORKTREE"
```

### Handling uncommitted changes

`git worktree remove` refuses if the worktree is dirty. If that happens:

```
questions: [{
  question: "Worktree <path> has uncommitted changes. Remove anyway?",
  header: "Dirty tree",
  options: [
    { label: "Cancel (recommended)", description: "Inspect the changes first - they may be work in progress" },
    { label: "Force remove", description: "Discard all uncommitted changes and remove the worktree" }
  ],
  multiSelect: false
}]
```

With `--auto`: never force. Stop and surface the dirty files.

"Force remove" passes `--force`. Only use this after explicit user
approval - per `~/.claude/CLAUDE.md`, destructive operations need
confirmation even when running in automated modes.

---

## Phase 5: Delete the local branch

```bash
git -C "$MAIN_REPO" branch -d "$TARGET_BRANCH"
```

`-d` (lowercase) refuses to delete unmerged branches. That is the safe
default.

### If -d fails (branch shows as unmerged locally)

This often happens right after a squash-merge: the local branch commits
are not yet reachable from `main` locally because we have not pulled.
Run Phase 6 first (fetch + pull), then retry `-d`.

If `-d` still fails after the pull, the branch has local commits that
never made it into the PR. Prompt:

```
questions: [{
  question: "Branch <name> has commits not merged to <default>. Force-delete?",
  header: "Unmerged",
  options: [
    { label: "Cancel (recommended)", description: "Keep the branch; investigate before deleting" },
    { label: "Force delete", description: "git branch -D - commits will be unrecoverable unless the branch is in the reflog" }
  ],
  multiSelect: false
}]
```

With `--auto`: refuse to force. `-D` destroys recoverable-only-via-reflog
commits and is never safe to automate.

---

## Phase 6: Fetch and fast-forward the default branch

```bash
git -C "$MAIN_REPO" fetch origin
git -C "$MAIN_REPO" checkout "$DEFAULT_BRANCH"
git -C "$MAIN_REPO" pull --ff-only origin "$DEFAULT_BRANCH"
```

### Why `--ff-only`

It refuses to create a merge commit if `HEAD` has diverged. Divergence
on the default branch in a cleanup workflow is always a surprise -
surface it rather than paper over it with a merge commit.

### If the fast-forward fails

The local default branch has commits not in origin. Prompt:

```
questions: [{
  question: "Local <default> has diverged from origin. Options:",
  header: "Diverged",
  options: [
    { label: "Inspect and cancel (recommended)", description: "Stop and show git log origin/<default>..<default>" },
    { label: "Reset to origin (destructive)", description: "git reset --hard origin/<default> - local commits will be lost" }
  ],
  multiSelect: false
}]
```

With `--auto`: cancel and report. Never reset --hard without explicit
approval (`~/.claude/CLAUDE.md` rule).

### If the user was not already on DEFAULT_BRANCH

Record the original branch so the final report notes the checkout
change. Do not silently leave the user on a different branch than they
started on unless the original branch was the one we just deleted.

---

## Phase 7: Sweep /implement artifact directories

`/implement` creates `/tmp/<REPO_NAME>-<EPIC_ID>/` with state.json and
a pair of PR artifacts keyed by epic ID and branch slug
(`<EPIC_ID>-<BRANCH_NAME>.md` and `<EPIC_ID>-<BRANCH_NAME>.txt`). After
merge, these are safe to remove unless the user wants to keep the PR
body for reference.

### Discovery

```bash
REPO_NAME="$(basename "$MAIN_REPO")"
find /tmp -maxdepth 1 -user "$(whoami)" -type d -name "${REPO_NAME}-bd-*" 2>/dev/null
```

The glob tolerates dotfile repo names (`/Users/npratt/.claude` →
`.claude-bd-<epic>`). The `-user` filter avoids touching directories
owned by other accounts.

### Match the target worktree to its artifact dir

If TARGET_WORKTREE has a corresponding state.json, it will contain
`epic_id`. Read it to get the exact artifact dir:

```bash
EPIC_ID="$(jq -r .epic_id "/tmp/${REPO_NAME}-<?>/state.json" 2>/dev/null)"
```

Iterate the candidate dirs from the `find` command above and read each
state.json to find the one whose `branch_name` matches TARGET_BRANCH.
That is the directory to sweep.

### Confirmation

Unless `--auto` is set and exactly one artifact dir matches
unambiguously, prompt:

```
questions: [{
  question: "Remove artifact directory /tmp/<repo>-<epic>/ (state.json and the PR draft files)?",
  header: "Artifact dir",
  options: [
    { label: "Remove (recommended)", description: "Sweep the artifact directory" },
    { label: "Keep", description: "Leave it for manual review or reuse of the PR body" }
  ],
  multiSelect: false
}]
```

### Leftover review/fix temp dirs

`/team-branch-review` and `/team-branch-fix` use
`/tmp/review-<team>/` and `/tmp/fix-<team>/`. Those skills normally
clean up after themselves, but a crashed run can leave orphans. List
any that match the deleted branch's team name pattern, prompt once for
a bulk sweep.

---

## Phase 8: Final report

Print a concise summary:

```
Post-merge cleanup complete.

Merged PR: <PR_URL or "not linked via gh">
Worktree removed: <path or "none">
Branch deleted: <branch>
Default branch: <name> now at <short sha> <subject>
Artifacts swept: <list of /tmp paths> (or "none")
Skipped: <anything that was declined or errored>
```

If any phase was skipped (dirty tree kept, unmerged branch not force-
deleted, diverged main not reset), list it under "Skipped" so the user
can follow up.

---

## Pairing with Monitor: waiting for a PR to merge

`/postmerge` assumes the PR is already merged. If the user wants to
kick off cleanup the moment the PR lands, pair the skill with the
Monitor tool to stream PR state:

```bash
# Watches PR <N>; emits exactly once on merge, close, or repeated gh
# failures. Then the session receives a notification and can invoke
# /postmerge.
while true; do
  out="$(gh pr view <N> --json state,mergedAt,url 2>&1)" || { echo "gh-error: $out"; exit 1; }
  state="$(echo "$out" | jq -r .state)"
  merged="$(echo "$out" | jq -r .mergedAt)"
  case "$state" in
    MERGED) echo "MERGED: $(echo "$out" | jq -r .url) at $merged"; exit 0 ;;
    CLOSED) echo "CLOSED without merge: $(echo "$out" | jq -r .url)"; exit 0 ;;
  esac
  sleep 60
done
```

Use Monitor with `persistent: true` for long watches:

```
Monitor(
  description: "PR #<N> merge watch",
  command: <above script>,
  persistent: true
)
```

The script exits on any terminal state (merged, closed, sustained gh
error). The exit produces a notification. On a MERGED event, the user
or the session can invoke `/postmerge <N>` to handle the cleanup.

This pairing is intentionally *not* baked into `/postmerge` itself. The
cleanup skill stays deterministic and idempotent; the watch-for-merge
concern belongs in Monitor because that is what Monitor is for. Keep
them composable.

---

## Error Handling

| Scenario | Recovery |
|----------|----------|
| Not in a git repo | Stop with clear message |
| No remote named origin | Stop - cannot fetch or verify |
| gh not available | Warn, fall back to git-based merge verification |
| Target branch == default branch | Refuse - never delete main |
| PR not merged (gh confirms) | Stop, report PR URL and state |
| PR merge status unverifiable | Prompt for explicit confirmation; `--auto` refuses |
| Worktree has uncommitted changes | Prompt; `--auto` refuses to force |
| Local branch has unmerged commits | Run Phase 6 first, retry; if still unmerged, prompt for -D |
| Default branch diverged from origin | Prompt; `--auto` refuses to reset |
| Artifact dir ambiguous (multiple candidates) | Prompt to pick one or skip |
| Artifact dir not owned by current user | Skip with note in report |

## Guidelines

- **Idempotent**: running `/postmerge` twice should be safe. If the
  worktree is already gone, skip Phase 4 cleanly. If the branch is
  already deleted, skip Phase 5. Only fail when there is something
  dangerous to skip.
- **Verify before destroying**: Phase 2 (PR merged?) and Phase 6
  (fast-forward, not merge) are the two safety gates. Do not relax
  either one in `--auto` mode.
- **Defer to the user on risky calls**: force-remove, force-delete,
  reset-hard all require explicit confirmation. `--auto` means "skip
  prompts on the safe path", not "take the fast path through danger".
- **Surface, do not silence**: if a phase is skipped, the final report
  names it. The user should never learn about skipped cleanup by
  noticing later.
- **Compose with Monitor for the wait**: `/postmerge` handles cleanup;
  Monitor handles the pre-merge watch. See the Pairing section.
