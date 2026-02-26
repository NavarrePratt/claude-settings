---
name: remember
description: Review session work and optionally update agent memory. Use to capture decisions, patterns, or insights - or just to reflect without changing anything.
---

# Memory Introspection

The user has triggered a memory review. They may have provided context, or they may just want you to reflect on the session.

## Context from user
$ARGUMENTS

## Locate Memory Directory

Your auto memory directory path is in your system prompt, in a line like:
> You have a persistent auto memory directory at `~/.claude/projects/<project>/memory/`.

Extract that path. If you cannot find it, use `~/.claude/projects/` and look for a subdirectory matching the current working directory.

Create the directory if it does not exist yet.

## Process

1. **Summarize what happened this session** - briefly, 2-3 sentences max.

2. **Read your current memory index** at `<memory-dir>/MEMORY.md` (it may not exist yet - that is fine).

3. **Read any topic files referenced in MEMORY.md** that are relevant to this session's work.

4. **Assess whether memory needs updating.** Consider:
   - Were decisions made that affect future sessions?
   - Did patterns emerge worth recording?
   - Were gotchas or mistakes encountered worth avoiding next time?
   - Did the user explicitly ask you to remember something specific?

5. **Report your assessment.** Show one of:

   **If no changes needed:**
   > "Memory is current. Nothing from this session warrants an update."

   Done. Stop here.

   **If changes are warranted:**
   Show exactly what you would add or modify, which file it goes in, and why. Format as a diff-like preview. Then ask: "Want me to apply this?"

6. **Wait for approval.** Only write to memory files if the user confirms. If they say no or suggest edits, adjust accordingly.

## Rules

- You are NOT obligated to update memory every time this is invoked. A session of routine work may produce nothing worth recording. That is normal.
- Prefer updating existing topic files over creating new ones. Only create a new file when the topic genuinely does not fit anywhere.
- If MEMORY.md does not exist yet, propose creating it with a clean index structure.
- Keep MEMORY.md under 150 lines. Push detail into topic files.
- Do not duplicate information already in project CLAUDE.md files.
- Do not record temporary debugging state or one-off fixes.
- Focus on decisions, patterns, preferences, and mistakes - things that change how you work in future sessions.
