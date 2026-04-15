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
   Use exact commands from discovery. Omit categories if no command exists.
4. Include note: "If implementation reveals new issues, create separate issues for investigation"

**Track all created issue IDs** for the publish step.

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

# Design Decisions
[Document key design decisions and tradeoffs from planning. For each decision:
what was chosen, what alternatives were considered, and why this approach won.
Focus on choices where a reasonable person might have decided differently.
This section feeds directly into the PR description - write it for reviewers.]

# Success Criteria
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

## Publish All Beads

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
