# br in Worktrees

Canonical pattern for invoking the `br` CLI from inside a git worktree where
`.beads/` is gitignored.

## Why br Fails in Worktrees

`br` auto-discovers its database by walking up from `cwd` looking for a
`.beads/` directory. In most repos, `.beads/` is gitignored (the database is
not a committed artifact), so `git worktree add` does not copy it. The
worktree's path chain has no `.beads/` to find, and every subsequent `br`
call fails with:

```
No beads directory found.
Run `br init` to create one.
```

## The Fix: --db Flag

Every `br` call made from inside a worktree targets the main repo's database
via the `--db` flag:

```bash
br --db "$MAIN_REPO_BEADS_DB" <command>
```

The flag is a top-level option, so it goes before the subcommand:

```bash
br --db "$MAIN_REPO_BEADS_DB" show bd-xxx --json
br --db "$MAIN_REPO_BEADS_DB" update bd-xxx --status in_progress
br --db "$MAIN_REPO_BEADS_DB" close bd-xxx --reason "..."
```

## Deriving MAIN_REPO_BEADS_DB

The path must be derived BEFORE the `cd` into the worktree, so
`git rev-parse --show-toplevel` returns the main repo root (not the worktree
root). Ask `br` for its database path first and fall back to the default
location only if `br where` cannot report it:

```bash
MAIN_REPO_BEADS_DB="$(br where --json 2>/dev/null | jq -r '.database_path // empty' 2>/dev/null || true)"
if [ -z "$MAIN_REPO_BEADS_DB" ]; then
  MAIN_REPO_BEADS_DB="$(git rev-parse --show-toplevel)/.beads/beads.db"
fi
```

Record the value as a skill-level variable (alongside `BASE_REF`,
`WORKTREE_PATH`, etc.) and reference it on every post-cd `br` call.

## Why Not a Shell Env Var?

Setting `export BEADS_DIR=...` (or any other shell env var) does not work
across Claude's Bash tool calls. The Bash tool spawns a fresh shell per
invocation, so exported variables do not persist between tool calls. A
per-call `--db` flag is explicit, survives tool-call boundaries, and makes
every call site self-documenting.

## Scope: Lead Agent Only

This pattern applies to the lead agent - the orchestrating session that runs
`cd` into the worktree. Subagents spawned inside the worktree do not call
`br` directly; they implement beads and report results back to the lead. If
a future workflow needs subagents that invoke `br`, pass `MAIN_REPO_BEADS_DB`
into the subagent prompt and follow the same pattern.

## Limitation: Default Database Name

The derivation calls `br where --json` first, so projects that configure a
non-default database name (via `.beads/config.yaml` or another mechanism)
are handled transparently whenever `br` can report the path itself. The
hardcoded `.beads/beads.db` fallback only runs when `br where --json` is
unavailable or does not return a `database_path` field - for example, on
older `br` builds. A project on such a build that also uses a non-default
database name will need to adjust the derivation for that project.
