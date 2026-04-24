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
| Commit log | `git log --oneline <BASE_REF>..HEAD` | Phase 5 |
| Review outcome | Review outcome record | Phase 4 |
| Verification results | Bead verification runs (Phase 3) or post-fix re-verification (Phase 4) | Phase 3/4 |
| Bead summary files | `/tmp/<REPO_NAME>-<EPIC_ID>/<BEAD_ID>-summary.md` (subagent mode) | Phase 3 |
| REPO_NAME and EPIC_ID | Phase 2 state / conversation context | Phase 0/2 |

## Template

The PR body template lives at `templates/pr-body.md`. It contains these
placeholders:

| Placeholder | Content |
|-------------|---------|
| CHANGES_SUMMARY | 2-3 sentences: what this PR accomplishes and why. Derived from the epic description, not a list of beads. |
| DESIGN_DECISIONS | Key tradeoffs and decisions made during implementation, with reasoning. Entry point for reviewers. |
| VERIFICATION_RESULTS | Checkbox list of verification commands. Checked items passed; unchecked items were not run. |
| FOLLOW_UP_DISCOVERIES | Bulleted list of TODOs, related bugs, and follow-up work surfaced during implementation. Sourced from the Discoveries section of each bead's summary file. |

## PR Title vs Body

The PR body renders from `templates/pr-body.md` using the placeholder
substitution above. The PR title is derived separately and passed to
`gh pr create --title` explicitly - never embedded in the body or left for
GitHub to infer. This avoids Conventional-Commits validators rejecting an
auto-inferred title that lacks a required prefix.

## Generating the PR Title (PR_TITLE)

### Constraints

- ≤70 characters on the final serialized title (including any prefix/scope)
- No trailing period
- Imperative mood, user-visible outcome first
- No bead IDs, no internal codenames (existing project-wide rule)

### Conventional Commits

Use Conventional Commits for the PR title only when the repo clearly
expects them. The type list and the "match recent commits" detection
rule are already documented in
[git-commit/SKILL.md](../../git-commit/SKILL.md) ("Commit Message
Format Detection"); apply the same judgment here, with two
PR-title-specific adjustments:

- **Sample the base branch, not the current branch.** A feature
  branch may not yet follow the repo's convention. Use:

  ```bash
  HISTORY_REF="$BASE_REF"
  git show-ref --verify --quiet "refs/remotes/origin/$BASE_REF" && HISTORY_REF="origin/$BASE_REF"
  git log --format=%s --no-merges -n 30 "$HISTORY_REF"
  ```

- **Look inside `.github/workflows/`** for direct evidence that the
  PR title itself is validated. A bare `commitlint` invocation is
  commit-message linting, not PR-title validation. If a PR-title
  validator is configured and its rules narrow the format beyond
  the standard types (required scope, restricted type list, lower
  length cap, etc.), ask the user for the exact required shape
  rather than guessing.

Type list uses the git-commit skill's set plus `revert` (for PRs that
revert a prior commit). If the signal is mixed - no workflow
validator, base-branch commits don't clearly follow CC - ask the user
instead of guessing.

### Subject Writing Guidance

- Imperative mood ("add", "fix", "surface") - not past tense or gerund
- User-visible outcome first - lead with what the PR does for a reader
- No trailing period
- Strip nothing substantive: RFC/AIP/standards references are part of the
  subject, not tracking tokens

Examples:

- Yes: `feat: surface AIP-193 error details on SDK exceptions`
- No: `feat: AIP-193 Error Details in SDK Exceptions.`

### Normalization

Applied before saving to `pr-title.txt`:

- Collapse internal whitespace; trim outer whitespace
- Enforce a single logical line (no embedded newlines)
- 70-char limit on the final serialized title, including any prefix/scope
- Strip any trailing period
- Preserve substantive subject content (do NOT strip RFC/AIP/standards
  references; the project-wide rule against bead IDs and internal codenames
  covers what should be removed)

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
Bounded the review-fix pipeline to one review pass and one fix pass rather
than looping until clean. Unbounded loops risk burning tokens on diminishing
returns when the reviewer and fixer disagree on style issues. A single
bounded pass catches real bugs while keeping cost predictable.

Chose to invoke /team-branch-review as a skill rather than calling the agent
team directly. This keeps the review pipeline decoupled from /implement so
either can evolve independently, at the cost of slightly less control over
reviewer configuration.
```

## Generating Verification Results (VERIFICATION_RESULTS)

List the verification commands as a GitHub-flavored markdown checkbox list.
Use the most recent run of each command.

**If Phase 4 made code changes** (fix commits exist): use the post-fix
re-verification results from Phase 4 Step 2, not the Phase 3 results. The
Phase 3 results are stale after fix changes.

**If Phase 4 made no code changes** (no fixes applied, or review was clean):
use the Phase 3 results directly.

Rules:

- Passing commands render as checked boxes: `` - [x] `cmd` ``
- Commands that could not be run in this environment render unchecked:
  `` - [ ] `cmd` ``. Unchecked means "deferred / not yet run", not "failed".
- Failing commands must not appear here. If a verification command fails,
  the pipeline should not have reached PR-description generation - stop
  and surface the failure instead of papering over it.

Example:

```markdown
- [x] `mise run lint`
- [x] `mise run test`
- [ ] `mise run test:e2e` (requires live cluster; run before merge)
```

If verification was not run (e.g., no verification section in bead
descriptions), write:
```
No explicit verification commands defined in bead descriptions.
```

## Generating Follow-up Discoveries (FOLLOW_UP_DISCOVERIES)

Each bead subagent writes a summary file at
`/tmp/<REPO_NAME>-<EPIC_ID>/<BEAD_ID>-summary.md` with a `## Discoveries`
section capturing TODOs, related bugs, and follow-up work surfaced during
implementation. These signals are the most valuable part of the summary
for the next reader — without preservation they die when `/tmp` is cleaned
up. Roll them into the PR body so they stay with the change.

1. Enumerate bead summary files in `/tmp/<REPO_NAME>-<EPIC_ID>/`. Use the
   list of completed beads from Phase 3 to know which summaries to expect.
2. For each summary, extract the `## Discoveries` section. Same regex
   pattern used for `## Verification` and `## Design Decisions` elsewhere:
   match from `## Discoveries` through the next `## ` heading or
   end-of-file.
3. Normalize each extracted bullet so the reader can trace provenance:
   prefix with `[from <BEAD_ID>]`, e.g. `- [from bd-xxx] Observed that the
   retry logic in foo.py:120 double-counts failures.`
4. Deduplicate identical bullets across summaries (different beads can
   surface the same underlying issue).
5. Aggregate into a single bulleted list and substitute into
   `FOLLOW_UP_DISCOVERIES`.

Rules:

- A bead summary with `## Discoveries` containing only `None` or empty
  content contributes nothing; skip silently.
- If no bead in the epic produced any discoveries, write:
  ```
  None identified during implementation.
  ```
- If the run used inline mode (no per-bead summary files exist), write:
  ```
  Inline implementation — Discoveries not structurally tracked.
  ```
  Inline mode keeps the work in the lead's conversation rather than
  spawning subagents, so there are no summary files to aggregate from.

Example:

```markdown
- [from bd-aaa] The error path in `src/auth/middleware.ts:78` swallows
  the underlying exception. Worth tracing in a follow-up.
- [from bd-aaa] `User.is_admin` is checked in two places with subtly
  different logic; consider consolidating.
- [from bd-bbb] The fixture loader assumes UTC; tests may drift on other
  runners.
```

Do NOT file new beads for these discoveries automatically. The PR body is
a human-readable channel; if the lead or user wants to track a discovery
as work, they run `/issue-create` explicitly.

## Output

Two artifacts, saved side by side in the directory created during Phase 2
(worktree setup). Both files share a base name of `<EPIC_ID>-<BRANCH_NAME>`
(recorded as **ARTIFACT_BASENAME** in worktree-setup.md Step 6) so
concurrent `/implement` sessions produce distinguishable filenames in
editor tabs:

- `/tmp/<REPO_NAME>-<EPIC_ID>/<EPIC_ID>-<BRANCH_NAME>.md` - rendered body
  from `templates/pr-body.md` with placeholders substituted
- `/tmp/<REPO_NAME>-<EPIC_ID>/<EPIC_ID>-<BRANCH_NAME>.txt` - derived title
  as a single line of plain text, already normalized per the rules above

Present both the title and the full body to the user in the conversation
for review. Allow iterative edits via conversation ("change the summary
to...", "use `fix:` not `feat:`", "add a note about X", etc.) and re-save
the affected file after each edit.

Explicit approval gate: Phase 6 Push-and-create-PR must not run until the
user has approved the full PR draft (both title and body). Approvals can
arrive across separate messages in an iterative edit flow; track both
artifacts as approved independently, then proceed only when both are
approved and the current content is what was approved.
