# Shared: Planning Context Loading

Reusable procedure for resolving an epic ID and loading its planning context
into a review or fix skill. Produces the `PLANNING_CONTEXT` placeholder value
consumed by reviewer prompt templates (and the equivalent block in skills
without templates).

Review skills render the result into a `## Planning Context` block immediately
after the PR Description (branch reviews) or Commit Context (commit review)
section, before any teammate roster or review brief. This block exists so
reviewers see intentional design choices from planning instead of redisputing
them from the diff.

## Epic ID Resolution

Check sources in order; first hit wins:

1. **Explicit argument**: If the skill was invoked with `--epic <EPIC_ID>`,
   use that value directly. No validation beyond a non-empty string — `br
   show` will surface a clear error if the ID is wrong.

2. **Branch name parse**: Branches created by `/implement` are named
   `<epic-id>-<slug>` (see commit `382f0dd Rename /implement PR artifacts
   with epic ID and branch slug`). Extract the prefix with:

   ```bash
   BRANCH=$(git branch --show-current)
   # Match the leading bead ID (bd- prefix followed by hex or short
   # identifier). Stop at the first hyphen after the ID segment.
   EPIC_ID=$(echo "$BRANCH" | sed -nE 's/^(bd-[a-zA-Z0-9]+).*/\1/p')
   ```

   If `EPIC_ID` is non-empty, verify it exists:

   ```bash
   br show "$EPIC_ID" --json > /dev/null 2>&1 && echo "resolved"
   ```

   If verification fails (not a real epic, or `br` is unavailable in this
   context), treat as no epic found and fall through.

3. **No epic**: Render the fallback described below. Do not error out — a
   reviewer with no planning context is still useful, just less targeted.

## Design Decisions Extraction

Once `EPIC_ID` resolves, fetch the description and extract the
`## Design Decisions` section using the same regex pattern used elsewhere
(`## Verification`, `## Discoveries`):

```bash
EPIC_JSON=$(br show "$EPIC_ID" --json)
EPIC_TITLE=$(echo "$EPIC_JSON" | jq -r '.[0].title // .title')
EPIC_DESC=$(echo "$EPIC_JSON" | jq -r '.[0].description // .description')
# Match from `## Design Decisions` through next `## ` heading or end-of-file.
DESIGN_DECISIONS=$(echo "$EPIC_DESC" | awk '
  /^## Design Decisions[[:space:]]*$/ { capture=1; next }
  capture && /^## / { capture=0 }
  capture { print }
')
```

If `DESIGN_DECISIONS` is empty (epic has no such section), substitute the
literal string `None captured.` — same convention used by
`EPIC_DESIGN_DECISIONS` in the bead-prompt template.

## Child Beads (optional)

Include child beads when `br` exposes them cheaply and the count is
manageable. The goal is to surface each child's intent (`## Goal` section)
so the reviewer understands what each commit cluster was trying to do —
not to recreate the full plan.

```bash
# Enumerate child beads of the epic. Use whatever subcommand your br
# version provides (e.g. `br show $EPIC_ID --json` with a `.children`
# field, or `br dep list $EPIC_ID`). If neither is available, skip this
# block entirely — epic-level context alone is still valuable.
```

For each child bead (cap at ~8 to keep the prompt bounded), fetch its
description and extract `## Goal` using the same awk pattern above with
`## Goal` substituted. Render as:

```
- bd-xxx: [bead title]
  **Goal**: [extracted goal line, or "Not captured." if the bead has no ## Goal]
```

If the epic has more than 8 children, include the first 8 (lowest bead IDs
typically map to earliest sequenced work) and append a one-line note:
`... and N more child beads (truncated for prompt size).`

## Rendering PLANNING_CONTEXT

Combine the pieces into a single block. The block is the string value
substituted into the reviewer prompt template's `PLANNING_CONTEXT`
placeholder.

**When epic resolved:**

```
**Epic**: EPIC_ID — EPIC_TITLE

### Design Decisions

DESIGN_DECISIONS

### Implementation Beads

- bd-xxx: [title]
  **Goal**: [goal]
- bd-yyy: [title]
  **Goal**: [goal]
```

Omit the `### Implementation Beads` subsection if child enumeration was
skipped or returned zero results. Keep the `### Design Decisions`
subsection in all epic-resolved cases (substituting `None captured.` when
the section was missing).

**When no epic resolved (fallback):**

```
No linked planning context available — reviewing against general code-quality heuristics only.
```

This exact string — one paragraph, no headings — so reviewers recognize
the pattern and do not hunt for missing data.

## Calling Convention

Skills that use this loader produce:

- `PLANNING_CONTEXT`: the rendered block, ready to substitute into the
  reviewer prompt template (or inject inline for skills without a
  template file)
- `EPIC_ID` and `EPIC_TITLE`: for downstream use (commit messages,
  report headers, etc.) — optional, consume if helpful

The loader is read-only — it never modifies the bead tracker or the
working tree.
