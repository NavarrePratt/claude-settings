---
allowed-tools: Bash(git *), Edit, Read, mcp__codex__codex
description: Thorough code review of all commits on current branch vs main using Codex
---

# Codex Branch Review

Thorough code review of all commits on the current branch compared to main using iterative collaboration with Codex. Use this command before creating a PR to get a rigorous review of all branch changes.

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`
- Base commit: !`git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null || echo "main"`

## Instructions

You are conducting a rigorous code review of all changes on this branch using the Codex MCP tool (`mcp__codex__codex`). This review covers the entire branch - all commits since diverging from main. Iterate with Codex until it approves the changes or you reach 5 review iterations.

### Step 1: Initial Review Request

Call `mcp__codex__codex` with these parameters:
- `sandbox`: `read-only`
- `approval-policy`: `never`
- `cwd`: The current working directory (ensures Codex operates in the correct repo)
- `prompt`: The review prompt below (substitute BRANCH_NAME and BASE_COMMIT from Context above)

**Review Prompt for Codex:**

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
Review incrementally to avoid overwhelming context:
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
4. Document any issues you disagree with (you'll include reasoning in re-review)

**Note**: After making fixes, you may need to amend the last commit or create a new fixup commit depending on the nature of the changes.

### Step 3: Re-Review Loop

After making fixes, call `mcp__codex__codex` again with the same parameters and a prompt containing:

1. Statement: "Review iteration [N] - re-reviewing after fixes"
2. List of issues addressed and how each was fixed
3. Any disagreements with reasoning
4. Instructions to re-run git diff to see the updated state
5. Same review criteria and response format (APPROVED/NEEDS REVISION)

**Repeat Steps 2-3 until:**
- Codex responds with "APPROVED", OR
- You reach iteration 5 (stop and report remaining issues)

### Step 4: Final Summary

Produce this summary:

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
[List each code change made to address review feedback, with file:line references and which commit was affected]

### Review Concerns Addressed
[Summarize the main categories of issues Codex identified and how they were resolved]

### Commit Recommendations
[Any suggestions for commit reorganization - squashing, splitting, rewording]

### Remaining Notes
[Any caveats, deferred issues, documented disagreements, or recommendations]
```

## Guidelines

- **Let Codex gather diffs**: Do not embed diffs in the prompt - Codex will run git commands itself
- **Always set cwd**: Pass the working directory to ensure Codex operates in the correct repository
- **Review the whole branch**: Consider how all commits work together, not just individual changes
- **Always re-review after fixes**: Codex should re-run git diff to see updated state
- **Preserve user intent**: Address concerns without changing the fundamental approach
- **Document disagreements**: If you think an issue is a false positive, explain why in the re-review and summary
- **Show your work**: The user should see exactly what changed and why
- **Handle errors**: If the Codex tool fails, report the error and stop
- **Commit hygiene**: If fixes are made, suggest whether to amend or create fixup commits

## Notes

- Each Codex call is a fresh session. Re-review prompts must include context about what changed.
- The `read-only` sandbox allows git commands but prevents file modifications by Codex itself.
- File edits are made by Claude (the outer agent) using the Edit tool, not by Codex.

$ARGUMENTS
