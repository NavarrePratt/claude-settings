# Complexity Rubric

Deterministic rubric for classifying whether a finding requires approach selection
(complex) or can use the default fix flow (trivial).

## Scoring

Score each finding against these criteria. Each criterion is worth +1 point.

| Criterion | +1 when |
|-----------|---------|
| **Multi-function scope** | Fix touches 3+ functions or methods |
| **Multiple valid approaches** | Code structure admits more than one reasonable fix strategy |
| **New patterns required** | Fix introduces patterns/abstractions not already present in the codebase |
| **Public API impact** | Fix changes exported functions, method signatures, or interface contracts |
| **Non-obvious fix** | Reviewer's suggested fix is lengthy (>10 lines) or requires domain-specific reasoning |

## Classification

| Score | Classification | Behavior |
|-------|---------------|----------|
| 0-1 | Trivial | Standard Fix/Skip/Defer/More info flow |
| 2+ | Complex | Generate 2-3 approaches for user selection |

**When uncertain, classify as complex.** False positives (showing approaches for a
trivial fix) waste a few seconds. False negatives (auto-fixing something that needed
deliberation) risk the wrong fix being applied.

## Evaluation Procedure

For each canonical finding:

1. **Read context**: Read ~50 lines around the finding location
2. **Score each criterion**: Check the five criteria above, award +1 for each that applies
3. **Record score**: Store `complexity_score` and `complexity_class` on the finding
4. **Route**: Trivial findings continue through the existing flow. Complex findings
   enter approach generation (see [approach-generation.md](approach-generation.md)).

## Examples

### Trivial (score 1)
- Missing nil check on a single return value: touches 1 function, one obvious fix,
  no new patterns, no API change, fix is 2 lines. Score: 0.

### Complex (score 3)
- Error handling refactor across an HTTP handler: touches handler + 2 helper functions (+1),
  could wrap errors vs return early vs use middleware (+1), changes response contract (+1).
  Score: 3.

### Edge case (score 2, classify complex)
- Race condition in concurrent map access: touches 2 functions, could use mutex vs
  sync.Map vs channel (+1), introduces sync pattern not present in codebase (+1).
  Score: 2 - complex.
