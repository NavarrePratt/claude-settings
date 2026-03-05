---
name: team-branch-fix
description: Implement fixes for code review findings using a parallel agent team. Accepts a review report (from /team-branch-review or pasted), interviews the user on which findings to fix, then spawns agents to implement fixes in parallel with Codex validation.
argument-hint: "[review report or path to report]"
---

# Team Branch Fix

Implement code fixes for review findings using a parallel agent team. Each agent owns an exclusive set of files, implements all fixes for those files, and self-validates with Codex MCP. Runs verification (lint/test/build) after all agents complete.

## Reference Files

This skill uses reference files in the `references/` directory. Load referenced files
with the Read tool before executing the relevant phase. If a reference file is missing,
use the inline fallback instructions in the phase description.

| Reference | Used in |
|-----------|---------|
| [complexity-rubric.md](references/complexity-rubric.md) | Phase 2 Step 1 - trivial vs complex classification |
| [approach-generation.md](references/approach-generation.md) | Phase 2 Step 1 - generating fix approaches for complex findings |
| [codex-validation.md](references/codex-validation.md) | Phase 2 Step 1 - two-step Codex review flow |
| [decision-schema.md](references/decision-schema.md) | Phases 1-3 - finding data model and field lifecycle |
| [blocked-findings.md](references/blocked-findings.md) | Phase 7.5 - blocked finding resolution protocol |

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`

## Instructions

You are the team lead coordinating parallel implementation of code fixes from a review report. You will interview the user to decide which findings to fix, group work by file ownership, spawn fix agents, and verify the results.

**CRITICAL: You MUST use the AskUserQuestion tool for ALL user-facing questions in Phase 2. Do NOT print questions as text output and wait for free-form responses. Every question about whether to fix, skip, or defer a finding MUST be an AskUserQuestion tool call with structured options. This is the only way to give the user a proper interactive experience.**

**CRITICAL: You MUST use the TeamCreate tool to create an agent team and the Task tool (with team_name parameter) to spawn fix agents. Each fixer is an independent Claude agent session. Do NOT attempt workarounds if these tools are missing.**

---

### Phase 0: Precondition Check

Before anything else, verify required tools and git state.

**Step 1: Tool availability**

1. **Check for the Task tool (subagent spawner).** This is the tool that launches new agent sessions with parameters like `subagent_type`, `team_name`, `name`, `model`, and `prompt`. It is NOT the same as TaskCreate/TaskList/TaskUpdate/TaskGet (those manage a task list). Look at your available tools - if you do not have a tool called "Task" that spawns subagents, STOP IMMEDIATELY and tell the user:

   "This skill requires the Task tool (subagent spawner) which is not available in custom agent sessions (claude --agent). Run this skill from a plain `claude` session instead."

   Do NOT attempt workarounds. Just stop.

2. **Check for the TeamCreate tool.** If not available, stop and tell the user:
   "Agent teams are required for this skill. Enable them by setting CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 in your Claude Code settings, then retry."

**Step 2: Git state**

Verify the git state is safe to proceed:

```bash
git status --short
git log --oneline $(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main)..HEAD
```

**Require both:**
1. **Clean working tree**: No uncommitted changes (staged or unstaged). If dirty, stop and tell the user to commit or stash their changes first.
2. **Commits on branch**: At least one commit ahead of main. If the branch has no commits vs main, there's nothing to fix - stop and tell the user.

Do NOT proceed past this phase unless both conditions are met.

**Step 3: Compute team name**

Compute a unique team name: take the branch name, replace `/` with `-`, truncate to 30 chars, then prefix with `fix-` and append `-` plus the first 6 chars of HEAD's commit hash. Example: branch `feat/add-auth` at commit `a1b2c3d` becomes `fix-feat-add-auth-a1b2c3`. Record this as **TEAM_NAME** and use it everywhere a team_name is needed.

Create temp directory for fixer results:
```bash
mkdir -p /tmp/fix-TEAM_NAME
```

### Phase 1: Gather Review Report

Obtain the review report from one of these sources (check in order):

1. **Conversation context**: If a `/team-branch-review` report exists in the conversation above, use it directly
2. **$ARGUMENTS**: If arguments are provided, treat them as the report or a path to one
3. **Ask the user**: If no report is available, ask the user to provide one

Parse the report and extract all findings into a structured list. Assign each finding a sequential `finding_id` (format: `f-1`, `f-2`, ...) and classify its `finding_scope`:

- `finding_id`: Sequential identifier (`f-{N}`) assigned during parsing
- `finding_scope`: One of:
  - `line` - targets a specific file:line location
  - `file` - targets a whole file (no specific line)
  - `cross-cutting` - systemic issue not tied to a specific file (e.g., "add error handling everywhere")
- Finding title
- File and line number
- Severity (Critical/High/Medium/Low)
- Category
- Issue description
- Suggested fix
- Codex validation status (Confirmed/Disputed/Adjusted)
- Found by (which reviewer)

See [Decision Schema](references/decision-schema.md) for the full finding data model including decision tracking fields populated in later phases.

### Phase 1.5: Finding Canonicalization

Before presenting findings to the user, deduplicate and group them so the interview operates on canonical findings rather than raw duplicates.

**Step 1: Classify scope**

For each finding, verify the `finding_scope` assigned in Phase 1:
- Has file AND line number -> `line`
- Has file but no line number -> `file`
- No specific file (or references multiple files as a pattern) -> `cross-cutting`

**Step 2: Group duplicates**

Apply grouping rules by scope:

- **`line`-scoped**: Group findings that share the same `(file, line)` AND describe the same underlying issue. Two findings from different reviewers catching the same nil dereference at the same location are duplicates. Two findings at the same line for different issues (e.g., nil check vs naming convention) are NOT duplicates.
- **`file`-scoped**: Group findings that share the same `file` AND describe the same issue.
- **`cross-cutting`**: Group findings that describe the same systemic concern, regardless of wording. For example, "add error handling to all API endpoints" and "missing error handling in API layer" are the same concern.

**Step 3: Assign canonical IDs**

For each group:
1. Assign a sequential `group_id` (`g-1`, `g-2`, ...)
2. Set `source_finding_ids` to the list of all member `finding_id` values
3. Pick one finding as the canonical representative (`is_canonical: true`) - prefer the one with the most specific suggested fix
4. Merge `found_by` lists (union of all members)
5. Use the highest `severity` among members
6. Use the most specific `suggested_fix` among members

**Step 4: Output**

Replace the raw finding list with the deduplicated canonical list. Each entry carries its `group_id` and `source_finding_ids` so downstream phases can trace back to the original findings. Only canonical findings (`is_canonical: true`) proceed to Phase 2.

Announce the dedup results to the user: "Parsed N findings, deduplicated to M canonical findings (K duplicates merged)."

### Phase 2: User Interview

**You MUST call the AskUserQuestion tool for every question below. Do NOT print questions as plain text.**

See also:
- [Complexity Rubric](references/complexity-rubric.md) - scoring criteria for trivial vs complex
- [Approach Generation](references/approach-generation.md) - how to draft fix approaches
- [Codex Validation](references/codex-validation.md) - two-step Codex review flow
- [Decision Schema](references/decision-schema.md) - data model for finding decisions

**Step 0: Infer session default**

Before processing findings, analyze the review report and branch context to infer a default approach style. Check these signals:

| Signal | Suggests |
|--------|----------|
| Branch name contains `fix/`, `hotfix/`, `patch/` | `minimal_patch` |
| Most findings are Critical/High severity | `minimal_patch` |
| Branch name contains `refactor/`, `cleanup/` | `refactor` |
| Most findings are Medium/Low with pattern issues | `refactor` |
| Commit messages mention stability, reliability | `defensive` |
| No strong signal | `minimal_patch` (safe default) |

Display the inference and reason to the user in assistant text, then offer a one-click override:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "Inferred default: [style] based on [reason]. Change?",
    header: "Fix style",
    options: [
      { label: "Keep [style]", description: "Use [style] as the default for all findings" },
      { label: "Minimal patches", description: "Smallest change that fixes each issue" },
      { label: "Defensive fixes", description: "Fix plus guards against related failures" },
      { label: "Case by case", description: "No default - choose approach for every finding individually" }
    ],
    multiSelect: false
  }]
```

"Case by case" means no session default is set. Effects:
- Complex findings still get approach generation, but no approach is pre-marked as "(Recommended)" since there's no default style to prefer.
- Trivial findings still use the standard Fix/Skip/Defer flow - they do NOT get approach generation.
- Fixer agents fall through to precedence rule 4 (minimal-change) instead of rule 3 (session default) for cases where no explicit approach was chosen.

The session default determines which approach is listed first (recommended position) when generating approaches for complex findings.

**Step 1: Critical & High findings**

For each confirmed Critical or High finding:

1. **Classify complexity**: Read ~50 lines of code around the finding and score against the [complexity rubric](references/complexity-rubric.md). Score >= 2 = complex. If uncertain, classify complex. *(Fallback if reference missing: +1 each for multi-function scope, multiple valid approaches, new patterns required, public API impact, non-obvious fix. Threshold: >= 2.)*

2. **If trivial** (score < 2): Use the standard flow - call AskUserQuestion:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "[Finding title] in [file:line] - [one-line issue description]. Fix this?",
    header: "High finding",
    options: [
      { label: "Fix (Recommended)", description: "Apply the suggested fix" },
      { label: "More info", description: "Investigate the code in depth before deciding" },
      { label: "Skip", description: "Intentional or not a real issue" },
      { label: "Defer", description: "Fix in a later session" }
    ],
    multiSelect: false
  }]
```

3. **If complex** (score >= 2): Generate 2 concrete approaches per [approach generation](references/approach-generation.md). Present via AskUserQuestion with markdown previews (~10 lines each). Include "Validate with Codex" option per [codex validation](references/codex-validation.md). Store chosen approach in [decision schema](references/decision-schema.md). Limit to 2 approaches so options stay within AskUserQuestion's 4-option maximum (2 approaches + "Validate with Codex" + "Skip").

   *(Fallback if reference files missing: draft 2 approaches from different styles - pick 2 of minimal_patch, defensive, refactor. Each needs a short label, one-line description, and ~10 line code preview. Session default goes first. Include "Validate with Codex" and "Skip" options. If user validates with Codex, call mcp__codex__codex with approaches + code context, then re-present with assessments baked in. Store decision.approach, decision.approach_detail, decision.approach_source.)*

**If the user selects "More info"** (trivial findings only):

Investigate the finding in depth:
1. Read the file around the flagged line with the Read tool (include generous context, ~50 lines around the issue)
2. Trace callers and dependents with Grep to show impact
3. Check git blame to understand why the code exists: `git log --oneline -3 -- <file>`
4. Present a summary to the user:
   - The actual code in question (with surrounding context)
   - What the reviewer flagged and why
   - What the suggested fix would change
   - Potential impact: what other code depends on this
   - Risk assessment: how likely is this to cause problems if left unfixed

Then re-ask with AskUserQuestion (without the "More info" option):
```
Call AskUserQuestion tool with:
  questions: [{
    question: "With the above context - fix [Finding title] in [file:line]?",
    header: "Fix decision",
    options: [
      { label: "Fix (Recommended)", description: "Apply the suggested fix" },
      { label: "Skip", description: "Intentional or not a real issue" },
      { label: "Defer", description: "Fix in a later session" }
    ],
    multiSelect: false
  }]
```

**Step 2: Disputed Critical & High findings**

For findings where Claude and Codex disagreed, resolve stance first, then handle complexity.

**Stance resolution**: Call AskUserQuestion to pick whose assessment to trust:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "[Finding title] in [file:line] - DISPUTED. Claude: [issue]. Codex: [counter]. Whose assessment?",
    header: "Disputed",
    options: [
      { label: "Trust Claude", description: "[Claude's position in one line]" },
      { label: "Trust Codex", description: "[Codex's position in one line]" },
      { label: "More info", description: "Investigate both positions before deciding" },
      { label: "Skip", description: "Not a real issue" }
    ],
    multiSelect: false
  }]
```

If "More info", follow the investigation procedure from Step 1 but also highlight the specific disagreement with code evidence for each position. Then re-ask without "More info".

After stance is resolved, record `decision.stance` (`claude` or `codex`).

**Then classify complexity** using the winning stance's suggested fix:
- **If trivial**: Use the stance winner's suggested fix directly. Set `decision.disposition: "fix"`.
- **If complex**: Generate 2 approaches within the chosen stance (approaches should align with the assessment the user trusted). Present via AskUserQuestion with markdown previews, same pattern as Step 1 complex flow.

**Step 3: Medium & Low findings**

Batch these by category using AskUserQuestion:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "N medium/low findings in [category]. How to handle?",
    header: "Med/Low",
    options: [
      { label: "Fix all (Recommended)", description: "Apply all suggested fixes in this category" },
      { label: "Review individually", description: "Decide on each finding separately" },
      { label: "Skip all", description: "Skip all findings in this category" }
    ],
    multiSelect: false
  }]
```

If **"Fix all"**: Batch approve all findings in the category. Fixer agents handle implementation using the suggested fixes.

If **"Review individually"**: Present each finding using the trivial-path AskUserQuestion (Fix/More info/Skip/Defer) - do NOT classify complexity up front for Med/Low findings. Complexity classification is deferred to the "More info" click:

- If user picks **"Fix"**: use suggested fix directly (no complexity check needed).
- If user picks **"More info"**: classify complexity. If complex (score >= 2), generate approaches with the same flow as Step 1 complex findings. If trivial, follow the standard investigation (read code, trace callers, git blame) then re-ask without "More info".

This is the opt-in path for approach selection on lower-severity findings. It avoids slowing the interview with approach generation for findings the user would have accepted as-is.

**Step 4: Summary**

Present the final fix plan to the user:
- N findings to fix (by severity)
- N findings skipped
- N findings deferred
- Files that will be modified
- For complex findings with chosen approaches: list the finding and chosen approach label

Call AskUserQuestion to confirm before proceeding:
```
Call AskUserQuestion tool with:
  questions: [{
    question: "Proceed with fixing N findings across M files?",
    header: "Confirm plan",
    options: [
      { label: "Proceed (Recommended)", description: "Spawn fix agents and implement all approved fixes" },
      { label: "Review plan again", description: "Go back and change fix decisions" },
      { label: "Cancel", description: "Abort without making changes" }
    ],
    multiSelect: false
  }]
```

### Phase 3: Build Fix Plan

Findings arriving here are already deduplicated via Phase 1.5 canonical groups. Each finding carries a `group_id` and `source_finding_ids` from canonicalization.

**Use canonical groups, not raw findings.** When grouping work for agents, operate on the canonical finding list (entries where `is_canonical: true`). The `group_id` is the stable identifier passed to fixer agents and used in results tracking.

**Early exit if nothing to fix.** If no canonical findings have `disposition == "fix"`, report "No fixes to apply - all findings were skipped or deferred." and skip to Phase 9 (with an empty fix summary).

**Preserve user decisions.** Each canonical finding carries its full `decision` from Phase 2. When building the fix plan, only include findings where `disposition == "fix"`. Forward these decision fields to the fixer agent:
- `decision.approach`: the label of the user's chosen approach (complex findings only)
- `decision.approach_detail`: the code preview showing the intended fix (complex findings only)
- `decision.approach_source`: whether the choice was `default`, `user_choice`, or `codex_validated`
- `decision.stance`: for disputed findings, whose assessment (`claude` or `codex`) the user trusted

The fixer agent must respect the chosen approach. For trivial findings without an explicit approach, the fixer uses the `suggested_fix` from the review report.

**Group** all approved canonical findings by file path. Each unique file (or tight cluster of related files) becomes one agent's work unit.

**Grouping rules:**
- All findings for the same file go to one agent
- If a fix requires changes in two files that import each other, group them together
- Cross-cutting findings (no specific file) go to a dedicated agent, or are distributed across agents if each instance targets identifiable files
- Aim for roughly balanced work across agents, but never split a file across agents
- Maximum ~8 files per agent to keep context manageable

Record the fix plan (include canonical IDs for traceability):
- Agent 1: [files] - [N findings] (group_ids: g-1, g-3, g-7)
- Agent 2: [files] - [N findings] (group_ids: g-2, g-4)
- ...

### Phase 4: Discover Verification Commands

Run a focused search to find exact development commands. Check in order:

1. `mise.toml` / `.mise.toml` (mise task runner)
2. `package.json` scripts / `pyproject.toml` / `Makefile` / `Justfile`
3. `.github/workflows` (CI jobs are authoritative)
4. `docs/CONTRIBUTING.md` or `README.md`

For each category, find the EXACT command:
- **Lint/format**: `mise run lint`, `golangci-lint run`, `ruff check .`, `npm run lint`, etc.
- **Static analysis**: `mise run check`, `mypy .`, `staticcheck ./...`, `tsc --noEmit`, etc.
- **Unit tests**: `mise run test`, `go test ./...`, `pytest`, `npm run test`, etc.
- **Integration/E2E**: `mise run test:e2e`, `pytest tests/e2e/`, `go test -tags=integration`, etc.

Store discovered commands for Phase 6.

### Phase 5: Spawn Fix Team

**If team creation fails** (TeamCreate returns an error): fall back to spawning a single Agent (no team) that implements all fixes sequentially. Use the same fixer prompt template but include all findings in one prompt. Skip team-related operations (TaskCreate, TaskList polling). Read the single agent's results directly.

1. **Create the team**:
   ```
   TeamCreate(team_name: "TEAM_NAME", description: "Fix implementation for BRANCH_NAME")
   ```

2. **Create tasks** for each agent using TaskCreate:
   - Subject: "Fix N findings in [file list summary]"
   - Description: Structured payload with file list and per-finding approach directives:
     ```
     Files: [file list]

     Finding f-N: [title]
       File: path:line
       Issue: [description]
       Suggested fix: [reviewer's suggestion]
       Approach: [user's chosen approach or "Use suggested fix"]
       Approach source: default | user_choice | codex_validated
     ```
   - activeForm: "Fixing [file list summary]"

3. **Spawn ALL fix agents in parallel** (all Task calls in a single message):

   ```
   Task(
     subagent_type: "general-purpose",
     team_name: "TEAM_NAME",
     name: "fixer-1",
     model: "opus",
     description: "Fix findings in [files]",
     prompt: "[Fix Agent Prompt Template]"
   )
   ```

### Fix Agent Prompt Template

Each fix agent receives this prompt (substitute FILE_LIST, FINDINGS, CWD, TEAM_NAME, and FIXER_NAME):

```
You are implementing code fixes for specific review findings. You have exclusive ownership of your assigned files - no other agent will touch them.

## Your Files (EXCLUSIVE OWNERSHIP)

FILE_LIST

Do NOT modify any files outside this list.

## Findings to Fix

FINDINGS

Each finding includes:
- **Finding ID**: canonical finding_id (f-N) from review report
- **File and line number**
- **Issue description**
- **Suggested fix**: the reviewer's original suggestion
- **Approach**: the user's chosen approach description, or "Use suggested fix" for trivial findings
- **Approach Source**: `default` | `user_choice` | `codex_validated`
- **Codex validation status**

## Process

### Step 1: Claim Your Task
Check TaskList for your assigned task. Claim it with TaskUpdate (set owner to your name, status to in_progress).

### Step 2: Understand Context
For each file you own:
1. Read the full file with the Read tool
2. Understand the surrounding code, not just the flagged lines
3. Check how your files relate to each other (imports, callers)

### Step 3: Implement Fixes
For each finding, in order of severity (Critical first, then High, Medium, Low):
1. Re-read the file BEFORE each fix - line numbers from the review report may have shifted due to earlier fixes in the same file
2. Find the actual code referenced by the finding (match by content, not just line number)
3. Implement the fix using the Edit tool
4. Verify the fix doesn't break surrounding code
5. If a fix would conflict with a higher-priority fix you already applied, skip it and note why

**Fix approach precedence** (use the first applicable rule):
1. **User's chosen approach** (when approach_source is `user_choice` or `codex_validated`): implement exactly what the approach describes. The user selected this approach for a reason.
2. **Suggested fix from review report** (when approach is "Use suggested fix" or approach_source is `default`): apply the reviewer's suggested fix.
3. **Session default style** (inferred by team lead in Phase 2 Step 0): if neither above applies, follow the session's default fix style (minimal_patch, refactor, or defensive).
4. **Minimal-change** (last resort): fix the issue with the smallest correct change.

**When the chosen approach allows refactoring** (e.g., "defensive refactor", "restructure"):
- Refactoring is allowed ONLY within your owned files
- No public API changes unless the approach explicitly calls for it
- Document the refactoring scope in your results file

**General quality rules:**
- Match existing code style exactly
- Don't add unnecessary comments explaining the fix

### Step 3.5: Handle Infeasible Approaches

After attempting all fixes, check if any finding's chosen approach was not viable after reading the code in context. This applies to both individually-approved findings and "Fix all" batch-approved findings.

**If an approach is not viable, do NOT silently pick an alternative.** Instead:
1. Mark the finding as **Blocked** in the results file
2. Include: why the approach is not viable given the actual code
3. Provide 2 concrete fallback options you considered (with brief descriptions)
4. Continue with remaining findings normally
5. Mark your task completed as usual - blocked findings are handled by the team lead

This is the safety net for hidden complexity. It is better to report a finding as blocked than to improvise a fix the user did not approve.

### Step 4: Self-Validate with Codex

After implementing all fixes, validate your changes using the Codex MCP tool.

Call `mcp__codex__codex` with:
- `sandbox`: `"read-only"`
- `approval-policy`: `"never"`
- `cwd`: "CWD"

Prompt:
```
Review these code changes for correctness. The changes implement fixes for specific review findings.

## Changes Made

[List each fix: what was changed and why]

## Verify

1. Run `git diff` to see all changes
2. For each changed file, verify:
   - The fix addresses the reported issue
   - No new issues were introduced
   - Code style is consistent
   - No regressions in surrounding logic
3. Report any problems found
```

If Codex finds issues with your fixes, correct them and re-validate.

If the Codex MCP tool is unavailable or times out, skip validation and note in your results file under Codex Validation: "Codex MCP unavailable - fixes not validated." Continue to Step 5.

### Step 5: Write Results to File

Write your results to `/tmp/fix-TEAM_NAME/FIXER_NAME.md` using the Write tool. Use this exact format:

## Fixes Applied
For each fix:
- **Finding ID**: f-N
- **Finding**: [original title]
- **File**: path:line
- **Approach used**: [chosen approach label, or "suggested fix"]
- **Change**: What was changed
- **Status**: Applied | Modified (how the fix differed from approach)

## Blocked Findings
For each finding where the chosen approach was not viable:
- **Finding ID**: f-N
- **Finding**: [original title]
- **File**: path:line
- **Chosen approach**: What the user selected
- **Why blocked**: Why this approach is not viable given the actual code
- **Fallback A**: [concrete alternative with brief description]
- **Fallback B**: [concrete alternative with brief description]

## Codex Validation
[Codex validation results - confirmed clean or issues found and corrected]

## Skipped Findings
[Any findings skipped due to conflicts with higher-priority fixes, with explanation]

After writing the results file, mark your task completed via TaskUpdate (status: completed). The team lead will read your results from the file after all fixers have finished.
```

### Phase 6: Wait for All Agents to Complete

**CRITICAL: You MUST NOT call TaskUpdate to change the status of any fixer task.** Only the fixer agent that owns a task may call TaskUpdate on it. If you believe a fixer is stuck, send them a message - do not mark their task completed yourself.

Execute this loop. Do not deviate from it.

```
loop:
    result = TaskList()
    if EVERY task in result has status "completed":
        break -> proceed to reading results files, then Phase 7.5
    else:
        call TaskList() again (continue loop)
```

**If ANY task status is not `completed`, your ONLY permitted next action is to call TaskList again.**

You MUST NOT do any of the following while any task is not `completed`:
- Run verification commands
- Send shutdown requests to any fixer
- Proceed to Phase 7.5, Phase 7, or Phase 8
- Present results to the user
- Call TaskUpdate on any fixer task

**Messages from fixers are NOT completion signals.** Fixers write their results to files and then call TaskUpdate to mark their task `completed`. The task status `completed` is the only exit condition for this loop.

**Timeout handling:** If a specific task has been `in_progress` for 20+ consecutive polls with no change, send ONE follow-up message to that fixer, then continue polling. After 3 follow-up messages to the same fixer with no status change, declare that fixer failed and proceed without their results (note the gap in the report).

**Collecting results:** Once ALL tasks show status `completed`, read each fixer's results file:

```
For each fixer:
    Read /tmp/fix-TEAM_NAME/{fixer-name}.md
```

Compile a summary of all results:
- Which findings were fixed (and how)
- Which findings were blocked (approach not viable, with fallback options)
- Which findings were skipped by agents (and why)
- Which agents failed (and what findings they owned)

### Phase 7.5: Resolve Blocked Findings

See [blocked-findings.md](references/blocked-findings.md) for the full resolution protocol, results file format, and follow-up fixer rules.

**Process summary:**

1. Parse blocked findings from fixer results files (`## Blocked Findings` section)
2. If none, skip to Phase 7
3. For each blocked finding, call AskUserQuestion with Fallback A, Fallback B, and Skip options
4. If any fallbacks approved: spawn follow-up fixers (same prompt template, same ownership, results to `{name}-followup.md`)
5. Poll follow-up fixers via Phase 6 loop
6. If follow-up fixer also blocks: report as **unresolved** (max 1 re-entry, no infinite loops)
7. Proceed to Phase 7

*(Fallback if reference file missing: for each blocked finding, present the fixer's two fallback options via AskUserQuestion. Spawn follow-up fixers for approved fallbacks using the Phase 5 prompt template. Max 1 follow-up round - if that also blocks, mark unresolved. Team lead never implements fixes directly.)*

### Phase 7: Verify

**Gating condition:** Only enter this phase when no pending follow-up fixer tasks remain. If Phase 7.5 spawned follow-up fixers, their polling and results collection must complete before verification runs. Verification covers ALL changes (original fixes + follow-up fixes).

Run the discovered verification commands:

```bash
# Run in order, stop and report if any fail
[discovered lint command]
[discovered static analysis command]
[discovered test command]
[discovered e2e command]  # if applicable
```

If verification fails:
1. Identify which fix caused the failure (check git diff by file)
2. Report the failure to the user with the failing command output
3. Do NOT attempt to auto-fix verification failures - let the user decide

### Phase 8: Cleanup

Shut down all teammates in parallel (all shutdown requests in a single message):

```
SendMessage(type: "shutdown_request", recipient: "fixer-1", content: "Fixes complete")
SendMessage(type: "shutdown_request", recipient: "fixer-2", content: "Fixes complete")
# ... all remaining fixers simultaneously
```

After all teammates confirm shutdown:
```
TeamDelete()
```

Clean up temp directory:
```bash
rm -rf /tmp/fix-TEAM_NAME
```

### Phase 9: Final Report and Commit Strategy

Present the summary to the user:

```markdown
## Fix Implementation Summary

### Overview
- Findings fixed: N / N approved
- Findings resolved via fallback: N (originally blocked, fixed after user chose fallback)
- Findings skipped by user after block: N (user chose Skip in Phase 7.5)
- Findings unresolved: N (follow-up fixer also blocked; max re-entry hit)
- Findings skipped during implementation: N (conflicts with higher-priority fixes)
- Findings deferred: N
- Files modified: N

### Fixes Applied
[For each fix: finding_id, finding title, file:line, approach used, what changed]

### Blocked Finding Resolution
[For each finding that was originally blocked by a fixer:]
- **Finding ID**: f-N
- **Finding**: [title]
- **File**: [file:line]
- **Original approach**: [what was first attempted]
- **Why blocked**: [fixer's reason]
- **Resolution**: Resolved via fallback [A/B] | Skipped by user | Unresolved (follow-up also blocked)
- **Fallback used** (if resolved): [which fallback and what changed]

### Verification Results
- Lint: PASS/FAIL
- Static analysis: PASS/FAIL
- Tests: PASS/FAIL
- E2E: PASS/FAIL (if applicable)

### Skipped During Implementation
[Any findings skipped due to conflicts with higher-priority fixes]

### Codex Validation
- All fixes validated: YES/NO
- Issues found and corrected during validation: N
```

**Commit strategy with unresolved findings:** If unresolved blocked findings remain (follow-up also blocked), note them in the commit summary but still offer commit options for the fixes that did succeed. The commit should not be held up by findings the user will handle manually.

**If verification passed**, call the AskUserQuestion tool for commit strategy:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "All fixes verified. How do you want to commit the changes?",
    header: "Commit",
    options: [
      { label: "Fixup into original commits (Recommended)", description: "Amend each fix into the branch commit that introduced the issue" },
      { label: "Single fix commit", description: "All fixes in one commit with a summary message" },
      { label: "Multiple commits", description: "Group fixes into separate logical commits" },
      { label: "Don't commit", description: "Leave changes uncommitted for manual review" }
    ],
    multiSelect: false
  }]
```

Once the user selects a strategy (other than "Don't commit"), spawn a subagent to handle the commits:

```
Task(
  subagent_type: "general-purpose",
  model: "sonnet",
  description: "Commit review fixes",
  prompt: "[Commit Agent Prompt - see below]"
)
```

Use Sonnet for the commit agent since this is mechanical work that doesn't need Opus.

**Commit Agent Prompt** (substitute STRATEGY, FIXES_APPLIED, and BASE_COMMIT):

```
You are committing code review fixes. The fixes are already applied and verified. Use the /commit skill for all commits.

Strategy: STRATEGY

Fixes applied:
FIXES_APPLIED

## Instructions by Strategy

### If "Fixup into original commits":
For each fix, identify which commit on the branch introduced the code being fixed:
1. Run `git log --oneline BASE_COMMIT..HEAD -- <file>` and `git blame <file>` for the fixed lines
2. Determine the target commit
3. Stage only the files for this fix: `git add <specific files>`
4. Create a fixup commit: `git commit --fixup=<target-commit-hash>`
5. Repeat for each fix

After all fixup commits are created, inform the user they can run `git rebase -i --autosquash BASE_COMMIT` to fold them in.

### If "Single fix commit":
Stage all changed files and use the /commit skill to create one commit summarizing all fixes.

### If "Multiple commits":
Group the fixes by logical concern (e.g., all security fixes together, all correctness fixes together).
For each group:
1. Stage only the files for that group: `git add <specific files>`
2. Use the /commit skill to create a commit for that group

Report back what commits were created.
```

**Don't commit**: Leave all changes in the working tree. Suggest the user review with `git diff`.

**If verification failed**, do NOT offer commit options. Report the failures and let the user decide how to proceed.

---

## Error Handling

| Scenario | Recovery |
|----------|----------|
| Fix agent fails or times out | Note which findings weren't fixed, continue with others |
| Codex MCP unavailable for an agent | Agent reports unvalidated fixes, noted in report |
| Codex unavailable during approach validation | Note unvalidated in option description, proceed without |
| Markdown previews don't render | Fall back to description-only options |
| Verification fails | Report failure details, do NOT auto-fix |
| Team spawn fails | Fall back to single-agent fix implementation |
| No findings approved by user | Report "No fixes to apply" and exit |
| User declines all fallback options for blocked finding | Mark as skipped in final report |
| Follow-up fixer also blocked (max re-entry) | Report as unresolved, user handles manually |
| Complexity classification timeout | Default to complex, proceed with approach generation |
| Reference file missing | Use inline fallback instructions in the phase description |

## Guidelines

- **All Claude agents use Opus**: Set `model: "opus"` for every spawned agent
- **Exclusive file ownership**: Never assign the same file to two agents
- **Parallel execution**: Spawn ALL fix agents simultaneously in one message
- **Respect approach precedence**: User's chosen approach > suggested fix > session default > minimal-change
- **Preserve user decisions**: Only fix what the user approved. Block (don't improvise) when approach is not viable.
- **Report blocked findings**: Fixer agents mark infeasible approaches as Blocked with fallback options for the team lead.
- **Self-validate**: Each agent validates with Codex before reporting done
- **Don't auto-fix verification failures**: Report and let the user decide
- **Never mark fixer tasks**: You MUST NOT call TaskUpdate on any fixer task. Only fixers mark their own tasks completed.
- **Results come from files**: Fixers write results to `/tmp/fix-TEAM_NAME/`. Do not use message content as results.
- **Bounded iteration**: Phase 7.5 allows at most 1 follow-up round. If a follow-up fixer also blocks, the finding becomes unresolved. No infinite loops.
- **Verify after all fixes**: Phase 7 verification runs only after all fixers (original + follow-up) have completed.

$ARGUMENTS
