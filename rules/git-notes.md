# Git Notes for Agent Context

After every commit, attach a git note with verbose agent-oriented context using `git notes add`.

Commit messages are for humans (explain why). Git notes are for future agents (raw process logs).

## Required Note Sections

```
## Conversation History
How user requests evolved and actual intent behind the work.

## Actions Taken
Files accessed, commands executed, edits made - step by step.

## Errors and Mistakes
Failures, misunderstandings, actual error output. Raw truth - this is the most valuable section.
Write "None" if genuinely inapplicable.

## Dead Ends
Approaches attempted and abandoned, with reasoning for why they did not work.

## Hints and Warnings
Non-obvious constraints, gotchas, fragile areas discovered during the work.

## Codebase Discoveries
Undocumented behaviors, conventions, or dependencies learned.

## Open Questions
Unresolved or deferred decisions.
```

Include specific file paths, line numbers, and error messages. Write assuming the reading agent has zero prior context.

## Information Safety - CRITICAL

Git notes are stored in refs/notes/commits and can be pushed to remotes. In public or externally-visible repositories, notes are fully exposed.

**NEVER include in git notes:**
- Internal Slack messages, channel names, or thread content
- Internal URLs (Confluence, Jira, Grafana, internal dashboards, VPN endpoints)
- Employee names, team names, or org structure details
- API keys, tokens, secrets, credentials, or service account identifiers
- Internal hostnames, IP addresses, or infrastructure details
- Customer names, account identifiers, or customer-specific data
- Internal process names, project codenames, or roadmap references
- Content from internal documents, design docs, or RFCs
- Bead IDs (bd-xxx), bead names, or references to the bead tracking system. Summarize what the work was about instead.

**Instead, write generically:**
- "per internal discussion" not "per #team-channel Slack thread"
- "the monitoring dashboard" not "grafana.internal/d/abc123"
- "a colleague suggested" not "Aaron mentioned in Slack"
- "internal tracking ticket" not "JIRA-1234"
- "tracked work item to fix config loading" not "bd-042: fix config loading"

**When unsure whether something is safe to include**: Stop and ask the user using AskUserQuestion before writing the note. Do not guess - the cost of leaking internal information into a public repo is high. Err on the side of asking.

## Pushing Notes

Do NOT push notes automatically. Notes follow the same rule as all remote operations - require explicit user approval. When the user is ready:

```bash
git push origin refs/notes/commits
```

## Scope

This rule applies to all repositories. The information safety requirements apply universally but are especially critical for public or externally-visible repositories.
