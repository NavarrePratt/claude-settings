---
allowed-tools: Bash(git *), Edit, Read, mcp__codex__codex
description: Thorough code review of active git changes using iterative Codex collaboration
---

# Codex Diff Review

Thorough code review of active git changes using iterative collaboration with Codex. Iterate with Codex until it explicitly approves the changes or you reach 5 review iterations.

## Context

- Current branch: !`git branch --show-current`
- Base comparison: !`git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null || echo "HEAD~1"`
- Git status: !`git status --short`
- Staged changes: !`git diff --cached --stat`
- Unstaged changes: !`git diff --stat`

## Current Diff

```
!`git diff HEAD`
```

## Instructions

You are conducting a rigorous code review using the Codex MCP tool (`mcp__codex__codex`). Iterate with Codex until it approves the changes or you reach 5 review iterations.

### Step 1: Initial Review Request

Call `mcp__codex__codex` with a prompt containing:

1. The review criteria below
2. The full diff from the Context section above
3. Clear response format instructions

**Review Criteria to Include:**

```
You are a senior code reviewer conducting a thorough review. Analyze the following git diff with extreme rigor.

Evaluate each change against these criteria:

1. **Correctness**: Logic errors, off-by-one bugs, incorrect assumptions
2. **Security**: Injection risks, auth gaps, input validation, data exposure
3. **Performance**: Unnecessary allocations, N+1 queries, resource leaks
4. **Error Handling**: Missing error paths, improper propagation, cleanup on failure
5. **Code Quality**: Naming, abstraction level, duplication, style consistency
6. **Edge Cases**: Nil dereferences, empty collections, concurrency, boundaries
7. **Testing**: Testability, obvious missing test cases

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
4. Document any issues you disagree with (you'll include reasoning in re-review)

### Step 3: Re-Review Loop

After making fixes, call `mcp__codex__codex` again with:

1. Statement: "Review iteration [N]"
2. List of issues addressed and how each was fixed
3. Any disagreements with reasoning
4. The updated diff: !`git diff HEAD`
5. Same response format instructions (APPROVED/NEEDS REVISION)

**Repeat Steps 2-3 until:**
- Codex responds with "APPROVED", OR
- You reach iteration 5 (stop and report remaining issues)

### Step 4: Final Summary

Produce this summary:

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
[Any caveats, deferred issues, documented disagreements, or recommendations]
```

## Guidelines

- **Always re-review after fixes**: Include all changes in the re-review request
- **Preserve user intent**: Address concerns without changing the fundamental approach
- **Document disagreements**: If you think an issue is a false positive, explain why in the re-review and summary
- **Show your work**: The user should see exactly what changed and why
- **Handle errors**: If the Codex tool fails, report the error and stop

$ARGUMENTS
