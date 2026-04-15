# Epic Resolution Algorithm

Resolve an epic ID into an ordered execution plan of implementable beads.

## Inputs

- `epic_id`: The epic bead ID (e.g., `bd-xxx`)
- All beads from `br list` + `br show` (batch) with parent fields
- Ready set from `br ready`

## Algorithm

### Step 1: Find All Descendants of the Epic

NOTE: `br dep tree` does NOT work for finding children. The dep tree command
uses `get_dependencies()` (outgoing deps from an issue) rather than
`get_dependents()` (incoming deps to an issue). Since parent-child deps are
stored as child->parent, querying from the parent returns nothing.

The `br show` command includes a `parent` field; `br list` does not. Use
`br show` with batch IDs to efficiently find all children.

```bash
# Step 1a: Get all bead IDs
br list --json --limit 0
# Extract id field from each item

# Step 1b: Batch fetch with parent fields
br show <id1> <id2> <id3> ... --json
# br show accepts multiple IDs in one call
# For large bead sets, process br show in batches to avoid
# argument length limits and reduce per-call response size.
```

Build the descendant set iteratively:
1. Initialize the set with just the epic ID
2. Scan all beads: if a bead's `parent` field matches any ID in the set, add it
3. Repeat step 2 until no new beads are added (fixed point)
4. Remove the epic itself from the final set

This matches atari's `buildDescendantSet()` at internal/workqueue/queue.go.

The `br show` output also includes `dependencies` and `dependents` arrays
with full metadata, which are needed for wave computation in Step 5.

### Step 2: Get Ready Beads

```bash
br ready --json --limit 0
```

Returns all beads with status=ready (unblocked, not deferred). The `--limit 0` flag
returns the full set without pagination.

### Step 3: Compute Intersection

Build the execution plan from beads that appear in BOTH:
1. The descendant set (from Step 1)
2. The ready set (from Step 2)

This intersection gives beads that belong to this epic AND are currently
unblocked and available for work.

```
descendant_ids = set(descendant_set.keys())
ready_ids = set(b.id for b in ready_set)
executable = [b for b in descendant_set.values() if b.id in ready_ids]
```

### Step 4: Classify All Descendants

Each bead in the descendant set (already fetched via br show in Step 1)
has full status information. Classify each:

- `status == "closed"` -> exclude from plan, record as "already done"
- `status == "in_progress"` -> warn user (may be claimed by another session)
- in ready set -> include in execution plan
- NOT in ready set and status open/deferred -> record as "blocked"

### Step 5: Compute Execution Waves from Blocking Dependencies

Since we can no longer use `depth` from `br dep tree`, compute waves from
the blocking dependency graph. Each bead from `br show` includes a
`dependencies` array with entries that have `dependency_type`.

Filter to `dependency_type == "blocks"` to find blocking deps (ignore
parent-child, which is structural not execution-ordering).

Algorithm:
```
1. For each executable bead, collect its blockers (dependency_type == "blocks")
   that are ALSO in the descendant set and not yet closed
2. Wave 1: beads with no open blockers within the epic
3. Wave 2: beads whose blockers are all in Wave 1
4. Wave N: beads whose blockers are all in earlier waves
5. Within each wave: sort by priority ascending
```

If no blocks dependencies exist (all beads are independent), put everything
in Wave 1 sorted by priority.

### Step 6: Handle Edge Cases

**No ready beads (all blocked)**:
Report blocked beads and their blocking dependencies. Use the batch-fetched
data from Step 1b (which includes `dependencies` and `dependents` arrays)
to identify what dependency is preventing each bead from becoming ready.
No additional `br show` calls are needed.

**Partially ready**:
Show which beads are ready (in the execution plan) and which are blocked.
Proceed with the ready beads - blocked beads will become ready as their
dependencies are completed.

**All done**:
If every descendant bead has status=closed, report:
"All beads in epic already closed - nothing to implement."

**In-progress beads**:
If any bead has status=in_progress, warn:
"bd-xxx is currently in_progress (may be claimed by another session).
Skipping to avoid duplicate work. If this is stale, reset it with
`br update bd-xxx --status open`."

### Step 7: Present Execution Plan

Display the plan to the user via assistant text:

```
Execution Plan for Epic: <epic title> (<epic_id>)

Wave 1:
  bd-xxx: Title (P2)
  bd-yyy: Title (P1)

Wave 2:
  bd-zzz: Title (P2)

Already completed: bd-aaa, bd-bbb
In progress (skipped): bd-ccc
Blocked (not ready): bd-ddd (blocked by bd-eee)

Total: N beads to implement across M waves.
Proceed?
```

### Step 8: Wait for User Confirmation

Use AskUserQuestion to get explicit confirmation before proceeding to
implementation phases:

```
Call AskUserQuestion tool with:
  questions: [{
    question: "Proceed with implementing N beads across M waves?",
    header: "Execute plan",
    options: [
      { label: "Proceed", description: "Start implementing beads in wave order" },
      { label: "Skip some", description: "Select specific beads to implement" },
      { label: "Cancel", description: "Abort without implementing anything" }
    ],
    multiSelect: false
  }]
```

If "Skip some": present each bead via AskUserQuestion for include/exclude.
Rebuild the plan with only included beads.

If "Cancel": exit the skill with no changes.
