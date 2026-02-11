---
name: team-branch-comment
description: Post review findings as PR comments on specific lines. Accepts a review report (from /team-branch-review or pasted), interviews the user on which findings to comment on, then posts targeted review comments via GitHub API.
argument-hint: "[review report or path to report]"
---

# Team Branch Comment

Post code review findings as GitHub PR review comments targeting specific file:line locations. Single-agent skill - no team infrastructure needed since this drafts text rather than editing files.

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`

## Instructions

You are posting review findings as targeted PR comments. You will verify the PR exists, gather findings, check for duplicate comments, interview the user on which findings to post, draft comments, and post them via `gh api`.

**CRITICAL: You MUST use the AskUserQuestion tool for ALL user-facing questions. Do NOT print questions as text output and wait for free-form responses. Every question about whether to comment, skip, or defer a finding MUST be an AskUserQuestion tool call with structured options.**

---

### Phase 0: Precondition Check

Before anything else, verify required tools and git state.

**Step 1: gh CLI availability**

```bash
gh --version
```

If `gh` is not available, stop and tell the user:
"This skill requires the GitHub CLI (gh). Install it from https://cli.github.com/ and authenticate with `gh auth login`."

**Step 2: Branch has commits**

```bash
git log --oneline $(git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main)..HEAD
```

If no commits ahead of main, stop and tell the user: "Current branch has no commits vs main. Nothing to comment on."

**Step 3: Find PR for current branch**

```bash
gh pr view --json number,url,headRefName,baseRefName
```

If no PR exists, stop and tell the user: "No PR found for the current branch. Create a PR first with `gh pr create`, then retry."

Record **PR_NUMBER** and **PR_URL** from the output.

**Step 4: Get repo identifiers**

```bash
gh repo view --json owner,name --jq '{owner: .owner.login, name: .name}'
```

Record **OWNER** and **REPO**.

**Step 5: Get HEAD commit SHA**

```bash
git rev-parse HEAD
```

Record **HEAD_SHA** for use in comment posting.

---

### Phase 1: Gather Review Report

Obtain the review report from one of these sources (check in order):

1. **Conversation context**: If a `/team-branch-review` or `/parallel-branch-review` report exists in the conversation above, use it directly
2. **$ARGUMENTS**: If arguments are provided, treat them as the report or a path to one
3. **Ask the user**: If no report is available, ask the user to provide one

**Preserve the raw report text** - save the full original report (markdown) as **RAW_REPORT** for embedding in the review body later. This ensures the complete context is always available on the PR even if individual comments are skipped or dropped.

Parse the report and extract all findings into a structured list:
- Finding title
- File and line number
- Severity (Critical/High/Medium/Low)
- Category
- Issue description
- Suggested fix
- Codex validation status (Confirmed/Disputed/Adjusted)
- Found by (which reviewer)

---

### Phase 2: Fetch Existing PR Comments

Fetch all review comments currently on the PR to detect duplicates:

```bash
gh api repos/OWNER/REPO/pulls/PR_NUMBER/comments --jq '.[] | {id, path, line, body: .body[:120]}'
```

Build a lookup of existing comments by file path -> [{line, body_snippet}].

For each finding, check if a comment already exists:
- Same file path
- Line number within 5 lines of the finding's location
- Body contains similar keywords or phrasing

Mark findings that match as **has_existing_comment** and record a snippet of the existing comment for display during the interview.

---

### Phase 3: User Interview

**You MUST call the AskUserQuestion tool for every question below. Do NOT print questions as plain text.**

**Step 1: Critical & High findings (no existing comment)**

For each confirmed Critical or High finding that does NOT have an existing comment, call the AskUserQuestion tool:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "[Finding title] in [file:line] - [one-line issue description]. Post comment?",
    header: "High finding",
    options: [
      { label: "Comment (Recommended)", description: "Post a review comment at this location" },
      { label: "More info", description: "Investigate the code in depth before deciding" },
      { label: "Skip", description: "Intentional or not worth commenting on" },
      { label: "Defer", description: "Handle in a later session" }
    ],
    multiSelect: false
  }]
```

**Step 2: Critical & High findings (with existing comment)**

For findings that already have a similar comment on the PR, shift the default:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "[Finding title] in [file:line] - [one-line description]. Note: existing comment at line N: \"[snippet]\"",
    header: "Has comment",
    options: [
      { label: "Skip (Recommended)", description: "Similar comment already exists on this line" },
      { label: "Comment anyway", description: "Post an additional comment despite the existing one" },
      { label: "More info", description: "Investigate the code and existing comment before deciding" }
    ],
    multiSelect: false
  }]
```

**If the user selects "More info" (in either Step 1 or Step 2):**

Investigate the finding in depth:
1. Read the file around the flagged line with the Read tool (~50 lines of context)
2. Trace callers and dependents with Grep to show impact
3. Check git blame: `git log --oneline -3 -- <file>`
4. If there is an existing comment, show the full comment body
5. Present a summary:
   - The actual code in question (with surrounding context)
   - What the reviewer flagged and why
   - What the suggested fix would change
   - Potential impact on other code
   - Risk assessment

Then re-ask with AskUserQuestion (without the "More info" option):
```
Call AskUserQuestion tool with:
  questions: [{
    question: "With the above context - post comment for [Finding title] in [file:line]?",
    header: "Decision",
    options: [
      { label: "Comment (Recommended)", description: "Post a review comment" },
      { label: "Skip", description: "Not worth commenting on" },
      { label: "Defer", description: "Handle later" }
    ],
    multiSelect: false
  }]
```

**Step 3: Disputed Critical & High findings**

For findings where Claude and Codex disagreed, call AskUserQuestion with both perspectives:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "[Finding title] in [file:line] - DISPUTED. Claude: [issue]. Codex: [counter]. Comment?",
    header: "Disputed",
    options: [
      { label: "Comment per Claude", description: "Post comment based on Claude's assessment" },
      { label: "Comment per Codex", description: "Post comment based on Codex's assessment" },
      { label: "More info", description: "Investigate both positions before deciding" },
      { label: "Skip", description: "Disagreement suggests it may not be clear-cut" }
    ],
    multiSelect: false
  }]
```

If "More info", follow the same investigation procedure as Step 1, but also highlight the specific disagreement between Claude and Codex with code evidence for each position. Then re-ask without the "More info" option.

**Step 4: Medium & Low findings**

Batch these by category using AskUserQuestion:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "N medium/low findings in [category]. How to handle?",
    header: "Med/Low",
    options: [
      { label: "Comment all (Recommended)", description: "Post comments for all findings in this category" },
      { label: "Review individually", description: "Decide on each finding separately" },
      { label: "Skip all", description: "Skip all findings in this category" }
    ],
    multiSelect: false
  }]
```

If "Review individually", present each finding with the same AskUserQuestion pattern as Step 1 (including "More info" option and duplicate detection from Step 2).

**Step 5: Summary confirmation**

Present the final plan:
- N findings to comment on (by severity)
- N findings skipped
- N findings deferred
- Target PR: PR_URL

Call AskUserQuestion to confirm:
```
Call AskUserQuestion tool with:
  questions: [{
    question: "Post N comments to PR #PR_NUMBER?",
    header: "Confirm",
    options: [
      { label: "Proceed (Recommended)", description: "Draft comments and review before posting" },
      { label: "Review plan again", description: "Go back and change decisions" },
      { label: "Cancel", description: "Abort without posting any comments" }
    ],
    multiSelect: false
  }]
```

---

### Phase 4: Draft Comments

For each approved finding, generate a comment body using this format:

```
[via Claude] **[Severity]: [Title]**

[Issue description - concise explanation of the problem]

**Suggestion:** [concrete fix suggestion with code if applicable]

_[Validation status] by Codex ([confidence]) - [rationale]_
_Found by: [reviewer name(s)]_
```

Rules for comment drafting:
- Every comment MUST start with `[via Claude]`
- Keep comments focused and actionable
- Include code suggestions using GitHub markdown code blocks when the fix is concrete
- For disputed findings, include the perspective the user chose (Claude or Codex)
- If multiple reviewers flagged the same issue, credit all of them

Collect all drafts into a list with their target file:line.

---

### Phase 5: Present Drafts & Confirm

Display all draft comments in a readable format:

```markdown
## Draft Comments for PR #PR_NUMBER

### Comment 1: [file:line]
[Full comment body]

### Comment 2: [file:line]
[Full comment body]

...

**Total: N comments (N inline, N general)**
```

Note: "general" comments are findings with no specific file:line - these will be included in the review body rather than as inline comments.

Call AskUserQuestion for final approval:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "Post these N comments as a single PR review on PR #PR_NUMBER?",
    header: "Post",
    options: [
      { label: "Post all (Recommended)", description: "Submit all comments as one PR review" },
      { label: "Edit comments first", description: "Review and edit individual comments before posting" },
      { label: "Cancel", description: "Abort without posting" }
    ],
    multiSelect: false
  }]
```

**If "Edit comments first":**

For each comment, present it and call AskUserQuestion:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "Comment for [file:line]: [first 60 chars of body]...",
    header: "Edit",
    options: [
      { label: "Post as-is", description: "Keep this comment unchanged" },
      { label: "Edit", description: "Provide replacement text (use Other to type)" },
      { label: "Skip", description: "Remove this comment from the batch" }
    ],
    multiSelect: false
  }]
```

If the user selects "Edit" (via the Other option), use their replacement text as the new comment body. Ensure the `[via Claude]` prefix is preserved - if the user's text doesn't include it, prepend it.

**Step: Choose review event type**

After all edits are finalized, call AskUserQuestion to choose the review event:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "What type of review to submit?",
    header: "Review type",
    options: [
      { label: "Comment (Recommended)", description: "Neutral review with comments - no approval or rejection" },
      { label: "Request changes", description: "Formally request changes on the PR" }
    ],
    multiSelect: false
  }]
```

Record the choice as **REVIEW_EVENT**: "COMMENT" or "REQUEST_CHANGES".

---

### Phase 6: Post Review

Submit all comments as a single PR review using the GitHub Reviews API. This groups all findings under one review instead of posting individual comments.

**Step 1: Build the review payload**

Construct a JSON file at `/tmp/pr-review-payload.json` with this structure:

```json
{
  "commit_id": "HEAD_SHA",
  "event": "REVIEW_EVENT",
  "body": "REVIEW_BODY",
  "comments": [
    {
      "path": "FILE_PATH",
      "line": LINE_NUMBER,
      "body": "COMMENT_BODY"
    }
  ]
}
```

Where:
- **commit_id**: The HEAD_SHA recorded in Phase 0
- **event**: The REVIEW_EVENT chosen by the user ("COMMENT" or "REQUEST_CHANGES")
- **body**: A review summary (see below)
- **comments**: Array of all inline comments with file:line locations

**Review body format:**

```
[via Claude] Code review: N findings across M files.

[If there are general findings with no file:line, include them here:]

---

**General findings:**

[Each general finding formatted the same as inline comments but listed in the review body]

---

<details>
<summary>Full review report</summary>

[RAW_REPORT - the complete original review report preserved verbatim]

</details>
```

If all findings have file:line locations and there are no general findings, omit the "General findings" section but always include the collapsible full report:
```
[via Claude] Code review: N findings across M files.

<details>
<summary>Full review report</summary>

[RAW_REPORT]

</details>
```

**Step 2: Write and post the payload**

Write the JSON payload to a temp file and submit via `gh api`:

```bash
gh api repos/OWNER/REPO/pulls/PR_NUMBER/reviews \
  --input /tmp/pr-review-payload.json
```

Use Python or `jq` to construct the JSON safely, avoiding shell escaping issues with comment bodies:

```bash
python3 -c "
import json, sys
payload = {
    'commit_id': 'HEAD_SHA',
    'event': 'REVIEW_EVENT',
    'body': '''REVIEW_BODY''',
    'comments': [
        {'path': 'file.py', 'line': 10, 'body': 'comment text'},
        # ... all inline comments
    ]
}
json.dump(payload, open('/tmp/pr-review-payload.json', 'w'))
"

gh api repos/OWNER/REPO/pulls/PR_NUMBER/reviews \
  --input /tmp/pr-review-payload.json
```

**Important:**
- All inline comments (those with file:line) go in the `comments` array
- Findings with no specific file:line go in the review `body` instead
- If the API returns 422 (Validation Failed), check that all file paths match the PR diff and line numbers are within the diff hunks. If a specific comment causes the failure, fall back to posting it as part of the review body instead, and retry with the remaining inline comments
- If the API returns 403, report the failure and suggest checking `gh` authentication and PR permissions
- Clean up `/tmp/pr-review-payload.json` after posting

---

### Phase 7: Final Report

Present the summary:

```markdown
## Review Summary for PR #PR_NUMBER

- Review type: [COMMENT or REQUEST_CHANGES]
- Inline comments: N
- General findings (in review body): N
- Skipped: N (already commented or user choice)
- Deferred: N
- Status: [Posted / Failed]

### Inline Comments
[For each: file:line - finding title]

### General Findings
[For each: finding title - included in review body]

### Review URL
[Link to the review if available from API response]
```

If the review failed, suggest the user check their `gh` authentication and PR permissions. If specific inline comments caused a 422, note which ones were moved to the review body on retry.

---

## Error Handling

| Scenario | Recovery |
|----------|----------|
| No PR for current branch | Stop with message to create PR first |
| gh CLI not available | Stop with install instructions |
| Review post fails (403) | Report failure, suggest checking gh auth and PR permissions |
| Review post fails (422) | Likely a bad file path or line not in diff. Move offending inline comments to the review body and retry |
| No findings approved | Report "No comments to post" and exit |
| Finding has no file:line | Include in review body instead of comments array |
| Rate limited by GitHub API | Wait and retry with backoff |
| Report has no parseable findings | Ask user to provide a valid review report |

## Guidelines

- **Single agent**: No team or subagents needed - handle everything in the main session
- **Single review**: All comments are posted as one PR review, not individual comments
- **[via Claude] prefix**: The review body and every inline comment MUST start with `[via Claude]`
- **Duplicate detection**: Always check existing PR comments before the interview
- **User approval before posting**: NEVER post the review without explicit user confirmation
- **Shell safety**: Use Python to build JSON payload - avoid shell escaping issues
- **Preserve user edits**: If the user modifies a comment draft, use their text exactly (with [via Claude] prefix)
- **No file modifications**: This skill only posts review comments - it does NOT edit code
- **Clean up temp files**: Remove `/tmp/pr-review-payload.json` after posting

## Chaining from Review

This skill works best when chained from a review:
1. Run `/team-branch-review` or `/parallel-branch-review` to generate findings
2. Run `/team-branch-comment` to post findings as PR comments
3. Optionally run `/team-branch-fix` to fix findings locally on your own branches

$ARGUMENTS
