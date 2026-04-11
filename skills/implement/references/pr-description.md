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
| CHANGES_LIST | Bulleted list of significant changes grouped by functional area. NOT grouped by bead or commit. |
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

## Generating the Changes List (CHANGES_LIST)

Group changes by functional area, not by bead or by commit.

1. Review `git diff --stat main..HEAD` to understand the scope
2. Read commit messages with `git log --oneline main..HEAD`
3. Categorize changes into functional areas (e.g., "Skill definition",
   "Templates", "Reference docs", "Error handling")
4. Write one bullet per significant change within each area
5. Omit trivial formatting or whitespace changes

Example:
```markdown
- Add review-fix-pipeline.md reference with pre-review checks, outcome
  parsing, and bounded iteration rule
- Wire Phase 4 into SKILL.md with /team-branch-review and /team-branch-fix
  invocation
- Add error handling entries for review pipeline failures
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
Review skipped: [reason - e.g., "no commits on branch" or "user opted out"].
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

Save the rendered template to `.claude/pr-descriptions/{branch-name}.md`
in the worktree. Create the directory if it does not exist.

Present the full rendered description to the user in the conversation for
review. Allow iterative edits via conversation ("change the summary to...",
"remove the bead table", etc.) and re-save after each edit.
