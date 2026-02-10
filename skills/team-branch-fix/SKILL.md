---
name: team-branch-fix
description: Implement fixes for code review findings using a parallel agent team. Accepts a review report (from /team-branch-review or pasted), interviews the user on which findings to fix, then spawns agents to implement fixes in parallel with Codex validation.
argument-hint: "[review report or path to report]"
---

# Team Branch Fix

Implement code fixes for review findings using a parallel agent team. Each agent owns an exclusive set of files, implements all fixes for those files, and self-validates with Codex MCP. Runs verification (lint/test/build) after all agents complete.

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`

## Instructions

You are the team lead coordinating parallel implementation of code fixes from a review report. You will interview the user to decide which findings to fix, group work by file ownership, spawn fix agents, and verify the results.

**CRITICAL: You MUST use the AskUserQuestion tool for ALL user-facing questions in Phase 2. Do NOT print questions as text output and wait for free-form responses. Every question about whether to fix, skip, or defer a finding MUST be an AskUserQuestion tool call with structured options. This is the only way to give the user a proper interactive experience.**

**CRITICAL: You MUST use the Teammate tool to create an agent team and the Task tool (with team_name parameter) to spawn fix agents. Each fixer is an independent Claude agent session. Do NOT attempt workarounds if these tools are missing.**

---

### Phase 0: Precondition Check

Before anything else, verify required tools and git state.

**Step 1: Tool availability**

1. **Check for the Task tool (subagent spawner).** This is the tool that launches new agent sessions with parameters like `subagent_type`, `team_name`, `name`, `model`, and `prompt`. It is NOT the same as TaskCreate/TaskList/TaskUpdate/TaskGet (those manage a task list). Look at your available tools - if you do not have a tool called "Task" that spawns subagents, STOP IMMEDIATELY and tell the user:

   "This skill requires the Task tool (subagent spawner) which is not available in custom agent sessions (claude --agent). Run this skill from a plain `claude` session instead."

   Do NOT attempt workarounds. Just stop.

2. **Check for the Teammate tool.** If not available, stop and tell the user:
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

Parse the report and extract all findings into a structured list:
- Finding title
- File and line number
- Severity (Critical/High/Medium/Low)
- Category
- Issue description
- Suggested fix
- Codex validation status (Confirmed/Disputed/Adjusted)
- Found by (which reviewer)

### Phase 2: User Interview

**You MUST call the AskUserQuestion tool for every question below. Do NOT print questions as plain text.**

**Step 1: Critical & High findings**

For each confirmed Critical or High finding, call the AskUserQuestion tool:

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

**If the user selects "More info":**

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

For findings where Claude and Codex disagreed, call AskUserQuestion with both perspectives:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "[Finding title] in [file:line] - DISPUTED. Claude: [issue]. Codex: [counter]. Fix?",
    header: "Disputed",
    options: [
      { label: "Fix per Claude", description: "Apply Claude's suggested fix" },
      { label: "Fix per Codex", description: "Apply Codex's suggested fix" },
      { label: "More info", description: "Investigate both positions before deciding" },
      { label: "Defer", description: "Fix in a later session" }
    ],
    multiSelect: false
  }]
```

If "More info", follow the same investigation procedure as Step 1, but also highlight the specific disagreement between Claude and Codex with code evidence for each position. Then re-ask without the "More info" option.

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

If "Review individually", present each finding with the same AskUserQuestion pattern as Step 1 (including "More info" option).

**Step 4: Summary**

Present the final fix plan to the user:
- N findings to fix (by severity)
- N findings skipped
- N findings deferred
- Files that will be modified

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

**First, deduplicate.** The review report may contain the same issue flagged by multiple reviewers (e.g., security and correctness reviewers both catch a nil dereference). Identify findings that reference the same file and line with the same underlying issue. Merge duplicates into a single finding, noting which reviewers flagged it. Use the most specific suggested fix among the duplicates.

**Then group** all approved findings by file path. Each unique file (or tight cluster of related files) becomes one agent's work unit.

**Grouping rules:**
- All findings for the same file go to one agent
- If a fix requires changes in two files that import each other, group them together
- Aim for roughly balanced work across agents, but never split a file across agents
- Maximum ~8 files per agent to keep context manageable

Record the fix plan:
- Agent 1: [files] - [N findings]
- Agent 2: [files] - [N findings]
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

1. **Create the team**:
   ```
   Teammate(operation: "spawnTeam", team_name: "TEAM_NAME", description: "Fix implementation for BRANCH_NAME")
   ```

2. **Create tasks** for each agent using TaskCreate:
   - Subject: "Fix N findings in [file list summary]"
   - Description: The agent's file list, findings, and suggested fixes
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

Each fix agent receives this prompt (substitute FILE_LIST, FINDINGS, and CWD):

```
You are implementing code fixes for specific review findings. You have exclusive ownership of your assigned files - no other agent will touch them.

## Your Files (EXCLUSIVE OWNERSHIP)

FILE_LIST

Do NOT modify any files outside this list.

## Findings to Fix

FINDINGS

Each finding includes:
- File and line number
- Issue description
- Suggested fix
- Codex validation status

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

**Fix quality rules:**
- Match existing code style exactly
- Minimal changes - fix the issue, don't refactor surrounding code
- If the suggested fix is wrong or incomplete, use your judgment for a better fix
- Don't add unnecessary comments explaining the fix

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

### Step 5: Write Results to File

Write your results to `/tmp/fix-TEAM_NAME/FIXER_NAME.md` using the Write tool. Use this exact format:

## Fixes Applied
For each fix:
- **Finding**: [original title]
- **File**: path:line
- **Change**: What was changed
- **Status**: Applied | Skipped (reason) | Modified (how the fix differed from suggestion)

## Codex Validation
[Codex validation results - confirmed clean or issues found and corrected]

## Skipped Findings
[Any findings that couldn't be fixed, with explanation]

After writing the results file, mark your task completed via TaskUpdate (status: completed). The team lead will read your results from the file after all fixers have finished.
```

### Phase 6: Wait for All Agents to Complete

**CRITICAL: You MUST NOT call TaskUpdate to change the status of any fixer task.** Only the fixer agent that owns a task may call TaskUpdate on it. If you believe a fixer is stuck, send them a message - do not mark their task completed yourself.

Execute this loop. Do not deviate from it.

```
loop:
    result = TaskList()
    if EVERY task in result has status "completed":
        break -> proceed to reading results files, then Phase 7
    else:
        call TaskList() again (continue loop)
```

**If ANY task status is not `completed`, your ONLY permitted next action is to call TaskList again.**

You MUST NOT do any of the following while any task is not `completed`:
- Run verification commands
- Send shutdown requests to any fixer
- Proceed to Phase 7 or Phase 8
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
- Which findings were skipped by agents (and why)
- Which agents failed (and what findings they owned)

### Phase 7: Verify

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
Teammate(operation: "cleanup")
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
- Findings skipped: N (with reasons)
- Findings deferred: N
- Files modified: N

### Fixes Applied
[For each fix: finding title, file:line, what changed]

### Verification Results
- Lint: PASS/FAIL
- Static analysis: PASS/FAIL
- Tests: PASS/FAIL
- E2E: PASS/FAIL (if applicable)

### Skipped During Implementation
[Any findings that agents couldn't fix, with explanations]

### Codex Validation
- All fixes validated: YES/NO
- Issues found and corrected during validation: N
```

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
| Verification fails | Report failure details, do NOT auto-fix |
| Team spawn fails | Fall back to single-agent fix implementation |
| No findings approved by user | Report "No fixes to apply" and exit |

## Guidelines

- **All Claude agents use Opus**: Set `model: "opus"` for every spawned agent
- **Exclusive file ownership**: Never assign the same file to two agents
- **Parallel execution**: Spawn ALL fix agents simultaneously in one message
- **Minimal changes**: Fix the reported issue, don't refactor surrounding code
- **Preserve user decisions**: Only fix what the user approved
- **Self-validate**: Each agent validates with Codex before reporting done
- **Don't auto-fix verification failures**: Report and let the user decide
- **Never mark fixer tasks**: You MUST NOT call TaskUpdate on any fixer task. Only fixers mark their own tasks completed.
- **Results come from files**: Fixers write results to `/tmp/fix-TEAM_NAME/`. Do not use message content as results.

$ARGUMENTS
