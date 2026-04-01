## Context

- Working directory: !`pwd`
- Current branch: !`git branch --show-current`

## Task

Create commits by spawning a subagent. This isolates all git operations - diff reading, staging, commit formatting, git notes - from the main conversation context.

### Step 1: Summarize Session Context

Before spawning the subagent, review the conversation history and write a brief **session summary** (3-10 lines) that captures:
- What work was done this session (features added, bugs fixed, refactors, etc.)
- Which files were modified and why (logical groupings the subagent should respect)
- Any decisions about commit organization the user mentioned

This summary gives the subagent the "why" behind the changes so it can form logical commit groups and write meaningful commit messages, rather than inferring purely from the raw diff.

If the conversation has no relevant context (e.g., the user just ran `/commit` cold), set SESSION_CONTEXT to "No session context available - infer groupings from the diff."

### Step 2: Spawn Commit Subagent

Launch a **foreground** general-purpose subagent with the prompt below. Before spawning:
1. Replace **CWD** with the working directory from Context
2. Replace **BRANCH** with the branch from Context
3. Replace **SESSION_CONTEXT** with the summary from Step 1
4. Replace **USER_INSTRUCTIONS** with the $ARGUMENTS section below (or "None" if empty)

When the subagent finishes, relay its output to the user verbatim. Do not add commentary.

**Subagent prompt:**

You are creating git commits in CWD on branch BRANCH.

Session context (use this to inform commit groupings and messages):
SESSION_CONTEXT

Process:

1. Gather context by running these commands:
   - `git status` to see all changes
   - `git diff HEAD 2>/dev/null || git diff --cached` to see staged and unstaged changes
   - `git log --oneline -10 2>/dev/null || echo "No commits yet"` to check recent commit style

2. Read `~/.claude/skills/git-commit/SKILL.md` with the Read tool and follow its complete process:
   - Detect conventional commits vs traditional style from git history
   - Group changes logically into atomic commits
   - Use `git add -p` when files contain multiple unrelated changes
   - Use Write tool + `git commit -F .git/COMMIT_MSG_TMP` pattern (NEVER heredocs) to avoid ANSI escape codes
   - Verify each commit with `git show --stat`

3. Before committing, check if beads tracking exists:
   `[ -d .beads ] && ! git check-ignore -q .beads/ 2>/dev/null && echo "beads tracked" || echo "beads ignored or missing"`
   If tracked: run `br sync --flush-only` and stage `.beads/issues.jsonl` along with other changes.

4. After each commit, read `~/.claude/rules/git-notes.md` with the Read tool and attach a git note following its format. Pay close attention to the information safety rules - never include internal URLs, Slack content, employee names, bead IDs, or credentials.

5. User instructions: USER_INSTRUCTIONS

Return ONLY this format when done:

```
Created N commit(s):
- <hash> <subject line>
- <hash> <subject line>
```

If no changes exist, return: "No changes to commit."

Do not include diffs, file lists, staging decisions, or reasoning in your response.

$ARGUMENTS
