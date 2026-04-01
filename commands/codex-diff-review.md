---
description: Thorough code review of active git changes using iterative Codex collaboration
---

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`
- Git status: !`git status --short`

## Task

Run a diff review by spawning a subagent. This isolates all Codex round-trips, diff reading, and fix iterations from the main conversation context.

Launch a **foreground** general-purpose subagent with the prompt below. Substitute:
- **CWD** with the working directory
- **USER_INSTRUCTIONS** with $ARGUMENTS below (or "None" if empty)

When the subagent finishes, relay its final summary to the user verbatim.

**Subagent prompt:**

You are conducting a rigorous code review of uncommitted changes in CWD.

You will iterate with Codex MCP until it approves the changes or you reach 5 review iterations.

### Step 1: Initial Review Request

Call `mcp__codex__codex` with:
- `sandbox`: `read-only`
- `approval-policy`: `never`
- `cwd`: CWD

Prompt for Codex:

```
You are a senior code reviewer conducting a thorough review of uncommitted git changes.

## Your Task

You have full git access. Gather the diff yourself to handle changes of any size.

### Step 1: Assess Scope

Run these commands to understand the changes:
- `git status --short` - see what files are modified
- `git diff --stat` - see line counts for unstaged changes
- `git diff --cached --stat` - see line counts for staged changes

### Step 2: Gather and Review

**For large diffs (>500 lines changed or >10 files):**
Review incrementally:
- Review by file: `git diff HEAD -- <filepath>`
- Focus on the most critical/complex files first

**For smaller diffs:**
- `git diff HEAD` - review all uncommitted changes at once

### Step 3: Apply Review Criteria

Evaluate each change against these criteria:

1. **Correctness**: Logic errors, off-by-one bugs, incorrect assumptions
2. **Security**: Injection risks, auth gaps, input validation, data exposure
3. **Performance**: Unnecessary allocations, N+1 queries, resource leaks
4. **Error Handling**: Missing error paths, improper propagation, cleanup on failure
5. **Code Quality**: Naming, abstraction level, duplication, style consistency
6. **Edge Cases**: Nil dereferences, empty collections, concurrency, boundaries
7. **Testing**: Testability, obvious missing test cases

### Output Format

For each issue found, provide:
- File: path and line number
- Severity: Critical / High / Medium / Low
- Category: Which criteria it violates
- Issue: Clear description
- Suggestion: Specific fix

End your response with one of:
- "APPROVED:" followed by confirmation if ready to commit
- "NEEDS REVISION:" followed by issue count if changes needed
```

### Step 2: Evaluate Response and Fix Issues

If Codex responds with "APPROVED": skip to Step 4.

If Codex responds with "NEEDS REVISION":
1. For each issue, evaluate if it's valid
2. Fix valid Critical/High issues immediately using the Edit tool
3. Fix valid Medium/Low issues if the fix is straightforward
4. Document any issues you disagree with (include reasoning in re-review)

### Step 3: Re-Review Loop

After making fixes, call `mcp__codex__codex` again with the same parameters and a prompt containing:

1. Statement: "Review iteration [N] - re-reviewing after fixes"
2. List of issues addressed and how each was fixed
3. Any disagreements with reasoning
4. Instructions to re-run `git diff HEAD` to see the updated state
5. Same review criteria and response format (APPROVED/NEEDS REVISION)

Repeat Steps 2-3 until Codex responds with "APPROVED" or you reach iteration 5.

### Step 4: Final Summary

Produce ONLY this summary (no other output):

```markdown
## Codex Diff Review Summary

### Outcome
[APPROVED / APPROVED WITH CAVEATS / MANUAL REVIEW REQUIRED]

### Review Statistics
- Iterations: [N]
- Issues found: [total]
- Issues fixed: [count]
- Disagreements documented: [count]

### Changes Made During Review
[List each code change made to address review feedback, with file:line references]

### Review Concerns Addressed
[Summarize the main categories of issues Codex identified and how they were resolved]

### Remaining Notes
[Any caveats, deferred issues, or recommendations]
```

### Guidelines

- Let Codex gather diffs via git commands - do not embed diffs in prompts
- Always re-review after fixes so Codex sees the updated state
- Preserve user intent - address concerns without changing the fundamental approach
- Document disagreements with reasoning
- Each Codex call is a fresh session - include context about what changed

User instructions: USER_INSTRUCTIONS

$ARGUMENTS
