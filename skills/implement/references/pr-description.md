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
| Commit log | `git log --oneline main..HEAD` | Phase 5 |
| Review outcome | Review outcome record | Phase 4 |
| Verification results | Bead verification runs | Phase 3 |

## Template

The PR body template lives at `templates/pr-body.md`. It contains these
placeholders:

| Placeholder | Content |
|-------------|---------|
| CHANGES_SUMMARY | 2-3 sentences: what this PR accomplishes and why. Derived from the epic description, not a list of beads. |
| DESIGN_DECISIONS | Key tradeoffs and decisions made during implementation, with reasoning. Entry point for reviewers. |
| REVIEW_SUMMARY | One-line or short block summarizing the review outcome from Phase 4. |
| VERIFICATION_RESULTS | List of verification commands run and their results (PASS/FAIL). |

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

## Generating the Review Summary (REVIEW_SUMMARY)

Derive from the review outcome record compiled in Phase 4.

Select the case that matches the Phase 4 review outcome record. The record
has three fields: outcome label, fixes status, and remaining issues. Map
directly from those fields - do not require information the record does not
capture.

**If no findings** (fixes: "none needed"):
```
Reviewed by multi-agent team: no actionable findings.
```

**If findings fixed** (fixes: "applied"):
```
Reviewed by multi-agent team: findings identified and fixed.
```

**If findings remain for manual review** (fixes: "partially applied" or "skipped", or remaining is non-empty):
```
Reviewed by multi-agent team: findings identified. Some remain for manual review.
```

**If review was skipped** (no commits, or user opted out):
```
Review skipped: [reason].
```

Do not include numeric counts of findings, fixes, or deferrals. The review
report contains the details; the PR summary should be qualitative. The agent
may add brief context from the conversation (e.g., "disputed findings
deferred") but the template should not require it.

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

## Output

Save the rendered template to `.claude/pr-descriptions/feat-<BRANCH_NAME>.md`
in the worktree. Create the directory if it does not exist.

Present the full rendered description to the user in the conversation for
review. Allow iterative edits via conversation ("change the summary to...",
"add a note about X", etc.) and re-save after each edit.
