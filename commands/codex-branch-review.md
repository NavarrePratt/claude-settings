---
allowed-tools: Bash(git *), Edit, Read, mcp__codex__codex
description: Thorough code review of all commits on current branch vs main using Codex
---

# Codex Branch Review

Thorough code review of all commits on the current branch compared to main using iterative collaboration with Codex. Use this command before creating a PR to get a rigorous review of all branch changes.

## Context

- Current branch: !`git branch --show-current`
- Base branch: !`git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null || echo "main"`
- Commits on branch: !`git log --oneline $(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null)..HEAD`
- Files changed: !`git diff --stat $(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null)..HEAD`

## Branch Diff

```
!`git diff $(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null)..HEAD`
```

## Commit Messages

```
!`git log --format="commit %h%nAuthor: %an%nDate: %ad%n%n%s%n%b%n---" $(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null)..HEAD`
```

## Instructions

You are conducting a rigorous code review of all changes on this branch using the Codex MCP tool (`mcp__codex__codex`). This review covers the entire branch - all commits since diverging from main. Iterate with Codex until it approves the changes or you reach 5 review iterations.

### Step 1: Initial Review Request

Call `mcp__codex__codex` with a prompt containing:

1. The review criteria below
2. The full branch diff from the Context section above
3. The commit messages for context on intent
4. Clear response format instructions

**Review Criteria to Include:**

```
You are a senior code reviewer conducting a thorough pre-PR review. Analyze the following branch diff with extreme rigor. This represents all changes on a feature branch that will be merged to main.

Consider both individual changes AND the branch as a whole:

1. **Correctness**: Logic errors, off-by-one bugs, incorrect assumptions
2. **Security**: Injection risks, auth gaps, input validation, data exposure
3. **Performance**: Unnecessary allocations, N+1 queries, resource leaks
4. **Error Handling**: Missing error paths, improper propagation, cleanup on failure
5. **Code Quality**: Naming, abstraction level, duplication, style consistency
6. **Edge Cases**: Nil dereferences, empty collections, concurrency, boundaries
7. **Testing**: Testability, obvious missing test cases
8. **Architecture**: Does the overall approach make sense? Are there better patterns?
9. **Commit Hygiene**: Are commits atomic and well-described? Should any be squashed or split?

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

After making fixes, call `mcp__codex__codex` again with:

1. Statement: "Review iteration [N]"
2. List of issues addressed and how each was fixed
3. Any disagreements with reasoning
4. The updated branch diff: !`git diff $(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main 2>/dev/null)..HEAD`
5. Same response format instructions (APPROVED/NEEDS REVISION)

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

- **Review the whole branch**: Consider how all commits work together, not just individual changes
- **Always re-review after fixes**: Include all changes in the re-review request
- **Preserve user intent**: Address concerns without changing the fundamental approach
- **Document disagreements**: If you think an issue is a false positive, explain why in the re-review and summary
- **Show your work**: The user should see exactly what changed and why
- **Handle errors**: If the Codex tool fails, report the error and stop
- **Commit hygiene**: If fixes are made, suggest whether to amend or create fixup commits

$ARGUMENTS
