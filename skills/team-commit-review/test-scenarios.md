# Test Scenarios: team-commit-review

Manual testing checklist for the team-commit-review skill.

---

## 1. Commit Targeting

Scenarios for commit targeting: parsing user arguments into concrete SHAs.

### 1.1 No argument (defaults to HEAD)

- Invoke `/team-commit-review` with no argument
- Expected: Skill resolves target to `git rev-parse HEAD` (single commit)
- Expected: Diff is computed as `PARENT..HEAD` (the HEAD commit's parent to HEAD)
- Verify: `commit_shas` contains exactly one SHA, matching HEAD

### 1.2 Single bare SHA (valid, HEAD-reachable)

- Invoke `/team-commit-review a1b2c3d` with a valid SHA that is an ancestor of HEAD
- Expected: Skill validates with `git cat-file -e SHA`, then confirms reachability with `git merge-base --is-ancestor SHA HEAD`
- Expected: Diff is computed between the commit's parent and the commit itself
- Verify: Review covers only the specified commit, not HEAD

### 1.3 "last N" with N > 1

- Invoke `/team-commit-review last 3` on a branch with at least 3 commits
- Expected: Skill resolves to `HEAD~3..HEAD` range
- Expected: Diff uses `git diff --name-status -M -C HEAD~3..HEAD`
- Expected: `git log --oneline --first-parent HEAD~3..HEAD` lists exactly 3 commits
- Verify: `commit_summary` contains all 3 commit subject lines

### 1.4 "last 1" (equivalent to HEAD)

- Invoke `/team-commit-review last 1`
- Expected: Resolves to `HEAD~1..HEAD`, which covers exactly the HEAD commit
- Verify: Behavior is equivalent to invoking with no argument

### 1.5 Invalid SHA (not found)

- Invoke `/team-commit-review 0000000deadbeef` with a SHA that does not exist
- Expected: `git cat-file -e` fails
- Expected: Skill stops in Phase 0 with message: "Commit SHA not found. Verify the SHA exists and is reachable from HEAD."

### 1.6 SHA not reachable from HEAD

- Create a detached commit or use a SHA from another branch that is not an ancestor of HEAD
- Invoke `/team-commit-review <unreachable-sha>`
- Expected: `git cat-file -e` succeeds but `git merge-base --is-ancestor` fails
- Expected: Skill stops in Phase 0 with message: "Commit SHA is not reachable from HEAD. Only HEAD-reachable commits can be reviewed."

### 1.7 Merge commit (rejected)

- Identify a merge commit (has 2+ parent lines in `git cat-file -p`)
- Invoke `/team-commit-review <merge-sha>`
- Expected: Skill stops in Phase 0 with message: "Commit SHA is a merge commit. Merge commits combine existing reviewed work and are not meaningful to review independently. Review the source branch instead with /team-branch-review."

### 1.8 Root commit (first commit in repo)

- Invoke `/team-commit-review <root-sha>` where the SHA is the very first commit (no parent)
- Expected: Skill diffs against the empty tree hash `4b825dc642cb6eb9a060e54bf899d69f82cf7c17`
- Expected: `git diff --name-status -M -C 4b825dc642cb6eb9a060e54bf899d69f82cf7c17 ROOT_SHA` is used
- Verify: All files in the root commit appear as Added (A status)

### 1.9 "last N" where N exceeds available history

- On a branch with 5 commits, invoke `/team-commit-review last 20`
- Expected: `git rev-parse HEAD~20` fails because history is too short
- Expected: Skill stops with a clear error (parsing failure or git error surfaced to user)

### 1.10 Unparseable argument

- Invoke `/team-commit-review foo bar baz` or `/team-commit-review HEAD~3..HEAD`
- Expected: Skill stops in Phase 0 with message: "Could not parse commit target. Expected: no argument (HEAD), 'last N' (e.g. last 3), or a bare SHA (e.g. a1b2c3d)."

---

## 2. Team Composition

Scenarios for team composition: reviewer count and specialization based on diff size.

### 2.1 Small commit (< 200 lines changed)

- Run against a commit with fewer than 200 lines added + removed
- Expected: 2 reviewers spawned: `reviewer-security` and `reviewer-pragmatism`
- Verify: No other reviewer names appear in the team

### 2.2 Medium commit (200-1000 lines changed)

- Run against a commit or range with 200-1000 lines added + removed
- Expected: 4 reviewers spawned: `reviewer-security`, `reviewer-correctness`, `reviewer-architecture`, `reviewer-simplicity`
- Verify: Exactly 4 agents created via TeamCreate

### 2.3 Large commit (1000+ lines changed)

- Run against a commit or range with more than 1000 lines added + removed
- Expected: 6 reviewers spawned: `reviewer-security`, `reviewer-correctness`, `reviewer-architecture`, `reviewer-simplicity`, `reviewer-performance`, `reviewer-testing`
- Verify: Exactly 6 agents created via TeamCreate

### 2.4 Empty commit (no file changes)

- Create a commit with `git commit --allow-empty -m "empty"` and review it
- Expected: lines_changed is 0, which falls under the small threshold (< 200)
- Expected: 2 reviewers spawned, but findings should be empty or trivially approved
- Verify: Skill does not crash on zero-line diff

---

## 3. Precondition Checks

### 3.1 Task tool unavailable (custom agent session)

- Run skill from a `claude --agent` session where the Task tool is not available
- Expected: Skill stops immediately with: "This skill requires the Task tool (subagent spawner) which is not available in custom agent sessions (claude --agent). Run this skill from a plain `claude` session instead."
- Expected: No workarounds attempted (no CLI commands, no direct Codex calls)

### 3.2 TeamCreate tool unavailable

- Run skill without `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` set
- Expected: Skill stops with: "Agent teams are required for this skill. Enable them by setting CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 in your Claude Code settings, then retry."

### 3.3 Not inside a git repository

- Run skill from a directory that is not a git working tree
- Expected: `git rev-parse --is-inside-work-tree` fails
- Expected: Skill stops with: "Not inside a git repository. Run this skill from within a git working tree."

### 3.4 Uncommitted changes in working tree

- Stage or modify files without committing, then run the skill
- Expected: `git status --porcelain` returns output
- Expected: Skill stops with: "Working tree has uncommitted changes. Commit or stash them before running a commit review."

---

## 4. Reviewer Workflow

### 4.1 Reviewers receive commit-scoped diff

- After team spawns, inspect the prompt substituted into each reviewer
- Expected: `DIFF_RANGE` is the exact commit-scoped range from Phase 1 (e.g. `HEAD~3..HEAD` or `PARENT..SHA`), not a branch diff like `main..HEAD`
- Expected: Reviewers run `git diff DIFF_RANGE -- <filepath>` to get per-file diffs
- Verify: Review covers only the targeted commits, not the entire branch

### 4.2 Codex validation runs in read-only sandbox

- Observe Codex MCP calls from reviewer agents
- Expected: Each reviewer calls `mcp__codex__codex` with `sandbox: "read-only"` and `approval-policy: "never"`
- Expected: `cwd` is set to the actual working directory
- Verify: Codex does not modify any files

### 4.3 Findings written to TEMP_DIR

- After reviewers complete, check the temp directory
- Expected: Each reviewer writes its findings to `TEMP_DIR/{reviewer-name}.md`
- Expected: File format matches the template structure (Raw Findings, Codex Validation Results, Summary, Notable Observations)
- Verify: File paths use the unique `TEMP_DIR` with timestamp suffix

### 4.4 Cross-domain tips delivered

- When a reviewer notices an issue in another reviewer's domain
- Expected: Reviewer sends a fire-and-forget `SendMessage` tip to the relevant teammate
- Expected: Reviewer also includes the finding in their own report tagged with `[cross-domain: THEIR_FOCUS_AREA]`
- Verify: Tips do not block the sending reviewer's progress

---

## 5. Outcome Determination and Deduplication

### 5.1 Outcome determination - APPROVED

- When no confirmed Critical or High findings remain after Codex validation
- Expected: Report outcome is "APPROVED"

### 5.2 Outcome determination - NEEDS REVISION

- When any confirmed Critical or High findings remain after Codex validation
- Expected: Report outcome is "NEEDS REVISION"

### 5.3 Outcome determination - MANUAL REVIEW REQUIRED

- When more than 50% of Critical/High findings are disputed between Claude and Codex
- Expected: Report outcome is "MANUAL REVIEW REQUIRED"

### 5.4 Deduplication of cross-reviewer findings

- When multiple reviewers report the same issue
- Expected: Finding appears once in the report with cross-references noting which reviewers caught it
- Expected: Conflicts resolved by favoring the position with stronger code evidence

---

## 6. Edge Cases

### 6.1 Large "last N" (> 2000 lines)

- Use `last N` targeting a range that exceeds 2000 lines changed
- Expected: Warning printed: "Warning: Combined diff is LINES lines. Review quality may degrade for very large diffs. Consider using /team-branch-review for branch-level review instead."
- Expected: Review continues despite the warning (not blocked)
- Expected: Warning is noted in the final report

### 6.2 Files with renames (R status)

- Review a commit that renames files (detected via `git diff --name-status -M -C`)
- Expected: Renamed files appear in the change inventory with R status
- Expected: Reviewers can access both old and new paths in their analysis

### 6.3 Files with deletions (D status)

- Review a commit that deletes files
- Expected: Deleted files appear in the change inventory with D status
- Expected: Reviewers can review the deletion context (what was removed and why it matters)

### 6.4 Binary files in commit

- Review a commit that includes binary file changes (images, compiled artifacts)
- Expected: Binary files appear in the change inventory
- Expected: Reviewers note binary files but do not attempt line-level analysis on them

### 6.5 Reviewer agent crashes or times out (partial failure)

- Simulate a reviewer failing to produce a findings file
- Expected: Lead tracks expected reviewer set against actual files produced
- Expected: Gap noted in the final report ("reviewer-X did not produce findings")
- Expected: Report proceeds with available findings from other reviewers

### 6.6 TeamCreate unavailable at spawn time

- Team creation fails after precondition checks pass (runtime failure)
- Expected: Skill falls back to single-agent review (runs codex-diff-review instead)
- Expected: User informed of the fallback

### 6.7 Codex MCP unavailable for a reviewer

- Reviewer's `mcp__codex__codex` call fails
- Expected: Reviewer reports unvalidated findings and notes that Codex validation was unavailable
- Expected: Final report notes which reviewers lacked Codex validation

### 6.8 No findings from any reviewer

- All reviewers complete but none report any issues
- Expected: Report outcome is "APPROVED - no issues found"
- Expected: Report includes a note about review coverage (which reviewers participated)

### 6.9 Temp directory collision

- Highly unlikely due to timestamp suffix, but if TEMP_DIR already exists
- Expected: Skill appends a random suffix to avoid collision
- Expected: Review proceeds normally

### 6.10 Team name computation

- Run on a branch like `feat/add-auth` at HEAD `a1b2c3d`
- Expected: Team name is `commit-review-feat-add-auth-a1b2c3` (branch slug: `/` replaced by `-`, truncated to 20 chars, plus `-` and first 6 chars of HEAD)
- Verify: Team name is valid for TeamCreate (no special characters beyond hyphens)
