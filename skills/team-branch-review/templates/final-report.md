# Branch Review: BRANCH_NAME

## Overview
- **Branch**: BRANCH_NAME -> main
- **Commits**: N commits
- **Files Changed**: N files (+X/-Y lines)
- **Review Team**: N Claude reviewers (Opus) + Codex validation
- **Reviewers**: [list of roles that participated]

## Outcome: [APPROVED | NEEDS REVISION | MANUAL REVIEW REQUIRED]

Determination:
- APPROVED: No confirmed Critical or High findings after Codex validation
- NEEDS REVISION: Any confirmed Critical or High findings remain
- MANUAL REVIEW REQUIRED: Significant disagreement (>50% of Critical/High disputed) between Claude and Codex

---

## Critical & High Findings

### [Finding Title]
- **File**: path:line
- **Severity**: Critical/High
- **Category**: [concern area]
- **Issue**: Description
- **Suggestion**: Concrete fix
- **Validation**: [Confirmed/Disputed] by Codex ([confidence]) - [rationale]
- **Found by**: [which reviewer(s)]

[Repeat for each]

---

## Medium & Low Findings

[Same format, grouped by category for readability]

---

## Codex-Only Findings

[Issues discovered by Codex that no Claude reviewer caught]

---

## Disputed Findings

[Findings where Claude and Codex disagreed, with both perspectives and evidence]

---

## Review Statistics
- Total findings (pre-validation): N
- Post-validation: N confirmed, N disputed, N severity-adjusted, N new from Codex
- By severity: Critical: N, High: N, Medium: N, Low: N
- Cross-reviewer overlap: N findings caught by 2+ reviewers
- Claude-Codex agreement rate: X%

## Reviewer Notes
[Notable observations from individual reviewers about overall code quality, patterns, or architectural concerns]

## Commit Recommendations
[Suggestions for commit organization if applicable]
