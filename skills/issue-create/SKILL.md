---
name: issue-create
description: Quickly create a single issue from conversation context. Use when you want to create a bead, file an issue, track this as a bead, make a ticket for something, or capture a work item. Lightweight alternative to issue-plan for one piece of work without full planning.
---

# Quick Bead Creation

Create a single issue from the current conversation context. This is a lightweight alternative to issue-plan for when you need to quickly capture one piece of work.

The skill uses a two-phase approach: ideation happens on the main thread (extracting requirements, clarifying with the user), then a subagent handles the mechanical creation work (codebase exploration, formatting, br commands). This keeps the main conversation focused on what matters.

## Context Sources

This command receives context from two sources:
1. **Conversation history** - All messages above inform requirements, decisions, and scope
2. **Arguments** - Additional instructions passed when invoking the command (see $ARGUMENTS at end)

---

## Phase 1: Ideation (Main Thread)

### Step 1: Extract from Conversation

Review the conversation history above to identify:
- **What**: The specific task or bug to address
- **Why**: The motivation or problem being solved
- **Scope**: What's included and excluded
- **Acceptance criteria**: How to know when it's done
- **Known files**: Any files already discussed or identified in conversation

If the user provided additional instructions below, incorporate those as well.

### Step 2: Clarify Ambiguities

Before handing off to the subagent, use AskUserQuestion to clarify anything uncertain. Infer as much as possible from conversation context, but ask about:

**Always infer from context, only ask if unclear:**
- Priority (default: P2/normal) - infer from urgency words like "critical", "blocking", "minor", "nice to have"
- Scope boundaries - if the task could reasonably be interpreted multiple ways

**Only ask if mentioned in conversation:**
- Dependencies - if other beads were discussed as related/blocking
- Implementation approach - if multiple valid approaches were discussed

Use quick dropdown options rather than open-ended questions. Example:
```
questions:
  - question: "What priority should this bead have?"
    header: "Priority"
    options:
      - label: "P2 - Normal (Recommended)"
        description: "Standard priority for most tasks"
      - label: "P1 - High"
        description: "Important, should be done soon"
      - label: "P0 - Critical"
        description: "Blocking issue, needs immediate attention"
      - label: "P3 - Low"
        description: "Nice to have, can wait"
```

### Step 3: Assemble Brief and Dispatch Subagent

Once requirements and user preferences are clear, assemble a creation brief and spawn a general-purpose subagent. The brief must be self-contained - the subagent has no access to conversation history.

Spawn the subagent with a prompt like:

```
Create a bead (issue) using the br CLI. Here is everything you need:

## Brief
- Title: [50 chars max, imperative voice]
- Priority: [P0-P4]
- What/Why: [1-4 sentences]
- Acceptance Criteria:
  - [criterion 1]
  - [criterion 2]
- Known Files: [files from conversation, or "none - discover from codebase"]
- Working Directory: [pwd]

## Your Tasks

1. **Discover verification commands** - Search in order:
   - mise.toml / .mise.toml (mise task runner)
   - package.json scripts / pyproject.toml / Makefile / Justfile
   - .github/workflows (CI jobs are authoritative)
   Report exact commands for: linting, tests, type checking, integration/e2e tests.

2. **Discover relevant files** (only if Known Files says "none" or is incomplete) -
   Find files that will need modification and related test files.

3. **Create the bead** using br create (deferred status until epic is linked).
   Use `## ` (double-hash) headings throughout — the /implement parser depends on this level.
   Required core sections: Goal, Why, Files, Acceptance Criteria, Verification.
   Optional extras when the brief or discovery surfaced the content: Design, Gotchas, Out of Scope.
   Omit optional sections entirely when empty — do not write empty headings.

   br create 'Title' --priority <N> --status deferred --description "$(cat <<'BEAD_DESC_EOF'
   ## Goal
   [One sentence — the outcome, not the task]

   ## Why
   [2-4 sentences from the brief — motivation and context]

   ## Files
   - path/to/file.ext:LINE-RANGE — [why this file, what changes here]

   ## Acceptance Criteria
   - [ ] [criterion from brief]

   ## Verification
   - [ ] `[discovered lint command]` passes
   - [ ] `[discovered test command]` passes

   If implementation reveals new issues, create separate issues for investigation.
   BEAD_DESC_EOF
   )" --json

4. **Create wrapping epic and link** (every bead gets its own epic).
   Use `## ` (double-hash) headings — `/implement` extracts the epic's
   `## Design Decisions` by regex if present, and single-hash headers will
   not match.
   ```bash
   br create '[topic name]' --type epic --priority <same as bead> --description "$(cat <<'BEAD_DESC_EOF'
   ## Overview
   Wrapping epic for [brief topic description].

   ## Scope
   [One sentence describing what this epic covers]

   ## Implementation Issues
   (linked below)

   ## Success Criteria
   All linked beads completed.
   BEAD_DESC_EOF
   )" --json
   # Extract epic ID from output, then link and publish:
   br dep add <new-bead-id> <epic-id> --type parent-child
   br update <new-bead-id> --status open
   ```

5. **Return a summary** in this format:
   Created: <bead-id>
   Title: <title>
   Priority: P<n>
   Status: open
   Description: <2-3 sentence summary>
   Linked to epic: <epic-id>

6. **Verify creation** - Run `br show <new-bead-id> --json` and confirm the
   title matches what was intended and the description is not truncated
   (heredoc edge cases can silently corrupt long descriptions). If the
   description appears truncated or the title does not match, add a warning
   line to the summary: `Warning: bead may have been created with truncated description`
```

### Step 4: Report Result

When the subagent returns, relay its summary to the user. If the bead description is short, show the full description instead of a summary.

---

## Example

**Conversation context:**
> User: "The br CLI panics when you pass an empty string to --title. This is
> blocking my workflow - I hit it twice today. The create command should
> validate the title before writing to the database."

**Resulting bead:**
- Title: `Validate title is non-empty in br create`
- Priority: P1 (user explicitly said "blocking my workflow")
- Description includes: reproduction steps, relevant file (`cli/commands/create.rs`),
  acceptance criteria (empty title returns error instead of panicking),
  verification commands (`cargo test`, `cargo clippy`)

---

## Additional Instructions

$ARGUMENTS
