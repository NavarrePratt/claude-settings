---
allowed-tools: Bash(git *), Edit, Read, mcp__codex__codex, AskUserQuestion
description: Interactive code review of uncommitted changes - confirms fixes with user before applying
---

# Codex Diff Review (User Interactive)

Thorough code review of active git changes (staged and unstaged) using Codex, with user confirmation before making any changes. Unlike the non-interactive version, this skill asks the user to approve each fix before applying it.

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`
- Git status: !`git status --short`

## Instructions

You are conducting a rigorous code review of uncommitted changes using the Codex MCP tool (`mcp__codex__codex`), but with user approval for all changes. This ensures the user maintains control over what gets modified.

### Step 1: Initial Review Request

Call `mcp__codex__codex` with these parameters:
- `sandbox`: `read-only`
- `approval-policy`: `never`
- `cwd`: The current working directory (from Context above)
- `prompt`: The review prompt below

**Review Prompt for Codex:**

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
Review incrementally to avoid overwhelming context:
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

### Step 2: Present Issues to User

If Codex responds with "APPROVED": skip to Step 5.

If Codex responds with "NEEDS REVISION", present the issues to the user using AskUserQuestion. Group issues by severity and let the user decide what to do.

**For each Critical/High issue, ask:**

```yaml
questions:
  - question: "[Issue description from Codex] - How should we handle this?"
    header: "[Severity]"
    options:
      - label: "Fix it"
        description: "Apply the suggested fix: [brief fix description]"
      - label: "Disagree"
        description: "I disagree with this finding - document why and skip"
      - label: "Defer"
        description: "Valid issue but fix later - note it in summary"
    multiSelect: false
```

**For Medium/Low issues, batch them:**

```yaml
questions:
  - question: "Codex found [N] medium/low severity issues. Which should we address now?"
    header: "Minor Issues"
    options:
      - label: "Fix all"
        description: "Apply all suggested fixes for medium/low issues"
      - label: "Review individually"
        description: "Let me decide on each issue separately"
      - label: "Skip all"
        description: "Note them in summary but don't fix now"
    multiSelect: false
```

### Step 3: Apply Approved Fixes

For each issue the user approved:
1. Use the Edit tool to apply the fix
2. Document what was changed

For disagreements, document the user's reasoning for the re-review.

### Step 4: Re-Review Loop

After making fixes, call `mcp__codex__codex` again with the same parameters and a prompt containing:

1. Statement: "Review iteration [N] - re-reviewing after fixes"
2. List of issues addressed and how each was fixed
3. Documented disagreements with user's reasoning
4. Instructions to re-run `git diff HEAD` to see the updated state
5. Same review criteria and response format (APPROVED/NEEDS REVISION)

**Repeat Steps 2-4 until:**
- Codex responds with "APPROVED", OR
- You reach iteration 5 (stop and report remaining issues)

### Step 5: Final Summary

Produce this summary:

```markdown
## Codex Diff Review Summary (User Interactive)

### Outcome
[APPROVED / APPROVED WITH CAVEATS / MANUAL REVIEW REQUIRED]

### Review Statistics
- Iterations: [N]
- Issues found: [total]
- Issues fixed (user approved): [count]
- Issues disagreed (user decision): [count]
- Issues deferred: [count]

### Changes Made During Review
[List each code change made, noting user approval]

### User Decisions
[Document each disagreement or deferral with user's reasoning]

### Review Concerns Addressed
[Summarize the main categories of issues Codex identified and how they were resolved]

### Remaining Notes
[Any caveats, deferred issues, or recommendations]
```

## Guidelines

- **Always get user approval**: Never make changes without explicit user consent
- **Present clear options**: Use AskUserQuestion with specific, actionable choices
- **Respect user decisions**: If user disagrees with Codex, document their reasoning
- **Let Codex gather diffs**: Do not embed diffs in the prompt - Codex will run git commands itself
- **Always set cwd**: Pass the working directory to ensure Codex operates in the correct repository
- **Show your work**: The user should see exactly what Codex found and what was changed
- **Handle errors**: If the Codex tool fails, report the error and stop

## Notes

- This is the interactive version - use `/codex-diff-review` for autonomous fixing
- Each Codex call is a fresh session. Re-review prompts must include context about what changed.
- The `read-only` sandbox allows git commands but prevents file modifications by Codex itself.
- File edits are made by Claude (the outer agent) using the Edit tool, not by Codex.

$ARGUMENTS
