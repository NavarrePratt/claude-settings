---
name: implement
description: >
  Implement an epic's beads in a worktree with logical commits, multi-agent
  review, fix pipeline, and PR description generation. Replaces the manual
  atari-start -> review -> fix -> clean-copy -> PR workflow.
argument-hint: "<epic-id> [branch-name]"
---

# Implement Epic

Resolve an epic into an ordered execution plan, then implement each bead in a
worktree with logical commits, review, and PR generation.

## Reference Files

This skill uses reference files in the `references/` directory. Load referenced
files with the Read tool before executing the relevant phase.

| Reference | Used in |
|-----------|---------|
| [epic-resolution.md](references/epic-resolution.md) | Phase 1 - intersection algorithm |
| [bead-implementation.md](references/bead-implementation.md) | Phase 2-3 - worktree setup and bead implementation |
| [commit-strategy.md](references/commit-strategy.md) | Phase 3 - logical commit creation |
| [review-fix-pipeline.md](references/review-fix-pipeline.md) | Phase 4 - review and fix pipeline |
| [pr-description.md](references/pr-description.md) | Phase 5 - PR description generation and Phase 6 handoff |

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`
- Git remotes: !`git remote -v | head -4`

## Instructions

You are implementing an epic's beads end-to-end. This skill orchestrates the
full workflow from resolving which beads to implement through to producing a
reviewable branch.

---

### Phase 0: Precondition Checks

Verify all prerequisites before proceeding. Run each check in order and stop
at the first failure with a clear error message.

**Check 1: br CLI available**

```bash
which br
```

If missing:
> "The `br` CLI is required but not found in PATH. Install it or ensure it is on your PATH."

**Check 2: Not already in a worktree**

```bash
git rev-parse --is-inside-work-tree
```

Then check whether `.git` is a file (worktree) or directory (main repo):

```bash
test -f "$(git rev-parse --show-toplevel)/.git" && echo "worktree" || echo "main-repo"
```

If the result is "worktree":
> "You are already inside a git worktree. Exit the current worktree first (`/exit-worktree` or `ExitWorktree`), then re-run `/implement`."

**Check 3: Epic ID provided**

Parse `$ARGUMENTS` to extract the epic ID (first whitespace-delimited token).

If `$ARGUMENTS` is empty or the first token does not look like a bead ID:
> "Usage: `/implement <epic-id> [branch-name]`
>
> Provide the epic's bead ID as the first argument. Example: `/implement bd-xxx`"

Record the epic ID as **EPIC_ID**.

**Check 4: Epic exists**

```bash
br show <EPIC_ID> --json
```

If the command fails or returns an error:
> "Epic `<EPIC_ID>` not found. Verify the ID with `br show <EPIC_ID>` or list epics with `br list --type epic`."

Record the epic title from the JSON output as **EPIC_TITLE**.

**Check 5: Epic has children**

Check whether any bead lists this epic as its parent using the parent-field
scan (same approach as Phase 1 Step 1, but only checking for existence):

```bash
br list --json --limit 0
```

Extract all IDs, then batch-fetch parent fields:

```bash
br show <id1> <id2> ... --json
```

Scan the results for any bead whose `parent` field matches EPIC_ID. If none
found:
> "Epic `<EPIC_ID>` (`<EPIC_TITLE>`) has no child beads. Add beads to the epic first:
> ```
> br create \"Task title\" --status deferred --description \"...\" --json
> br dep add <new-bead-id> <EPIC_ID> --type parent-child
> ```"

**Check 6: No dependency cycles**

```bash
br dep cycles
```

If the output indicates cycles exist:
> "Dependency cycles detected in the bead graph. Resolve them before proceeding:
> ```
> <cycle output>
> ```
> Use `br dep remove` to break the cycle."

**Check 7: Parse optional branch name**

Parse the second whitespace-delimited token from `$ARGUMENTS` as an optional
branch name. If not provided, it will be generated later (in Phase 2) from
the epic title.

Record as **BRANCH_NAME** (or null if not provided).

All preconditions passed. Proceed to Phase 1.

---

### Phase 1: Resolve Epic to Execution Plan

See [epic-resolution.md](references/epic-resolution.md) for the full algorithm.
Read that file with the Read tool before executing this phase.

**Step 1: Find all beads under this epic**

NOTE: `br dep tree` does NOT work for finding children. It queries
`get_dependencies()` (what an issue depends on), not `get_dependents()`
(what depends on an issue). For parent-child relationships, children point
to their parent, so querying from the parent returns nothing.

Instead, use `br show` which includes the `parent` field (unlike `br list`
which omits it).

Algorithm:

1. Get all bead IDs:
   ```bash
   br list --json --limit 0
   ```
   Extract the `id` field from each item.

2. Batch-fetch parent fields (br show accepts multiple IDs):
   ```bash
   br show <id1> <id2> <id3> ... --json
   ```
   Process in batches of ~50 IDs if the list is large.

3. Build the descendant set iteratively:
   - Initialize the set with just EPIC_ID
   - Scan all beads: if a bead's `parent` field matches any ID in the set,
     add that bead to the set
   - Repeat until no new beads are added (handles nested sub-epics)
   - Remove EPIC_ID itself from the final descendant set

This matches atari's `buildDescendantSet()` in internal/workqueue/queue.go.

Record each descendant's status, priority, and dependencies from the
`br show` output.

**Step 2: Get ready beads**

```bash
br ready --json --limit 0
```

Record the full ready set.

**Step 3: Compute intersection**

Find beads that appear in BOTH the descendant set AND the ready set. These
are the beads that belong to this epic and are currently unblocked.

**Step 4: Classify all descendants**

For each bead in the descendant set, classify by status:

- **closed**: exclude from plan, record as "already done"
- **in_progress**: warn user - may be claimed by another session, skip to avoid
  duplicate work
- **ready** (in ready set): include in execution plan
- **open/deferred** (not in ready set): record as "blocked", identify blocking
  dependency

**Step 5: Determine execution order**

Sort executable beads using their blocking dependencies to compute waves:

- **Wave 1**: beads with no blocking dependencies (or whose blockers are all
  already closed)
- **Wave 2**: beads whose blockers are all in Wave 1
- **Wave N**: beads whose blockers are all in earlier waves

Within each wave, sort by priority ascending (P0 before P1 before P2).

If no dependency chain exists (all beads are independent), put them all in
Wave 1 and sort by priority.

**Step 6: Handle edge cases**

- **No ready beads**: report blocked beads and their blockers, exit
- **Partially ready**: show ready vs blocked, proceed with ready beads
- **All done**: report "all beads in epic already closed", exit
- **Truncated tree**: warn about depth cutoff if any node has `truncated: true`
- **In-progress beads**: warn and skip, suggest `br update <id> --status open`
  if stale

**Step 7: Present execution plan**

Display to the user:

```
Execution Plan for Epic: <EPIC_TITLE> (<EPIC_ID>)

Wave 1:
  bd-xxx: Title (P2)
  bd-yyy: Title (P1)

Wave 2:
  bd-zzz: Title (P2)

Already completed: bd-aaa, bd-bbb
In progress (skipped): bd-ccc
Blocked (not ready): bd-ddd (blocked by bd-eee)

Total: N beads to implement across M waves.
```

**Step 8: Wait for user confirmation**

Use AskUserQuestion to confirm before proceeding:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "Proceed with implementing N beads across M waves?",
    header: "Execute plan",
    options: [
      { label: "Proceed", description: "Start implementing beads in wave order" },
      { label: "Skip some", description: "Select specific beads to implement" },
      { label: "Cancel", description: "Abort without implementing anything" }
    ],
    multiSelect: false
  }]
```

If "Skip some": present each bead via AskUserQuestion for include/exclude,
then rebuild the plan with only included beads.

If "Cancel": exit the skill with no changes.

If "Proceed": continue to Phase 2.

---

### Phase 2: Create Worktree

**Step 1: Compute branch name**

If **BRANCH_NAME** was provided in `$ARGUMENTS`: use it as-is.

Otherwise, slugify **EPIC_TITLE**:
- Convert to lowercase
- Replace spaces with hyphens
- Strip characters that are not alphanumeric or hyphens
- Collapse consecutive hyphens
- Truncate to 40 characters

Record as **BRANCH_NAME**.

**Step 2: Check for existing worktree**

```bash
git worktree list
```

Check if `.claude/worktrees/implement-<BRANCH_NAME>` already exists in the
worktree list or on the filesystem.

If it exists, use AskUserQuestion:

```
Call AskUserQuestion tool with:
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

If "Reuse": change working directory to the existing worktree path and skip
to Step 5.

If "Abort": exit the skill.

Also check if the branch name already exists:

```bash
git branch --list "feat/<BRANCH_NAME>"
```

If the branch exists but no worktree uses it, warn the user and offer to use
a different name or reuse the branch.

**Step 3: Create the worktree**

```bash
git worktree add .claude/worktrees/implement-<BRANCH_NAME> -b feat/<BRANCH_NAME> main
```

Record the full worktree path as **WORKTREE_PATH**.

**Step 4: Change working directory**

Change the working directory to WORKTREE_PATH. All subsequent commands execute
inside the worktree.

**Step 5: Verify environment**

Verify the beads database is accessible from the worktree:

```bash
br where
```

This should show the main repo's `.beads/` directory. If it fails, the worktree
may not share the bead configuration. Warn the user.

Verify the worktree is clean:

```bash
git status --porcelain
```

Should be empty. If not, warn about unexpected files.

**Step 6: Determine implementation mode**

Read [bead-implementation.md](references/bead-implementation.md) with the Read tool.

Count the total beads in the execution plan (from Phase 1).

- If 5 or fewer beads: use **inline mode** (lead agent implements directly)
- If 6 or more beads: use **subagent mode** (spawn one agent per bead)

For subagent mode, create the summary directory:

```bash
mkdir -p /tmp/implement-<BRANCH_NAME>
```

Record the implementation mode for Phase 3.

---

### Phase 3: Implement Beads

Read [bead-implementation.md](references/bead-implementation.md) and
[commit-strategy.md](references/commit-strategy.md) with the Read tool before
starting this phase.

Process beads wave by wave, in the order determined by Phase 1.

**For each wave:**

Initialize a tracking list for this wave: beads completed, beads skipped.

**For each bead in the wave:**

Execute the per-bead cycle from bead-implementation.md:

1. **Claim**: `br update <bead_id> --status in_progress --json`

2. **Load context**: `br show <bead_id> --json`
   - Extract the full description
   - Parse the `## Verification` section to find verification commands
     (backtick-enclosed commands in checkbox items)

3. **Implement** (mode-dependent):

   **Inline mode**: implement the bead directly.
   - Read the bead description carefully
   - Use Read, Edit, Write, Bash, Grep, Glob tools
   - Follow existing project patterns
   - Stay within the bead's scope

   **Subagent mode**: read templates/bead-prompt.md and substitute variables:
   - Replace `BEAD_ID` with the bead identifier
   - Replace `BEAD_TITLE` with the bead title
   - Replace `BEAD_DESCRIPTION` with the full description
   - Replace `BEAD_PARENT` with the epic ID (EPIC_ID)
   - Replace `WORKTREE_PATH` with the worktree absolute path
   - Replace `VERIFICATION_COMMANDS` with extracted verification commands
   - Replace `PRIOR_SUMMARIES` with concatenated summaries from prior beads
     (or "No prior beads implemented yet." for the first bead)
   - Replace `BRANCH_NAME` with the branch name

   Spawn the agent with a 10-minute timeout expectation:
   ```
   Agent(
     description: "Implement <bead_id>",
     prompt: <rendered template>,
     mode: "bypassPermissions"
   )
   ```

   If the agent has not completed after 10 minutes, treat as a timeout:
   reset the bead to open status and record as skipped.

   After agent completes, read the summary file:
   `/tmp/implement-<BRANCH_NAME>/<bead_id>-summary.md`

4. **Verify**: run each extracted verification command.
   - If all pass: proceed to close
   - If any fail: attempt one fix, then re-run all verification
   - If still failing: handle as failure

5. **Close on success**:
   ```bash
   br close <bead_id> --reason "<what was accomplished>"
   ```
   Record bead as completed in wave tracking.

6. **Handle failure**:
   ```bash
   br update <bead_id> --status open --notes "SKIPPED: <what failed>"
   ```
   Record bead as skipped in wave tracking.

**After all beads in the wave are processed:**

Follow the commit strategy from commit-strategy.md:

1. Survey changes: `git diff --stat` and `git diff --name-only`

2. Group changes by functional area:
   - Same-concern files together
   - Test files with the code they test
   - Unrelated changes in separate commits
   - Never group more than 5 beads worth of changes per commit

3. For each logical group, stage the relevant files and use `/commit`
   (the git-commit skill) to create the commit.

4. If the wave had only one bead: commit immediately with all its changes.

5. If all beads in the wave were skipped: check for partial changes.
   Revert incomplete work with `git checkout -- <files>` if it would
   break the build. Otherwise leave it for the user to review.

**After all waves are processed:**

Report a summary:

```
Implementation Complete

Completed: bd-xxx, bd-yyy, bd-zzz
Skipped: bd-aaa (reason), bd-bbb (reason)
Commits: N commits on feat/<BRANCH_NAME>

Worktree: <WORKTREE_PATH>
```

If any beads were skipped, note that they were reset to open status and
can be retried.

---

### Phase 4: Review and Fix Pipeline

Read [review-fix-pipeline.md](references/review-fix-pipeline.md) with the Read
tool before executing this phase.

After all beads are implemented and committed (Phase 3 complete), run an
automated review and fix cycle using the /team-branch-review and
/team-branch-fix skills.

**Step 1: Pre-Review Check**

Verify the branch is reviewable:

```bash
git status --porcelain
```

If uncommitted changes exist, run /commit first. Then verify commits exist:

```bash
git log --oneline main..HEAD
```

If no commits exist vs main, skip the review pipeline entirely and proceed
to Phase 5.

See review-fix-pipeline.md Check 1 and Check 2 for the full procedure
including the AskUserQuestion flow for dirty-tree edge cases.

**Step 2: Invoke Review**

Run /team-branch-review to review all commits on this branch.

The review skill detects branch and base commit automatically. It spawns
reviewer agents, validates findings with Codex, and produces a report.

**Step 3: Compute Actionable Findings and Act**

Read the review report from the conversation. Record the outcome label:

```
## Outcome: [APPROVED | NEEDS REVISION | MANUAL REVIEW REQUIRED]
```

The outcome label is recorded for the PR description but does NOT gate fix
execution. Instead, parse the findings table and compute the actionable
finding count:

- **Actionable**: Confirmed + Severity-adjusted + non-disputed Codex-only
- **Not actionable**: Disputed (reviewer disagreement requires human judgment)

**If no actionable findings**: skip the fix pipeline. For disputed-only cases
(MANUAL REVIEW REQUIRED), present disputed findings to user and offer
"Skip" or "Abort".

**If actionable findings exist** (any outcome label): run /team-branch-fix
with `--auto` flag and the review report. The `--auto` flag auto-selects
actionable findings and skips disputed ones, keeping /implement autonomous.

After fix completes, proceed to Step 4.

See review-fix-pipeline.md for the full actionable finding definition,
branching logic, and AskUserQuestion templates.

**Step 4: Post-Fix Cleanup**

After the fix pass completes:

1. Verify clean git state: `git status --porcelain`
2. If uncommitted changes remain and the fix pass reported success,
   run /commit. If the fix pass reported verification failures, do NOT
   auto-commit - surface the dirty state to the user via AskUserQuestion
   and let them decide whether to commit, revert, or fix manually.
3. Record review outcome for Phase 5 PR description:
   - Outcome label (APPROVED, NEEDS REVISION, MANUAL REVIEW REQUIRED)
   - Whether fixes were applied, deferred, or skipped
   - Any critical findings that remain

**Bounded Iteration Rule**

**CRITICAL: Do NOT re-run /team-branch-review after /team-branch-fix
completes.** One review + one fix pass is the maximum. Reasoning:

- /team-branch-review spawns 2-6 agents with Codex validation (expensive)
- /team-branch-fix spawns N fixer agents with Codex validation (expensive)
- Looping risks unbounded cost and context exhaustion

If critical findings remain after the fix pass, report them in the Phase 5
PR description rather than looping. The user can always run another review
cycle manually after /implement completes.

---

### Phase 5: PR Description Generation

Read [pr-description.md](references/pr-description.md) with the Read tool
before executing this phase.

Generate a PR description from the full implementation context and save it
for use when creating the PR.

**Step 1: Gather Inputs**

Collect all context needed for the PR description. These should already be
available from prior phases:

1. Epic context: **EPIC_ID** and **EPIC_TITLE** from Phase 0
2. Epic description: from `br show <EPIC_ID> --json` (Phase 0)
3. Review outcome record from Phase 4 (or "review skipped" if Phase 4 was skipped)

Run this command to gather the commit log:

```bash
git log --oneline main..HEAD
```

**Step 2: Generate Description**

Read templates/pr-body.md with the Read tool. Substitute all placeholders
using the algorithm documented in references/pr-description.md:

- **CHANGES_SUMMARY**: 2-3 sentences derived from the epic description.
  Explain what this PR accomplishes and why. Do not mention bead IDs or
  internal tracking. Frame in terms of user-facing or system-level impact.

- **DESIGN_DECISIONS**: key tradeoffs and decisions made during
  implementation, with reasoning. This is the primary entry point for
  reviewers. Do not list files changed or restate the diff. See
  `references/pr-description.md` for the full algorithm.

- **REVIEW_SUMMARY**: derive from the Phase 4 review outcome record.
  Use qualitative descriptions, not numeric counts. See
  `references/pr-description.md` for the full set of cases.

- **VERIFICATION_RESULTS**: list of verification commands run during Phase 3
  with PASS/FAIL results. If no verification commands were defined in bead
  descriptions, write "No explicit verification commands defined in bead
  descriptions."

**Step 3: Save and Present**

Create the PR description directory and save the rendered description:

```bash
mkdir -p .claude/pr-descriptions
```

Save to `.claude/pr-descriptions/feat-<BRANCH_NAME>.md` (in the worktree).

Present the full rendered description to the user in the conversation.
Allow iterative edits via conversation ("change the summary to...",
"add a note about X", etc.) and re-save after each edit.

---

### Phase 6: User Handoff

Present a final summary and offer the user action options.

**Step 1: Final Summary**

Present a complete summary of what was accomplished:

```
Implementation Complete

Epic: <EPIC_TITLE> (<EPIC_ID>)
Branch: feat/<BRANCH_NAME>
Worktree: <WORKTREE_PATH>

Beads: implemented and closed (list skipped beads if any)
Commits: <git log --oneline main..HEAD>
Review: <review outcome summary>
PR description: .claude/pr-descriptions/feat-<BRANCH_NAME>.md
```

**Step 2: Action Options**

Use AskUserQuestion to present next steps:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "Implementation complete. What would you like to do?",
    header: "Next step",
    options: [
      { label: "Push and create PR", description: "Push branch to origin and create PR with generated description" },
      { label: "Push only", description: "Push branch to origin, create PR manually later" },
      { label: "Stay in worktree", description: "Keep working in this worktree" },
      { label: "Return to main", description: "Switch back to main repo directory" }
    ],
    multiSelect: false
  }]
```

**Step 3: Execute Choice**

**Push and create PR**:

This requires explicit user approval per CLAUDE.md rules (never push or
create PRs without approval). The AskUserQuestion selection counts as initial
intent but confirm before executing:

1. Show what will be pushed: `git log --oneline main..HEAD`
2. Confirm: "Push feat/<BRANCH_NAME> to origin and create PR?"
3. If confirmed:
   ```bash
   git push -u origin feat/<BRANCH_NAME>
   gh pr create --title "<EPIC_TITLE>" --body-file .claude/pr-descriptions/feat-<BRANCH_NAME>.md
   ```
4. Report the PR URL to the user.

**Push only**:

1. Confirm: "Push feat/<BRANCH_NAME> to origin?"
2. If confirmed:
   ```bash
   git push -u origin feat/<BRANCH_NAME>
   ```
3. Report: "Branch pushed. Create PR when ready."

**Stay in worktree**:

No action needed. Report:

> "Staying in worktree at <WORKTREE_PATH>. Run further commands as needed."

**Return to main**:

Change working directory back to the original repository root (the directory
from Phase 0, before entering the worktree):

```bash
cd <original working directory>
```

Report:

> "Returned to main. Worktree preserved at <WORKTREE_PATH>.
> To remove the worktree later: `git worktree remove <WORKTREE_PATH>`"

---

## Error Handling

| Scenario | Recovery |
|----------|----------|
| br CLI not installed | Stop with install instructions |
| Already in worktree | Stop, tell user to exit worktree first |
| Epic not found | Stop with verification command |
| No children in epic | Stop with instructions to add beads |
| Dependency cycles | Stop with cycle details and resolution command |
| No ready beads | Report blockers, exit |
| All beads already closed | Report completion, exit |
| Bead in_progress by other session | Warn and skip |
| Truncated dep tree | Warn about depth limit |
| Worktree already exists | Offer to reuse or abort |
| Branch name already exists | Offer alternative name or reuse |
| br where fails in worktree | Warn about bead config accessibility |
| Bead claim fails | Skip bead, continue to next |
| Verification command not found | Skip that check, warn in output |
| Subagent crashes or times out | Reset bead to open, skip |
| All beads in wave skipped | Log warning, proceed to next wave |
| Git conflicts in worktree | Stop, report to user |
| Review skill unavailable (no TeamCreate/Task) | Skip review, warn user, proceed to Phase 5 |
| Review produces no report | Skip fix pipeline, proceed to Phase 5 with warning |
| Fix skill fails mid-pipeline | Report partial results, proceed to Phase 5 |
| Review aborted by user | Stop pipeline, leave branch as-is |
| PR description directory not writable | Warn, print description to conversation only |
| git diff/log fails in worktree | Use available context, note gaps in PR description |
| /commit skill fails (pre-commit hooks, staging) | Report the failure, let user resolve manually |
| gh pr create fails | Report error, provide manual PR creation command |
| git push fails | Report error, suggest user push manually |
| User edits PR description iteratively | Re-save to file after each edit |

## Guidelines

- **Wave ordering**: implement depth-1 beads before depth-2 beads
- **Priority within waves**: lower priority number = implement first
- **Resume support**: detect already-closed beads and skip them
- **No duplicate work**: skip in_progress beads to avoid conflicts
- **User confirmation**: always confirm before starting implementation
- **Bounded depth**: use max-depth 10 for dep tree queries
- **Inline vs subagent**: 5 or fewer beads = inline, 6+ = subagent
- **Commit at wave boundaries**: not after each individual bead
- **No bead IDs in commits**: internal tracking detail, not for git history
- **No counts in commits**: counts go stale before the commit is pushed
- **Worktree isolation**: all work happens in .claude/worktrees/implement-*
- **One review cycle**: never re-run /team-branch-review after /team-branch-fix
- **Auto-mode for fixes**: /team-branch-fix is always invoked with --auto from /implement
- **Skill invocation**: invoke /team-branch-review and /team-branch-fix via the Skill() tool, not natural language text
- **Review is post-implementation**: review runs after all beads are implemented, not per-bead
- **Clean state before review**: all changes must be committed before review starts
- **PR description from epic**: the summary comes from the epic description, not from bead titles
- **Design decisions from epic**: PR design decisions derive from the epic description, not from implementation details
- **No bead details in PR descriptions**: PR descriptions must not mention bead IDs, skip reasons, or resume-state bookkeeping. If omitted work changes scope, describe in product or system terms only
- **No numeric stats in PR descriptions**: do not include file counts, line counts, finding counts, or diff stats. The diff shows what changed. Counts go stale
- **Push requires approval**: never push to remote without explicit user confirmation
- **PR creation requires approval**: never create a PR without explicit user confirmation
- **Iterative PR editing**: allow user to refine the PR description via conversation before saving

$ARGUMENTS
