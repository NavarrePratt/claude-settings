# Grug Cut Pass Report - {{BRANCH_NAME}}

## Summary

- Scope: {{SCOPE}}
- Findings: {{N_SLOP}} Slop, {{N_ACCEPTABLE}} Acceptable, {{N_BORDERLINE}} Borderline
- Applied: {{N_APPLIED}} cuts amended into origin commits
- Kept: {{N_KEPT}} findings
- Deferred: {{N_DEFERRED}} findings (see Deferred section)
- Blocked: {{N_BLOCKED}} findings (see Blocked section)
- Verification: {{VERIFICATION_STATUS}} ({{VERIFICATION_DETAIL}})
- Optional review chain: {{REVIEW_CHAIN_STATUS}}

## Applied Cuts

{{APPLIED_TABLE}}

Example row:
| Finding | File | Tag | Origin commit |
|---------|------|-----|---------------|
| Abstract base class with single implementation | pkg/auth/middleware.go | single_impl_abc | 3f2a1b7 |

## Kept

Findings the user chose not to cut, with their rationale.

{{KEPT_TABLE}}

## Deferred

Findings the user wants to revisit but not cut now. Useful as follow-up work items.

{{DEFERRED_TABLE}}

Example row:
| Finding | File | Note |
|---------|------|------|
| Potentially premature generic | pkg/store/repo.go | "Want to check with @teammate first" |

## Blocked

Findings where applying the cut caused a test failure or rebase conflict, and the user accepted the block.

{{BLOCKED_TABLE}}

## Auto-Mode Decision Log

{{AUTO_DECISION_LOG}}

Populated only when `--auto` was used. Lists each auto-approved finding with the anti-pattern tag that triggered approval.

Example:
```
f-1  dead_code                   auto-approved
f-3  commented_code              auto-approved
f-7  single_impl_abc             auto-approved
f-12 unreachable_error_handling  auto-approved
f-18 premature_abstraction       asked user (not on allowlist) -> user chose Cut
```

## Acceptable Complexity (for reference)

What Codex considered complex but ruled as justified. Useful for you to see what was checked and approved.

{{ACCEPTABLE_TABLE}}

## Next Steps

- Review the branch with `git log --oneline {{BASE_REF}}..HEAD`
- If satisfied, submit the PR via your normal flow (this skill never pushes or creates PRs)
- Optional: run `/team-branch-review` for correctness validation before submission
