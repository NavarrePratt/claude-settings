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

**Always ask:**
- Epic linkage - "Should this be linked to an existing epic?" (provide list if epics exist)

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
- Epic Linkage: [epic ID, or "none"]
- Working Directory: [pwd]

## Your Tasks

1. **Discover verification commands** - Search in order:
   - mise.toml / .mise.toml (mise task runner)
   - package.json scripts / pyproject.toml / Makefile / Justfile
   - .github/workflows (CI jobs are authoritative)
   Report exact commands for: linting, tests, type checking, integration/e2e tests.

2. **Discover relevant files** (only if Known Files says "none" or is incomplete) -
   Find files that will need modification and related test files.

3. **Create the bead** using br create:
   br create "Title" --priority <N> --description "$(cat <<'EOF'
   # Description
   [What and why from brief]

   # Relevant Files
   - path/to/file - [reason]

   # Acceptance Criteria
   - [ ] [criterion from brief]

   # Verification
   - [ ] `[discovered lint command]` passes
   - [ ] `[discovered test command]` passes

   If implementation reveals new issues, create separate issues for investigation.
   EOF
   )" --json

4. **Epic linkage** (if epic ID provided):
   br dep add <new-bead-id> <epic-id> --type parent-child

5. **Return a summary** in this exact format:
   Created: <bead-id>
   Title: <title>
   Priority: P<n>
   Status: open
   Description: <2-3 sentence summary>
   Linked to epic: <epic-id or "None">
```

### Step 4: Report Result

When the subagent returns, relay its summary to the user. If the bead description is short, show the full description instead of a summary.

---

## Additional Instructions

$ARGUMENTS
