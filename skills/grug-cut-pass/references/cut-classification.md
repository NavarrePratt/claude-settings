# Cut Classification Taxonomy

How Codex and the user categorize each finding during a grug cut pass. Every finding lands in exactly one of three buckets. The bucket drives the auto-mode decision and the user prompt framing.

## Slop

Code that adds complexity without justification. Default action: cut.

**Signals that indicate slop:**

- **Premature abstraction**: abstract base class, interface, or trait with a single implementation; "factory" with one product; "strategy" with one strategy
- **Dead code**: unreferenced functions, methods, imports, variables, or parameters in freshly-added code (not pre-existing)
- **Commented-out code blocks**: blocks larger than 2 lines that were commented rather than deleted
- **Unreachable error handling**: try/except for exceptions the code path cannot raise; nil-checks on values just assigned non-nil; defensive guards for conditions ruled out by the type system
- **Indirection without benefit**: a method that only delegates to another method; a wrapper class that only holds one field and exposes it unchanged; a constant referring to another constant
- **YAGNI violations**: parameters added "for future flexibility"; hooks for callers that do not exist; config knobs with no code path that reads them
- **Rule-of-three violations in reverse**: extracting a helper used exactly once; splitting a single cohesive function into three that always run in sequence
- **Generic machinery for non-generic use**: type parameters on a class that is used with exactly one type; `interface{}` / `any` parameters narrowed back to one type at every call site

**Anti-pattern tags** (used for auto-mode allowlist matching):
- `dead_code`
- `commented_code`
- `single_impl_abc`
- `unreachable_error_handling`
- `unused_param_new_code`
- `premature_abstraction`
- `indirection_no_benefit`
- `yagni_hook`
- `premature_generic`

## Acceptable Complexity

Code that looks complex but is justified. Default action: keep, no user prompt needed beyond the auto-report summary.

**Signals that justify complexity:**

- **Measurement-backed optimization**: caching, batching, or complex data structures with profiler evidence in comments, commit message, or linked issue
- **Framework contract**: code shape required by the framework being used (e.g., Django model Meta inner class, Kubernetes controller reconcile signature, React hook rules)
- **Narrow interface over complex state**: a class hiding genuinely complex state behind a small public surface (state machines, coordinators, connection pools)
- **Cross-cutting concern with real consumers**: abstraction with two or more live implementations OR one implementation plus a clearly planned second with a tracked work item
- **Concurrency-required structure**: locks, channels, atomic operations present because of demonstrated concurrent access, not speculation
- **Domain modeling**: class hierarchies or enums that mirror real-world categories the domain actually distinguishes (payment types, account states, document kinds)

Acceptable complexity findings appear in the Codex output so the user can see what Codex considered and ruled out, but Phase 4 does not prompt on them.

## Borderline

Could go either way. Default action: ask the user.

**Signals of borderline:**

- **Abstraction with one current impl but clear second coming**: base class for HTTP client with only `OktaClient`, but the PR description mentions adding `Auth0Client` next
- **Defensive handling that might be reachable**: nil-check on a value from a third-party library whose contract is unclear
- **Helper extracted for one call site but improves readability**: a 20-line block factored into a well-named 20-line helper; no deduplication benefit, but the call site is clearer
- **Generic with one instantiation but domain naturally polymorphic**: `Repository<T>` with only `Repository<User>` today, but domain model clearly has other entities
- **Config knob wired but not exercised**: a setting reachable from the call path that flips real behavior, but no test covers the non-default branch

Borderline findings always prompt the user in Phase 4, even in `--auto` mode. The user is the judge on these.

## Output Format for Codex

Codex returns findings as a YAML array. Each finding must include:

```yaml
- id: f-1
  file: "pkg/auth/middleware.go"
  line: 42
  line_end: 58                    # optional, for block findings
  classification: "Slop"          # Slop | Acceptable | Borderline
  confidence: "High"              # High | Medium | Low
  anti_pattern_tag: "single_impl_abc"  # required for Slop, omit for others
  title: "Abstract base class with single implementation"
  description: >
    AuthHandler is an abstract class with one concrete subclass (OktaAuthHandler).
    The abstraction adds indirection without enabling a second implementation.
  suggested_cut: >
    Delete AuthHandler. Rename OktaAuthHandler to AuthHandler. Update the 3 call
    sites in pkg/server/routes.go.
  acceptable_reason: null         # populated only for Acceptable; brief justification
```

## Auto-mode Allowlist

In `--auto` mode, findings that meet ALL of the following are auto-approved for cutting:

1. `classification == "Slop"`
2. `confidence == "High"`
3. `anti_pattern_tag` is one of: `dead_code`, `commented_code`, `single_impl_abc`, `unreachable_error_handling`, `unused_param_new_code`

Everything else routes to AskUserQuestion in Phase 4. The allowlist is intentionally conservative: it covers cuts where the slop is mechanically detectable and low-risk, and leaves judgment calls to the user.
