# Codex Approach Validation

Two-step UX for validating fix approaches with Codex before the user commits to a choice.

## When This Applies

Only for complex findings (complexity score >= 2) during Phase 2 approach selection.
The "Validate with Codex" option appears alongside the generated approaches in the
first AskUserQuestion call.

## Two-Step Flow

### Step 1: Present approaches with validation option

The initial AskUserQuestion includes "Validate with Codex" as a non-final option
(before "Skip"):

```
options: [
  { label: "Approach A (Recommended)", description: "...", markdown: "..." },
  { label: "Approach B", description: "...", markdown: "..." },
  { label: "Validate with Codex", description: "Get Codex assessment before choosing" },
  { label: "Skip", description: "Don't fix this finding" }
]
```

### Step 2: If user selects "Validate with Codex"

1. **Call Codex MCP** with the approaches and code context:

```
mcp__codex__codex:
  sandbox: "read-only"
  approval-policy: "never"
  prompt: |
    Review these proposed fix approaches for a code review finding.

    ## Finding
    [title] in [file:line]
    [description]

    ## Current Code
    [~50 lines of code context around the finding]

    ## Proposed Approaches

    ### Approach A: [label]
    [markdown preview]

    ### Approach B: [label]
    [markdown preview]

    ## Evaluate
    For each approach:
    1. Will it correctly fix the reported issue?
    2. Does it introduce any new problems?
    3. Which approach do you recommend and why?
    4. Any modifications you'd suggest to the approaches?
```

2. **Re-present approaches with Codex assessment** baked into the descriptions.
   Do NOT include "Validate with Codex" in the second round:

```
# Assistant text before the question:
"Codex assessment: [summary of Codex feedback]"

options: [
  { label: "Approach A (Recommended)", description: "[original] - Codex: [assessment]", markdown: "..." },
  { label: "Approach B", description: "[original] - Codex: [assessment]", markdown: "..." },
  { label: "Skip", description: "Don't fix this finding" }
]
```

3. **Record the decision** with `approach_source: "codex_validated"` to indicate
   the user chose after seeing Codex feedback.

## What Codex Validation Does NOT Do

- It does not add new approaches (Codex evaluates, it does not generate alternatives)
- It does not override the user's choice (the user always makes the final call)
- It does not block progress (if Codex MCP is unavailable, inform the user and
  re-present without assessments - let them choose based on the original descriptions)

## Error Handling

If Codex MCP is unavailable or times out:

```
"Codex validation unavailable. Proceeding with the original approaches."
```

Re-present the same options without "Validate with Codex" and without assessments.
Set `approach_source: "user_choice"` (not `codex_validated`).
