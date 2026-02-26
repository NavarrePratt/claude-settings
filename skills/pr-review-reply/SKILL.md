---
name: pr-review-reply
description: Reply to PR comments after fixes are applied via team-branch-fix
argument-hint: "[path to import report]"
---

# PR Review Reply

Reply to original PR review comments after `/team-branch-fix` has applied fixes. Maps each fix back to the original comment using Comment ID metadata from the import report, drafts contextual replies referencing the fixing commit, and posts them after user approval.

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`

## Instructions

You are replying to PR review comments after fixes have been applied. You will locate the import report, parse findings with comment metadata, determine which were fixed, draft replies, get user approval, and post them via `gh api`.

**CRITICAL: You MUST use the AskUserQuestion tool for ALL user-facing questions. Do NOT print questions as text output and wait for free-form responses.**

---

### Phase 0: Prerequisites

**Step 1: gh CLI**

```bash
gh --version
```

If not available, stop: "This skill requires the GitHub CLI (gh). Install it from https://cli.github.com/ and authenticate with `gh auth login`."

**Step 2: Get repo identifiers**

```bash
gh repo view --json owner,name --jq '{owner: .owner.login, name: .name}'
```

Record **OWNER** and **REPO**.

**Step 3: Find PR for current branch**

```bash
gh pr view --json number,url,headRefName,baseRefName
```

If no PR exists, stop: "No PR found for the current branch. Create a PR first, then retry."

Record **PR_NUMBER** and **PR_URL**.

---

### Phase 1: Locate Report

Find the import report using this priority:

1. **$ARGUMENTS**: If an argument is provided, treat it as a file path and read it.
2. **Temp file**: Look for the most recent matching file:
   ```bash
   ls -t /tmp/pr-review-import-*.md 2>/dev/null | head -1
   ```
3. **Conversation context**: If a `/pr-review-import` report exists in the conversation above, use it directly.
4. **Ask the user**: If none found, call AskUserQuestion:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "No import report found. Provide the path to the import report file, or paste the report contents.",
    header: "Report needed",
    options: [
      { label: "Let me provide a path", description: "I'll type the file path" },
      { label: "Cancel", description: "Abort - I need to run /pr-review-import first" }
    ],
    multiSelect: false
  }]
```

If "Cancel", stop immediately with a note to run `/pr-review-import` first.

Record the report content as **RAW_REPORT**.

---

### Phase 2: Parse Findings

Extract findings from RAW_REPORT that have **Comment ID** and **Comment Type** fields. Only findings with these fields can be replied to.

For each qualifying finding, record:
- **title**: Finding title
- **file_line**: File:line location
- **comment_id**: The Comment ID value
- **comment_type**: `review_comment`, `issue_comment`, or `review_body`
- **source_url**: The Source URL
- **severity**: Critical/High/Medium/Low
- **issue**: The Issue description
- **suggestion**: The Suggestion (if present)

If no findings have Comment ID fields, stop: "No findings with Comment ID metadata found. This report may not have been produced by `/pr-review-import`."

Report the count: "Found N findings with comment metadata."

---

### Phase 3: Identify Fixed Findings

Determine which findings were actually addressed by recent commits.

**Strategy 1: team-branch-fix output in conversation**

If `/team-branch-fix` output exists in the conversation above (look for "Fix Implementation Summary" or fixer result files), parse it for:
- Findings listed under "Fixes Applied" with Status: Applied
- Map each back to the import report findings by title and file:line match

**Strategy 2: Git diff analysis**

If no team-branch-fix output is available, fall back to diff analysis:

```bash
# Get the PR base SHA
gh api repos/OWNER/REPO/pulls/PR_NUMBER --jq '.base.sha'
```

Record **BASE_SHA**.

```bash
# Get files changed since the fix work started
# Use the most recent merge-base or the base SHA
git diff --name-only BASE_SHA..HEAD
```

For each finding, check if its file appears in the changed files list. If the file was changed, check whether the specific region around the finding's line was modified:

```bash
git diff BASE_SHA..HEAD -- <file_path>
```

Look for hunks that overlap with the finding's line number (+/- 10 lines). Mark findings as:
- **fixed**: File was changed and the relevant region was modified
- **unclear**: File was changed but the region around the finding's line was not touched
- **not_fixed**: File was not changed at all

**Step: Confirm ambiguous cases**

For findings marked **unclear**, present them to the user via AskUserQuestion:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "[Finding title] in [file:line] - file was modified but the flagged region was not directly changed. Was this addressed?",
    header: "Unclear fix",
    options: [
      { label: "Yes, it was fixed", description: "The fix addressed this finding (perhaps indirectly)" },
      { label: "No, skip it", description: "This finding was not addressed" }
    ],
    multiSelect: false
  }]
```

**Step: User deselection**

Present the full list of findings identified as fixed and let the user deselect any:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "N findings identified as fixed. Review the list and deselect any you don't want to reply to:\n\n[numbered list: title - file:line - severity]",
    header: "Confirm replies",
    options: [
      { label: "Reply to all", description: "Post replies for all N fixed findings" },
      { label: "Deselect some", description: "Let me pick which ones to reply to" },
      { label: "Cancel", description: "Don't post any replies" }
    ],
    multiSelect: false
  }]
```

If "Deselect some", present each finding individually with Post/Skip options via AskUserQuestion.

If "Cancel", stop immediately.

Record the final list of findings to reply to as **REPLY_TARGETS**.

---

### Phase 4: Find Addressing Commits

For each finding in REPLY_TARGETS, find the commit that fixed it:

```bash
git log --oneline BASE_SHA..HEAD -- <file_path>
```

Take the most recent commit that modified the finding's file. Record the **short SHA** (first 7 chars) and **commit subject** for each.

If multiple findings share the same file and the same commit fixed them all, reuse the same SHA.

---

### Phase 5: Draft Replies

For each finding in REPLY_TARGETS, draft a reply message.

**Format:**

```
[via Claude] Fixed in <short-sha> - <contextual explanation>
```

The contextual explanation should:
- Reference the specific code change made (what was modified)
- Explain the approach chosen (why this fix)
- Be concise but informative (1-3 sentences)
- Read naturally as a response to the original reviewer comment

**To write good contextual explanations**, read the actual diff for each fix:

```bash
git show <full-sha> -- <file_path>
```

Use the diff content to describe what changed concretely, not generically.

**Example replies:**

- `[via Claude] Fixed in a1b2c3d - Added nil check before accessing the response body. The handler now returns a 502 with a structured error instead of panicking on upstream failures.`
- `[via Claude] Fixed in f4e5d6c - Switched from string concatenation to parameterized query. All user inputs in this handler now go through the prepared statement path.`
- `[via Claude] Fixed in 7a8b9c0 - Extracted the timeout value to a constant and increased it from 5s to 30s based on observed p99 latency for this endpoint.`

---

### Phase 6: User Review

Present all draft replies grouped by severity (Critical/High first).

For each reply, show:
- Original comment summary (first ~80 chars of the finding's issue description)
- Draft reply text
- Target: comment type and ID

Use AskUserQuestion for each reply:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "Reply to: [file:line] - [finding title]\nOriginal: [first 80 chars of issue]\nDraft: [draft reply text]",
    header: "[Severity]",
    options: [
      { label: "Post as-is", description: "Send this reply" },
      { label: "Edit", description: "Modify the reply text (use Other to type)" },
      { label: "Skip", description: "Don't reply to this comment" }
    ],
    multiSelect: false
  }]
```

If "Edit", use the user's replacement text. Ensure the `[via Claude]` prefix is preserved - if the user's text doesn't include it, prepend it.

After reviewing all replies, present the final summary:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "Post N replies to PR #PR_NUMBER?",
    header: "Confirm",
    options: [
      { label: "Post all", description: "Send all N approved replies" },
      { label: "Cancel", description: "Abort without posting" }
    ],
    multiSelect: false
  }]
```

If "Cancel", stop immediately.

---

### Phase 7: Post Replies

For each approved reply, post using the correct endpoint based on Comment Type.

**review_comment**:

```bash
jq -n --arg body "REPLY_TEXT" '{body: $body, in_reply_to: COMMENT_ID}' | \
  gh api repos/OWNER/REPO/pulls/PR_NUMBER/comments --input - -X POST
```

Note: Uses `in_reply_to` in the JSON body to thread the reply under the original comment.

**issue_comment**:

```bash
gh api repos/OWNER/REPO/issues/PR_NUMBER/comments \
  -X POST -f body="REPLY_TEXT"
```

Since issue comments don't support threading, include a quote of the original and source URL for context:

```
> Re: [original comment first line, truncated to 100 chars]
> ([source_url])

REPLY_TEXT
```

**review_body**:

```bash
gh api repos/OWNER/REPO/issues/PR_NUMBER/comments \
  -X POST -f body="REPLY_TEXT"
```

Include a reference to the original review:

```
> Re: review comment ([source_url])

REPLY_TEXT
```

**Error handling per reply:**
- If a POST fails with 404, the comment may have been deleted. Log the failure and continue.
- If a POST fails with 403, report auth/permission issue and stop posting further replies.
- If a POST fails with 422, log the error details and continue with remaining replies.

Use `jq` with `--arg` for safe JSON escaping of reply bodies:

```bash
jq -n --arg body "REPLY_TEXT" '{body: $body, in_reply_to: COMMENT_ID}' | \
  gh api repos/OWNER/REPO/pulls/PR_NUMBER/comments --input - -X POST
```

---

### Phase 8: Summary

Report the results:

```markdown
## Reply Summary for PR #PR_NUMBER

- Replies posted: N
- Replies skipped: N
- Failures: N

### Posted Replies
[For each: file:line - finding title - commit SHA - comment link]

### Skipped
[For each: file:line - finding title - reason]

### Failures
[For each: file:line - finding title - error details]
```

If there were failures, suggest checking `gh auth status` and PR permissions.

---

## Error Handling

| Scenario | Recovery |
|----------|----------|
| No PR for current branch | Stop with message to create PR first |
| gh CLI not available | Stop with install instructions |
| No import report found | Ask user to provide path or run /pr-review-import |
| Report has no Comment IDs | Stop with message about needing /pr-review-import output |
| No findings identified as fixed | Report "No fixed findings found" and exit |
| Comment deleted (404) | Log and continue with remaining replies |
| Auth failure (403) | Stop and report permission issue |
| Rate limited | Wait and retry with backoff |

## Guidelines

- **Single agent**: No team or subagents needed
- **[via Claude] prefix**: Every reply MUST start with `[via Claude]`
- **User approval before posting**: NEVER post replies without explicit user confirmation
- **Shell safety**: Use `jq` with `--arg` to build JSON payloads
- **No file modifications**: This skill only posts replies - it does NOT edit code
- **Correct endpoints**: review_comment uses `/pulls/PR_NUMBER/comments` with `in_reply_to: COMMENT_ID` in the JSON body, issue_comment and review_body use `/issues/PR_NUMBER/comments`

## Chaining

This skill works best when chained after a fix workflow:
1. Run `/pr-review-import <PR>` to import comments into a report
2. Run `/team-branch-fix` to implement fixes
3. Run `/pr-review-reply` to reply to original comments with fix references

$ARGUMENTS