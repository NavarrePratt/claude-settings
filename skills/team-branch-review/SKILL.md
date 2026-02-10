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
- **team_name**: Compute a unique team name: take the branch name, replace `/` with `-`, truncate to 30 chars, then prefix with `review-` and append `-` plus the first 6 chars of HEAD's commit hash. Example: branch `feat/add-auth` at commit `a1b2c3d` becomes `review-feat-add-auth-a1b2c3`. Use this value everywhere a team_name is needed.

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
   Teammate(operation: "spawnTeam", team_name: "TEAM_NAME", description: "Branch review of BRANCH_NAME")
   ```

2. **Create temp directory** for reviewer findings:
   ```bash
   mkdir -p /tmp/review-TEAM_NAME
   ```

3. **Create tasks** for each reviewer using TaskCreate. Each task should include:
   - Subject: "Review FOCUS_AREA for BRANCH_NAME"
   - Description: The reviewer's focus area, base commit hash, and list of changed files
   - activeForm: "Reviewing FOCUS_AREA"

4. **Load templates and reviewer briefs.** Before spawning, read the following files using the Read tool:

   **Prompt templates** (from `~/.claude/skills/team-branch-review/templates/`):
   - `templates/reviewer-prompt.md` - The prompt template for reviewer agents (used in step 4 below)
   - `templates/final-report.md` - The report format for Phase 5 synthesis (read now, use later)

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

   Read ALL relevant brief files and both template files.

5. **Spawn ALL reviewers in parallel** (send all Task calls in a single message):

   For each reviewer, take the loaded reviewer prompt template and substitute all placeholders, then pass the result as the Task prompt:

   ```
   Task(
     subagent_type: "general-purpose",
     team_name: "TEAM_NAME",
     name: "reviewer-security",
     model: "opus",
     description: "Security review",
     prompt: "[reviewer-prompt.md with all placeholders substituted]"
   )
   ```

   Repeat for each reviewer. All spawns happen simultaneously.

### Reviewer Prompt Template

Each reviewer receives the prompt from `~/.claude/skills/team-branch-review/templates/reviewer-prompt.md` (loaded in Phase 3, step 4).

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
| TEAM_NAME | The team name computed in Phase 1 (used for findings file path) |
| REVIEWER_NAME | The reviewer's name, e.g. `reviewer-security` (used for findings file path) |
| TEAM_ROSTER | List of all OTHER reviewers (omit current). Format each as: `- reviewer-name: Focus description` |

### Phase 4: Wait for All Tasks to Complete

**CRITICAL: You MUST NOT call TaskUpdate to change the status of any reviewer task.** Only the reviewer agent that owns a task may call TaskUpdate on it. If you believe a reviewer is stuck, send them a message - do not mark their task completed yourself.

Execute this loop. Do not deviate from it.

```
loop:
    result = TaskList()
    if EVERY task in result has status "completed":
        break -> proceed to reading findings files, then Phase 5
    else:
        call TaskList() again (continue loop)
```

**If ANY task status is not `completed`, your ONLY permitted next action is to call TaskList again.**

You MUST NOT do any of the following while any task is not `completed`:
- Write, draft, or begin synthesizing the report
- Send shutdown requests to any reviewer
- Proceed to Phase 5 or Phase 6
- Present findings to the user
- Call TaskUpdate on any reviewer task

**Messages from reviewers are NOT completion signals.** Reviewers write their findings to files and then call TaskUpdate to mark their task `completed`. Any messages you receive from reviewers (checkpoint notifications, cross-domain tips) are progress signals only. The task status `completed` is the only exit condition for this loop.

**Understanding checkpoint messages:** Reviewers send a "CHECKPOINT: Entering Codex validation..." message before entering the long-running Codex MCP call. Use these to track reviewer progress:
- Reviewer has NOT sent a checkpoint message: still doing primary review (fast phase)
- Reviewer HAS sent a checkpoint message but task not yet completed: inside Codex validation (slow phase, be patient)
- Task shows `completed`: reviewer is done, findings file is ready

**Timeout handling:** If a specific task has been `in_progress` for 20+ consecutive polls with no change AND the reviewer has not yet sent a checkpoint message, send ONE follow-up message to that reviewer, then continue polling. If the reviewer HAS sent a checkpoint message (meaning they are in the Codex validation phase), be more patient - allow 40+ polls before sending a follow-up. After 3 follow-up messages to the same reviewer with no status change, declare that reviewer failed and proceed without their findings (note the gap in the report).

**Collecting findings:** Once ALL tasks show status `completed`, read each reviewer's findings file:

```
For each reviewer:
    Read /tmp/review-TEAM_NAME/{reviewer-name}.md
```

Compile all findings into a single list tagged by reviewer.

### Phase 5: Synthesize Final Report

You (the lead) produce the final report directly using the findings read from `/tmp/review-TEAM_NAME/` files in Phase 4. Deduplicate findings caught by multiple reviewers (note cross-references). Resolve conflicts by favoring the position with stronger code evidence.

Use the report format from `~/.claude/skills/team-branch-review/templates/final-report.md` (loaded in Phase 3, step 4). Substitute BRANCH_NAME and fill in all sections with the compiled findings.

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

**Step 4: Clean up temp directory:**
```bash
rm -rf /tmp/review-TEAM_NAME
```

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
- **Never mark reviewer tasks**: You MUST NOT call TaskUpdate on any reviewer task. Only reviewers mark their own tasks completed.
- **Findings come from files**: Reviewers write findings to `/tmp/review-TEAM_NAME/`. Do not use message content as findings.

## Chaining to Fix Implementation

After presenting the report, if the outcome is NEEDS REVISION or MANUAL REVIEW REQUIRED, add this note:

> To implement fixes for these findings, run `/team-branch-fix`. It will let you choose which findings to fix, then spawn agents to implement the changes in parallel.

$ARGUMENTS
