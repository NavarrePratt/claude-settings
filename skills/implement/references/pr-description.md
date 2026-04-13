# PR Description Generation

Algorithm for generating a PR description from the implementation context
collected during Phases 1-4 of /implement.

## Inputs

All inputs are gathered during prior phases and available in the lead agent's
conversation context.

| Input | Source | Phase |
|-------|--------|-------|
| Epic title and ID | br show output | Phase 0 |
| Epic description | br show output | Phase 0 |
| Implemented beads (IDs, titles, close reasons) | Wave tracking | Phase 3 |
| Skipped beads (IDs, titles, skip reasons) | Wave tracking | Phase 3 |
| Commit log | `git log --oneline main..HEAD` | Phase 5 |
| Diff stats | `git diff --stat main..HEAD` | Phase 5 |
| Lines changed | `git diff --shortstat main..HEAD` | Phase 5 |
| Review outcome | Review outcome record | Phase 4 |
| Verification results | Bead verification runs | Phase 3 |

## Template

The PR body template lives at `templates/pr-body.md`. It contains these
placeholders:

| Placeholder | Content |
|-------------|---------|
| CHANGES_SUMMARY | 2-3 sentences: what this PR accomplishes and why. Derived from the epic description, not a list of beads. |
| DESIGN_DECISIONS | Key tradeoffs and decisions made during implementation, with reasoning. Entry point for reviewers. |
| BEAD_TABLE | Markdown table of all beads with ID, title, and status (Closed or Skipped with reason). |
| REVIEW_SUMMARY | One-line or short block summarizing the review outcome from Phase 4. |
| VERIFICATION_RESULTS | List of verification commands run and their results (PASS/FAIL). |
| DIFF_STATS | Output of `git diff --shortstat main..HEAD` - lines added/removed/files changed. |

## Generating the Summary (CHANGES_SUMMARY)

The summary explains **why** this PR exists. Derive it from the epic
description, not from individual bead titles.

1. Read the epic description from Phase 0 context
2. Write 2-3 sentences that answer: "What does this PR accomplish and why?"
3. Do not mention bead IDs or internal tracking
4. Frame in terms of user-facing or system-level impact

Example:
> Add the review and fix pipeline to the /implement skill. This automates
> post-implementation quality checks by running /team-branch-review followed
> by /team-branch-fix with bounded iteration, catching issues before PR
> creation.

## Generating Design Decisions (DESIGN_DECISIONS)

This section is the primary entry point for reviewers - both human and agent.
Surface the key tradeoffs and decisions so reviewers can critique the reasoning
before diving into code. Agent-based reviewers benefit from having tradeoff
context upfront instead of rediscovering it from the diff.

1. Read the epic description from Phase 0 context - the "Design Decisions"
   section is the primary source, capturing tradeoffs from the planning phase
2. Supplement with bead descriptions, which explain why each piece of work
   exists and any per-bead approach decisions
3. Add any implementation-phase decisions that arose during Phases 2-4
   (these are supplementary - the planning-phase decisions carry more weight)
4. Identify decisions where a reasonable person might have chosen differently
5. Write each decision as a short paragraph: the choice made, alternatives
   considered (if any), and the reasoning
6. Focus on architectural choices, API design, error handling strategy,
   performance tradeoffs - not mechanical details
7. Do NOT list files changed or restate what the diff shows

Example:
```markdown
Bounded the review-fix loop to 2 iterations rather than running until clean.
Unbounded loops risk burning tokens on diminishing returns when the reviewer
and fixer disagree on style issues. Two passes catches real bugs while keeping
cost predictable.

Chose to invoke /team-branch-review as a skill rather than calling the agent
team directly. This keeps the review pipeline decoupled from /implement so
either can evolve independently, at the cost of slightly less control over
reviewer configuration.
```

## Generating the Bead Table (BEAD_TABLE)

Create a markdown table with all beads from the execution plan.

```markdown
| Bead | Title | Status |
|------|-------|--------|
| bd-xxx | First task title | Closed |
| bd-yyy | Second task title | Closed |
| bd-zzz | Third task title | Skipped: verification failed |
```

- Use the bead ID and title from the execution plan
- Status is "Closed" for successfully implemented beads
- Status is "Skipped: <reason>" for beads that were skipped in Phase 3
- Include already-closed beads from the resume case with "Previously closed"

## Generating the Review Summary (REVIEW_SUMMARY)

Derive from the review outcome record compiled in Phase 4.

**If review was APPROVED**:
```
Reviewed by multi-agent team: APPROVED, no critical or high issues found.
```

**If review found issues and fixes were applied**:
```
Reviewed by multi-agent team: N findings identified, M fixed, K deferred.
```

**If review was skipped** (no commits, --skip-review, or user choice):
```
Review skipped: [reason - e.g., "no commits on branch" or "user opted out"]
```

**If review was MANUAL REVIEW REQUIRED and user skipped fixes**:
```
Review flagged disputed findings. User opted to skip automated fixes.
N unresolved findings noted for manual review.
```

Include counts of fixed/skipped/deferred/unresolved only when non-zero.

## Generating Verification Results (VERIFICATION_RESULTS)

List the verification commands that were run during Phase 3 bead implementation.
Use the last known state (from the most recent run of each command).

```markdown
- `mise run lint` - PASS
- `mise run test` - PASS
- `mise run build` - PASS
```

If verification was not run (e.g., no verification section in bead
descriptions), write:
```
No explicit verification commands defined in bead descriptions.
```

## Generating Diff Stats (DIFF_STATS)

Run and include the output of:

```bash
git diff --shortstat main..HEAD
```

Example output:
```
15 files changed, 487 insertions(+), 23 deletions(-)
```

## Output

Save the rendered template to `.claude/pr-descriptions/feat-<BRANCH_NAME>.md`
in the worktree. Create the directory if it does not exist.

Present the full rendered description to the user in the conversation for
review. Allow iterative edits via conversation ("change the summary to...",
"remove the bead table", etc.) and re-save after each edit.
