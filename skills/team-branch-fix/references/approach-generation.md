# Approach Generation

Guidelines for generating concrete fix approaches when a finding is classified as complex
(complexity score >= 2).

## Principles

1. **Concrete, not abstract**: Each approach shows actual code changes (max ~10 lines diff per option)
2. **Differentiated tradeoffs**: Approaches must differ meaningfully - not just cosmetic variations
3. **Respect session default**: The first approach should align with the inferred session
   default (minimal_patch / defensive / refactor) unless that style is inappropriate for the finding

## Session Default Inference (Phase 2, Step 0)

Before processing any findings, infer a default approach style for the session:

| Signal | Suggests |
|--------|----------|
| Branch name contains `fix/`, `hotfix/`, `patch/` | `minimal_patch` |
| Most findings are severity Critical/High | `minimal_patch` |
| Branch name contains `refactor/`, `cleanup/` | `refactor` |
| Most findings are Medium/Low with pattern issues | `refactor` |
| Commit messages mention stability, reliability | `defensive` |
| Mix of severities, no strong signal | `minimal_patch` (safe default) |

The session default determines which approach is listed first (recommended position).
Users can override per-finding.

## Approach Styles

### minimal_patch
- Smallest change that fixes the issue
- No structural changes, no new abstractions
- Preserves existing patterns even if imperfect

### defensive
- Fixes the issue plus adds guards against related failures
- Adds validation, nil checks, or bounds checking where warranted
- Moderate scope increase over minimal_patch

### refactor
- Restructures code to eliminate the root cause
- May introduce new patterns or abstractions
- Largest scope but most thorough

## Generating Approaches

For each complex finding:

1. **Read code context** (~50 lines around the finding)
2. **Draft 2-3 approaches** from different styles (at least 2 of: minimal_patch, defensive, refactor)
3. **For each approach**:
   - Write a short label (2-5 words): e.g., "Add nil guard", "Wrap with errgroup", "Extract validator"
   - Write a one-line description for the AskUserQuestion option
   - Write a markdown preview showing the key code change (~10 lines max)
   - Note the tradeoff in 1 sentence
4. **Order approaches**: session default first, then by ascending scope
5. **Always include "Skip" as the final option**

## Presentation Format

Before calling AskUserQuestion, output assistant text explaining:
- What the finding is and why it's complex
- What each approach trades off

Then call AskUserQuestion with markdown previews:

```
AskUserQuestion:
  questions: [{
    question: "[Finding title] in [file:line] - which approach?",
    header: "Approach",
    options: [
      {
        label: "[Approach 1 label] (Recommended)",
        description: "[one-line tradeoff]",
        markdown: "[~10 line code preview]"
      },
      {
        label: "[Approach 2 label]",
        description: "[one-line tradeoff]",
        markdown: "[~10 line code preview]"
      },
      {
        label: "Validate with Codex",
        description: "Get Codex assessment before choosing"
      },
      {
        label: "Skip",
        description: "Don't fix this finding"
      }
    ],
    multiSelect: false
  }]
```

## Storing the Decision

After the user selects an approach, update the finding's decision:

```yaml
decision:
  disposition: "fix"
  approach: "Add nil guard"           # label of chosen approach
  approach_detail: |                  # the markdown preview content
    if resp == nil {
        return fmt.Errorf("nil response from auth service")
    }
  approach_source: "user_choice"      # default | user_choice | codex_validated
  complexity_score: 3
  complexity_class: "complex"
```

If the user selects "Skip", set `disposition: "skip"` with no approach fields.

## Cross-cutting Findings

For cross-cutting findings classified as complex:
- Show one representative instance in the code preview
- Note how many locations are affected
- Approaches should describe the pattern to apply, not just one fix site
