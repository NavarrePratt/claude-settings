---
name: issue-plan-user
description: Plan issues by interviewing the user with probing questions. Best when you want direct user input to refine requirements before implementation.
---

# User Interview Planning

Plan work by interviewing the user in-depth about their plans using probing, non-obvious questions.

## Context Sources

This command receives context from two sources:
1. **Conversation history** - All messages above inform requirements, decisions, and scope
2. **Arguments** - Additional instructions passed when invoking the command (see $ARGUMENTS at end)

## Tools Used

- **AskUserQuestion** - Primary tool for in-depth user interviews
- **Task (Explore subagent)** - Codebase exploration to inform questions
- **br CLI** - Issue creation after planning is complete

---

## Overview

This is an interactive planning process where you interview the user to fully flesh out their plan before creating beads. Unlike issue-plan and issue-plan-codex which use AI-to-AI debate, this command uses direct user dialogue to refine requirements.

## Step 1: Understand the Plan

Review the entire conversation to understand what plan is being discussed. Identify:
- The core goal or feature being planned
- Any constraints or requirements already mentioned
- Technical context from prior discussion
- Open questions or unclear areas

## Step 2: Initial Exploration (Optional)

If the plan involves code changes and you need context to ask better questions, run a **quick** Explore query (model: "haiku") to understand:
- Relevant existing code and patterns
- How similar features are implemented
- Potential integration points

Skip this step if the conversation already provides sufficient technical context.

## Step 3: Interview the User

Interview the user about this plan in detail using the AskUserQuestion tool. Probe across multiple dimensions:

**Technical implementation:**
- How should this integrate with existing code?
- What patterns or conventions should it follow?
- Are there performance considerations?

**UI and UX (if applicable):**
- How should users interact with this?
- What feedback should the system provide?
- What happens in error states?

**Scope and boundaries:**
- What is explicitly out of scope?
- Are there phases or increments to consider?
- What is the minimum viable implementation?

**Edge cases and assumptions:**
- What inputs or states could cause problems?
- What assumptions are we making about the environment?
- How should the system behave in unexpected situations?

**Risks and dependencies:**
- What could go wrong?
- What does this depend on?
- What other work might be affected?

**Testing and verification:**
- How will we know this works correctly?
- What should be tested manually vs automatically?
- Are there integration concerns?

### Interview Guidelines

- **Ask non-obvious questions** - Probe deeper into things the user might not have considered
- **Challenge assumptions** - Question unstated beliefs about how things should work
- **Ask about the hard parts** - Focus on areas that seem complex or risky
- **Use multiple rounds** - Continue interviewing until the plan is fully fleshed out
- **Be very in-depth** - This is not a superficial checklist; dig into specifics

### Using AskUserQuestion Effectively

Structure questions with clear options when possible:
```
questions:
  - question: "How should errors be surfaced to the user?"
    header: "Error UX"
    options:
      - label: "Toast notification"
        description: "Non-blocking notification that auto-dismisses"
      - label: "Modal dialog"
        description: "Blocking dialog requiring user acknowledgment"
      - label: "Inline error"
        description: "Error displayed next to the relevant input"
      - label: "Status bar"
        description: "Persistent status area showing current state"
    multiSelect: false
```

For open-ended questions, provide example answers as options with "Other" for custom input.

## Step 4: Synthesize the Plan

After the interview is complete, synthesize everything discussed into a comprehensive plan:

1. **Summary** - What we are building and why
2. **Scope** - What is included and excluded
3. **Technical approach** - How it will be implemented
4. **Key decisions** - Important choices made during the interview
5. **Edge cases addressed** - How we handle unusual situations
6. **Testing strategy** - How we verify correctness
7. **Risks and mitigations** - What could go wrong and how we prevent it

Present this synthesis to the user and confirm it accurately captures the plan.

## Step 5: Discover Verification Commands

### Discover Verification Commands

Run a focused Explore query to find exact development commands:
```
Find the ACTUAL commands used in this project for verification. Search in order:
1. mise.toml / .mise.toml (mise task runner - https://github.com/jdx/mise)
2. package.json scripts / pyproject.toml / Makefile / Justfile
3. .github/workflows (CI jobs are authoritative)
4. docs/CONTRIBUTING.md or README.md

For each category, report the EXACT command string:
- Linting/formatting:
  - Task runners: `mise run lint`, `make lint`, `just lint`
  - Python: `ruff check .`, `ruff format --check .`, `black --check .`, `flake8`, `isort --check-only .`
  - Go: `golangci-lint run`
  - JS/TS: `npm run lint`, `eslint .`
- Static analysis / type checking:
  - Task runners: `mise run check`, `mise run typecheck`, `make typecheck`
  - Python: `mypy .`, `mypy src/`, `pyright`, `basedpyright`
  - Go: `staticcheck ./...`, `go vet ./...`
  - JS/TS: `npm run typecheck`, `tsc --noEmit`
- Unit tests:
  - Task runners: `mise run test`, `make test`, `just test`
  - Python: `pytest`, `pytest tests/unit/`, `pytest -v`, `python -m pytest`
  - Go: `go test ./...`, `go test -v ./...`
  - JS/TS: `npm run test`, `jest`, `vitest`
- Integration/E2E tests:
  - Task runners: `mise run test:e2e`, `mise run test:integration`, `make integration`
  - Python: `pytest tests/e2e/`, `pytest tests/integration/`, `pytest -m integration`, `pytest -m e2e`
  - Go: `go test -tags=integration ./...`
  - JS/TS: `npm run test:e2e`, `playwright test`

Output format: "CATEGORY: [exact command]"
Stop searching a category once you find an authoritative source.
```

## Step 6: Create Beads

### Create Issues (Deferred)

Create issues using `br create` with `--status deferred` to prevent atari from picking them up before planning is complete.

For each issue:
```bash
br create "Title" --status deferred --description "..." --json
# Track the IDs for later publishing
```

Each issue must:
1. Have clear acceptance criteria (what success looks like)
2. Be scoped to complete in one session
3. End with verification notes using **discovered commands** (not generic phrases):
   ```
   ## Verification
   - [ ] `[discovered lint command]` passes
   - [ ] `[discovered static analysis command]` passes
   - [ ] `[discovered test command]` passes
   - [ ] `[discovered e2e command]` passes (if applicable)
   ```
   Use exact commands from Phase 1 discovery. Omit categories if no command exists.
4. Include note: "If implementation reveals new issues, create separate issues for investigation"

**Track all created issue IDs** for the publish step.

### Final Verification Issue (Deferred)

After creating all implementation issues, create one final issue to run the full test suite:

1. **Create the issue** with deferred status:
   ```bash
   br create "Run full test suite for [feature] (final verification)" --status deferred --description "..." --json
   ```
   - Description: Verify all changes work together by running the complete test suite
   - Include the discovered e2e/integration command from Phase 1
   - Acceptance criteria: All tests pass, no regressions introduced

2. **Set up dependencies**:
   Use `br dep add <final-issue> <implementation-issue> --type blocks` for EACH implementation issue.
   This ensures the final verification runs only after all implementation work is complete.

Example:
```bash
# If implementation issues are bd-001, bd-002, bd-003 and final is bd-004:
br dep add bd-004 bd-001 --type blocks
br dep add bd-004 bd-002 --type blocks
br dep add bd-004 bd-003 --type blocks
```

### Create Epic

After all issues are created and dependencies set, create an epic as a summary of the planned work.

**Epic Priority and Selection Mode**: When atari uses `selection_mode: top-level` (the default), epics compete by priority. The epic with the **lowest priority number** (highest priority) gets all its work done first before moving to the next epic. Set epic priority based on when you want this work completed relative to other epics:
- P0-P1: Urgent work that should be done before other planned work
- P2 (default): Normal priority, processed in creation order among equals
- P3-P4: Lower priority, will be worked after higher-priority epics complete

```bash
br create "[feature/task name]" --type epic --priority <N> --description "$(cat <<'EOF'
# Overview
[Brief description of the overall work being planned]

# Scope
[What this epic covers]

# Implementation Issues
- bd-xxx: [issue title]
- bd-xxx: [issue title]
- bd-xxx: Run full E2E/integration test suite (final verification)

# Verification Commands
- Lint: `[discovered lint command]`
- Static analysis: `[discovered static analysis command]`
- Tests: `[discovered test command]`
- E2E: `[discovered e2e command]`

# Key Trade-offs
[Document major trade-offs from collaborative debate]

# Success Criteria
All implementation issues closed and E2E verification passes.
EOF
)" --json
```

Link all created issues to the epic as children:
```bash
br dep add bd-xxx <epic-id> --type parent-child
# ... repeat for each implementation issue
```

Check epic progress: `br epic status`

### Publish All Beads

After the epic is created and all dependencies are set, publish all beads by transitioning them from deferred to open status. This makes them available to `br ready` and atari.

```bash
# Publish all deferred beads created during this planning session
for id in $all_bead_ids; do
  br update $id --status open
done
```

**Important**: Only publish after:
- All implementation issues are created (deferred)
- All dependencies are set up
- Epic is created and children linked
- You have verified the dependency graph is correct

This ensures atari will not pick up any beads until the entire plan is ready and properly sequenced.

## Step 7: Output Summary

After creating and publishing beads, output a clear summary:

```
Created X bead(s) from user interview:

- bd-xxx: [title] (P2, open)
- bd-xxx: [title] (P2, open, blocked by bd-xxx)
- bd-xxx: [title] - final verification (P2, open, blocked by all above)

Epic: bd-xxx - [epic title]

Key decisions from interview:
- [Decision 1]
- [Decision 2]

Ready for implementation.
```

---

## Additional Instructions

$ARGUMENTS
