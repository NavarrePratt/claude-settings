---
description: Thorough code review of all commits on current branch vs main using Codex
---

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`
- Base commit: !`git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null || echo "main"`

## Task

Run a full branch review by spawning a subagent. This isolates all Codex round-trips, diff reading, and fix iterations from the main conversation context.

Launch a **foreground** general-purpose subagent with the prompt below. Substitute:
- **CWD** with the working directory
- **BRANCH_NAME** with the current branch
- **BASE_COMMIT** with the base commit
- **USER_INSTRUCTIONS** with $ARGUMENTS below (or "None" if empty)

When the subagent finishes, relay its final summary to the user verbatim.

**Subagent prompt:**

You are conducting a rigorous code review of all changes on branch BRANCH_NAME since BASE_COMMIT, in working directory CWD.

You will iterate with Codex MCP until it approves the changes or you reach 5 review iterations.

### Step 1: Initial Review Request

Call `mcp__codex__codex` with:
- `sandbox`: `read-only`
- `approval-policy`: `never`
- `cwd`: CWD

Prompt for Codex:

```
You are a senior code reviewer conducting a thorough pre-PR review of a feature branch.

Branch: BRANCH_NAME
Base: BASE_COMMIT

## Your Task

You have full git access. Gather the diff yourself to handle branches of any size.

### Step 1: Assess Scope

Run these commands to understand the branch:
- `git log --oneline BASE_COMMIT..HEAD` - see all commits
- `git diff --stat BASE_COMMIT..HEAD` - see files changed and line counts

### Step 2: Gather and Review

**For large branches (>500 lines changed or >10 files):**
Review incrementally:
- Review commit-by-commit: `git show <commit_hash>`
- Or review by file: `git diff BASE_COMMIT..HEAD -- <filepath>`
- Focus on the most critical/complex files first

**For smaller branches:**
- `git diff BASE_COMMIT..HEAD` - review the full diff

### Step 3: Apply Review Criteria

Evaluate against these criteria:

1. **Correctness**: Logic errors, off-by-one bugs, incorrect assumptions
2. **Security**: Injection risks, auth gaps, input validation, data exposure
3. **Performance**: Unnecessary allocations, N+1 queries, resource leaks
4. **Error Handling**: Missing error paths, improper propagation, cleanup on failure
5. **Code Quality**: Naming, abstraction level, duplication, style consistency
6. **Edge Cases**: Nil dereferences, empty collections, concurrency, boundaries
7. **Testing**: Testability, obvious missing test cases
8. **Architecture**: Does the overall approach make sense? Are there better patterns?
9. **Commit Hygiene**: Are commits atomic and well-described? Should any be squashed or split?

### Output Format

For each issue found, provide:
- File: path and line number
- Severity: Critical / High / Medium / Low
- Category: Which criteria it violates
- Issue: Clear description
- Suggestion: Specific fix

End your response with one of:
- "APPROVED:" followed by confirmation if ready to merge
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
4. Instructions to re-run git diff to see the updated state
5. Same review criteria and response format (APPROVED/NEEDS REVISION)

Repeat Steps 2-3 until Codex responds with "APPROVED" or you reach iteration 5.

### Step 4: Final Summary

Produce ONLY this summary (no other output):

```markdown
## Codex Branch Review Summary

### Branch Info
- Branch: [branch name]
- Commits: [count] commits since main
- Files changed: [count]

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

### Commit Recommendations
[Any suggestions for commit reorganization]

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
