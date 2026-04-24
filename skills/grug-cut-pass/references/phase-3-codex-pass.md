# Phase 3: Codex Grug Pass

Invoke Codex MCP with the grug-brain prompt template, parse the findings, and save to disk for Phase 4.

## Step 1: Substitute the Codex prompt template

Read `templates/codex-prompt.md` and replace these placeholders with Phase 1 outputs:

| Placeholder | Source |
|-------------|--------|
| `{{BASE_REF}}` | From Phase 0 |
| `{{SCOPE_DESCRIPTION}}` | "current branch only" or "whole git-spice stack" |
| `{{COMMITS}}` | Contents of `$TMP_DIR/commits.txt` |
| `{{FILE_LIST}}` | Contents of `$TMP_DIR/file-list.txt` |
| `{{DIFF_TEXT}}` | Contents of `$TMP_DIR/diff.patch` |
| `{{UPSTACK_IMPACT}}` | Contents of `$TMP_DIR/upstack-impact.md` if whole-stack, else empty |
| `{{PLANNING_CONTEXT}}` | Rendered planning_context from Phase 1 (via `~/.claude/skills/shared/planning-context.md`). Falls back to "No linked planning context available — reviewing against general code-quality heuristics only." when no epic resolves. |

If `DIFF_TEXT` exceeds ~300K characters, truncate per-file to the first 2000 lines of each file's diff and note the truncation in the prompt.

Save the substituted prompt to `$TMP_DIR/codex-prompt-substituted.md` for debugging.

## Step 2: Invoke Codex MCP

Call `mcp__codex__codex` with:

```json
{
  "prompt": "<substituted prompt>",
  "sandbox": "read-only",
  "approval-policy": "never",
  "cwd": "<repo root>"
}
```

Do NOT specify the `model` parameter - let the global Codex configuration pick the default model (per global rules).

## Step 3: Parse Codex response

Codex returns markdown containing a YAML block of findings (format defined in the prompt template and `cut-classification.md`). Extract the YAML between the ```yaml and ``` fences.

Parse and validate each finding has these required fields:
- `id` (sequential; reassign if missing)
- `file`
- `line`
- `classification` in {Slop, Acceptable, Borderline}
- `confidence` in {High, Medium, Low}
- `title`
- `description`
- `suggested_cut`

For `Slop` findings, `anti_pattern_tag` is required. For `Acceptable`, `acceptable_reason` is required.

If a finding is malformed, include it in a `validation-warnings.txt` file but drop it from the findings list - do not prompt the user on garbage data.

## Step 4: Save findings

Write valid findings to `$TMP_DIR/findings.yaml`:

```yaml
findings:
  - id: f-1
    file: pkg/auth/middleware.go
    line: 42
    # ...
```

Also write a terse summary to the user:

```
Codex grug pass: <N_slop> Slop, <N_acceptable> Acceptable, <N_borderline> Borderline.
Review details saved to <TMP_DIR>/findings.yaml.
```

## Step 5: Handle no-findings case

If `findings.yaml` contains zero entries (or only `Acceptable` entries):
- Print: "No slop detected. Branch looks clean from a grug perspective."
- Skip to Phase 8 and write a "no-op" report.
- Do not advance to Phase 4.

## Error Handling

| Failure | Action |
|---------|--------|
| Codex invocation times out | Retry once with a smaller prompt (per-file truncation to 1000 lines). If second attempt fails, abort with "Codex timed out; try a smaller scope (single branch or subset of files)." |
| Codex returns non-YAML output | Save raw response to `$TMP_DIR/codex-raw.txt`, abort with "Codex response not parseable; see $TMP_DIR/codex-raw.txt" |
| All findings are malformed | Abort; surface `validation-warnings.txt` |
| Codex MCP unavailable | Already caught in Phase 0; if it drops mid-run, abort cleanly |
