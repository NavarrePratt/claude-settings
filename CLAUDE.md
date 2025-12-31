# Global Instructions

This file documents workflow standards, issue tracking practices, and code quality expectations.

See detailed rules in:
- @rules/issue-tracking.md - bd CLI patterns and issue management
- @rules/session-protocol.md - Session procedures and quality gates

# Quick Reference

## Session Startup (User-Initiated Only)

Run these commands ONLY when the user explicitly requests session initialization (e.g., "start session", "check for work", "what's ready?"). Do NOT run automatically after context compaction - if you were working on something before compaction, continue that work.

```bash
pwd && bd prime && bd ready --json && git log --oneline -5 && git status
```

## Issue Workflow

```bash
bd ready --json                           # Find work
bd update bd-xxx --status in_progress     # Claim it
# ... do work ...
bd close bd-xxx --reason "Completed..."   # Close with reason
```

## Git Commits

Use `/commit` slash command for all commits—creates atomic, well-formatted commits matching project style.

# Issue Tracking Summary

Track all work with `bd`. Create issues for test failures and bugs. Record meticulous notes for history.

**Priority levels**: 0=critical, 1=high, 2=normal, 3=low, 4=backlog

**Creating issues**: Title 50 chars max, imperative voice. Verbose descriptions with relevant files and snippets.

**Closing issues**: Always provide `--reason` with what was done and how verified. Never close if tests fail or implementation is partial.

**Dependencies**: `bd dep add A B --type blocks` means A must complete before B.

# Session Protocol Summary

**Startup (user-initiated only)**: `bd prime` -> `bd ready` -> review git state. Do NOT run after context compaction.

**Work**: One issue at a time. Commit after each. Verify end-to-end.

**Completion**: File remaining work as issues. Close completed issues. Push to remote.

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
