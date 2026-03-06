---
name: amps-bites
description: Generate weekly AMPS Bites updates for the Applied Training team by pulling GitHub and Slack activity. Use when the user mentions AMPS bites, weekly update, team status update, weekly summary, or wants to post to the eng-amps-ext-staff thread. Also trigger when the user says things like "what did we do this week" or "summarize the team's work".
---

# AMPS Bites Generator

Generate a weekly status update for the Applied Training (AT) team by collecting GitHub and Slack activity, synthesizing it into concise bullets, and posting to the weekly AMPS Bites thread.

## Team Members

| Name | GitHub | Slack display name |
|------|--------|--------------------|
| Aaron Batilo | abatilo | abatilo |
| Navarre Pratt | NavarrePratt | npratt |

## Tracked Projects

Always check for updates on these projects. If there's meaningful progress, include a bullet. If nothing happened that week, skip it - don't force a mention.

| Project | What to look for |
|---------|-----------------|
| Aviato / CoreWeave Sandbox | Customer-facing name is "CoreWeave Sandbox". Repos: aviato, aviato-deploy, cwsandbox-client, kabinet-charts. Slack: #aviato is the best channel for project-level updates. Look for: feature work, customer onboarding, private preview milestones, docs updates |

## Output Format

The update is a short reply in the weekly AMPS Bites thread in #eng-amps-ext-staff. Format:

```
AT:
 Bullet one - concise summary of meaningful work
 Bullet two - another item
 Bullet three - etc
```

The audience is senior staff across AMPS - they want to know what moved the needle this week, not a changelog. Write like you're telling a coworker what happened over coffee. 3-5 bullets max.

Guidelines:
- Project-level milestones, not technical stats (no "merged 28 PRs")
- Plain language, no jargon-heavy PR titles
- Don't list everything - pick what matters
- Casual and direct, not promotional. This is not a promo packet.
- If a tracked project had progress, lead with it

## Procedure

### Step 1: Collect GitHub Activity

Use the `gh` CLI to pull PRs from the past 7 days for each team member across the coreweave org. Run these in parallel:

```bash
# Merged PRs - abatilo
gh search prs --author=abatilo --owner=coreweave --merged-at=">=$(date -v-7d +%Y-%m-%d)" --limit=50 --json title,repository,url,closedAt

# Merged PRs - NavarrePratt
gh search prs --author=NavarrePratt --owner=coreweave --merged-at=">=$(date -v-7d +%Y-%m-%d)" --limit=50 --json title,repository,url,closedAt

# Opened PRs (not yet merged) - abatilo
gh search prs --author=abatilo --owner=coreweave --created=">=$(date -v-7d +%Y-%m-%d)" --state=open --limit=30 --json title,repository,url,createdAt

# Opened PRs (not yet merged) - NavarrePratt
gh search prs --author=NavarrePratt --owner=coreweave --created=">=$(date -v-7d +%Y-%m-%d)" --state=open --limit=30 --json title,repository,url,createdAt
```

Note: macOS `date` uses `-v-7d` for 7 days ago. If that fails, try `date -d '7 days ago'` (GNU).

### Step 2: Collect Slack Activity

Use the Slack MCP tools. Run these searches in parallel:

**Per-person activity** (broad search across all channels):
- `mcp__slack__conversations_search_messages` with `filter_users_from` set to each user's display name (@abatilo, @npratt)
- Set `filter_date_after` to 7 days ago (YYYY-MM-DD format)
- Limit to 20-30 messages per person

**Tracked project channels** (for project-level context the team may not have authored):
- Pull recent history from channels listed in the tracked projects table (e.g., #aviato)
- Use `mcp__slack__conversations_history` with limit of 1w
- This catches updates from other contributors, customer feedback, and decisions that inform the project bullet

Skim messages for themes: what projects are being discussed, what shipped, blockers, decisions. Ignore casual chatter and short replies - focus on substantive messages about work.

### Step 3: Synthesize

Start with the tracked projects table - check if any had meaningful progress this week. Then look at the rest of the activity for anything else worth mentioning.

Think at the project level: what milestones were hit, what's now possible that wasn't before, what moved forward in a way that matters. Group related PRs and Slack threads into a single narrative bullet. 3-5 bullets max.

Good bullet: "Aviato rename to CoreWeave Sandbox is done - client, docs, and charts all updated"
Bad bullet: "Merged 28 PRs across cwsandbox-client, aviato, ark, actions, docs, kabinet-charts"

Do NOT include:
- PR counts or technical stats
- Raw PR titles or commit messages
- Trivial changes (typo fixes, dependency bumps)
- Work that hasn't meaningfully progressed
- Anything that reads like a promo packet or changelog

### Step 4: Present Draft

Present two sections. The first is the actual update ready to post. The second is a reference section with the raw activity so the user can jog their memory and tweak the draft.

```
## Draft

AT:
 [bullet 1]
 [bullet 2]
 ...

## Activity Reference

**[Team member name]**
PRs merged:
- [PR title] (repo) - [url]
- ...

PRs open:
- [PR title] (repo) - [url]
- ...

Slack highlights:
- [channel] - [brief summary of substantive message]
- ...

**[Next team member]**
...
```

The draft is what gets posted. The activity reference is just for the user to scan and decide if anything was missed or mischaracterized. Ask if they want to adjust anything before posting.

### Step 5: Find Thread and Post

After the user approves:

1. Search #eng-amps-ext-staff for the current week's AMPS Bites thread:
   - Use `mcp__slack__conversations_search_messages` with search_query "AMPS Bites" or "AMPS bites", filter_in_channel #eng-amps-ext-staff, filter_date_on today
   - The thread parent message typically contains "AMPS Bites" or a Slackbot reminder about posting updates

2. Post the update as a reply using `mcp__slack__conversations_add_message`:
   - channel_id: the channel ID from the search result
   - thread_ts: the timestamp of the parent message
   - payload: the approved update text

If no thread is found for today, check the last few days - the thread might have been posted earlier in the week. If still not found, show the final text and tell the user to post it manually.

Always confirm with the user before posting to Slack.
