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

Load referenced files with the Read tool before executing the relevant phase.

| Reference | Used in |
|-----------|---------|
| [preconditions.md](references/preconditions.md) | Phase 0 - prerequisite checks and BASE_REF discovery |
| [epic-resolution.md](references/epic-resolution.md) | Phase 1 - intersection algorithm and wave computation |
| [worktree-setup.md](references/worktree-setup.md) | Phase 2 - worktree creation and environment setup |
| [bead-implementation.md](references/bead-implementation.md) | Phase 3 - implementation, verification, and commit strategy |
| [review-fix-pipeline.md](references/review-fix-pipeline.md) | Phase 4 - review and fix pipeline |
| [pr-description.md](references/pr-description.md) | Phase 5 - PR description generation |

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

Read [preconditions.md](references/preconditions.md) before executing.

Run all checks from the reference file in order, stopping at first failure:

1. br CLI available
2. Not already in a worktree
3. Epic ID provided (parse from `$ARGUMENTS`)
4. Epic exists (record **EPIC_ID** and **EPIC_TITLE**)
5. No dependency cycles
6. Parse optional branch name from `$ARGUMENTS`
7. Discover **BASE_REF** (default branch detection)

Note: the "epic has children" check is deferred to Phase 1. The epic-resolution
algorithm reports "no children found" as an early exit if the parent-field scan
in Step 1 finds no descendants.

All subsequent references to the base branch use BASE_REF.

---

### Phase 1: Resolve Epic to Execution Plan

Read [epic-resolution.md](references/epic-resolution.md) before executing.

Follow the full algorithm from the reference file:

1. Find all descendants of the epic via parent-field scan
2. Get ready beads from `br ready --json --limit 0`
3. Compute intersection (descendants that are ready)
4. Classify all descendants by status
5. Compute execution waves from blocking dependencies
6. Handle edge cases (no ready beads, all done, in-progress conflicts)
7. Present execution plan to user, including estimated agent count:
   `Estimated agents: N implementation + review/fix pipeline (~2-6 reviewers + N fixers)`
   This is a rough estimate based on bead count and mode - the exact number
   depends on review findings. The point is to set expectations.
8. Wait for user confirmation via AskUserQuestion

If "Cancel": exit with no changes.
If "Skip some": let user select beads, rebuild plan.
If "Proceed": continue to Phase 2.

**Optional: Create progress tasks**

If TaskCreate is available, create tasks for visibility:
- One parent task per phase (mark Phase 0-1 as completed)
- One task per bead in the execution plan
- Update task status as phases and beads progress

If TaskCreate is not available (tool not loaded), skip all task operations
silently. Task tracking is a UX enhancement, not a correctness requirement.
Never let a task-tool failure block the implementation pipeline. Only the
lead agent creates or updates tasks - subagents never touch TaskCreate.

---

### Phase 2: Create Worktree

Read [worktree-setup.md](references/worktree-setup.md) before executing.

Follow the reference file to:

1. Compute **BRANCH_NAME** (from `$ARGUMENTS` or slugified EPIC_TITLE)
2. Check for existing worktree (offer reuse or abort)
3. Create worktree: `git worktree add .claude/worktrees/implement-<BRANCH_NAME> -b feat/<BRANCH_NAME> <BASE_REF>`
4. Change working directory to **WORKTREE_PATH**
5. Verify environment (br access, clean state)
6. Decide implementation mode (see bead-implementation.md for criteria)

**State file for context protection**

After Phase 2 completes, write a state file to
`/tmp/<REPO_NAME>-<EPIC_ID>/state.json` to survive context compaction. This
is context protection, not a full resume system - it helps the agent recall
where it was. Use `mktemp -d` or verify directory ownership before writing to
avoid symlink attacks on the predictable `/tmp` path.

`<REPO_NAME>` is the repository directory name (REPO_NAME, computed in
worktree-setup.md Step 1).

```json
{
  "schema_version": 1,
  "epic_id": "<EPIC_ID>",
  "epic_title": "<EPIC_TITLE>",
  "branch_name": "feat/<BRANCH_NAME>",
  "base_ref": "<BASE_REF>",
  "worktree_path": "<WORKTREE_PATH>",
  "repo_name": "<REPO_NAME>",
  "last_completed_phase": "phase-2",
  "execution_plan": { "waves": [...] },
  "bead_statuses": { "<bead_id>": { "status": "pending" } },
  "review_outcome": null
}
```

Write points:
- After Phase 2 (worktree ready): initial state with all beads pending
- During Phase 3 (after each bead): update bead_statuses and last_completed_phase
- After Phase 4 (review done): update review_outcome

Note: `bead_statuses` stores execution status only (pending, done, skipped), not
skip reasons or other details. After context compaction, re-fetch skip reasons
and bead details from `br show <id> --json` rather than relying on in-memory state.

The state file lives in /tmp (outside the worktree) to avoid dirtying git.

---

### Phase 3: Implement Beads

Read [bead-implementation.md](references/bead-implementation.md) before
starting this phase.

Process beads wave by wave, in the order determined by Phase 1.

**For each wave:**

Initialize tracking for this wave: beads completed, beads skipped.

**For each bead in the wave:**

Execute the per-bead cycle from bead-implementation.md:

1. **Claim**: `br update <bead_id> --status in_progress --json`
2. **Load context**: `br show <bead_id> --json` - extract description and
   verification commands
3. **Implement**: inline or subagent mode per Phase 2 decision
4. **Verify**: run extracted verification commands, assess errors by type
5. **Close**: `br close <bead_id> --reason "..."` on success
6. **Skip**: `br update <bead_id> --status open --notes "SKIPPED: ..."` on failure

**Commits:** follow the commit strategy from bead-implementation.md to create
logical commits using `/commit`.

**After all waves:** report implementation summary (completed, skipped,
commits, worktree path).

---

### Phase 4: Review and Fix Pipeline

Read [review-fix-pipeline.md](references/review-fix-pipeline.md) before
executing this phase.

**Step 1: Pre-Review Check**

```bash
git status --porcelain
```

If uncommitted changes exist, run /commit first. Then verify commits exist:

```bash
git log --oneline <BASE_REF>..HEAD
```

If no commits exist vs BASE_REF, skip the review pipeline entirely.

See review-fix-pipeline.md for the full dirty-tree handling procedure.

**Step 2: Invoke Review**

```
Skill(skill: "team-branch-review")
```

**Step 3: Compute Actionable Findings and Act**

Record the outcome label from the review report. Parse findings to compute
the actionable finding count per review-fix-pipeline.md.

- **No actionable findings**: skip fix pipeline
- **Actionable findings exist**: run fix pipeline with `--auto` flag

```
Skill(skill: "team-branch-fix", args: "--auto <review report>")
```

**Step 4: Post-Fix Cleanup**

1. Verify clean git state
2. Commit remaining changes if fix pass succeeded, or surface to user if
   verification failures remain
3. Squash fix commits into their parent commits (see review-fix-pipeline.md
   Post-Fix Squash section)
4. Assess whether targeted re-review is needed for non-trivial fixes
5. Record review outcome for Phase 5

See review-fix-pipeline.md for review iteration guidance and squash strategy.

---

### Phase 5: PR Description Generation

Read [pr-description.md](references/pr-description.md) before executing.

**Step 1: Gather Inputs**

Collect from prior phases: epic context, review outcome, verification results.

```bash
git log --oneline <BASE_REF>..HEAD
```

**Step 2: Generate Description**

Read templates/pr-body.md and substitute placeholders per
references/pr-description.md:

- **CHANGES_SUMMARY**: 2-3 sentences from epic description. Why this PR exists.
- **DESIGN_DECISIONS**: key tradeoffs and reasoning. Primary reviewer entry point.
- **REVIEW_SUMMARY**: qualitative review outcome (no numeric counts).
- **VERIFICATION_RESULTS**: commands run with PASS/FAIL results.

**Step 3: Save and Present**

Save to `.claude/pr-descriptions/feat-<BRANCH_NAME>.md` in the worktree.
Present to user for iterative editing.

---

### Phase 6: User Handoff

**Step 0: Resolve Epic Status**

After all beads are processed, resolve the epic's status based on bead
completion, not review outcome. Review quality is captured in Phase 4 and
surfaced in the PR description - epic status tracks whether work is done.

Classify all descendant beads by their current status. If context was
compacted, re-derive the descendant list from `br show <EPIC_ID> --json`
(the parent-field scan from Phase 1 is repeatable). The `bead_statuses` in
the state file is a compaction safety net for execution plan beads only - it
is not the source of truth. Re-query `br show <id> --json` for every
descendant to get current statuses.

Classify by current status (checked at handoff time, not plan time):

- **Completed**: current status is closed
- **Skipped**: current status is open with SKIPPED notes (failed verification)
- **In-progress**: current status is in_progress (treat as not-closed)
- **Blocked/other**: any remaining status (open without skip notes, deferred, etc.)

**If ALL descendant beads are closed** (completed count equals total descendants):

```bash
br close <EPIC_ID> --reason "All beads implemented via /implement"
```

No user confirmation needed - the user approved the plan in Phase 1.

**If some beads were skipped, remain blocked, or are in-progress elsewhere** (partial completion):

```bash
br update <EPIC_ID> --notes "$(cat <<'EOF'
COMPLETED: <list>. SKIPPED: <list with reasons>. BLOCKED: <list>. IN_PROGRESS: <list>.
EOF
)"
```

The epic stays open for a future session to pick up remaining work.

**If br close or br update fails**: warn but do not block the handoff. Epic
status resolution is best-effort.

Record the epic status outcome (closed, partial, or error) for the summary.

**Step 1: Final Summary**

```
Implementation Complete

Epic: <EPIC_TITLE> (<EPIC_ID>)
Epic status: <closed | updated with notes (partial completion) | unchanged (error)>
Branch: feat/<BRANCH_NAME>
Worktree: <WORKTREE_PATH>

Beads: implemented and closed (list skipped beads if any)
Commits: <git log --oneline <BASE_REF>..HEAD>
Review: <review outcome summary>
PR description: .claude/pr-descriptions/feat-<BRANCH_NAME>.md
```

**Step 2: Action Options**

Use AskUserQuestion:

```
questions: [{
  question: "Implementation complete. What would you like to do?",
  header: "Next step",
  options: [
    { label: "Push and create PR", description: "Push branch and create PR with generated description" },
    { label: "Push only", description: "Push branch, create PR manually later" },
    { label: "Stay in worktree", description: "Keep working in this worktree" },
    { label: "Return to main", description: "Switch back to main repo directory" }
  ],
  multiSelect: false
}]
```

**Step 3: Execute Choice**

**Push and create PR**: requires explicit user confirmation per CLAUDE.md.
Show commits, confirm, then push and create PR with body-file.

**Push only**: confirm, push with `-u`.

**Stay in worktree**: no action needed.

**Return to main**: change directory back to original repo root. Report
worktree preservation path.

---

## Error Handling

| Scenario | Recovery |
|----------|----------|
| br CLI not installed | Stop with install instructions |
| Already in worktree | Stop, tell user to exit first |
| Epic not found | Stop with verification command |
| No children in epic | Stop with instructions to add beads |
| Dependency cycles | Stop with cycle details |
| No ready beads | Report blockers, exit |
| All beads already closed | Report completion, exit |
| Bead in_progress by other session | Warn and skip |
| Worktree already exists | Offer reuse or abort |
| Branch name already exists | Offer alternative name or reuse |
| br where fails in worktree | Warn, suggest BR_HOME |
| Bead claim fails | Skip bead, continue to next |
| Verification command not found | Skip that check, warn |
| Subagent crashes or hangs | Reset bead to open, skip |
| All beads in wave skipped | Log warning, proceed to next wave |
| Git conflicts in worktree | Stop, report to user |
| Review skill unavailable | Skip review, warn, proceed to Phase 5 (note degraded state in PR) |
| Review produces no report | Skip fix pipeline, warn (note degraded state in PR) |
| Fix skill fails mid-pipeline | Report partial results, proceed to Phase 5 (note degraded state in PR) |
| Review aborted by user | Stop pipeline, leave branch as-is |
| /commit skill fails | Report failure, let user resolve |
| gh pr create fails | Report error, provide manual command |
| git push fails | Report error, suggest manual push |

## Guidelines

- **Wave ordering**: implement earlier waves before later waves
- **Priority within waves**: lower priority number = implement first
- **Resume support**: detect already-closed beads and skip them
- **No duplicate work**: skip in_progress beads to avoid conflicts
- **User confirmation**: always confirm before starting implementation
- **No bead IDs in commits**: internal tracking detail, not for git history
- **No counts in commits**: counts go stale before the commit is pushed
- **Worktree isolation**: all work happens in .claude/worktrees/implement-*
- **Review default**: one full review + one fix pass, targeted re-review only for non-trivial fixes
- **Auto-mode for fixes**: /team-branch-fix always invoked with --auto
- **Skill invocation**: invoke review/fix via the Skill() tool
- **Review is post-implementation**: review runs after all beads, not per-bead
- **Clean state before review**: all changes must be committed before review
- **PR description from epic**: summary comes from epic, not bead titles
- **No bead details in PR descriptions**: no bead IDs, skip reasons, or bookkeeping
- **No numeric stats in PR descriptions**: no file/line/finding counts
- **Degraded verification**: when review or fix is skipped/failed, the PR description must explicitly note the degraded verification state so reviewers know
- **Push requires approval**: never push without explicit user confirmation
- **PR creation requires approval**: never create PR without confirmation
- **Iterative PR editing**: allow user to refine description before saving
- **Epic status resolution**: auto-close the epic when all beads are done, update notes on partial completion. Best-effort - never block handoff on a br failure

$ARGUMENTS
