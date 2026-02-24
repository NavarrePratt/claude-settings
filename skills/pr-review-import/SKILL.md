---
name: pr-review-import
description: Import PR review comments into a review report for team-branch-fix
argument-hint: "<PR URL or number> [filter instructions]"
---

# PR Review Import

Import review comments from a GitHub PR and convert them into the standard review report format consumed by `/team-branch-fix` and `/team-branch-comment`.

## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`

## Instructions

You are importing PR review comments and converting them into the standard branch review report format. This is a single-agent skill - no team or subagents needed.

**This skill does NOT edit files.** It produces a review report only.

---

### Phase 0: Prerequisites Guard

**Step 1: gh authentication**

```bash
gh auth status
```

If not authenticated, stop and tell the user: "GitHub CLI is not authenticated. Run `gh auth login` first."

**Step 2: GitHub repo check**

```bash
gh repo view --json owner,name --jq '{owner: .owner.login, name: .name}'
```

If this fails, stop and tell the user: "Current directory is not a GitHub repository (or gh cannot detect one)."

Record **OWNER** and **REPO** from the output.

---

### Phase 1: Parse Input

The user's input comes from `$ARGUMENTS`. Parse it to extract:

1. **PR identifier**: Either a full URL (`https://github.com/org/repo/pull/123`) or a bare number (`123`)
   - From a URL: extract owner, repo, and PR number. These override the OWNER/REPO from Phase 0.
   - From a bare number: use OWNER/REPO from Phase 0.
2. **Filter instructions**: Any text after the PR identifier. This is free-text that Claude interprets naturally (e.g., "only from @user", "ignore bot comments", "only security-related").

Record **PR_NUMBER** and any **FILTER_TEXT**.

---

### Phase 2: Branch Guard

Compare local HEAD against the PR's head SHA to ensure we're on the right branch.

**Step 1: Fetch PR head SHA**

```bash
gh api repos/OWNER/REPO/pulls/PR_NUMBER --jq '.head.sha'
```

Record **PR_HEAD_SHA**.

**Step 2: Get local HEAD**

```bash
git rev-parse HEAD
```

Record **LOCAL_HEAD**.

**Step 3: Compare**

If PR_HEAD_SHA != LOCAL_HEAD:
- Warn the user: "Branch mismatch: PR head is `PR_HEAD_SHA` but local HEAD is `LOCAL_HEAD`."
- Ask to confirm override via AskUserQuestion:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "PR head SHA does not match local HEAD. Continue anyway?",
    header: "Branch mismatch",
    options: [
      { label: "Continue", description: "Proceed despite SHA mismatch - report will still reference PR comments" },
      { label: "Abort", description: "Stop and let me check out the correct branch" }
    ],
    multiSelect: false
  }]
```

If the user selects "Abort", stop immediately.

---

### Phase 3: Fetch Comments

Fetch all three comment types with pagination:

```bash
# Review comments (inline code comments)
gh api --paginate repos/OWNER/REPO/pulls/PR_NUMBER/comments

# Reviews (top-level review bodies)
gh api --paginate repos/OWNER/REPO/pulls/PR_NUMBER/reviews

# Issue comments (general PR discussion)
gh api --paginate repos/OWNER/REPO/issues/PR_NUMBER/comments
```

Parse the JSON responses and record the total count across all three sources.

---

### Phase 4: Soft Cap

If total comments across all three sources > 100:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "This PR has N total comments. Processing all of them may take a while. Continue?",
    header: "Many comments",
    options: [
      { label: "Continue", description: "Process all N comments" },
      { label: "Abort", description: "Stop - I'll re-run with a filter" }
    ],
    multiSelect: false
  }]
```

If the user selects "Abort", stop immediately.

---

### Phase 5: Apply Filters

If FILTER_TEXT is non-empty, apply the user's filter instructions to the collected comments. Claude interprets these naturally:

- "only from @user" - keep only comments authored by that user
- "ignore bot comments" - remove comments from bot accounts
- "only security-related" - keep only comments discussing security topics
- Any other free-text filter

**Default behavior (no filter)**: Include ALL comments, including bot comments.

---

### Phase 6: Collapse Threads

Group review comments into threads:

1. For each review comment with `in_reply_to_id`, walk the chain to find the root comment
2. Group all replies under their root comment
3. Merge reply text in `created_at` order into the parent finding's description, preserving each reply as a quoted block with author attribution
4. Use the root comment's `path` and `line`/`original_line` for the finding's file location
5. Use the root comment's `id` as the Comment ID
6. Collect ALL unique authors across the thread into the "Found by" field: `@user1, @user2`

---

### Phase 7: Drop Non-Actionable

Remove comments that are not actionable:

- Empty bodies (null or whitespace-only)
- Pure approval signals: "LGTM", "Looks good", "Looks good to me", thumbs-up-only
- Pure emoji reactions with no text content

Keep everything else - even short comments may contain actionable feedback.

---

### Phase 8: Classify Findings

For each remaining comment, classify:

- **Severity**: Critical / High / Medium / Low
  - Critical: Security vulnerabilities, data loss, crashes
  - High: Logic errors, incorrect behavior, missing error handling
  - Medium: Code quality, style issues, minor bugs
  - Low: Nits, suggestions, documentation

- **Category**: Security, Performance, Logic, Style, Documentation, Error Handling, Testing, Architecture, Correctness, or other appropriate category

Use the comment content and context to determine appropriate classification. When a thread has discussion that refines or resolves the issue, factor that into the severity.

---

### Phase 9: Optional Codex Validation

```
Call AskUserQuestion tool with:
  questions: [{
    question: "Run Codex validation on imported findings? (validates each finding against the codebase)",
    header: "Codex validation",
    options: [
      { label: "Yes", description: "Validate each finding with Codex MCP - adds confidence scores" },
      { label: "No", description: "Skip validation - faster, findings reported as-is" }
    ],
    multiSelect: false
  }]
```

**If yes**: For each finding, call `mcp__codex__codex` asking it to validate whether the finding is legitimate given the current codebase. Add a `Validation` field to each finding with the result (Confirmed/Disputed and rationale).

**If no**: Omit the `Validation` field from findings entirely.

---

### Phase 10: Generate Report

Format the report to match the team-branch-review output structure. Use the template from `~/.claude/skills/team-branch-review/templates/final-report.md` as the canonical format.

**Key differences from a standard team-branch-review report:**

1. The "Review Team" line says `Imported from PR #N` instead of reviewer count
2. Each finding includes three extra fields:
   - `- **Comment ID**: 1234567890`
   - `- **Comment Type**: review_comment|issue_comment|review_body`
   - `- **Source URL**: https://github.com/OWNER/REPO/pull/PR_NUMBER#discussion_rNNNN` (or appropriate URL)

**File/line handling:**

| Comment type | File field value |
|---|---|
| Review comment with valid `line` | `path:line` |
| Review comment with `line=null` (outdated) | `path:original_line (outdated)` |
| Multi-line review comment (`start_line` differs from `line`) | `path:start_line-line` |
| Review body (top-level review text) | `(general)` |
| Issue comment (PR discussion) | `(general)` |

**Report structure:**

```markdown
# Branch Review: BRANCH_NAME

## Overview
- **Branch**: BRANCH_NAME -> main
- **Commits**: N commits
- **Files Changed**: N files (+X/-Y lines)
- **Review Team**: Imported from PR #PR_NUMBER
- **Reviewers**: [list of unique comment authors]

## Outcome: [APPROVED | NEEDS REVISION | MANUAL REVIEW REQUIRED]

Determination:
- APPROVED: No Critical or High findings
- NEEDS REVISION: Any Critical or High findings remain
- MANUAL REVIEW REQUIRED: Findings are ambiguous or contested in thread discussion

---

## Critical & High Findings

### [Finding Title]
- **File**: path:line
- **Severity**: Critical/High
- **Category**: [concern area]
- **Issue**: Description (from comment body, with thread context merged)
- **Suggestion**: Concrete fix (if suggested in the comment)
- **Validation**: [Only if Codex validation was run] [Confirmed/Disputed] by Codex ([confidence]) - [rationale]
- **Found by**: @author1, @author2
- **Comment ID**: 1234567890
- **Comment Type**: review_comment
- **Source URL**: https://github.com/...

[Repeat for each]

---

## Medium & Low Findings

[Same format, grouped by category]

---

## Review Statistics
- Total comments fetched: N
- After filtering: N
- After thread collapsing: N
- After dropping non-actionable: N
- Final findings: N
- By severity: Critical: N, High: N, Medium: N, Low: N
- By type: review_comment: N, issue_comment: N, review_body: N
```

**Outcome determination:**
- APPROVED: No Critical or High severity findings
- NEEDS REVISION: Any Critical or High severity findings exist
- MANUAL REVIEW REQUIRED: Findings have significant back-and-forth in threads suggesting disagreement

---

### Phase 11: Output

1. **Display** the full report in the conversation
2. **Write** the report to `/tmp/pr-review-import-OWNER-REPO-PR_NUMBER.md`
3. **Report** the temp file path to the user

After presenting the report, add this note:

> **Next steps:**
> - `/team-branch-fix` - Implement fixes for these findings
> - `/team-branch-comment` - Post findings back as PR review comments (useful for cross-posting or re-posting refined comments)

---

## Error Handling

| Scenario | Recovery |
|----------|----------|
| gh not authenticated | Stop with message to run `gh auth login` |
| Not in a GitHub repo | Stop with message |
| PR not found (404) | Stop with message: "PR #N not found in OWNER/REPO" |
| No comments on PR | Report "No review comments found on PR #N" and exit |
| All comments filtered out | Report "All N comments were filtered out. Try a broader filter." |
| Branch mismatch | Warn and ask for override |
| Codex MCP unavailable | Skip validation, note in report |
| API rate limiting | Wait and retry with backoff |

## Guidelines

- **Single agent**: No team or subagents needed
- **Read-only**: This skill does NOT edit any files - it only reads PR data and produces a report
- **Thread awareness**: Collapsed threads are the primary unit of work, not individual comments
- **Preserve original text**: Include the actual comment text in findings, don't paraphrase
- **Faithful classification**: Classify based on comment content, not assumptions
- **Compatible output**: The report MUST be parseable by `/team-branch-fix` and `/team-branch-comment`

$ARGUMENTS