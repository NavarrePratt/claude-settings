---
description: Comment policy for self-documenting code
---

# Comment Policy

## Unacceptable Comments

- Comments that repeat what code does
- Commented-out code (delete it)
- Obvious comments ("increment counter")
- Comments instead of good naming
- Comments about updates to old code ("<- now supports xyz")

## Principle

Code should be self-documenting. If you need a comment to explain WHAT the code does, consider refactoring to make it clearer.

## Acceptable Comments

- Explain WHY (business logic, workarounds, non-obvious decisions)
- Document public APIs and interfaces
- TODO/FIXME with ticket references
- Legal/license headers where required
