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
git log --oneline main..HEAD
```

If output is empty, the branch has no commits vs main. Nothing to review:

> "No commits on this branch vs main. Skipping review."

Skip the entire review pipeline and proceed to the next phase.

## Invoke Review

Invoke the review skill using the Skill tool:

```
Skill(skill: "team-branch-review")
```

The review skill detects branch and base commit automatically via its Context
section. It spawns reviewer agents, validates findings with Codex, and produces
a report with one of three outcomes.

## Parse Outcome

After /team-branch-review completes, read the review report from the
conversation. The report follows the template at
`~/.claude/skills/team-branch-review/templates/final-report.md`.

The outcome line appears as:

```
## Outcome: [APPROVED | NEEDS REVISION | MANUAL REVIEW REQUIRED]
```

### Outcome: APPROVED

No confirmed Critical or High findings after Codex validation.

Action: skip the fix pipeline entirely. Record for the PR description:

```
Review outcome: APPROVED
Findings: No critical or high issues found.
```

Proceed to the next phase.

### Outcome: NEEDS REVISION

Confirmed Critical or High findings remain.

Action: invoke the fix pipeline with --auto to bypass user interview questions:

```
Skill(skill: "team-branch-fix", args: "--auto <paste the review report>")
```

The `--auto` flag makes the fix skill fully autonomous: confirmed findings are
auto-approved, disputed findings are auto-skipped (requires human judgment),
Med/Low findings are batch-approved, and fixup commit strategy is used with
auto-rebase. Push and PR creation remain human-gated in Phase 6.

The fix skill handles:
- Auto-decisions for all findings (logged to Auto-Mode Decision Log)
- Fixer agent spawning
- Codex validation
- Commit strategy (fixup with auto-rebase, single commit fallback on failure)

After /team-branch-fix completes, proceed to Post-Fix Cleanup.

### Outcome: MANUAL REVIEW REQUIRED

Significant disagreement (>50% of Critical/High disputed) between Claude
and Codex reviewers.

Action: present the disputed findings to the user and ask how to proceed.

Use AskUserQuestion:
```
questions: [{
  question: "Review found disputed findings that need human judgment. How to proceed?",
  header: "Disputed",
  options: [
    { label: "Fix confirmed findings", description: "Run /team-branch-fix for non-disputed findings only" },
    { label: "Skip review fixes", description: "Proceed without fixing - note unresolved findings in PR" },
    { label: "Abort", description: "Stop the implement pipeline, leave branch as-is" }
  ],
  multiSelect: false
}]
```

**If "Fix confirmed findings"**:

```
Skill(skill: "team-branch-fix", args: "--auto <paste the review report, noting disputed findings should be skipped>")
```

After fix completes, proceed to Post-Fix Cleanup. Record disputed findings
as unresolved in the review outcome.

**If "Skip review fixes"**:

Record all findings as unresolved in the review outcome. Proceed to the next
phase with a note:

```
Review outcome: MANUAL REVIEW REQUIRED (fixes skipped by user)
Unresolved findings: N critical/high (disputed)
```

**If "Abort"**:

Stop the implement pipeline. Leave the branch as-is. Report:

> "Pipeline aborted at review stage. Branch `feat/<BRANCH_NAME>` remains in
> the worktree with all implementation commits intact."

## Post-Fix Cleanup

After the fix pass completes (or was skipped):

### Step 1: Verify Clean Git State

```bash
git status --porcelain
```

If uncommitted changes exist (fix agents may have left unstaged work):

> "Post-fix uncommitted changes detected. Running /commit."

Invoke /commit to commit remaining changes.

### Step 2: Record Review Outcome

Compile the review outcome for use in the PR description:

```
Review outcome: [APPROVED | NEEDS REVISION (fixed) | MANUAL REVIEW REQUIRED]
Total findings: N
Fixed: N
Skipped: N
Deferred: N
Unresolved: N (if any critical findings remain)
```

This record is passed to the PR description generation phase.

## Bounded Iteration Rule

**Do NOT re-run /team-branch-review after /team-branch-fix completes.**

One review + one fix pass is the maximum. Reasoning:
- /team-branch-review spawns 2-6 agents with Codex validation (expensive)
- /team-branch-fix spawns N fixer agents with Codex validation (expensive)
- Looping risks unbounded cost and context exhaustion

If critical findings remain after the fix pass, report them in the PR
description and final handoff rather than looping. The user can always run
`/team-branch-review` manually for a second pass.
