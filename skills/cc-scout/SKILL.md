---
name: cc-scout
description: Scan Slack, Twitter/X, and Claude Code changelogs for new skills, features, patterns, and config ideas - then compare against current setup and suggest changes. Also detects staleness in existing rules and skills. Use when the user says things like "what's new in claude code", "scan for CC updates", "check for config improvements", "any new skills this week", "is my setup stale", or "cc scout". Run this periodically (weekly or biweekly) to stay current.
---

# CC Scout

Scan external sources for Claude Code news, community skills, usage patterns, and changelog updates. Compare findings against the current setup and produce actionable suggestions - both new things to adopt and existing config that may have gone stale.

## Arguments

Accepts optional arguments. Combine as needed.

| Argument | Effect | Example |
|----------|--------|---------|
| `Nd` | Set scan window (default: 7 days) | `/cc-scout 14d` |
| `--scheduled` | Non-interactive mode: skip prompts, post report to Slack self-DM | `/cc-scout --scheduled` |
| `--load` | Load a previous report from Slack self-DM and enter interactive mode | `/cc-scout --load` |

Examples:
- `/cc-scout` - interactive scan, last 7 days
- `/cc-scout 14d` - interactive scan, last 14 days
- `/cc-scout --scheduled` - autonomous scan, posts report to self-DM
- `/cc-scout --scheduled 14d` - autonomous scan, 14-day window
- `/cc-scout --load` - load most recent report from self-DM, discuss interactively
- `/cc-scout --load last monday` - load the report from last Monday
- `/cc-scout --load Mar 24` - load the report closest to March 24

## Execution Modes

### Interactive (default)

Present the report in conversation. Ask the user which suggestions to implement. Wait for decisions before making changes.

### Scheduled (`--scheduled`)

Running autonomously via a remote trigger with no user present. In this mode:

1. Run all phases (scan, staleness detection, synthesis) as normal
2. Do NOT prompt the user or use AskUserQuestion - there is nobody to answer
3. Write the full report as a Slack message to the self-DM (`@npratt`)
4. If the report is too long for a single message, split into multiple messages in sequence
5. End with a one-line summary of how many new discoveries and staleness alerts were found

The Slack self-DM serves as an inbox the user checks at their convenience.

### Load (`--load`)

Load a previously posted report from the Slack self-DM and enter interactive mode. Skips all scanning phases - just reads and presents the existing report.

**Finding the report:**

1. Search the self-DM for reports: use `mcp__slack__conversations_search_messages` with `filter_in_im_or_mpim: "@npratt"` and `search_query: "CC Scout Report"`
2. The user may provide a hint after `--load` in plain text (a date, a day of the week, "latest", or anything else). Interpret it naturally to find the right report. No hint means most recent.
3. Fetch the full thread using `mcp__slack__conversations_replies` with the matched message's `thread_ts` on channel `@npratt`

**After loading:**

1. Combine all thread replies into a single report
2. Present it in conversation (same as interactive mode output)
3. Ask the user which suggestions they want to implement
4. Proceed with implementation as directed

## Phase 1: Parallel Scan

Launch these four scans as parallel subagents. Each returns a structured findings report.

### 1A: Slack Scanner

Scan Slack channels for Claude Code discussion - custom skills shared, workflow patterns, configuration tips, permission setups, MCP integrations, and feature announcements.

**Default channels**: #claude-coders

The user can specify additional channels per-invocation. If they do, scan those too.

**How to scan**:
1. Use `mcp__slack__channels_list` to resolve channel IDs
2. Use `mcp__slack__conversations_history` with the time window
3. For threads with replies (especially high-reaction messages), fetch replies with `mcp__slack__conversations_replies`
4. Focus on substantive messages - skip casual chatter, short replies, and emoji-only responses

**Extract**:
- Custom skills or plugins shared (with links/code if available)
- CLAUDE.md patterns or templates
- Permission configurations (settings.json patterns)
- MCP server setups
- Workflow automation (hooks, shell functions, integrations)
- Feature announcements or org-level changes
- Tips that changed how someone works

**Upweighted contributors** - give these people's messages extra prominence in the report. They are known to share high-signal CC content:
- Aaron Batilo (abatilo)

Tag upweighted content with `[UPWEIGHTED]` in the findings.

### 1B: Twitter/X Scanner

Use WebSearch to find recent posts from Claude Code team members and related accounts. Search for each account individually.

**Accounts to scan**:
- Boris Cherny (@bcherny) - Claude Code team
- Thariq (@trq212) - Claude Code team
- Claude AI (@claudeai) - official account
- Anthropic (@AnthropicAI) - company account

**Search queries** (run in parallel):
- `from:bcherny claude code OR "claude code" OR skill OR hook` (+ time-scoped)
- `from:trq212 claude code OR "claude code" OR skill` (+ time-scoped)
- `from:claudeai code OR developer OR CLI OR update` (+ time-scoped)
- `from:AnthropicAI claude code OR developer` (+ time-scoped)

If WebSearch doesn't support `from:` filtering well, fall back to broader searches like `"boris cherny" claude code` or `bcherny claude code`.

**Extract**:
- New feature announcements
- Tips or patterns shared
- Links to blog posts, docs, or demos
- Upcoming changes or deprecations

### 1C: Changelog Scanner

Fetch the Claude Code changelog and extract entries from the scan window.

**Source**: https://code.claude.com/docs/en/changelog

Use WebFetch to retrieve the page. Extract entries dated within the time window. For each entry, capture:
- Date
- Feature/change name
- Summary of what changed
- Whether it affects existing tool names, APIs, or behaviors

### 1D: Setup Inventory

Inventory the current Claude Code configuration. This runs in parallel with the external scans.

**Scan these locations**:
- `~/.claude/skills/*/SKILL.md` - all skills (name + description)
- `~/.claude/rules/*.md` - all rules (filename + first heading)
- `~/.claude/commands/*.md` - all commands
- `~/.claude/agents/*.md` - all agents
- `~/.claude/settings.json` - current settings
- `~/.claude/CLAUDE.md` - main instructions
- MCP server configuration

Produce a flat inventory list with file paths and one-line descriptions.

## Phase 2: Staleness Detection

After the changelog scan completes, cross-reference changelog entries against existing rules and skills to detect staleness.

**What to check**:
- Tool names referenced in rules/skills that were renamed or deprecated in the changelog
- Behaviors that rules work around that the harness now handles natively (the rule is no longer needed)
- Settings or configuration patterns that have new/better alternatives
- API changes that affect MCP server configurations
- Skills that reference features or patterns that have been superseded

For each staleness finding, note:
- Which rule/skill file is affected
- What specific content may be stale
- What changelog entry triggered the finding
- Suggested action (update, remove, or verify)

This is one of the most valuable parts of the scan. A rule that references a deprecated tool name or works around a bug that was fixed is actively harmful - it gives the model wrong instructions.

## Phase 3: Synthesize and Compare

After all scans complete, consolidate findings into a single report.

### Report Structure

```markdown
## New Discoveries

### Skills and Plugins
[Skills shared by community members, with attribution and links]

### Patterns and Workflows
[Configuration patterns, shell tricks, workflow automations]

### Feature Announcements
[New CC features from changelog + Twitter/X]

### Notable Community Discussion
[High-signal threads, especially from upweighted contributors]

## Staleness Alerts

[Rules or skills that may need updating based on changelog changes]
[Ordered by severity: broken references first, then stale workarounds, then minor drift]

## Suggested Changes

1. [Effort] *Suggestion title* (Source) - What to change and why
2. [Effort] *Next suggestion* (Source) - Explanation
...
```

### Ranking Suggestions

Order suggestions by value-to-effort ratio:
1. **Staleness fixes** first - these are actively wrong and should be fixed
2. **Low-effort, high-value** additions (new rules, settings changes)
3. **Medium-effort** additions (new skills, workflow changes)
4. **Nice-to-have** items (things that are interesting but not urgent)

### Comparison Logic

For each external finding, check:
- Does the user already have something equivalent? If yes, skip or note as validation
- Does it conflict with an existing rule or pattern? If yes, flag the tension
- Is the effort worth the payoff given the user's setup complexity? Be honest - not every shiny thing needs adoption

## Phase 4: Deliver

### Interactive mode

Show the report in conversation. After presenting:
- Ask if any suggestions stand out as worth implementing now
- Offer to implement specific changes if the user picks them
- Do NOT start implementing changes without the user choosing which ones to pursue

### Scheduled mode (`--scheduled`)

Post the report to the Slack self-DM (`@npratt`) using `mcp__slack__conversations_add_message` as a thread:

1. **Parent message**: One-line header only: `CC Scout Report: {date range} - {N} discoveries, {M} staleness alerts, {K} suggestions`
2. **Thread reply**: Skills and Plugins
3. **Thread reply**: Patterns and Workflows
4. **Thread reply**: Feature Announcements
5. **Thread reply**: Staleness Alerts
6. **Thread reply**: Suggested Changes

To post thread replies, capture the `thread_ts` from the parent message response and pass it to subsequent `conversations_add_message` calls.

Slack mrkdwn formatting rules - these are critical for readability:
- Do not use markdown tables. Slack does not render them. Use numbered lists instead.
- Each bullet point must be on its own line. Use `\n` between items.
- Keep each thread reply focused on one section. Short messages render better than long ones.
- Use `*bold*` for section headers and item titles (Slack mrkdwn bold, not markdown `**bold**`).
- Use `\n\n` between the section header and the first item.

If any single reply exceeds ~3000 chars, split it into additional thread replies. Prefer more shorter messages over fewer long ones.
