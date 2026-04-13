# Bead Implementation Procedure

Per-bead claim, load, implement, verify, close/skip cycle for the /implement skill.

## Implementation Mode Decision

The lead agent decides implementation mode based on epic size:

- **Inline** (5 or fewer beads in execution plan): the lead agent implements each
  bead directly using Read, Edit, Write, Bash, Grep, Glob tools. No subagents.
- **Subagent** (6+ beads): spawn one Agent per bead. The lead reads summary files
  between beads to maintain cross-bead context.

This threshold balances context window pressure against coordination overhead.
Small epics fit comfortably in a single context. Large epics benefit from
isolated contexts per bead plus summary-based continuity.

## Per-Bead Cycle

### Step 1: Claim

```bash
br update <BEAD_ID> --status in_progress --json
```

This prevents duplicate work if another session picks up the same bead.

### Step 2: Load Context

```bash
br show <BEAD_ID> --json
```

Extract from the JSON:
- `description` - full bead description with acceptance criteria
- `dependencies` - any upstream beads (for context on what was already done)

Parse the `## Verification` section from the description to extract verification
commands. These are typically formatted as:
```
## Verification
- [ ] `command1` passes
- [ ] `command2` passes
```

Extract the backtick-enclosed commands into a list for Step 4.

### Step 3: Implement

#### Inline Mode (small epics)

The lead agent implements directly:

1. Read the bead description carefully
2. Use Read, Edit, Write, Bash, Grep, Glob tools to implement
3. Follow existing project patterns (check CLAUDE.md, existing code)
4. Stay focused on the bead's scope - do not expand beyond acceptance criteria

#### Subagent Mode (large epics)

Spawn a foreground agent per bead with a 10-minute timeout:

```
Agent(
  description: "Implement <BEAD_ID>",
  prompt: <rendered bead-prompt.md template>,
  mode: "bypassPermissions"
)
```

If the agent has not completed after 10 minutes of wall time, treat it as a
timeout: reset the bead to open status and record as skipped.

Template variables to substitute in templates/bead-prompt.md:
- `BEAD_ID` - the bead identifier
- `BEAD_TITLE` - short title from bead
- `BEAD_DESCRIPTION` - full description text
- `BEAD_PARENT` - parent epic ID (for discovery issue linking)
- `WORKTREE_PATH` - absolute path to the worktree
- `VERIFICATION_COMMANDS` - extracted verification commands from description
- `PRIOR_SUMMARIES` - concatenated summaries from previously completed beads

After the agent completes, read its summary file:
```
/tmp/implement-<BRANCH_NAME>/<BEAD_ID>-summary.md
```

Accumulate summaries for PRIOR_SUMMARIES in subsequent bead prompts.

### Step 4: Verify

Run each verification command extracted in Step 2.

**If all pass**: proceed to Step 5 (close).

**If any fail**: attempt to fix the issue.
- In inline mode: fix directly
- In subagent mode: the agent should have already attempted fixes; if the lead
  finds failures, attempt one targeted fix

Then re-run ALL verification commands once more.

**If still failing after retry**: proceed to Step 6 (handle failure).

### Step 5: Close on Success

```bash
br close <BEAD_ID> --reason "<brief summary of what was done>"
```

The reason should be a concise description of the implementation, not a list
of files changed. The diff captures the what; the reason captures the why.

### Step 6: Handle Failure

```bash
br update <BEAD_ID> --status open --notes "SKIPPED: <what failed and why>"
```

Record the bead as skipped in the execution tracking. Include:
- Which verification command(s) failed
- What was attempted to fix it
- Why the retry also failed

Continue to the next bead. Skipped beads do not block subsequent waves unless
they have explicit blocks dependencies on later beads.

## Cross-Bead Context (Subagent Mode)

When using subagents, the lead maintains continuity through summary files:

1. Each subagent writes a summary to /tmp/implement-<BRANCH_NAME>/<BEAD_ID>-summary.md
2. Summary format:
   - Files created or modified (paths only)
   - What the bead accomplished (one paragraph)
   - Discoveries or follow-up needed
3. Before spawning the next subagent, the lead reads ALL prior summaries
4. Prior summaries are injected into the next bead prompt via PRIOR_SUMMARIES

This preserves cross-bead awareness without holding full implementation context
in any single agent window.

## Error Recovery

| Scenario | Recovery |
|----------|----------|
| br update fails (bead already claimed) | Warn user, skip bead |
| Subagent crashes or times out | Reset bead to open, record as skipped |
| Verification command not found | Skip that check, warn in output |
| All beads in a wave skipped | Log warning, proceed to next wave |
| Git conflicts in worktree | Stop execution, report to user |
