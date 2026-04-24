---
name: grug-cut-pass
description: Cuts unnecessary complexity (slop) from a branch before PR submission and amends the cuts back into the original commits. Full workflow that applies changes: runs a Codex grug-brain pass to find premature abstraction, dead code, unreachable defensive checks, YAGNI hooks, and indirection without benefit; walks the user through cut/keep/defer decisions one-by-one; amends cuts into origin commits via interactive rebase; verifies tests still pass. Use this skill proactively whenever the user wants to clean up, trim, tighten, or cut slop from a branch before they submit a PR. Triggers include "grug cut pass", "grug pass", "cut the slop", "trim the fat", "clean up before PR", "yak shaving removal", "prune this branch", "amend slop out of commits", "remove over-engineering", "grug-brain sweep", "pre-PR cleanup", or any phrasing about removing complexity before review. Unlike /grug-review which only produces a checklist, this skill actually applies the cuts and rebases - reach for it when the user wants action, not just feedback.
argument-hint: "[--auto] [--rebase] [--single-commit] [--epic <EPIC_ID>]"
---

# Grug Cut Pass

Eight-phase workflow that identifies and removes unnecessary complexity from a branch before PR submission. Codex does the detection with grug-brain framing; the user decides each cut; cuts amend back into their origin commits; tests verify the result.

This skill is separate from `/grug-review` (which is a read-only checklist) and `/team-branch-review` / `/team-branch-fix` (which target correctness/security/architecture). Grug cut pass targets over-engineering: premature abstraction, YAGNI violations, unreachable defensive code, indirection without benefit.

## Reference Files

Load with Read before executing the relevant phase.

| Reference | Used in |
|-----------|---------|
| [cut-classification.md](references/cut-classification.md) | Phase 3 - Slop/Acceptable/Borderline taxonomy |
| [cut-decision-schema.md](references/cut-decision-schema.md) | Phases 3-5 - finding data model |
| [phase-0-preconditions.md](references/phase-0-preconditions.md) | Phase 0 - tool + git state checks |
| [phase-1-scope.md](references/phase-1-scope.md) | Phase 1 - scope detection and upstack impact map |
| [phase-3-codex-pass.md](references/phase-3-codex-pass.md) | Phase 3 - Codex invocation mechanics |
| [phase-5-apply-cuts.md](references/phase-5-apply-cuts.md) | Phase 5 - interactive rebase and amend flow |
| Reused: `team-branch-fix/references/blocked-findings.md` | Phase 6 - fallback protocol when a cut breaks tests |

## Templates

| Template | Used in |
|-----------|---------|
| [codex-prompt.md](templates/codex-prompt.md) | Phase 3 - prompt substituted with FILE_LIST, BASE_REF, diff |
| [final-report.md](templates/final-report.md) | Phase 8 - summary format |

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`
- Arguments: $ARGUMENTS

## Instructions

You are the orchestrator for the cut pass. The user runs this skill on a branch they are about to PR and wants to remove slop before review. Your job is to detect candidates, get user decisions, apply cuts safely, and verify the branch still works.

**CRITICAL: Use AskUserQuestion for every user-facing question in Phases 1, 2, 4, 6, and 7. Do NOT print questions as free text and wait for typed answers. Structured options only.**

**CRITICAL: Never push, never create PRs, never submit to remote. This skill operates locally. Explicit user action elsewhere handles remote operations.**

Parse arguments:
- `--auto` - apply the conservative auto-approve path in Phase 4; still asks on borderline findings
- `--rebase` - run Phase 2 automatically; otherwise Phase 2 prompts via AskUserQuestion
- `--single-commit` - Phase 5 produces one cut-pass commit rather than amending origin commits
- `--epic <EPIC_ID>` - explicit epic to load planning context from; falls back to branch-name parse otherwise

---

### Phase 0: Preconditions

Load [phase-0-preconditions.md](references/phase-0-preconditions.md) and run its checks:

- Git repo check
- Working tree clean (stash or abort if dirty)
- Codex MCP available
- BASE_REF discovery
- Git-spice detection (sets STACKED flag)

If any check fails, stop with an actionable message. Do not attempt workarounds.

### Phase 1: Scope Detection

Load [phase-1-scope.md](references/phase-1-scope.md).

Default scope: current branch vs BASE_REF. If STACKED and current branch is part of a gs stack, ask the user (AskUserQuestion) whether to scope to current branch only or whole stack. Whole-stack adds an upstack impact map to the findings context passed to Codex.

Build FILE_LIST from `git diff --name-only <BASE_REF>..HEAD`. If the file list is empty, report "no commits on branch" and exit.

After FILE_LIST is built, follow `~/.claude/skills/shared/planning-context.md` to produce **planning_context** using the `--epic` flag value (if passed) or the branch-name parse fallback. This block is passed through to Codex in Phase 3 so slop detection respects intentional design decisions — a single-impl abstraction the plan explicitly chose for a second implementation is Acceptable, not Slop.

### Phase 2: Mechanical Rebase

If `--rebase` flag is set, run rebase automatically. Otherwise AskUserQuestion: "Rebase onto latest origin/<default> before the cut pass?" with options `Rebase` / `Skip`.

If user chose Rebase:
- `git fetch origin <DEFAULT_BRANCH>`
- `git rebase origin/<DEFAULT_BRANCH>` (single branch) or `gs repo sync && gs stack restack` (whole stack)
- On conflict: stop the pipeline, surface conflicted files, instruct user to resolve and re-invoke

Skip cleanly if branch is already up to date.

### Phase 3: Codex Grug Pass

Load [phase-3-codex-pass.md](references/phase-3-codex-pass.md) and [cut-classification.md](references/cut-classification.md).

Substitute variables into [codex-prompt.md](templates/codex-prompt.md):
- FILE_LIST, BASE_REF, DIFF_TEXT
- UPSTACK_IMPACT (empty unless whole-stack scope)

Invoke `mcp__codex__codex` with:
- `approval-policy: "never"`
- `sandbox: "read-only"`
- prompt: the substituted template

Parse Codex response as YAML findings. Save to `/tmp/grug-cut-<BRANCH_SLUG>/findings.yaml`. Each finding conforms to [cut-decision-schema.md](references/cut-decision-schema.md).

If no findings are returned, skip to Phase 8 and report "no slop detected".

### Phase 4: User Walk-Through

Canonicalize findings: merge overlapping cuts (same file, adjacent lines) into a single decision.

**If `--auto`:**
- Auto-approve findings where `classification == Slop` AND `confidence == High` AND `anti_pattern_tag` is in the allowlist (dead_code, commented_code, single_impl_abc, unreachable_error_handling, unused_param_new_code)
- For all other findings (Slop+Medium, Slop+Low, any Borderline), ask user
- Log auto-decisions to `/tmp/grug-cut-<BRANCH_SLUG>/auto-decisions.log` for the final report

**Interactive path:**
- Present Slop findings first (sorted by confidence), then Borderline
- For each, AskUserQuestion with options `Cut` / `Keep` / `Defer`
- `Defer` records a follow-up item but does not apply the cut

Save decisions to `/tmp/grug-cut-<BRANCH_SLUG>/decisions.yaml`.

If user chose `Cut` on zero findings, skip Phase 5 and go to Phase 7.

### Phase 5: Apply Cuts

Load [phase-5-apply-cuts.md](references/phase-5-apply-cuts.md).

**Default (amend-per-origin):**
- Group approved cuts by origin commit via `git blame -L <line>,<line> -- <file>`
- If blame spans multiple commits for a single cut, ask user which commit to amend into
- Process origin commits in reverse chronological order with `git rebase -i`
- Apply cuts with Edit, `git add`, `git commit --amend --no-edit`, `git rebase --continue`
- On rebase conflict: stop, preserve state, offer `git rebase --abort` or manual continue

**`--single-commit` flag:**
- Apply all cuts in working tree
- Single `git commit -m "grug: cut pass"`

**Stacked branches (whole-stack scope):**
- Apply cuts per branch
- Run `gs stack restack` after each branch's cuts land
- Never `gs stack submit` (no remote ops)

### Phase 6: Verify

Discover verification commands using the same logic as `team-branch-fix` (scan mise.toml / package.json / Makefile / .github/workflows / docs for lint, test, type-check commands).

Run in order: lint, type-check, tests. Stop on first failure.

**On failure:**
- Identify the likely breaking cut (last-applied cut heuristic or bisect)
- AskUserQuestion with options `Revert this cut` / `Keep and fix separately` / `Abort pipeline`
- If the block pattern matches `blocked-findings.md` (cut was load-bearing), use that protocol's Fallback A/B/Skip flow

Record verification outcome in `/tmp/grug-cut-<BRANCH_SLUG>/verification.log`.

### Phase 7: Optional Chain

AskUserQuestion: "Cuts applied and verified. Run /team-branch-review for correctness validation?" with options `Run review` / `Skip`.

If `Run review`, invoke `/team-branch-review`. If that produces findings, offer to invoke `/team-branch-fix`. Follow the same invocation pattern used by `/implement` Phase 4.

### Phase 8: Cleanup and Report

Write the final report using [final-report.md](templates/final-report.md) as the template. Fields:
- Findings counts by classification
- Cuts applied / kept / deferred
- Verification result
- Optional review outcome
- Auto-mode decision log (if `--auto` was set)

Remove `/tmp/grug-cut-<BRANCH_SLUG>/` directory.

Do not push. Do not create a PR. Tell the user the branch is ready for them to submit.

---

## Error Recovery Quick Reference

| Failure | Action |
|---------|--------|
| Dirty tree in Phase 0 | Offer stash or abort |
| Codex unavailable in Phase 3 | Fall back to `/grug-review` with notice; abort cut pass |
| Rebase conflict (Phase 2 or 5) | Stop, surface files, user resolves manually |
| Tests fail in Phase 6 | Revert-cut / keep-and-fix-separately / abort |
| Load-bearing cut (blocks tests) | Use blocked-findings.md Fallback A/B/Skip |
| State file missing mid-phase | Offer restart from Phase 0 or abort |
| User Ctrl+C | `git rebase --abort` if a rebase is in progress; preserve any completed amends |
