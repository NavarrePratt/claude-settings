# Finding Decision Schema

Data model for findings as they flow through the fix pipeline. Fields are populated
progressively across phases.

## Finding Record

```yaml
finding:
  # Identity (Phase 1)
  finding_id: "f-1"              # sequential ID assigned during parsing
  finding_scope: line             # line | file | cross-cutting

  # Canonicalization (Phase 1.5)
  group_id: "g-1"                # canonical group after dedup
  source_finding_ids: ["f-1"]    # all finding_ids merged into this group
  is_canonical: true             # true for the representative finding in a group

  # Review data (Phase 1 - from report)
  title: "Nil pointer dereference"
  file: "pkg/auth/handler.go"
  line: 42
  severity: "High"               # Critical | High | Medium | Low
  category: "correctness"
  description: "err is not checked before using resp"
  suggested_fix: "Add nil check for resp before access"
  codex_status: "Confirmed"      # Confirmed | Disputed | Adjusted
  found_by: ["security-reviewer", "correctness-reviewer"]

  # Complexity classification (Phase 2)
  complexity_score: 0              # 0-5, sum of rubric criteria
  complexity_class: "trivial"      # trivial (score < 2) | complex (score >= 2)

  # User decision (Phase 2)
  decision:
    disposition: null              # fix | skip | defer (set during interview)
    approach: null                 # label of chosen approach (complex findings only)
    approach_detail: null          # code preview of chosen approach (forwarded to fixer)
    approach_source: null          # default | user_choice | codex_validated
    stance: null                   # claude | codex (disputed findings only: whose assessment won)

  # Implementation result (Phase 5 - set by fixer agent)
  implementation:
    status: null                   # applied | blocked | skipped
    approach_used: null            # what was actually implemented (approach label or "suggested fix")
    blocked_reason: null           # why the approach was not viable (blocked only)
    fallback_a: null               # first concrete alternative considered (blocked only)
    fallback_b: null               # second concrete alternative considered (blocked only)
```

## Field Lifecycle

| Field | Set in | Updated in |
|-------|--------|------------|
| `finding_id` | Phase 1 | never |
| `finding_scope` | Phase 1 | never |
| `group_id` | Phase 1.5 | never |
| `source_finding_ids` | Phase 1.5 | never |
| `is_canonical` | Phase 1.5 | never |
| `title` through `found_by` | Phase 1 | Phase 1.5 (found_by merged) |
| `complexity_score` | Phase 2 | never |
| `complexity_class` | Phase 2 | never |
| `decision.*` | Phase 2 | Phase 3 (preserved during grouping) |
| `implementation.*` | Phase 5 | never |

## Scope Classification

- **line**: Finding references a specific file and line number
- **file**: Finding references a file but no specific line (e.g., "this file lacks error handling")
- **cross-cutting**: Finding describes a systemic pattern across multiple files or the whole codebase

## Canonicalization Rules (Phase 1.5)

### Line-scoped findings
Group by `(file, line, normalized_issue)`. Two findings match when they reference the
same file and line and describe the same underlying issue (even if worded differently by
different reviewers).

### File-scoped findings
Group by `(file, normalized_issue)`.

### Cross-cutting findings
Group by issue description similarity. Two cross-cutting findings merge when they
describe the same systemic concern regardless of wording.

### Merge behavior
When findings merge into a canonical group:
- `group_id` is assigned as `g-{N}` (sequential)
- `source_finding_ids` collects all member `finding_id` values
- `found_by` is the union of all member `found_by` lists
- `suggested_fix` uses the most specific suggestion among members
- `severity` uses the highest severity among members
- `decision` from user interview takes precedence over defaults
