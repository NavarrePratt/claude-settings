# Cut Decision Schema

Data model for findings as they flow through the cut pass. Simpler than `team-branch-fix`'s decision-schema because the "approach" is always the same (delete the slop), so no complexity rubric or approach-generation step is needed.

## Finding Record

```yaml
finding:
  # Identity (Phase 3 - populated by Codex)
  id: "f-1"                          # sequential ID assigned during parse
  file: "pkg/auth/middleware.go"
  line: 42
  line_end: 58                       # optional, for multi-line cuts

  # Classification (Phase 3 - populated by Codex)
  classification: "Slop"             # Slop | Acceptable | Borderline
  confidence: "High"                 # High | Medium | Low
  anti_pattern_tag: "single_impl_abc"  # required for Slop; null otherwise

  # Description (Phase 3)
  title: "Abstract base class with single implementation"
  description: "..."                 # what the slop is
  suggested_cut: "..."               # how to remove it
  acceptable_reason: null            # populated only for Acceptable

  # Canonicalization (Phase 4 pre-prompt)
  canonical_id: "c-1"                # groups overlapping cuts (same file, adjacent lines)
  merged_finding_ids: ["f-1"]        # all findings merged into this canonical

  # User decision (Phase 4)
  decision:
    disposition: null                # cut | keep | defer
    decided_by: null                 # auto | user
    defer_note: null                 # if deferred: why
    auto_reason: null                # if decided_by=auto: which allowlist tag matched

  # Origin commit discovery (Phase 5 pre-apply)
  origin_commits: []                 # sha(s) from git blame; user picks primary if > 1
  primary_origin: null               # commit sha to amend into (single-commit mode: "HEAD+new")

  # Application result (Phase 5 - set after cut applied)
  applied:
    status: null                     # applied | blocked | skipped | reverted
    blocked_reason: null             # cut broke a test or had rebase conflict
    fallback_a: null                 # if blocked: first fallback option
    fallback_b: null                 # if blocked: second fallback option

  # Blocked resolution (Phase 6 - set if tests failed after cut)
  resolution:
    outcome: null                    # revert | keep_and_fix_separately | abort | null
    chosen_fallback: null            # if using blocked-findings protocol
```

## Field Lifecycle

| Field | Set in | Notes |
|-------|--------|-------|
| `id` through `acceptable_reason` | Phase 3 | Populated from Codex YAML output |
| `canonical_id`, `merged_finding_ids` | Phase 4 pre-prompt | Overlapping cuts merged into one decision |
| `decision.*` | Phase 4 | Auto-mode sets auto_reason; interactive sets decided_by=user |
| `origin_commits`, `primary_origin` | Phase 5 pre-apply | From git blame; user picks if multiple commits touched the lines |
| `applied.*` | Phase 5 | Set by the amend/rebase step |
| `resolution.*` | Phase 6 | Set only if verification failed |

## Canonicalization Rules (Phase 4 pre-prompt)

Findings are merged into a single canonical when all of:

- Same `file`
- Line ranges overlap OR are within 3 lines of each other
- Same or compatible `anti_pattern_tag`

When merging:
- Keep the highest-confidence finding as the canonical
- Union the line range (`line` = min, `line_end` = max)
- Concatenate descriptions with " | " separator
- `merged_finding_ids` preserves the original Codex IDs for traceability

This prevents the user from being asked three times about what is effectively one cut.

## Disposition Values

- **cut**: apply the suggested_cut in Phase 5
- **keep**: do not modify; the finding appears in the final report under "Kept"
- **defer**: do not modify now; `defer_note` captures why (e.g. "need to check with teammate first"); the finding appears in the final report under "Deferred" so the user can revisit

No "fix" disposition exists because there is no alternative approach - either the code is slop (cut it) or it is not (keep or defer).

## Auto-mode Semantics

When `--auto` is set, findings where all three hold:

1. `classification == "Slop"`
2. `confidence == "High"`
3. `anti_pattern_tag` is in the allowlist (see `cut-classification.md`)

Are set to `decision.disposition = "cut"`, `decided_by = "auto"`, `auto_reason = anti_pattern_tag` without prompting.

All other findings still go through AskUserQuestion. Auto-mode is aggressive only on mechanically-detectable slop; judgment calls still require the user.
