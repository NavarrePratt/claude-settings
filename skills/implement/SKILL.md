---
name: implement
description: >
  Implement an epic's beads in a worktree with logical commits, multi-agent
  review, fix pipeline, and PR description generation. Use when you want to
  implement this epic, burn down the beads, execute the plan, or start working
  on an epic.
argument-hint: "<epic-id> [branch-name] [--auto]"
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
| [../shared/br-in-worktree.md](../shared/br-in-worktree.md) | br flag pattern used across all post-cd phases (Phases 2, 3, 6) |

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
4b. Epic is not already claimed by another session (warn if in_progress)
5. No dependency cycles
6. Parse optional branch name and `--auto` flag from `$ARGUMENTS`
7. Discover **BASE_REF** (default branch detection)
8. Derive **MAIN_REPO_BEADS_DB** (path to main repo's `.beads/beads.db`,
   used on every post-cd `br` call via `--db`)

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

All `br` calls in Phase 2 and every subsequent phase use the `--db` flag to
target the main repo's database (`MAIN_REPO_BEADS_DB` from Phase 0). The
worktree has no `.beads/` directory because it is gitignored - see
[shared/br-in-worktree.md](../shared/br-in-worktree.md) for rationale.

Follow the reference file to:

1. Compute **BRANCH_NAME** (from `$ARGUMENTS` or slugified EPIC_TITLE)
2. Check for existing worktree (offer reuse or abort)
3. Create worktree: `git worktree add .claude/worktrees/implement-<BRANCH_NAME> -b feat/<BRANCH_NAME> <BASE_REF>`
4. Change working directory to **WORKTREE_PATH**
5. Verify clean state (Phase 0 already proved the beads db exists)
5b. Claim the epic: `br --db "$MAIN_REPO_BEADS_DB" update <EPIC_ID> --status in_progress --notes "CLAIMED: ..."` (see worktree-setup.md Step 5b)
6. Decide implementation mode (see bead-implementation.md for criteria)

**State file for context protection**

After Phase 2 completes, write a state file to
`/tmp/<REPO_NAME>-<EPIC_ID>/state.json` to survive context compaction. This
is context protection, not a full resume system - it helps the agent recall
where it was. Verify directory ownership before writing to avoid symlink
attacks on the predictable `/tmp` path.

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
  "main_repo_beads_db": "<MAIN_REPO_BEADS_DB>",
  "auto_mode": false,
  "last_completed_phase": "phase-2",
  "execution_plan": { "waves": [...] },
  "bead_statuses": { "<bead_id>": { "status": "pending" } },
  "review_outcome": null
}
```

The `main_repo_beads_db` field persists the path across context compaction
so post-compaction recovery can reconstruct `br --db "$MAIN_REPO_BEADS_DB" ...`
calls without re-deriving it.

Write points:
- After Phase 2 (worktree ready): initial state with all beads pending
- During Phase 3 (after each bead): update bead_statuses and last_completed_phase
- After Phase 4 (review done): update review_outcome

Note: `bead_statuses` stores execution status only (pending, done, skipped), not
skip reasons or other details. After context compaction, re-fetch skip reasons
and bead details from `br --db "$MAIN_REPO_BEADS_DB" show <id> --json` rather
than relying on in-memory state.

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

1. **Claim**: `br --db "$MAIN_REPO_BEADS_DB" update <bead_id> --status in_progress --json`
2. **Load context**: `br --db "$MAIN_REPO_BEADS_DB" show <bead_id> --json` -
   extract description and verification commands
3. **Implement**: inline or subagent mode per Phase 2 decision
4. **Verify**: run extracted verification commands, assess errors by type
5. **Close**: `br --db "$MAIN_REPO_BEADS_DB" close <bead_id> --reason "..."` on success
6. **Skip**: `br --db "$MAIN_REPO_BEADS_DB" update <bead_id> --status open --notes "SKIPPED: ..."` on failure

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
- **Actionable findings exist**: run fix pipeline. If **AUTO_MODE** is true,
  pass `--auto` for autonomous fix selection. If false, invoke without
  `--auto` so the user can interactively approve each finding.

```
# AUTO_MODE=true:
Skill(skill: "team-branch-fix", args: "--auto <review report>")

# AUTO_MODE=false:
Skill(skill: "team-branch-fix", args: "<review report>")
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

**Step 2: Derive PR Title**

Follow [pr-description.md "Generating the PR Title"](references/pr-description.md)
to derive the title:

1. Detect Conventional Commits - workflow grep first, then git log
   inference, ask the user when evidence falls in the ambiguous band.
   Exact thresholds and the strictness escape hatch live in the reference.
2. Determine CHANGE_KIND for the PR as a whole. Ask the user via
   AskUserQuestion when multiple kinds are equally plausible.
3. Map CHANGE_KIND to a prefix (when Conventional Commits applies) and
   compose an imperative ≤70-char subject with no trailing period.
4. Apply the normalization rules from the reference.
5. Save the final one-line title to `/tmp/<REPO_NAME>-<EPIC_ID>/<EPIC_ID>-<BRANCH_NAME>.txt` (i.e. `$ARTIFACT_DIR/$ARTIFACT_BASENAME.txt`).

**Step 3: Generate Body**

Read templates/pr-body.md and substitute placeholders per
references/pr-description.md:

- **CHANGES_SUMMARY**: 2-3 sentences from epic description. Why this PR exists.
- **DESIGN_DECISIONS**: key tradeoffs and reasoning. Primary reviewer entry point.
- **VERIFICATION_RESULTS**: GitHub-flavored checkbox list. `[x]` for commands that passed; `[ ]` for commands deferred (not yet run). Failures must not appear - stop the pipeline instead.

**Step 4: Save and Present**

Save the body to `/tmp/<REPO_NAME>-<EPIC_ID>/<EPIC_ID>-<BRANCH_NAME>.md`
(i.e. `$ARTIFACT_DIR/$ARTIFACT_BASENAME.md`) and keep the title at
`/tmp/<REPO_NAME>-<EPIC_ID>/<EPIC_ID>-<BRANCH_NAME>.txt` from Step 2.
Present both to the user and allow iterative editing; re-save the affected
file after each edit.

Do not allow Phase 6 Push-and-create-PR until the user has approved the
full PR draft (both title and body). Approvals can arrive across separate
messages in an iterative edit flow; track both artifacts as approved
independently, then proceed only when both are approved and the current
content is what was approved.

---

### Phase 6: User Handoff

**Step 0: Resolve Epic Status**

After all beads are processed, resolve the epic's status based on bead
completion, not review outcome. Review quality is captured in Phase 4 and
surfaced in the PR description - epic status tracks whether work is done.

Classify all descendant beads by their current status. If context was
compacted, re-derive the descendant list from
`br --db "$MAIN_REPO_BEADS_DB" show <EPIC_ID> --json` (the parent-field scan
from Phase 1 is repeatable). The `bead_statuses` in the state file is a
compaction safety net for execution plan beads only - it is not the source
of truth. Re-query `br --db "$MAIN_REPO_BEADS_DB" show <id> --json` for
every descendant to get current statuses.

Classify by current status (checked at handoff time, not plan time):

- **Completed**: current status is closed
- **Skipped**: current status is open with SKIPPED notes (failed verification)
- **In-progress**: current status is in_progress (treat as not-closed)
- **Blocked/other**: any remaining status (open without skip notes, deferred, etc.)

**If ALL descendant beads are closed** (completed count equals total descendants):

```bash
br --db "$MAIN_REPO_BEADS_DB" close <EPIC_ID> --reason "All beads implemented via /implement"
```

No user confirmation needed - the user approved the plan in Phase 1.

**If some beads were skipped, remain blocked, or are in-progress elsewhere** (partial completion):

```bash
br --db "$MAIN_REPO_BEADS_DB" update <EPIC_ID> --status open --notes "$(cat <<'EOF'
COMPLETED: <list>. SKIPPED: <list with reasons>. BLOCKED: <list>. IN_PROGRESS: <list>.
EOF
)"
```

The `--status open` resets the claim from Phase 2 so `br ready` can
surface the epic again. The notes record partial progress for the next
session.

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
PR title: <contents of /tmp/<REPO_NAME>-<EPIC_ID>/<EPIC_ID>-<BRANCH_NAME>.txt>
PR description: /tmp/<REPO_NAME>-<EPIC_ID>/<EPIC_ID>-<BRANCH_NAME>.md
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
Show commits and the derived title, confirm, then push and create the PR
with both title and body passed explicitly (see
[pr-description.md "Generating the PR Title"](references/pr-description.md)):

```bash
ARTIFACT_DIR="/tmp/<REPO_NAME>-<EPIC_ID>"
ARTIFACT_BASENAME="<EPIC_ID>-<BRANCH_NAME>"
test -s "$ARTIFACT_DIR/$ARTIFACT_BASENAME.txt" || { echo "Error: missing PR title" >&2; exit 1; }
test -s "$ARTIFACT_DIR/$ARTIFACT_BASENAME.md" || { echo "Error: missing PR body" >&2; exit 1; }
awk 'NR>1 || length($0)>70 {exit 1}' "$ARTIFACT_DIR/$ARTIFACT_BASENAME.txt" \
  || { echo "Error: PR title file is multi-line or >70 chars" >&2; exit 1; }

PR_URL=$(gh pr create \
  --base "$BASE_REF" \
  --title "$(cat "$ARTIFACT_DIR/$ARTIFACT_BASENAME.txt")" \
  --body-file "$ARTIFACT_DIR/$ARTIFACT_BASENAME.md")
echo "$PR_URL"
PR_NUMBER=$(printf '%s\n' "$PR_URL" | grep -oE '[0-9]+$' | tail -1)
```

After `gh pr create` succeeds, auto-invoke /watch-pr with the parsed PR
number so the user gets background follow-through on CI, reviews, and
merge events without a second prompt. watch-pr is read-only (cannot post
comments, merge, or push) so no additional approval is needed:

```
Skill(skill: "watch-pr", args: "<PR_NUMBER>")
```

If PR_NUMBER is empty (parsing failed) or the Skill invocation errors,
warn the user with the raw PR URL and the manual command
`/watch-pr <N>`, then finish the handoff. The PR was created
successfully; watch-pr is a convenience, not a correctness requirement.

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
| Epic already in_progress (claimed) | Warn with claim notes, offer abort or take-over via AskUserQuestion |
| Epic claim fails (br error) | Warn, continue implementation - epic just will not be visible as in_progress |
| No children in epic | Stop with instructions to add beads |
| Dependency cycles | Stop with cycle details |
| No ready beads | Report blockers, exit |
| All beads already closed | Report completion, exit |
| Bead in_progress by other session | Warn and skip |
| Worktree already exists | Offer reuse or abort |
| Branch name already exists | Offer alternative name or reuse |
| br call fails in worktree | Ensure `--db "$MAIN_REPO_BEADS_DB"` is set; see shared/br-in-worktree.md |
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
| watch-pr invocation fails after PR create | Warn with PR URL and manual `/watch-pr <N>` command, finish handoff (PR is already created) |

## Guidelines

- **Wave ordering**: implement earlier waves before later waves - dependencies require it and later beads may build on earlier changes
- **Priority within waves**: lower priority number = implement first - matches user intent for what matters most
- **Resume support**: detect already-closed beads and skip them - enables picking up where a previous session left off
- **No duplicate work**: skip in_progress beads to avoid conflicts - another session may be actively working on them
- **User confirmation**: always confirm before starting implementation - the execution plan may need adjustment based on user context
- **No bead IDs in commits**: internal tracking detail, not meaningful to git history readers
- **No counts in commits**: counts go stale before the commit is pushed - the diff shows exactly what changed
- **Worktree isolation**: all work happens in .claude/worktrees/implement-* - protects the main repo from incomplete changes
- **Review default**: one full review + one fix pass, targeted re-review only for non-trivial fixes - bounds token cost while catching real issues
- **Auto-mode for fixes**: /team-branch-fix receives --auto only when /implement is invoked with --auto - user's autonomy preference propagates through the pipeline
- **Skill invocation**: invoke review/fix via the Skill() tool - skills manage their own context and agent lifecycle
- **Review is post-implementation**: review runs after all beads, not per-bead - reviewing the complete change catches cross-bead interactions
- **Clean state before review**: all changes must be committed before review - reviewers need committed diffs, not working tree state
- **PR description from epic**: summary comes from epic context, not bead titles - the epic captures the why, beads are implementation detail
- **No bead details in PR descriptions**: no bead IDs, skip reasons, or bookkeeping - PR readers care about the change, not the tracking system
- **No numeric stats in PR descriptions**: no file/line/finding counts - counts go stale during editing and add no value over the diff itself
- **Degraded verification**: when review or fix is skipped/failed, the PR description must explicitly note the degraded verification state so reviewers know
- **Push requires approval**: never push without explicit user confirmation - remote operations are irreversible and affect shared state
- **PR creation requires approval**: never create PR without confirmation - PRs are visible to the team and trigger notifications
- **Iterative PR editing**: allow user to refine description before saving - generated descriptions benefit from human judgment on tone and emphasis
- **Epic status resolution**: auto-close the epic when all beads are done, update notes on partial completion. Best-effort - never block handoff on a br failure
- **Epic claim**: mark the epic in_progress in Phase 2 after worktree creation so concurrent /implement sessions (and `br ready` scans) see it as taken. Reset to open in Phase 6 on partial completion so work can be resumed later
- **Auto-watch after PR creation**: after a successful "Push and create PR" in Phase 6, auto-invoke /watch-pr with the new PR number. The user opted into publishing the PR, and watch-pr is read-only (it cannot post comments, merge, or push) so no additional approval is needed. If watch-pr cannot be started, warn with the PR URL and the manual `/watch-pr <N>` command, then finish the handoff

$ARGUMENTS
