---
name: issue-plan-codex
description: Plan issues using autonomous AI debate between Claude (opus) and Codex. Heavyweight and thorough - best for complex architectural work, large refactors, or ambiguous problems that need deep analysis. Use when the work is too important for the lightweight issue-plan or when you need maximum rigor in discovery and planning without user involvement.
---

# Planning Issues (Codex)

Plan and create issues for complex work requiring thorough discovery and multi-round collaborative debate.

## Context Sources

This command receives context from two sources:
1. **Conversation history** - All messages above inform requirements, decisions, and scope
2. **Arguments** - Additional instructions passed when invoking the command (see $ARGUMENTS at end)

## Tools Used

- **Task (Explore subagent)** - Thorough codebase exploration (inherits global model)
- **Task (Plan subagent)** - Implementation design (inherits global model)
- **mcp__codex__codex** - Cross-reference discovery (uses Codex global default)
- **br CLI** - Issue creation, status management, and dependencies

---

## Overview

This is a two-phase process: discovery first, then planning with collaborative debate.

## Phase 1: Discovery

Use BOTH approaches for comprehensive discovery:

### Claude Explore Agents
Use the Explore subagent with "very thorough" setting to understand:
1. All code related to this work (run up to 3 parallel explorations)
2. Current architecture, patterns, and conventions

### Verification Command Discovery

Run a focused Explore query using the verification command discovery process in `../shared/bead-workflow.md` (see "Verification Command Discovery" section).

### Codex Discovery
Use the codex MCP tool for additional discovery:
```
mcp__codex__codex
prompt: "Explore [topic]. Find all relevant code, patterns, edge cases, and potential issues. Report findings comprehensively."
```
Cross-reference Codex findings with Explore results to ensure nothing is missed.

## Phase 1.5: Discovery Synthesis

Before planning, consolidate findings into a brief summary:
- **Architecture overview**: Key patterns, conventions, and constraints discovered
- **Testing setup**: Where tests live, how to run them, what coverage exists
- **Verification commands**: Exact commands for lint, static analysis, test, e2e (from discovery)
- **Known risks**: Edge cases, gotchas, or blockers identified during discovery

This summary becomes the input for Phase 2.

## Phase 2: Planning with Collaborative Debate

Use multi-round refinement for thorough planning:

### Step 1: Initial Plan
Use the Plan subagent to design implementation approach based on discovery synthesis.

### Step 2: Collaborative Debate (2-4 rounds, until consensus or escalation)
Claude (Opus) and Codex debate back-and-forth to refine the plan:

**Round 1 - Dual Critique**:
- **Claude (Opus)**: List 5-10 specific gaps, risks, or edge cases in the plan. For each, explain why it matters.
- **Codex**: Use `mcp__codex__codex`:
  ```
  prompt: "Review this implementation plan: [plan]. List 5-10 specific gaps, conflicts, or risks. For each issue: (1) What could break? (2) What assumption might be wrong? (3) Suggest a concrete mitigation."
  ```
- Synthesize both critiques. If >3 critical issues overlap, they are high-priority fixes.

**Round 2 - Address & Counter**:
- **Claude (Opus)**: Propose specific revisions for each Round 1 concern. State which you accept, reject (with rationale), or defer.
- **Codex**: Use `mcp__codex__codex`:
  ```
  prompt: "Claude proposes these revisions: [revisions]. For each: (1) Does it actually solve the concern? (2) What breaks if Claude's assumption is wrong? (3) Suggest 1-2 concrete alternatives for weak points."
  ```
- Integrate valid counterpoints. If fundamental disagreement on architecture, pause and re-examine discovery findings.

**Round 3 - Final Consensus** (skip if Round 2 achieved consensus):
- **Claude (Opus)**: Present refined plan with all incorporated feedback. List any unresolved disagreements.
- **Codex**: Use `mcp__codex__codex`:
  ```
  prompt: "Final plan review: [plan]. Verify: (1) All discovered edge cases addressed or explicitly deferred? (2) Error/failure paths defined? (3) Testing strategy clear? (4) Dependencies sequenced correctly? List any gaps."
  ```
- If consensus: Proceed. If disagreement on implementation detail: Choose simpler/safer option, note as future optimization.

**Round 4 - Escalation** (only if Round 3 has unresolved critical issues):
- Re-examine discovery findings to identify which assumptions caused the conflict.
- Choose the approach with fewer unknowns. Document the trade-off explicitly.

### Quality Gate
Before creating issues, confirm:
- [ ] All discovered edge cases addressed or explicitly deferred with rationale
- [ ] Error paths defined (what happens when X fails?)
- [ ] Testing strategy covers new code
- [ ] Design decisions documented with reasoning (feeds into epic and PR description)

### Create Beads

Read and follow the bead creation process in `../shared/bead-workflow.md`. This covers:
1. **Create Issues (Deferred)** - Create implementation beads with acceptance criteria
2. **Final Verification Issue** - Create a gating issue that depends on all implementation beads
3. **Create Epic** - Summarize the planned work as an epic with all children linked
4. **Publish All Beads** - Transition from deferred to open once the dependency graph is complete

### Attach Debate Transcript

After the epic exists (its ID comes back from the Create Epic step), preserve the full debate as a comment on the epic. The conclusions landed in the epic's `## Design Decisions`, but the arguments and counterarguments — *why* specific tradeoffs were accepted or rejected — are the reasoning future readers (implementers, reviewers, a later planner revisiting this work) need when judging whether the chosen approach still holds under new context.

Compose the transcript from the debate rounds above, using this structure. Omit rounds that did not run:

```
# Planning Debate Transcript — <EPIC_ID>

## Round 1: Dual Critique

### Claude's critique
[verbatim list from Round 1]

### Codex response
[verbatim from mcp__codex__codex output]

### Synthesis
[what overlapped, what was accepted, what was deferred]

## Round 2: Address & Counter
[same shape: Claude's proposed revisions, Codex's counter, synthesis]

## Round 3: Final Consensus
[Claude's refined plan, Codex's validation, any unresolved disagreements]

## Round 4: Escalation
[only if triggered — the assumption that caused the conflict and the safer path chosen]

## Final consensus
[what was agreed; any points left unresolved and the reason for leaving them]
```

Write the transcript to `/tmp/<EPIC_ID>-debate.md` and attach:

```bash
br comments add <EPIC_ID> --file /tmp/<EPIC_ID>-debate.md
```

Skip this step only when the debate was trivial (one round, immediate agreement). Whenever there was real back-and-forth — disputed edge cases, rejected alternatives, assumptions that had to be re-examined — attach. A 4-8K-word transcript is typical and manageable as a comment.

## Output Summary

After creating and publishing beads, output a clear summary:

```
Created X bead(s) from AI debate planning (opus + codex):

- bd-xxx: [title] (P2, open)
- bd-xxx: [title] (P2, open, blocked by bd-xxx)
- bd-xxx: [title] - final verification (P2, open, blocked by all above)

Epic: bd-xxx - [epic title]

Design decisions from debate:
- [Decision 1] - Chose X over Y because Z
- [Decision 2] - Deferred Z for future optimization

Ready for implementation.
```

## Handling Failures

When discovery or planning reveals blocking issues:
1. Create a P0 meta issue titled: "Create plan for [blocker-topic]"
2. Description must include:
   - What was blocking and why it matters
   - Instruction to use Explore subagent for discovery
   - Instruction to use Plan subagent to design fix
   - Instruction to create implementation issues via issue-tracking skill
3. Any implementation issues spawned from meta issues are also P0

---

## Additional Instructions

$ARGUMENTS
