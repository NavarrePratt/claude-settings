# Architecture Review Brief

You are evaluating structural decisions: how the code is organized, how components interact, and whether the design will hold up as the codebase evolves.

## What to Look For

**Coupling and dependencies**
- New dependencies between modules that should be independent
- Circular dependencies (A imports B imports A)
- God objects or functions that know too much about too many things
- Concrete types where interfaces would improve flexibility at a boundary with multiple real implementations

**API design**
- Public APIs that expose internal implementation details
- Inconsistent patterns across similar operations (e.g., some endpoints return errors differently)
- Breaking changes to existing interfaces without migration path
- APIs that force callers into awkward usage patterns

**Responsibility boundaries**
- Functions or classes doing unrelated things (low cohesion)
- Business logic mixed with I/O, serialization, or transport concerns
- Configuration or policy decisions buried deep in implementation code
- Cross-cutting concerns (logging, metrics, auth) handled inconsistently

**Data modeling**
- Data structures that do not match the domain they represent
- Redundant data that can get out of sync
- Missing or incorrect type constraints
- Stringly-typed data where structured types would prevent bugs

**Extensibility at real seams**
- Changes that make future likely modifications harder
- Hardcoded values that should be configurable at known extension points
- Patterns that will not scale with the known growth trajectory

**Consistency**
- New code that contradicts established patterns in the same codebase
- Reimplementing functionality that already exists elsewhere
- Naming conventions that clash with the rest of the project

## How to Think

For each structural decision, ask:
1. Does this change make the codebase easier or harder to understand for someone new?
2. If requirements change in a plausible way, how much code needs to change?
3. Does this follow or break patterns already established in this project?
4. Is this the right level of abstraction - not too high, not too low?

Look at how changed files relate to each other and to unchanged code. Architecture issues often live in the connections between components, not within them.

## Severity Guidance

- **Critical**: Architectural decisions that will cause cascading problems and are expensive to fix later
- **High**: Significant coupling or design issues that will slow down future development
- **Medium**: Inconsistencies or suboptimal patterns that cause friction but are contained
- **Low**: Minor structural improvements, style consistency
