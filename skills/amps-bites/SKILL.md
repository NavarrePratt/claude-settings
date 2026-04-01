---
name: amps-bites
description: Generate weekly AMPS Bites updates for the Applied Training team by pulling GitHub and Slack activity. Use when the user mentions AMPS bites, weekly update, team status update, weekly summary, or wants to generate an AMPS bites draft. Also trigger when the user says things like "what did we do this week" or "summarize the team's work".
---

# AMPS Bites Generator

Generate a weekly status update by scanning GitHub PRs and Slack channels, then synthesizing into concise bullets.

This skill runs as a subagent to keep the raw API responses (dozens of PRs, Slack messages) out of the main conversation context.

## Instructions

### Step 1: Spawn Collection Subagent

Read `references/instructions.md` (relative to this skill's directory) with the Read tool to get the full procedure.

Then launch a **foreground** general-purpose subagent with a prompt that includes:
1. The full content from `references/instructions.md`
2. The working directory (for any repo-local context)
3. Any user arguments from $ARGUMENTS below

The subagent handles all GitHub API calls, Slack MCP scanning, synthesis, and draft formatting.

### Step 2: Present Results

When the subagent returns, present its output (draft + activity reference) to the user. Ask if they want to adjust anything before posting.

$ARGUMENTS
