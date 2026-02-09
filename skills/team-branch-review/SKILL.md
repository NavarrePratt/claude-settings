---
name: team-branch-review
description: Comprehensive branch review using a parallel agent team with Codex validation. Use when reviewing all commits on a branch before creating a PR, especially for large or complex branches that benefit from multi-perspective review.
argument-hint: "[optional extra instructions]"
---

# Team Branch Review

Comprehensive code review of all commits on the current branch compared to main. Spawns a team of parallel Claude (Opus) reviewers - each specializing in a different review concern. Each reviewer independently validates their own findings using the Codex MCP, then the lead synthesizes the final report.

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`
- Base commit: !`git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null || echo "main"`

## Instructions

You are the team lead conducting a comprehensive multi-agent code review.

**CRITICAL: You MUST use the Teammate tool to create an agent team and the Task tool (with team_name parameter) to spawn Claude reviewer agents. Each reviewer is an independent Claude agent session, NOT a Codex MCP call. Do NOT substitute direct mcp__codex__codex calls for agent teammates. The Codex MCP is used only BY the reviewer agents internally to validate their own findings - you (the lead) do not call Codex directly.**

**This skill does NOT edit files.** It produces a review report only.

You will:
1. Spawn a team of Claude (Opus) reviewer agents using the Teammate and Task tools
2. Each reviewer explores the code deeply and validates their findings with Codex MCP
3. You collect all validated findings and synthesize the final report

---

### Phase 0: Precondition Check

Before doing anything else, verify you have the required tools:

1. **Check for the Task tool (subagent spawner).** This is the tool that launches new agent sessions with parameters like `subagent_type`, `team_name`, `name`, `model`, and `prompt`. It is NOT the same as TaskCreate/TaskList/TaskUpdate/TaskGet (those manage a task list). Look at your available tools - if you do not have a tool called "Task" that spawns subagents, STOP IMMEDIATELY and tell the user:

   "This skill requires the Task tool (subagent spawner) which is not available in custom agent sessions (claude --agent). Run this skill from a plain `claude` session instead."

   Do NOT attempt workarounds (CLI commands, direct Codex calls, single-agent review). Just stop.

2. **Check for the Teammate tool.** If not available, stop and tell the user:
   "Agent teams are required for this skill. Enable them by setting CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 in your Claude Code settings, then retry."

Do NOT proceed past this phase unless both tools are confirmed available.

---

### Phase 1: Analyze Branch Scope

Substitute BASE_COMMIT from Context above, then run these git commands:

```bash
git diff --shortstat BASE_COMMIT..HEAD
git log --oneline BASE_COMMIT..HEAD
git diff --stat BASE_COMMIT..HEAD
```

Record:
- **lines_changed**: Total lines added + removed
- **files_changed**: List of all changed file paths
- **commit_count**: Number of commits
- **base_commit**: The exact commit hash (use this everywhere, not "main")

### Phase 2: Determine Team Composition

Based on lines_changed:

**Small (< 200 lines)** - 2 reviewers:

| Name | Focus |
|------|-------|
| `reviewer-security` | Security, correctness, logic errors, edge cases, input validation |
| `reviewer-pragmatism` | Architecture, code quality, unnecessary complexity, premature abstraction, YAGNI, over-engineering, locality of behavior |

**Medium (200-1000 lines)** - 4 reviewers:

| Name | Focus |
|------|-------|
| `reviewer-security` | Vulnerabilities, injection, auth gaps, input validation, data exposure |
| `reviewer-correctness` | Logic errors, off-by-one, nil dereferences, concurrency, edge cases |
| `reviewer-architecture` | Design patterns, abstraction, coupling, API design, modularity |
| `reviewer-simplicity` | Over-engineering, premature abstraction, YAGNI, unnecessary complexity, locality of behavior, Chesterton's Fence |

**Large (1000+ lines)** - 6 reviewers:

| Name | Focus |
|------|-------|
| `reviewer-security` | Vulnerabilities, injection, auth gaps, input validation, data exposure |
| `reviewer-correctness` | Logic errors, off-by-one, nil dereferences, race conditions |
| `reviewer-architecture` | Design patterns, coupling, API design, modularity |
| `reviewer-simplicity` | Over-engineering, premature abstraction, YAGNI, unnecessary complexity, locality of behavior, Chesterton's Fence |
| `reviewer-performance` | Allocations, N+1 queries, resource leaks, algorithmic complexity |
| `reviewer-testing` | Coverage gaps, testability, missing test cases, test quality |

### Phase 3: Spawn Team and Reviewer Agents

1. **Create the team**:
   ```
   Teammate(operation: "spawnTeam", team_name: "branch-review", description: "Branch review of BRANCH_NAME")
   ```

2. **Create tasks** for each reviewer using TaskCreate. Each task should include:
   - Subject: "Review FOCUS_AREA for BRANCH_NAME"
   - Description: The reviewer's focus area, base commit hash, and list of changed files
   - activeForm: "Reviewing FOCUS_AREA"

3. **Load reviewer briefs.** Before spawning, read the brief file for each reviewer from `~/.claude/skills/team-branch-review/reviewers/`. The mapping is:

   | Reviewer name | Brief file |
   |---|---|
   | `reviewer-security` | `reviewers/security.md` |
   | `reviewer-correctness` | `reviewers/correctness.md` |
   | `reviewer-architecture` | `reviewers/architecture.md` |
   | `reviewer-simplicity` | `reviewers/simplicity.md` |
   | `reviewer-pragmatism` | `reviewers/pragmatism.md` |
   | `reviewer-performance` | `reviewers/performance.md` |
   | `reviewer-testing` | `reviewers/testing.md` |

   Read ALL relevant brief files (use the Read tool). Substitute each file's full content as the FOCUS_BRIEF in that reviewer's prompt.

4. **Spawn ALL reviewers in parallel** (send all Task calls in a single message):

   ```
   Task(
     subagent_type: "general-purpose",
     team_name: "branch-review",
     name: "reviewer-security",
     model: "opus",
     description: "Security review",
     prompt: "[Reviewer Prompt Template with FOCUS substituted]"
   )
   ```

   Repeat for each reviewer. All spawns happen simultaneously.

### Reviewer Prompt Template

Each reviewer receives this prompt (substitute FOCUS_AREA, FOCUS_DESCRIPTION, FOCUS_BRIEF, BRANCH_NAME, BASE_COMMIT, FILE_LIST, and CWD):

```
You are a senior code reviewer specializing in FOCUS_AREA, part of a parallel review team.

## Assignment

Branch: BRANCH_NAME (base: BASE_COMMIT)
Changed files:
FILE_LIST

Your exclusive focus: FOCUS_DESCRIPTION

## Review Brief

FOCUS_BRIEF

## Process

### Step 1: Claim Your Task
Check TaskList for your assigned task. Claim it with TaskUpdate (set owner to your name, status to in_progress).

### Step 2: Primary Review
1. Understand the branch:
   - Run `git log --oneline BASE_COMMIT..HEAD` to see commits
   - Run `git diff --stat BASE_COMMIT..HEAD` for scope
2. For each changed file relevant to your focus:
   - Read the diff: `git diff BASE_COMMIT..HEAD -- <filepath>`
   - Read surrounding context with the Read tool for the full picture
   - Trace dependencies and callers with Grep and Glob
   - For large files, read the entire file to understand how the change fits
3. Think deeply. Look for subtle issues, not just obvious ones. Consider interactions between changed files.
4. Collect ALL findings - err on the side of reporting too much rather than too little.
5. Send a milestone message to the team lead so they know you are entering the long-running Codex validation phase:

   SendMessage(type: "message", recipient: "team-lead", summary: "FOCUS_AREA primary review done", content: "MILESTONE: Primary review complete. Found N findings across M files. Starting Codex validation.")

   Replace N and M with your actual counts. This message is mandatory before proceeding to Step 3.

### Step 3: Codex Validation

After sending the milestone message, validate your own findings using the Codex MCP tool.

Call `mcp__codex__codex` with:
- `sandbox`: `"read-only"`
- `approval-policy`: `"never"`
- `cwd`: "CWD"

Use this prompt (substitute YOUR_FINDINGS with your actual findings from Step 2):

```
You are a senior code reviewer validating findings from a FOCUS_AREA specialist. Rigorously challenge each finding: confirm real issues, flag false positives, correct severity ratings, and catch anything missed within FOCUS_AREA.

Branch: BRANCH_NAME (base: BASE_COMMIT)

## Findings to Validate

YOUR_FINDINGS

## Your Task

### Part 1: Validate Each Finding

For EACH finding, examine the actual code yourself:
1. Run `git diff BASE_COMMIT..HEAD -- <file>` to see the change
2. Read surrounding context to understand the full picture
3. Determine your verdict:

### Finding: [original title]
- **Verdict**: Confirmed | Disputed | Severity Adjusted | Enhanced
- **Confidence**: High | Medium | Low
- **Rationale**: Specific reasoning with code evidence for your verdict
- **Adjusted Severity**: [only if different from original]
- **Additional Context**: [related issues, broader implications, or missing nuance]

### Part 2: Find Missed Issues

After validating, check the same files for FOCUS_AREA issues the primary review missed.
Report new findings using the same format, prefixed with "NEW -".

### Part 3: Validation Summary

- Findings confirmed: N / total
- Findings disputed: N / total
- Severity adjustments: N
- New findings added: N
```

If the Codex MCP call fails, report your unvalidated findings and note that Codex validation was unavailable.

### Step 4: Report

Send the team lead (via SendMessage, type: "message") your complete results in this format:

## Raw Findings

For each finding:

### [Short descriptive title]
- **File**: path/to/file:line_number
- **Severity**: Critical | High | Medium | Low
- **Category**: [specific concern within FOCUS_AREA]
- **Issue**: Clear description of the problem
- **Evidence**: The specific code snippet or diff excerpt showing the issue
- **Suggestion**: Concrete fix or improvement
- **Confidence**: High | Medium | Low

## Codex Validation Results

[Full Codex validation output from Step 3]

## Summary
- Raw findings: Critical: N, High: N, Medium: N, Low: N
- After Codex validation: N confirmed, N disputed, N adjusted, N new

## Notable Observations
[Broader observations about patterns, quality, or concerns in your focus area]

Then mark your task completed via TaskUpdate (status: completed).
```

### Phase 4: Wait for All Tasks to Complete

Execute this loop. Do not deviate from it.

```
loop:
    result = TaskList()
    if EVERY task in result has status "completed":
        break -> proceed to Phase 5
    else:
        call TaskList() again (continue loop)
```

**If ANY task status is not `completed`, your ONLY permitted next action is to call TaskList again.**

You MUST NOT do any of the following while any task is not `completed`:
- Write, draft, or begin synthesizing the report
- Send shutdown requests to any reviewer
- Proceed to Phase 5 or Phase 6
- Present findings to the user

**Having received messages from all reviewers is NOT a reason to proceed.** Reviewers call SendMessage BEFORE calling TaskUpdate to mark their task `completed`. A message arriving does not mean the reviewer is done. The task status `completed` is the only exit condition for this loop.

**Understanding milestone messages:** Reviewers send a "MILESTONE: Primary review complete... Starting Codex validation." message before entering the long-running Codex MCP call. Use these to track reviewer progress:
- Reviewer has NOT sent a milestone message: still doing primary review (fast phase)
- Reviewer HAS sent a milestone message but no findings yet: inside Codex validation (slow phase, be patient)
- Reviewer HAS sent findings: review is substantively done, waiting for TaskUpdate

**Timeout handling:** If a specific task has been `in_progress` for 20+ consecutive polls with no change AND the reviewer has not yet sent a milestone message, send ONE follow-up message to that reviewer, then continue polling. If the reviewer HAS sent a milestone message (meaning they are in the Codex validation phase), be more patient - allow 40+ polls before sending a follow-up. After 3 follow-up messages to the same reviewer with no status change, declare that reviewer failed and proceed without their findings (note the gap in the report).

Once ALL tasks show status `completed`, compile all findings into a single list tagged by reviewer.

### Phase 5: Synthesize Final Report

You (the lead) produce the final report directly. Deduplicate findings caught by multiple reviewers (note cross-references). Resolve conflicts by favoring the position with stronger code evidence.

Use this report format:

```markdown
# Branch Review: BRANCH_NAME

## Overview
- **Branch**: BRANCH_NAME -> main
- **Commits**: N commits
- **Files Changed**: N files (+X/-Y lines)
- **Review Team**: N Claude reviewers (Opus) + Codex validation
- **Reviewers**: [list of roles that participated]

## Outcome: [APPROVED | NEEDS REVISION | MANUAL REVIEW REQUIRED]

Determination:
- APPROVED: No confirmed Critical or High findings after Codex validation
- NEEDS REVISION: Any confirmed Critical or High findings remain
- MANUAL REVIEW REQUIRED: Significant disagreement (>50% of Critical/High disputed) between Claude and Codex

---

## Critical & High Findings

### [Finding Title]
- **File**: path:line
- **Severity**: Critical/High
- **Category**: [concern area]
- **Issue**: Description
- **Suggestion**: Concrete fix
- **Validation**: [Confirmed/Disputed] by Codex ([confidence]) - [rationale]
- **Found by**: [which reviewer(s)]

[Repeat for each]

---

## Medium & Low Findings

[Same format, grouped by category for readability]

---

## Codex-Only Findings

[Issues discovered by Codex that no Claude reviewer caught]

---

## Disputed Findings

[Findings where Claude and Codex disagreed, with both perspectives and evidence]

---

## Review Statistics
- Total findings (pre-validation): N
- Post-validation: N confirmed, N disputed, N severity-adjusted, N new from Codex
- By severity: Critical: N, High: N, Medium: N, Low: N
- Cross-reviewer overlap: N findings caught by 2+ reviewers
- Claude-Codex agreement rate: X%

## Reviewer Notes
[Notable observations from individual reviewers about overall code quality, patterns, or architectural concerns]

## Commit Recommendations
[Suggestions for commit organization if applicable]
```

### Phase 6: Cleanup

**CRITICAL: You MUST clean up the team before finishing. Skipping cleanup leaves orphaned agent processes.**

**Step 1: Send shutdown requests** to all teammates in parallel (send ALL in a single message):

```
SendMessage(type: "shutdown_request", recipient: "reviewer-security", content: "Review complete")
SendMessage(type: "shutdown_request", recipient: "reviewer-correctness", content: "Review complete")
# ... all remaining reviewers simultaneously
```

**Step 2: Wait for shutdown confirmations.** After sending shutdown requests, you will receive shutdown confirmation messages from each teammate. Wait until you have received confirmations from all teammates before proceeding. If a teammate does not respond within a reasonable time, send the shutdown request again.

**Step 3: Clean up the team** only after all teammates have confirmed shutdown:
```
Teammate(operation: "cleanup")
```

If cleanup fails with "active members" error, wait and retry. Do NOT use `rm -rf` as a workaround - it leaves orphaned processes. If cleanup still fails after 3 retries, report the issue to the user rather than forcing deletion.

---

## Error Handling

| Scenario | Recovery |
|----------|----------|
| Reviewer agent fails or times out | Note the gap in the report, continue with remaining findings |
| Codex MCP unavailable for a reviewer | Reviewer reports unvalidated findings, noted in report |
| Team spawn fails | Fall back to single-agent review (run codex-branch-review instead) |
| No findings from any reviewer | Report "APPROVED - no issues found" with note about review coverage |

## Guidelines

- **All Claude agents use Opus**: Set `model: "opus"` for every spawned agent
- **Parallel execution**: Spawn ALL reviewer agents simultaneously in one message
- **Fixed base commit**: Use the exact hash from Phase 1 everywhere, never "main"
- **Distributed Codex validation**: Each reviewer calls Codex MCP to validate their own findings (runs in parallel)
- **Don't embed diffs in prompts**: Let agents and Codex gather diffs via git commands themselves
- **Deep exploration**: Reviewers should use Read, Grep, Glob extensively - not just skim diffs
- **No severity inflation**: Findings should use honest, appropriate severity levels
- **Phase 4 is a hard gate**: Do NOT write the report until every task status is `completed`. Receiving messages from reviewers does not satisfy this requirement. This is non-negotiable.

## Chaining to Fix Implementation

After presenting the report, if the outcome is NEEDS REVISION or MANUAL REVIEW REQUIRED, add this note:

> To implement fixes for these findings, run `/team-branch-fix`. It will let you choose which findings to fix, then spawn agents to implement the changes in parallel.

$ARGUMENTS
