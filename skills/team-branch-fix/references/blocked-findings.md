# Blocked Findings

Protocol for handling findings where the fixer agent could not implement the user's
chosen approach. Covers the results file format, team lead resolution, and follow-up
fixer spawning.

## Blocked Finding Results Format

Fixers report blocked findings in their results file under `## Blocked Findings`:

```markdown
## Blocked Findings
For each finding where the chosen approach was not viable:
- **Finding ID**: f-N
- **Finding**: [original title]
- **File**: path:line
- **Chosen approach**: What the user selected
- **Why blocked**: Why this approach is not viable given the actual code
- **Fallback A**: [concrete alternative with brief description]
- **Fallback B**: [concrete alternative with brief description]
```

Key rules for fixers:
- Do NOT silently pick an alternative when an approach is not viable
- Mark the finding as Blocked and provide exactly 2 concrete fallback options
- Continue with remaining findings normally
- Mark task completed as usual - the team lead handles blocked findings

## Phase 7.5 Resolution Protocol

After collecting all fixer results (end of Phase 6), before running verification.

### Step 1: Parse blocked findings

Scan each fixer's results file for the `## Blocked Findings` section. Collect all
blocked entries with their finding ID, file, reason, and fallback options.

If no blocked findings exist, skip directly to Phase 7.

### Step 2: Present fallbacks to user

For each blocked finding, call AskUserQuestion:

```
AskUserQuestion:
  questions: [{
    question: "[Finding title] in [file:line] - fixer could not implement chosen approach.
               Reason: [why blocked]. Choose a fallback?",
    header: "Blocked",
    options: [
      { label: "Fallback A", description: "[fixer's first alternative]" },
      { label: "Fallback B", description: "[fixer's second alternative]" },
      { label: "Skip", description: "Leave this finding unfixed" }
    ],
    multiSelect: false
  }]
```

Record each user decision:
- **Fallback selected**: Queue for follow-up implementation. Update the finding's decision:
  `approach` = chosen fallback label, `approach_detail` = fallback description,
  `approach_source` = `user_choice`.
- **Skip**: Mark as skipped-after-block. No further action.

### Step 3: Spawn follow-up fixers (if any fallbacks approved)

If all blocked findings were skipped, proceed to Phase 7.

If any fallbacks were approved:

1. Create follow-up tasks via TaskCreate with subject
   "Follow-up: fix N findings with fallback approaches"
2. Spawn follow-up fixer agents with the same prompt template as Phase 5. Differences:
   - Each finding's `approach` and `approach_detail` reflect the user's chosen fallback
   - Same exclusive ownership rules (same files as original fixer)
   - Results written to `/tmp/fix-TEAM_NAME/{fixer-name}-followup.md`
3. Follow-up fixers go through the same Codex self-validation step

### Step 4: Poll follow-up fixers

Re-enter the Phase 6 polling loop until all follow-up tasks show `completed`.
Same timeout and follow-up message rules apply.

### Step 5: Check follow-up results

Read follow-up results files. If any follow-up fixer also reports Blocked:
- Do NOT spawn another round (max 1 re-entry enforced)
- Report the finding as **unresolved** - user handles manually

## Iterative Orchestration Flow

```
Phase 6 (poll original fixers)
  -> collect results
  -> Phase 7.5 (check blocked findings)
     -> if blocked findings resolved with follow-ups:
        -> Phase 6 polling (follow-up fixers only)
        -> collect follow-up results
        -> Phase 7.5 re-check (max 1 re-entry; new blocks become unresolved)
     -> Phase 7 (verify all changes: original + follow-up)
```

## Follow-up Fixer Invariants

- Same prompt template, same Codex self-validation, same results format
- Write to same directory (different filename: `{name}-followup.md`)
- Team lead NEVER directly implements fixes (preserves exclusive ownership,
  validation workflow, and audit trail)
- Max 1 re-entry: if follow-up fixer also blocks on the same finding, report as unresolved

## Finding Resolution Outcomes

After Phase 7.5, each originally-blocked finding has one outcome:

| Outcome | Meaning |
|---------|---------|
| `resolved_fallback` | User chose a fallback, follow-up fixer applied it |
| `skipped_by_user` | User chose Skip in Phase 7.5 |
| `unresolved` | Follow-up fixer also blocked (max re-entry hit) |
