# Bead Implementation Procedure

Per-bead claim, load, implement, verify, close/skip cycle for the /implement
skill, plus commit strategy guidance.

## Implementation Mode Decision

The lead agent decides implementation mode based on the work characteristics:

**Favor inline when:** few beads, tightly coupled work sharing files or
context, or when the lead needs deep understanding of all changes for later
review and PR phases.

**Favor subagent when:** multiple independent beads, different modules or
files, or context budget is a concern.

The lead evaluates these tradeoffs at runtime rather than following a fixed
bead count threshold.

## Per-Bead Cycle

### Step 1: Claim

```bash
br --db "$MAIN_REPO_BEADS_DB" update <BEAD_ID> --status in_progress --json
```

This prevents duplicate work if another session picks up the same bead.

### Step 2: Load Context

```bash
br --db "$MAIN_REPO_BEADS_DB" show <BEAD_ID> --json
```

Extract from the JSON:
- `description` - full bead description with acceptance criteria
- `dependencies` - any upstream beads (for context on what was already done)

Parse the `## Verification` section from the description to extract
verification commands. These are typically formatted as:
```
## Verification
- [ ] `command1` passes
- [ ] `command2` passes
```

Extract the backtick-enclosed commands into a list for Step 4.

#### Fetch Parent Epic Design Decisions

Parent epics document design choices that apply to every child bead. Fetch
the parent and extract its `## Design Decisions` section so the
implementer sees the "why" alongside the "what":

```bash
br --db "$MAIN_REPO_BEADS_DB" show <PARENT_EPIC_ID> --json
```

Apply the same regex extraction pattern used for `## Verification` — match
from `## Design Decisions` through the next `## ` heading or end of
description. Bind the extracted text to the `EPIC_DESIGN_DECISIONS`
template variable used by `templates/bead-prompt.md`.

Fallbacks:
- Parent epic has no `## Design Decisions` section → render `EPIC_DESIGN_DECISIONS`
  as the literal string `None captured.` (the template assumes non-empty)
- Bead has no parent epic (rare for /implement, since epic resolution is
  the entry point) → render `EPIC_DESIGN_DECISIONS` as `Not applicable.`

### Step 3: Implement

#### Inline Mode

The lead agent implements directly:

1. Read the bead description carefully
2. Use Read, Edit, Write, Bash, Grep, Glob tools to implement
3. Follow existing project patterns (check CLAUDE.md, existing code)
4. Stay focused on the bead's scope - do not expand beyond acceptance criteria

#### Subagent Mode

Spawn one Agent per bead, wait for completion, read the summary, then
spawn the next. Sequential execution keeps the worktree in a consistent
state and avoids file-level conflicts between concurrent agents.

```
Agent(
  description: "Implement <BEAD_ID>",
  prompt: <rendered template for bead>,
  mode: "bypassPermissions"
)
```

**Why bypassPermissions**: subagents need to run verification commands, edit
files, and execute build tooling without interactive prompts. The worktree
provides isolation from the main repo, limiting blast radius.

**Timeout caveat**: Agent() is a blocking foreground call with no enforced
timeout parameter. The "10-minute timeout" is aspirational guidance for
subagent prompt design (telling the agent to be concise), not an enforced
mechanism. If an agent hangs, the lead agent hangs too.

**Recovery from a hung subagent**: the user can interrupt the session
(Ctrl+C or equivalent). The interrupted session cannot perform cleanup -
both the subagent and the lead are terminated, leaving the bead in
in_progress status. Recovery happens in the next session or via manual
intervention:

From the main repo (plain `br` resolves the beads DB automatically):

```bash
br update <BEAD_ID> --status open --notes "SKIPPED: subagent appeared hung, user interrupted previous session"
```

From inside a worktree (pass `--db` to point at the main repo's beads DB):

```bash
br --db "$MAIN_REPO_BEADS_DB" update <BEAD_ID> --status open --notes "SKIPPED: subagent appeared hung, user interrupted previous session"
```

Run `br list --status in_progress` to surface beads that were left
in_progress, including any stuck from a prior hung session. Reset with
`br update <id> --status open`. The bead remains available for a future
session to pick up.

Template variables to substitute in templates/bead-prompt.md:
- `BEAD_ID` - the bead identifier
- `BEAD_TITLE` - short title from bead
- `BEAD_DESCRIPTION` - full description text
- `BEAD_PARENT` - parent epic ID
- `EPIC_DESIGN_DECISIONS` - parent epic's `## Design Decisions` section, extracted the same way as `## Verification`; falls back to `None captured.` when the epic lacks the section, or `Not applicable.` when there is no parent epic
- `WORKTREE_PATH` - absolute path to the worktree
- `VERIFICATION_COMMANDS` - extracted verification commands
- `PRIOR_SUMMARIES` - context from previously completed beads (capped; see Cross-Bead Context)
- `REPO_NAME` - repository directory name
- `EPIC_ID` - epic bead ID (for artifact namespace)

After the agent completes, read its summary file:
```
/tmp/<REPO_NAME>-<EPIC_ID>/<BEAD_ID>-summary.md
```

Accumulate summaries for PRIOR_SUMMARIES in subsequent bead prompts.

### Step 4: Verify

**Verification contract**: in subagent mode, the subagent runs bead
verification commands as part of its implementation (see bead-prompt.md
instruction 3). Step 4 applies when the lead implements inline, or as a
confirmation check after a subagent reports results. The lead separately
runs post-wave integration verification (see Post-Wave Verification below).

Run each verification command extracted in Step 2.

**If zero verification commands were extracted** (no `## Verification` section
in the bead description): do not silently skip verification. Instead:

1. Check for project-level verification commands (from CLAUDE.md, Makefile,
   mise tasks, package.json scripts, or other discovered tooling).
2. If project-level commands exist, run those as a fallback.
3. If no verification is available at all, warn the user and get explicit
   confirmation to proceed without verification via AskUserQuestion.

**If all pass**: proceed to Step 5 (close).

**If any fail**: assess the error type before deciding on retry.

- **Trivial failures** (typos, missing imports, wrong paths): fix and re-run.
  These are mechanical errors that the fix addresses completely.
- **Deeper failures** (test failures from architectural mismatches, integration
  issues, missing dependencies): fail fast. The bead can be retried later with
  more context. Attempting a quick fix for a structural problem wastes time.

The lead assesses the error to decide, not a fixed retry count.

### Step 5: Close on Success

```bash
br --db "$MAIN_REPO_BEADS_DB" close <BEAD_ID> --reason "<brief summary of what was done>"
```

The reason should be concise - the diff captures the what; the reason
captures the why.

### Step 6: Handle Failure

```bash
br --db "$MAIN_REPO_BEADS_DB" update <BEAD_ID> --status open --notes "SKIPPED: <what failed and why>"
```

Record the bead as skipped. Include which verification failed, what was
attempted, and why it did not work.

Continue to the next bead. Skipped beads do not block subsequent waves
unless they have explicit blocks dependencies.

## Cross-Bead Context (Subagent Mode)

When using subagents, the lead maintains continuity through summary files:

1. Each subagent writes a summary to `/tmp/<REPO_NAME>-<EPIC_ID>/<BEAD_ID>-summary.md`
2. Summary format:
   - Files created or modified (paths only)
   - What the bead accomplished (one paragraph)
   - Discoveries or follow-up needed
3. Before spawning the next subagent, the lead reads all prior summaries
4. Prior summaries are injected into the next bead prompt via PRIOR_SUMMARIES

### Capping Prior Context

PRIOR_SUMMARIES grows with each bead. To keep subagent prompts within
reasonable token budgets:

- **Current wave + immediately preceding wave**: include full summaries from
  beads in these two waves.
- **Older waves**: include only the "Files Changed" section from each summary.
  The file list provides enough context for path-level awareness without
  carrying full narrative from every prior bead.

This keeps total PRIOR_SUMMARIES size roughly proportional to two waves of
work rather than the entire epic history.

**Example**: An epic has three waves. When spawning a bead in Wave 3:

```
PRIOR_SUMMARIES for Wave 3 bead:

--- Wave 2 (full summary) ---
## bd-ccc: Add validation middleware
Files changed: src/middleware/validate.ts, src/routes/api.ts
Added request validation middleware using zod schemas. Wired into
the API router for all POST endpoints.

--- Wave 1 (file list only) ---
## bd-aaa
Files changed: src/models/user.ts, src/db/migrations/003_add_email.sql
## bd-bbb
Files changed: src/config/defaults.ts
```

Wave 2 gets full summaries (narrative + files) because its changes are
recent enough to matter for implementation decisions. Wave 1 is reduced
to file lists - enough to avoid path conflicts without burning tokens
on stale narrative.

## Post-Wave Verification

After all beads in a wave complete and BEFORE committing, the lead runs
integration verification across the wave's changes. This catches cross-bead
interactions that individual bead verification cannot detect.

1. Collect the union of all verification commands from beads in the wave
   (extracted from each bead's `## Verification` section in Step 2).
2. Deduplicate commands (multiple beads may share the same lint/test/build).
3. Run each unique command in the worktree.
4. **If all pass**: proceed to commit.
5. **If any fail**: diagnose the failure. If it is a cross-bead integration
   issue (e.g., conflicting imports, incompatible changes), fix it before
   committing. If the failure is unrelated to the wave's changes, note it
   and proceed.

This verification is the lead's responsibility. Subagents run their own bead
verification during implementation (bead-prompt.md instruction 3), but the
lead re-runs verification at the wave boundary to catch integration issues.

## Commit Strategy

The lead agent decides when and how to commit based on the work.

### Default: Wave-Boundary Commits

Accumulate changes across a wave, then group into logical commits at the
boundary. This is the default because it produces the cleanest history when
beads are tightly coupled within a wave.

### Simplification: Per-Bead Commits

When beads are clearly independent (different modules, no shared files),
commit after each bead completes. This is simpler and maximizes atomicity.

### Grouping Heuristic (for wave-boundary commits)

After all beads in a wave complete, review accumulated changes and group
into logical commits:

1. **Survey**: `git diff --stat` and `git diff --name-only`

2. **Group by functional area**:
   - Same-concern files together (source + config, handler + route)
   - Test files with the code they test
   - Unrelated changes in separate commits
   - Each commit should introduce a single coherent idea

3. **Create commits**: stage files for each logical group and use `/commit`
   to create the commit with proper formatting.

### Skipped Beads

Before committing, check for partial changes from skipped beads. Revert
incomplete work with `git checkout -- <files>` if it would break the build.
Otherwise leave it for the user to review.

### Commit Message Content

- Explain why the change was made, not what files were changed
- Reference the epic being implemented for context
- Do not include bead IDs (internal tracking detail)
- Do not include specific counts of items (counts go stale)

## Error Recovery

| Scenario | Recovery |
|----------|----------|
| br update fails (bead already claimed) | Warn user, skip bead |
| Subagent crashes (control returns to lead) | Lead resets bead to open, records as skipped |
| Subagent appears hung (no progress) | User interrupts session (Ctrl+C); bead left in_progress - next session or manual `br update` resets to open |
| Verification command not found | Skip that check, warn in output |
| All beads in a wave skipped | Log warning, proceed to next wave |
| Git conflicts in worktree | Stop execution, report to user |
| Commit history too messy to salvage | Use /clean-copy to rewrite the branch with clean narrative history |
