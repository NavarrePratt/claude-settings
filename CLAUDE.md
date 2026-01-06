# Global Instructions

This file documents workflow standards, issue tracking practices, and code quality expectations.

See detailed rules in:
- @rules/issue-tracking.md - bd CLI patterns and issue management
- @rules/session-protocol.md - Session procedures and quality gates

# Quick Reference

## Git Commits

Use `/commit` slash command for all commits—creates atomic, well-formatted commits matching project style.

# Bead Creation Boundary

After creating a bead (via /bd-create skill OR manual `bd create`):
- Report the bead ID
- Return to previous task IMMEDIATELY
- Do NOT start working on the newly created bead
- Do NOT investigate, edit files, or implement anything for it

The bead will be picked up later by atari or worked on in a future session.
Exception: Only continue working if user explicitly says "and work on it now".

# Issue Tracking Summary

Track all work with `bd`. Create issues for test failures and bugs. Record meticulous notes for history.

**Priority levels**: 0=critical, 1=high, 2=normal, 3=low, 4=backlog

**Creating issues**: Title 50 chars max, imperative voice. Verbose descriptions with relevant files and snippets.

**Closing issues**: Always provide `--reason` with what was done and how verified. Never close if tests fail or implementation is partial.

**Dependencies**: `bd dep add A B --type blocks` means A must complete before B.

# Quality Gates

Before committing:
- Code compiles/lints without errors
- All tests pass
- No hardcoded secrets
- Changes are minimal and focused

# Code Style

- Read before modifying
- Match existing patterns
- Minimal changes only
- Delete unused code completely
- No over-engineering
- No emojis. No em dashes - use hyphens or colons instead.

# Communication

- Be explicit and direct
- Provide context (why, not just what)
- Use positive framing
- Be concise

# GitHub Interactions

**NEVER post comments, replies, or any content to GitHub (PRs, issues, discussions) without explicit user approval.**

Before submitting anything to the GitHub API:
1. Show the user exactly what will be posted
2. Wait for explicit approval (e.g., "yes", "go ahead", "post it")
3. Only then execute the API call

This applies to: PR comments, PR review replies, issue comments, and any other GitHub write operations.

## Comment Formatting

- Always prefix comments with `[via Claude]` to indicate they were written by Claude
- When replying to an existing comment, post as a reply (not a new comment in the main thread)

## Replying to PR Review Comments

To reply to a PR review comment, use the `/replies` endpoint with the comment ID:

```bash
# CORRECT - posts as a reply to an existing comment
gh api repos/OWNER/REPO/pulls/PR_NUMBER/comments/COMMENT_ID/replies \
  -X POST -f body="[via Claude] Your reply here"

# WRONG - posts as a new comment in the main thread
gh api repos/OWNER/REPO/issues/PR_NUMBER/comments \
  -X POST -f body="[via Claude] Your reply here"
```

To find comment IDs, fetch PR comments first:
```bash
gh api repos/OWNER/REPO/pulls/PR_NUMBER/comments --jq '.[] | {id, user: .user.login, body: .body[:80]}'
```

# MCP Tools

## Codex MCP

When using the `mcp__codex__codex` tool for code reviews or other tasks:
- Do NOT manually specify the `model` parameter unless the user explicitly requests a specific model
- Let the global Codex configuration handle the default model selection
- Only override the model when the command or user explicitly calls for it

# Principals

- Assumptions are the enemy. Never guess numerical values - benchmark instead of estimating. When uncertain, measure.
  Say "this needs to be measured" rather than inventing statistics.
- **Interaction**: Clarify unclear requests, then proceed autonomously. Only ask for help when scripts timeout (>2min) or genuine blockers arise.
- **Ground truth clarification**: For non-trivial tasks, reach ground truth understanding before coding. Simple tasks execute immediately.
  Complex tasks (refactors, new features, ambiguous requirements) require clarification first: research codebase, ask targeted questions,
  confirm understanding, persist the plan, then execute autonomously. 
- **First principals reimplementation**: Building from scratch can beat adapting legacy code when implementations are in wrong languages,
  carry historical baggage, or need architectural rewrites. Understand domain at spec level, choose optimal stack,
  implement incrementally with human verification.
- **Constraint persistence**: When user defines constraints ("never X", "always Y", "from now on"), immediately persist to projects local
  CLAUDE.md. Acknowledge, write, confirm.


<atari-managed>
# BD Integration

Use the bd CLI to track work across sessions.

See detailed rules in:
- @rules/issue-tracking.md - bd CLI patterns and issue management
- @rules/session-protocol.md - Session procedures and quality gates

## Quick Reference

### Session Startup (User-Initiated Only)

Run these commands ONLY when the user explicitly requests session initialization (e.g., "start session", "check for work", "what's ready?"). Do NOT run automatically after context compaction - if you were working on something before compaction, continue that work.

```bash
pwd && bd prime && bd ready --json && git log --oneline -5 && git status
```

### Issue Workflow

```bash
bd ready --json                           # Find work
bd update bd-xxx --status in_progress     # Claim it
# ... do work ...
bd close bd-xxx --reason "Completed..."   # Close with reason
```

### Git Commits

Use `/commit` slash command for all commits - creates atomic, well-formatted commits matching project style.

## Issue Tracking Summary

Track all work with `bd`. Create issues for test failures and bugs. Record meticulous notes for history.

**Priority levels**: 0=critical, 1=high, 2=normal, 3=low, 4=backlog

**Creating issues**: Title 50 chars max, imperative voice. Verbose descriptions with relevant files and snippets.

**Closing issues**: Always provide `--reason` with what was done and how verified. Never close if tests fail or implementation is partial.

**Dependencies**: `bd dep add A B --type blocks` means A must complete before B.

## Session Protocol Summary

**Startup (user-initiated only)**: `bd prime` -> `bd ready` -> review git state. Do NOT run after context compaction.

**Work**: One issue at a time. Commit after each. Verify end-to-end.

**Completion**: File remaining work as issues. Close completed issues. Push to remote.

## Quality Gates

Before committing:
- Code compiles/lints without errors
- All tests pass
- No hardcoded secrets
- Changes are minimal and focused
</atari-managed>
