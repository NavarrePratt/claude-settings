---
name: parallel-branch-review
description: Comprehensive branch review using parallel background agents with Codex validation. Each reviewer runs as an independent background task - results are collected only after all agents complete, eliminating premature synthesis. Use for thorough pre-PR review of any branch size.
argument-hint: "[optional extra instructions]"
---

# Parallel Branch Review

Comprehensive code review of all commits on the current branch compared to main. Spawns parallel Claude (Opus) reviewer agents as background tasks - each specializing in a different review concern. Each reviewer independently validates their own findings using the Codex MCP, then the lead collects all results and synthesizes the final report.

Unlike the team-based review, this skill uses structural completion guarantees: the lead physically cannot see reviewer output until all agents have finished. This eliminates premature synthesis.

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`
- Base commit: !`git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null || echo "main"`

## Instructions

You are the lead conducting a comprehensive multi-agent code review using parallel background tasks.

**CRITICAL: You MUST use the Task tool with `run_in_background: true` to spawn reviewer agents, then TaskOutput with `block: true` to collect their results. Do NOT use TeamCreate, Teammate, SendMessage, TaskCreate, TaskList, or TaskUpdate. This skill does not use agent teams.**

**This skill does NOT edit files.** It produces a review report only.

You will:
1. Spawn parallel Claude (Opus) reviewer agents as background tasks
2. Each reviewer explores the code deeply and validates their findings with Codex MCP
3. You block on TaskOutput until ALL reviewers complete, then synthesize the final report

---

### Phase 0: Precondition Check

Verify you have the Task tool (the tool that launches new agent sessions with parameters like `subagent_type`, `run_in_background`, `model`, and `prompt`). This is NOT the same as TaskCreate/TaskList/TaskUpdate (those manage a task list and are NOT used in this skill).

If you do not have the Task tool, STOP and tell the user:

"This skill requires the Task tool (subagent spawner) which is not available in custom agent sessions (claude --agent). Run this skill from a plain `claude` session instead."

Do NOT proceed unless confirmed available.

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

### Phase 3: Spawn Reviewer Agents

1. **Load templates and reviewer briefs.** Before spawning, read the following files using the Read tool:

   **Prompt template** (from `~/.claude/skills/parallel-branch-review/templates/`):
   - `templates/reviewer-prompt.md` - The prompt template for reviewer agents

   **Final report template** (for Phase 5, read now):
   - `~/.claude/skills/team-branch-review/templates/final-report.md`

   **Reviewer briefs** (from `~/.claude/skills/team-branch-review/reviewers/`):

   | Reviewer name | Brief file |
   |---|---|
   | `reviewer-security` | `reviewers/security.md` |
   | `reviewer-correctness` | `reviewers/correctness.md` |
   | `reviewer-architecture` | `reviewers/architecture.md` |
   | `reviewer-simplicity` | `reviewers/simplicity.md` |
   | `reviewer-pragmatism` | `reviewers/pragmatism.md` |
   | `reviewer-performance` | `reviewers/performance.md` |
   | `reviewer-testing` | `reviewers/testing.md` |

   Read ALL relevant brief files, the reviewer prompt template, and the final report template.

2. **Spawn ALL reviewers as background tasks in parallel** (send all Task calls in a single message):

   For each reviewer, take the loaded reviewer prompt template and substitute all placeholders, then pass the result as the Task prompt:

   ```
   Task(
     subagent_type: "general-purpose",
     model: "opus",
     run_in_background: true,
     description: "FOCUS_AREA review",
     prompt: "[reviewer-prompt.md with all placeholders substituted]"
   )
   ```

   Repeat for each reviewer. All spawns happen simultaneously in a single message.

3. **Record task IDs.** Each background Task call returns a task_id. Record all task IDs for Phase 4.

### Reviewer Prompt Template

Each reviewer receives the prompt from `~/.claude/skills/parallel-branch-review/templates/reviewer-prompt.md` (loaded above).

Substitute these placeholders before passing to each reviewer:

| Placeholder | Value |
|---|---|
| FOCUS_AREA | The reviewer's specialization name |
| FOCUS_DESCRIPTION | One-line description of their focus |
| FOCUS_BRIEF | Full content from the reviewer's brief file |
| BRANCH_NAME | Current branch name |
| BASE_COMMIT | Exact commit hash from Phase 1 |
| FILE_LIST | All changed file paths, one per line |
| CWD | Working directory |

### Phase 4: Collect All Results

**Call TaskOutput for ALL reviewers in parallel** (send all calls in a single message):

```
TaskOutput(task_id: "reviewer-security-task-id", block: true, timeout: 600000)
TaskOutput(task_id: "reviewer-correctness-task-id", block: true, timeout: 600000)
# ... one for each reviewer
```

This is a structural completion gate. You will not receive any results until ALL reviewers have finished. Do NOT attempt to read output files directly - use TaskOutput.

**If a reviewer times out** (TaskOutput returns a timeout), note the gap in the final report and proceed with the results you have.

Once all TaskOutput calls return, compile all findings into a single list tagged by reviewer.

### Phase 5: Synthesize Final Report

You (the lead) produce the final report directly. Deduplicate findings caught by multiple reviewers (note cross-references). Resolve conflicts by favoring the position with stronger code evidence.

Use the report format from `~/.claude/skills/team-branch-review/templates/final-report.md` (loaded in Phase 3). Substitute BRANCH_NAME and fill in all sections with the compiled findings.

---

## Error Handling

| Scenario | Recovery |
|----------|----------|
| Reviewer agent fails or times out | Note the gap in the report, continue with remaining findings |
| Codex MCP unavailable for a reviewer | Reviewer reports unvalidated findings, noted in report |
| Task tool unavailable | Stop and tell the user to run from a plain `claude` session |
| No findings from any reviewer | Report "APPROVED - no issues found" with note about review coverage |

## Guidelines

- **All Claude agents use Opus**: Set `model: "opus"` for every spawned agent
- **Parallel execution**: Spawn ALL reviewer agents simultaneously in one message
- **Parallel collection**: Call ALL TaskOutput simultaneously in one message
- **Fixed base commit**: Use the exact hash from Phase 1 everywhere, never "main"
- **Distributed Codex validation**: Each reviewer calls Codex MCP to validate their own findings (runs in parallel)
- **Don't embed diffs in prompts**: Let agents and Codex gather diffs via git commands themselves
- **Deep exploration**: Reviewers should use Read, Grep, Glob extensively - not just skim diffs
- **No severity inflation**: Findings should use honest, appropriate severity levels
- **No team tools**: Do NOT use TeamCreate, Teammate, SendMessage, TaskCreate, TaskList, or TaskUpdate

## Chaining to Fix Implementation

After presenting the report, if the outcome is NEEDS REVISION or MANUAL REVIEW REQUIRED, add this note:

> To implement fixes for these findings, run `/team-branch-fix`. It will let you choose which findings to fix, then spawn agents to implement the changes in parallel.

$ARGUMENTS
