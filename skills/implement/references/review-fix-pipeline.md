# Review and Fix Pipeline

Post-implementation review using /team-branch-review and /team-branch-fix,
wired into /implement as an automatic quality gate.

## Overview

One review pass followed by at most one fix pass. No re-review after fixes.
The user can always run another review cycle manually after /implement completes.

## Pre-Review Check

Before invoking review, verify the branch is in a reviewable state.

### Check 1: Clean Working Tree

```bash
git status --porcelain
```

If output is non-empty, uncommitted changes exist. Invoke /commit to commit
them before proceeding:

> "Uncommitted changes detected. Running /commit to commit before review."

Run /commit (the git-commit skill). After it completes, re-check:

```bash
git status --porcelain
```

If still dirty after /commit (user may have declined to commit), warn:

> "Working tree still has uncommitted changes. These will not be included in
> the review. Proceed anyway?"

Use AskUserQuestion:
```
questions: [{
  question: "Uncommitted changes remain. Proceed with review of committed code only?",
  header: "Dirty tree",
  options: [
    { label: "Proceed", description: "Review committed changes only, uncommitted changes ignored" },
    { label: "Abort", description: "Stop and handle uncommitted changes manually" }
  ],
  multiSelect: false
}]
```

If "Abort": stop the pipeline, return to user.

### Check 2: Commits Exist on Branch

```bash
git log --oneline <BASE_REF>..HEAD
```

If output is empty, the branch has no commits vs BASE_REF. Nothing to review:

> "No commits on this branch vs BASE_REF. Skipping review."

Skip the entire review pipeline and proceed to the next phase.

## Invoke Review

Invoke the review skill using the Skill tool:

```
Skill(skill: "team-branch-review")
```

The review skill detects branch and base commit automatically via its Context
section. It spawns reviewer agents, validates findings with Codex, and produces
a report with one of three outcomes.

## Parse Outcome and Compute Actionable Findings

After /team-branch-review completes, read the review report from the
conversation. The report follows the template at
`~/.claude/skills/team-branch-review/templates/final-report.md`.

The outcome line appears as:

```
## Outcome: [APPROVED | NEEDS REVISION | MANUAL REVIEW REQUIRED]
```

Record the outcome label for the PR description, but do NOT use it to gate
fix execution. Instead, compute the actionable finding count.

### Actionable Finding Definition

A finding is **actionable** if it is:
- Confirmed (any severity) - both Claude and Codex agree
- Severity-adjusted - Codex changed severity but confirmed the issue
- Codex-only (not disputed) - new finding from Codex validation

A finding is **NOT actionable** if it is:
- Disputed - Claude and Codex disagree on whether the issue exists

### Step 1: Compute Actionable Finding Count

Parse the review report's findings table. Count findings that match the
actionable criteria above. Record:
- `actionable_count`: total actionable findings
- `disputed_count`: total disputed findings

### Step 2: Branch on Actionable Count

**If actionable_count == 0 and disputed_count == 0**:

No findings at all. Record for PR description:

```
Review outcome: APPROVED
Fixes: none needed
Remaining: none
```

Proceed to Post-Fix Cleanup (skip fix pipeline).

**If actionable_count == 0 and disputed_count > 0** (any outcome label - disputed-only):

All findings are disputed - no automated fixes possible. Present disputed
findings to the user:

Use AskUserQuestion:
```
questions: [{
  question: "Review found only disputed findings (reviewer disagreement). No automated fixes available. How to proceed?",
  header: "Disputed",
  options: [
    { label: "Skip", description: "Proceed without fixes - note disputed findings in PR" },
    { label: "Abort", description: "Stop the implement pipeline, leave branch as-is" }
  ],
  multiSelect: false
}]
```

**If "Skip"**: record disputed findings as unresolved, proceed to Post-Fix Cleanup.

**If "Abort"**: stop the implement pipeline. Leave the branch as-is. Report:

> "Pipeline aborted at review stage. Branch `feat/<BRANCH_NAME>` remains in
> the worktree with all implementation commits intact."

**If actionable_count > 0** (any outcome label):

Invoke the fix pipeline. The invocation mode depends on the caller:

- **From /implement** (autonomous): pass `--auto` to skip the interactive
  finding interview. The fix skill will auto-select all actionable findings
  and skip disputed ones.

  ```
  Skill(skill: "team-branch-fix", args: "--auto <paste the review report>")
  ```

- **Standalone** (interactive): no `--auto` flag. The fix skill presents
  each finding to the user for include/exclude.

  ```
  Skill(skill: "team-branch-fix", args: "<paste the review report>")
  ```

The fix skill handles:
- Finding selection (auto or interactive)
- Fixer agent spawning
- Codex validation
- Commit strategy (fixup/single/multiple)

If the outcome is MANUAL REVIEW REQUIRED with mixed findings (both actionable
and disputed), present disputed findings to the user before invoking fixes:

> "Review found disputed findings alongside actionable ones. Disputed findings
> will be skipped by the fix pipeline. Actionable findings will be fixed."

After /team-branch-fix completes, proceed to Post-Fix Cleanup.

### Error Handling

| Scenario | Recovery |
|----------|----------|
| /team-branch-fix returns "No fixes to apply" | Proceed to Post-Fix Cleanup normally (auto-mode skipped all disputed) |
| Fix verification fails under APPROVED | Record failure in review outcome, proceed with warning |
| MANUAL REVIEW REQUIRED with no actionable findings | Skip fix pipeline, record disputed findings for PR |

## Post-Fix Cleanup

After the fix pass completes (or was skipped):

### Step 1: Verify Clean Git State

```bash
git status --porcelain
```

If uncommitted changes exist (fix agents may have left unstaged work):

**If the fix pass reported success** (all fixes applied, no verification failures):

> "Post-fix uncommitted changes detected. Running /commit."

Invoke /commit to commit remaining changes.

**If the fix pass reported verification failures**: do NOT auto-commit.
Surface the dirty state to the user via AskUserQuestion:

```
questions: [{
  question: "Fix pass had verification failures. Uncommitted changes remain. How to proceed?",
  header: "Failed fixes",
  options: [
    { label: "Commit anyway", description: "Commit the changes despite verification failures" },
    { label: "Revert changes", description: "Discard all uncommitted fix changes" },
    { label: "Leave for manual review", description: "Keep changes uncommitted, proceed to PR description" }
  ],
  multiSelect: false
}]
```

### Step 2: Re-Run Verification Commands

If the fix pass made code changes (fix commits exist, uncommitted changes
were committed in Step 1, or uncommitted fix changes remain in the working
tree), re-run the verification commands from Phase 3 (the union of all bead
verification commands).

**Note**: if the user chose "Leave for manual review" in Step 1, uncommitted
changes are present but not committed. Still re-run verification - the
results inform the PR description regardless of commit state.

```bash
# Run each verification command from Phase 3
# e.g., mise run lint, mise run test, mise run build
```

**If all pass**: proceed to Step 3.

**If any fail**: surface failures to the user via AskUserQuestion:

```
questions: [{
  question: "Post-fix verification failed. Some checks do not pass after fix changes. How to proceed?",
  header: "Verification failure",
  options: [
    { label: "Proceed anyway", description: "Continue to PR description - note failures for reviewers" },
    { label: "Abort", description: "Stop pipeline, leave branch for manual investigation" }
  ],
  multiSelect: false
}]
```

If "Proceed anyway": record failures in the review outcome. If "Abort": stop.

### Step 3: Record Review Outcome

Compile the review outcome for use in the PR description:

```
Review outcome: [APPROVED | NEEDS REVISION (fixed) | MANUAL REVIEW REQUIRED]
Fixes: [applied | partially applied | skipped | none needed]
Remaining: [any critical findings that remain, or "none"]
Verification: [pass | fail (details) - from post-fix re-verification]
```

This record is passed to the PR description generation phase.

## Post-Fix Squash

After the fix pass completes and any cleanup commits are made, fix commits
should be squashed into the original commits they fix. The PR should show
clean history, not "add feature X" followed by "fix review finding in X."

**Approaches** (lead chooses based on complexity):

- **git rebase with fixup**: for simple cases where each fix maps to a clear
  parent commit. Use `git rebase --autosquash` with fixup commits, or manual
  rebase commands (NOT `git rebase -i` which requires interactive input).
- **/clean-copy rewrite**: if the commit history is messy after fixes, use
  /clean-copy to rewrite the branch with clean narrative history.

Skip squashing if the fix pass made no commits or only trivial changes.

## Iteration Boundary

One /team-branch-review pass followed by at most one /team-branch-fix pass.
No re-review after fixes. This boundary exists for cost control: review and
fix passes consume significant tokens, and unbounded looping risks diminishing
returns when the reviewer and fixer disagree on style issues.

If critical findings remain after the fix pass, report them in the PR
description rather than looping. The user can always run `/team-branch-review`
manually for additional passes.
