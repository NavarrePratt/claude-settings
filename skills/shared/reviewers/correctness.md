# Correctness Review Brief

You are looking for code that will break at runtime. Not style, not architecture - actual bugs.

## What to Look For

**Logic errors**
- Off-by-one in loops, slices, and boundary conditions
- Wrong comparison operators (< vs <=, == vs ===)
- Negation errors in conditionals (inverted boolean logic)
- Short-circuit evaluation hiding side effects
- Switch/match fallthroughs that should not fall through

**Nil/null/undefined handling**
- Dereferencing potentially nil pointers or optional values
- Missing nil checks after map lookups, type assertions, or channel receives
- Functions that return (value, error) where the value is used without checking error first
- Optional chaining that silently swallows important failures

**Error handling**
- Errors caught but not handled (empty catch blocks, ignored return values)
- Error messages that lose context (wrapping without the original error)
- Panic/throw in library code that should return errors
- Cleanup code (defer, finally) that does not run on all exit paths

**Concurrency**
- Shared mutable state without synchronization
- Race conditions in read-modify-write sequences
- Goroutine/thread leaks (spawned but never joined or cancelled)
- Deadlock potential from inconsistent lock ordering
- Channel operations that can block forever

**Edge cases**
- Empty collections, zero-length strings, zero values
- Unicode handling (multi-byte characters, normalization)
- Large inputs that could cause integer overflow or excessive memory use
- Time zone and daylight saving time assumptions
- File system edge cases (symlinks, permissions, path traversal)

**State management**
- Mutations to shared data structures without copying
- Stale closures capturing loop variables
- Initialization order dependencies that could break

## How to Think

For each function or method changed, ask:
1. What are the valid inputs? What happens with invalid ones?
2. What can fail? Is every failure handled on every path?
3. If this runs concurrently, what breaks?
4. What does the caller expect? Does this code actually deliver that?

Read the tests (if any) for the changed code. What cases do they cover? What cases are missing?

## Severity Guidance

- **Critical**: Data corruption, crash in production, silent wrong results
- **High**: Failure under common edge cases, resource leaks under load
- **Medium**: Failure under uncommon conditions, degraded behavior
- **Low**: Defensive improvements, potential future bugs from fragile patterns
