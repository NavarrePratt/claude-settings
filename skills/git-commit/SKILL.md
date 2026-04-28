---
name: git-commit
description: Create logically grouped, atomic git commits with well-formatted commit messages following best practices. Use this skill when you need to commit changes to a git repository with proper message formatting and atomic grouping. Before creating commits, reflect on the work just completed and surface any open questions or unresolved decisions (judgment calls made without checking, deferred edge cases, defaults picked when multiple options were viable, trade-offs that could reasonably go the other way) via AskUserQuestion so they get discussed with the user rather than silently committed.
allowed-tools:
  - Bash
  - Read
  - Edit
---

# Git Commit Skill

This skill helps you create well-structured, atomic git commits with properly formatted commit messages.

## When to Use This Skill

Use this skill when:
- You need to commit changes to a git repository
- You want to create atomic, logically grouped commits
- You need to follow commit message best practices
- You have multiple changes that should be split into separate commits
- You need to use git partial adds (git add -p) for fine-grained control

## Task Overview

Based on the current git status and changes, create a set of logically grouped, atomic commits.
Be specific with each grouping, and keep scope minimal. Leverage partial adds to
make sure that multiple changes within a single file aren't batched into
commits with unrelated changes.

## Process

1. **Analyze Current State**
   - Check git status to see staged and unstaged changes
   - Review git diff to understand what has changed
   - Check recent commits (`git log --oneline -20`) to understand:
     - Whether the project uses conventional commits (e.g., `feat:`, `fix:`, `docs:`)
     - The project's commit message style and conventions
     - Typical subject line length and formatting patterns

2. **Group Changes Logically**
   - Identify related changes that should be committed together
   - Separate unrelated changes into different commits
   - Use `git add -p` for partial adds when a file contains multiple logical changes

3. **Create Commits**
   - Stage the appropriate changes for each commit
   - Write commit messages following the best practices below
   - Verify each commit is atomic and complete

## Commit Message Format Detection

**IMPORTANT**: Before writing any commits, analyze the recent git history to determine the project's commit style:

- **Check for Conventional Commits**: Look for patterns like `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`, `style:`, `perf:`, `ci:`, `build:`
- **Match the existing style**: If 80% or more of recent commits follow conventional commits, use that format
- **Be consistent**: Match the capitalization, punctuation, and structure of existing commits

### Conventional Commits Format

If the project uses conventional commits, follow this structure:

```
<type>[(optional scope)]: <description>

[optional body]

[optional footer(s)]
```

**Common types:**
- `feat`: A new feature
- `fix`: A bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, missing semicolons, etc.)
- `refactor`: Code changes that neither fix bugs nor add features
- `perf`: Performance improvements
- `test`: Adding or updating tests
- `build`: Changes to build system or dependencies
- `ci`: Changes to CI configuration
- `chore`: Other changes that don't modify src or test files

**Examples:**
- `feat: add user authentication`
- `fix: resolve null pointer in login handler`
- `docs: update API documentation`
- `refactor(auth): simplify token validation logic`

## Git Commit Message Best Practices

Follow these seven rules for excellent commit messages (adjust for conventional commits if used):

1. **Separate subject from body with a blank line** - Critical for readability
2. **Limit subject line to 50 characters** - Forces concise summaries
3. **Capitalize the subject line** - Consistent formatting
4. **Do not end subject line with a period** - It's a title, not a sentence
5. **Use imperative mood in subject** - "Add feature" not "Added feature"
   - Test: Subject should complete "If applied, this commit will _____"
6. **Wrap body at 72 characters** - Ensures readability in terminals
7. **Use a body only when it adds real context** - Most commits need only a subject. When a body is useful, explain why or a non-obvious consequence, not how.

### Message Structure

```
<subject: concise summary, imperative, capitalized, no period>

<optional body: one short paragraph explaining why or a non-obvious consequence>

<footer: references to issues, breaking changes, etc.>
```

### Key Principles

- **Atomic commits**: Each commit should represent one logical change
- **Context is bounded**: Explain WHY only when the subject and diff do not already make it clear
- **Future-proof**: Write for someone (including future you) reading this months later
- **Consistency**: Maintain uniform style across the project

### Body Length

Default to no body. Use a body only when the commit needs context the
subject and diff do not provide.

When present, keep the body to one short paragraph, usually 1-4
sentences. Use two paragraphs only for genuinely risky or surprising
changes. Do not exceed ~100 words without a specific reason.

Don't:
- Walk through files, functions, helpers, or implementation steps
- Enumerate tests added or recap what each test asserts
- Pre-answer reviewer questions or list alternatives considered
- Repeat the subject in prose
- Turn the commit body into a PR description

Put broad motivation, review context, design history, and detailed test
coverage in the PR description or linked design doc, not the commit
message.

### Examples

**Good Examples (Traditional Style):**
- `Refactor subsystem X for readability`
- `Remove deprecated methods from UserService`
- `Fix null pointer exception in login handler`
- `Add user authentication middleware`

**Good Examples (Conventional Commits):**
- `feat: add user authentication middleware`
- `fix: resolve null pointer exception in login handler`
- `refactor: improve subsystem X readability`
- `chore: remove deprecated methods from UserService`

**Bad Examples:**
- `fixed stuff`
- `Changes`
- `wip`
- `Update file.js`
- `feat added new feature` (incorrect format - missing colon)
- `Add 7 tests for auth module` (fragile count - goes stale if tests change before push)
- `Update 3 config files for new logging` (unnecessary count - the diff shows which files)

### No Fragile Counts

Never include specific counts of items (tests, files, functions, endpoints, etc.) in commit
subjects or bodies. These counts go stale before the commit is even pushed - a rebase, amend,
or fixup changes the number and the message becomes a lie. Describe what was done, not how many.

## Implementation Steps

1. Run `git status` to see current state
2. Run `git diff HEAD` to see all changes (if HEAD doesn't exist yet, use `git diff --cached` instead)
3. Run `git log --oneline -20` to analyze recent commit style (skip if no commits yet)
   - **Determine if conventional commits are used** (look for `type:` prefix patterns)
   - Note the typical capitalization and formatting style
   - Identify any project-specific conventions
4. Identify logical groupings of changes
5. For each logical group:
   - Stage the relevant changes (use `git add -p` if needed)
   - Create the commit. Single-line: `git commit -m "subject"`. Multi-line: pass a heredoc directly to `-m`:
     ```bash
     git commit -m "$(cat <<'EOF'
     subject line

     body paragraph explaining why
     EOF
     )"
     ```
   - Verify the commit with `git show`
6. After all commits, run `git status` to verify nothing important was missed

## Escape Codes in Commit Messages

Commit messages should be plain ASCII. If you ever see garbage like `^[[1m^[[30m` or `[0m[35m` in `git log`, ANSI codes leaked in — usually from piping colored CLI output (`grep --color`, `ls --color`, issue tracker descriptions) into a message. Don't copy colored terminal output into commit bodies; paraphrase instead.

To sanity-check a commit:
```bash
git log -1 --format=%B | grep -c $'\x1b'
```
`0` means the message is clean. Any higher number means an ESC byte (start of an ANSI sequence) is present.

## Notes

- **ALWAYS check recent git history first** to determine if conventional commits are used
- **Match the project's existing style** - consistency is more important than personal preference
- DO NOT push to remote unless explicitly asked
- Always verify authorship and commit details before amending
- Use `git add -p` for interactive staging when files contain multiple unrelated changes
- Keep commits focused and atomic - one logical change per commit
- If in doubt about whether to use conventional commits, look at the last 20-30 commits for patterns

