# Design Decision Support: Test Scenarios and Expected Behavior

Synthetic review report and behavior traces for validating all 9 scenarios.
Feed this report to `/team-branch-fix` on a branch named `fix/auth-improvements`
with commits ahead of main.

## Synthetic Review Report

### Finding 1: Missing nil check (Trivial, High)
- **File**: pkg/auth/handler.go:42
- **Severity**: High
- **Category**: correctness
- **Description**: `resp` is used without nil check after `authService.Validate()`
- **Suggested fix**: Add `if resp == nil { return fmt.Errorf("nil response") }` before line 43
- **Codex status**: Confirmed
- **Found by**: correctness-reviewer
- **Exercises**: Scenario 1 (trivial finding, no approach generation)

### Finding 2: Error handling across HTTP handler (Complex, Critical)
- **File**: pkg/api/server.go:110
- **Severity**: Critical
- **Category**: error-handling
- **Description**: Handler catches errors inconsistently - some return 500, some panic, some silently continue
- **Suggested fix**: Refactor error handling to use consistent error middleware pattern across all routes. Create error types for client vs server errors, wrap with context, and use a central error handler.
- **Codex status**: Confirmed
- **Found by**: architecture-reviewer
- **Exercises**: Scenario 2 (complex critical, approach generation) + Scenario 3 (validate with Codex)

### Finding 3: Duplicate of Finding 2 (dedup target)
- **File**: pkg/api/server.go:110
- **Severity**: High
- **Category**: error-handling
- **Description**: Error responses are inconsistent across API endpoints
- **Suggested fix**: Standardize error response format
- **Codex status**: Confirmed
- **Found by**: api-reviewer
- **Exercises**: Scenario 6 (deduplication)

### Finding 4: Unused import (Trivial, Medium)
- **File**: pkg/auth/handler.go:5
- **Severity**: Medium
- **Category**: code-quality
- **Description**: `fmt` imported but only used in debug logging that was removed
- **Suggested fix**: Remove unused import
- **Codex status**: Confirmed
- **Found by**: lint-reviewer
- **Exercises**: Scenario 4 (Med/Low, part of batch)

### Finding 5: Missing context propagation (Complex, Medium)
- **File**: pkg/auth/middleware.go:28
- **Severity**: Medium
- **Category**: correctness
- **Description**: Request context is not propagated to downstream service calls, preventing timeout/cancellation
- **Suggested fix**: Thread context.Context through all service method signatures and pass req.Context() from HTTP handlers. This requires changing 4+ function signatures and updating all callers.
- **Codex status**: Confirmed
- **Found by**: correctness-reviewer
- **Exercises**: Scenario 4 (Med/Low "More info" -> reveals complexity -> approach generation)

### Finding 6: Race condition in session cache (Disputed, Critical)
- **File**: pkg/auth/cache.go:65
- **Severity**: Critical
- **Category**: concurrency
- **Description**: Session cache accessed from multiple goroutines without synchronization
- **Suggested fix**: Add sync.RWMutex to protect cache reads/writes
- **Codex status**: Disputed - Codex says the cache is only accessed from a single goroutine due to the HTTP handler's serial processing
- **Found by**: concurrency-reviewer
- **Exercises**: Scenario 5 (disputed finding with potential approach generation)

### Finding 7: Missing error log (Trivial, Low)
- **File**: pkg/auth/handler.go:87
- **Severity**: Low
- **Category**: observability
- **Description**: Authentication failure path does not log the error before returning
- **Suggested fix**: Add `log.Error("auth failed", "error", err)` before the return
- **Codex status**: Confirmed
- **Found by**: observability-reviewer
- **Exercises**: Scenario 4 (Med/Low batch) + Scenario 1 (trivial)

### Finding 8: Hardcoded timeout (Trivial, Low)
- **File**: pkg/auth/client.go:15
- **Severity**: Low
- **Category**: configuration
- **Description**: HTTP client timeout is hardcoded to 30s
- **Suggested fix**: Make timeout configurable via environment variable or config
- **Codex status**: Confirmed
- **Found by**: config-reviewer
- **Exercises**: Scenario 4 (Med/Low batch)

---

## Expected Behavior Traces

### Phase 0: Precondition Check
- Branch `fix/auth-improvements` -> team name: `fix-fix-auth-improvemen-XXXXXX`
- Verify clean working tree, commits ahead of main

### Phase 1: Parse findings
- 8 findings parsed: f-1 through f-8
- Scope: all `line` scoped

### Phase 1.5: Canonicalization (Scenario 6)
- f-2 and f-3 share (pkg/api/server.go, 110) and describe same error handling issue
- Grouped as g-1 (canonical: f-2, more specific fix) with source_finding_ids: [f-2, f-3]
- found_by merged: [architecture-reviewer, api-reviewer]
- severity: Critical (highest of Critical, High)
- **Expect**: "Parsed 8 findings, deduplicated to 7 canonical findings (1 duplicate merged)."

### Phase 2 Step 0: Session default (Scenario 8)
- Branch: `fix/auth-improvements` -> contains `fix/` -> suggests `minimal_patch`
- **Expect**: AskUserQuestion: "Inferred default: minimal_patch based on branch name fix/..."
- User keeps or overrides

### Phase 2 Step 1: Critical & High findings

**g-1 (Error handling, Critical - was f-2+f-3)** - Scenario 2
- Read ~50 lines of pkg/api/server.go around line 110
- Score: multi-function scope (+1), multiple approaches (+1), new patterns (+1) = score 3 = COMPLEX
- **Expect**: Generate 2 approaches:
  - "Add error middleware" (minimal_patch-aligned if default kept)
  - "Restructure handlers" (refactor)
- **Expect**: AskUserQuestion with markdown previews, "Validate with Codex", "Skip"
- If user picks "Validate with Codex" (Scenario 3): Codex called, approaches re-presented with assessments, no Validate option second time

**f-6 (Race condition, Critical, DISPUTED)** - Scenario 5
- **Expect**: AskUserQuestion for stance: Trust Claude / Trust Codex / More info / Skip
- User picks stance -> record decision.stance
- Then classify complexity: mutex vs sync.Map (+1), introduces sync pattern (+1) = score 2 = COMPLEX
- **Expect**: Generate 2 approaches within chosen stance

**f-1 (Nil check, High, Confirmed)** - Scenario 1
- Read ~50 lines of pkg/auth/handler.go around line 42
- Score: single function (0), one obvious fix (0), no new patterns (0), no API change (0), 2 lines (0) = score 0 = TRIVIAL
- **Expect**: AskUserQuestion: Fix (Recommended) / More info / Skip / Defer
- **Expect**: NO approach options presented
- **Expect**: Fixer receives "Use suggested fix"

### Phase 2 Step 3: Medium & Low findings (Scenario 4)

**Batch: correctness [f-5]**
- AskUserQuestion: "1 medium finding in correctness. How to handle?"
- If "Review individually" then "More info" on f-5:
  - Classify complexity: 4+ function signatures (+1), multiple approaches (+1), no new patterns (0), public API impact (+1) = score 3 = COMPLEX
  - **Expect**: approach generation for f-5

**Batch: code-quality [f-4]**
- AskUserQuestion: fix all / review individually / skip all

**Batch: observability + configuration [f-7, f-8]**
- AskUserQuestion: fix all / review individually / skip all

### Phase 3: Build Fix Plan (Scenario 6 continued)
- Only canonical findings with disposition=="fix" proceed
- g-1 carries decision.approach from user selection
- Decision preserved through grouping

### Phase 5: Spawn fixers with approach directives
- Each finding's task payload includes Approach field
- Complex findings: user's chosen approach label + detail
- Trivial findings: "Use suggested fix"

### Phase 6-7.5: Blocked finding resolution (Scenario 7)
- If a fixer marks g-1's error handling approach as Blocked:
  - Results file has ## Blocked Findings section with 2 fallbacks
  - Phase 7.5: AskUserQuestion with Fallback A / Fallback B / Skip
  - If fallback chosen: follow-up fixer spawned
  - Re-enters Phase 6 polling
  - If follow-up also blocks: unresolved (max 1 re-entry)
  - Final report includes resolution status

### Phase 9: Final report
- Summary includes: fixed, resolved via fallback, skipped after block, unresolved, deferred
- Commit strategy offered if verification passes

---

## Error Handling Scenarios (Scenario 9)

### 9a: Codex unavailable during validation
- User picks "Validate with Codex" for g-1
- mcp__codex__codex call fails/times out
- **Expect**: "Codex validation unavailable. Proceeding with the original approaches."
- Options re-presented without "Validate with Codex", without assessments
- approach_source set to "user_choice" (not "codex_validated")

### 9b: Reference file missing
- If complexity-rubric.md is missing when classifying f-1
- **Expect**: inline fallback used: "+1 each for multi-function scope, multiple valid approaches, new patterns required, public API impact, non-obvious fix. Threshold: >= 2."

### 9c: User declines all fallbacks
- Fixer blocks on g-1, presents fallbacks
- User picks "Skip" for all blocked findings
- **Expect**: finding marked as skipped in final report
- **Expect**: "Findings skipped by user after block: N" in summary

### 9d: Fixer Codex unavailable
- Fixer cannot reach mcp__codex__codex during self-validation
- **Expect**: "Codex MCP unavailable - fixes not validated" in results file
- **Expect**: final report notes unvalidated fixes

### 9e: No findings approved
- User skips/defers all findings in Phase 2
- Phase 3: zero findings with disposition=="fix"
- **Expect**: "No fixes to apply - all findings were skipped or deferred."
- **Expect**: skip to Phase 9 with empty summary
