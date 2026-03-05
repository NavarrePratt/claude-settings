# Claude Code User Configuration

Personal `~/.claude` settings for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

## Structure

| Path | Purpose |
|------|---------|
| `CLAUDE.md` | Global instructions loaded into every conversation |
| `settings.json` | Claude Code settings (model, permissions, env vars, plugins) |
| `rules/` | Modular instruction sets referenced from CLAUDE.md |
| `skills/` | Custom slash-command skills (issue planning, code review, etc.) |
| `commands/` | Lightweight slash commands (commit, discover, codex review) |
| `mcp-servers/` | MCP server configuration (docker-compose + env) |
| `agents/` | Agent definitions |

## Rules

- `comments.md` - When comments are acceptable (why, not what)
- `cw-cli.md` - CoreWeave CLI reference
- `git-spice.md` - Stacked PR workflow with `gs`
- `grug-brain.md` - Anti-complexity development philosophy
- `issue-tracking.md` - `br` CLI patterns for issue management
- `python.md` - Python conventions (uv, style)
- `session-protocol.md` - Session startup, progress, and completion
- `testing.md` - Test real behavior, not coverage numbers

## Skills

Custom skills invoked via `/skill-name`:

- **Issue planning**: `issue-plan`, `issue-plan-codex`, `issue-plan-hybrid`, `issue-plan-user`
- **Code review**: `team-branch-review`, `parallel-branch-review`, `grug-review`, `codex-*-review`
- **Review follow-up**: `team-branch-fix`, `team-branch-comment`, `pr-review-reply`, `pr-review-import`
- **Workflow**: `git-commit`, `issue-create`, `remember`, `repo-explore`, `discover`
- **Documentation**: `diataxis-documentation`, `humanizer`
- **CoreWeave**: `cw-repo`, `cw-scaffold`, `cw-explore`, `cw-dev`
- **Infrastructure**: `kubernetes`

## Key Conventions

- All work tracked via [`br`](https://github.com/Dicklesworthstone/beads_rust) (beads_rust) - beads are picked up and implemented by [atari](https://github.com/NavarrePratt/atari)
- Commits created through `/commit` skill
- No git push or GitHub writes without explicit approval
- Python uses `uv` for everything
