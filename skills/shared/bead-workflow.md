# Shared Bead Workflow

Common procedures for verification discovery and bead creation across all issue-plan skills.

## Core Principle: All Beads Require an Epic

Every bead must belong to an epic before it is published. Standalone beads are not allowed.

Each planning skill invocation creates its own epic and links all beads under it.
This ensures organizational consistency across the tracker, enables the /implement
skill (which resolves work through epic descendants), and prevents orphaned beads
that are invisible to epic-level status reporting.

## Verification Command Discovery

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

## Create Issues (Deferred)

Create issues using `br create` with `--status deferred` to prevent atari from picking them up before planning is complete.

### Carry-forward principle

You just produced a rich plan through investigation (Explore, Codex debate, user interview, or all three). That investigation is the PRIMARY SOURCE for bead descriptions. When creating each bead, **copy the relevant Design / Files / Gotchas / Test Strategy content from the synthesis directly into the description**. Do not re-describe. Paste and adapt. The implementer executing this bead autonomously will see only what you write here — not the conversation history, not the synthesis document, not your Explore findings.

A bead description that reads like a one-paragraph ticket is a sign the carry-forward did not happen. Go back and move the relevant planning content into the description before moving on.

### Bead description template

Every bead description uses this structure. `## ` (double-hash) headings throughout — the implementer's parser depends on this.

**Required core sections** (always present):

```
## Goal
[One sentence — the outcome, not the task. What is true about the system after this bead is done.]

## Why
[2-4 sentences — the motivation. Link to the parent epic's goal if not obvious from Goal alone.]

## Design
[The technical approach, copied from the synthesis. For non-trivial choices, note alternatives considered and why this path won. This is the most important section — it is the plan, not a summary of the plan.]

## Files
[Concrete paths discovered during planning. Include line ranges where helpful as anchor points.]
- path/to/file.ext:45-120 — [why this file, what changes here]
- path/to/other.ext — [why]

## Acceptance Criteria
- [ ] [testable boolean condition — what is true when this bead is done]
- [ ] [...]

## Verification
- [ ] `[discovered lint command]` passes
- [ ] `[discovered static analysis command]` passes
- [ ] `[discovered test command]` passes
- [ ] `[discovered e2e command]` passes (if applicable)
```

**Optional extras** — include any that planning surfaced, omit the rest entirely (do not write empty headings):

```
## Patterns to Follow
[Existing conventions the implementer should mirror. Cite by file path with line numbers where possible — the implementer should read the real source, not a summary of it.]

## Test Strategy
[What tests to add or update, where. Distinguish unit / integration / regression. Name the test files explicitly.]

## Out of Scope
[Explicit "do not touch" boundaries. Often the highest-leverage section for autonomous execution — tells the implementer where to stop.]

## Gotchas
[Non-obvious constraints found during planning: race conditions, deprecated APIs, fragile dependencies, test environment quirks, edge cases the plan handles in a specific way.]

## Cross-bead Notes
[If this bead depends on or constrains siblings in the same epic, say so. Reference sibling bead IDs where known.]
```

Also include this closing note at the end of every description:

> If implementation reveals new issues, create separate issues for investigation.

### Size signal

Target description length: **250-1000 words** for a typical bead. A one-paragraph description means planning was incomplete — go back and investigate before creating. Beads much over 1000 words are a sign this should be split into sibling beads.

### Example invocation

```bash
br create "Title" --status deferred --description "$(cat <<'BEAD_DESC_EOF'
## Goal
...

## Why
...

## Design
...

## Files
- ...

## Acceptance Criteria
- [ ] ...

## Verification
- [ ] `...` passes

If implementation reveals new issues, create separate issues for investigation.
BEAD_DESC_EOF
)" --json
# Track the returned ID for later publishing
```

**Track all created issue IDs** for the publish step.

### Optional: evidence-preservation comments

Planning often produces rich artifacts that should outlive the planning session but would bloat a description. Attach them as comments on the relevant issue — per-bead for bead-specific evidence, on the epic for cross-epic evidence that applies to every child.

**Per-bead artifacts** (attach to the bead): raw Explore findings with file paths and line numbers; interview answers that clarified one specific design choice; reproductions for a bug being tracked.

```bash
br comments add <bead-id> --file /tmp/<bead-id>-investigation.md
```

**Epic-level artifacts** (attach to the epic): full Codex debate transcripts; interview recordings covering the whole plan; architecture sketches that span multiple beads. See the skill-specific instructions in `issue-plan/SKILL.md` and `issue-plan-codex/SKILL.md` for the debate-transcript pattern.

```bash
br comments add <epic-id> --file /tmp/<epic-id>-debate.md
```

Write the artifact to a temp file first, then attach. Skip this step when the evidence would not meaningfully help a future reader — not every bead needs an investigation trace, and a trivial one-round debate does not need a transcript. Attach whenever the raw evidence captures reasoning that the distilled description cannot.

## Final Verification Issue (Deferred)

After creating all implementation issues, create one final issue to run the full test suite:

1. **Create the issue** with deferred status:
   ```bash
   br create "Run full test suite for [feature] (final verification)" --status deferred --description "..." --json
   ```
   - Description: Verify all changes work together by running the complete test suite
   - Include the discovered e2e/integration command
   - Acceptance criteria: All tests pass, no regressions introduced

2. **Set up dependencies**:
   Use `br dep add <issue> <depends-on> --type blocks` where issue depends on depends-on.
   The first argument is the issue that WAITS, the second is the issue that must complete first.

Example:
```bash
# If implementation issues are bd-001, bd-002, bd-003 and final verification is bd-004:
# bd-004 (final) depends on each implementation issue:
br dep add bd-004 bd-001 --type blocks
br dep add bd-004 bd-002 --type blocks
br dep add bd-004 bd-003 --type blocks
# Sequential chain: bd-002 depends on bd-001, bd-003 depends on bd-002:
br dep add bd-002 bd-001 --type blocks
br dep add bd-003 bd-002 --type blocks
```

## Create Epic

After all issues are created and dependencies set, create an epic as a summary of the planned work.

**Epic Priority and Selection Mode**: When atari uses `selection_mode: top-level` (the default), epics compete by priority. The epic with the **lowest priority number** (highest priority) gets all its work done first before moving to the next epic. Set epic priority based on when you want this work completed relative to other epics:
- P0-P1: Urgent work that should be done before other planned work
- P2 (default): Normal priority, processed in creation order among equals
- P3-P4: Lower priority, will be worked after higher-priority epics complete

Use `## ` (double-hash) headings throughout — `/implement` extracts the epic's `## Design Decisions` section by regex and surfaces it in the implementer's prompt.

```bash
br create "[feature/task name]" --type epic --priority <N> --description "$(cat <<'EOF'
## Overview
[Brief description of the overall work being planned]

## Scope
[What this epic covers]

## Implementation Issues
- bd-xxx: [issue title]
- bd-xxx: [issue title]
- bd-xxx: Run full E2E/integration test suite (final verification)

## Verification Commands
- Lint: `[discovered lint command]`
- Static analysis: `[discovered static analysis command]`
- Tests: `[discovered test command]`
- E2E: `[discovered e2e command]`

## Design Decisions
[Document key design decisions and tradeoffs from planning. For each decision:
what was chosen, what alternatives were considered, and why this approach won.
Focus on choices where a reasonable person might have decided differently.
This section feeds directly into the PR description AND is surfaced in every
child bead's implementer prompt — write it for both audiences.]

## Success Criteria
All implementation issues closed and E2E verification passes.
EOF
)" --json
```

Link all created issues to the epic as children (child depends on parent):
```bash
br dep add bd-xxx <epic-id> --type parent-child
# ... repeat for each implementation issue
```

Check epic progress: `br epic status`

## Pre-publish self-check

Before transitioning beads from deferred to open, read each bead description and honestly answer these four questions:

1. Could a fresh implementer with no conversation context tell **which files to touch**?
2. Could they tell **what "done" looks like** (concrete, testable)?
3. Could they tell **what NOT to touch** — the scope boundaries?
4. Could they tell **which existing patterns to mirror**, with specific file references?

If any answer is "no," revise the description before publishing. These four capabilities are what separate a plan-mode-quality bead from a ticket stub. The carry-forward from your synthesis should already have addressed them — if it did not, something was dropped between planning and creation.

Run this check per-bead, not per-epic. A well-described epic with thin children still produces stuck implementers.

## Publish All Beads

After the epic is created, all dependencies are set, AND the pre-publish self-check passes, publish all beads by transitioning them from deferred to open status. This makes them available to `br ready` and atari.

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
- The pre-publish self-check above has passed for every bead

This ensures atari will not pick up any beads until the entire plan is ready and properly sequenced.
