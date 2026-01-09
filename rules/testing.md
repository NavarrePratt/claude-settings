# Testing Philosophy

## What Makes a Great Test

A great test covers behavior users depend on. It tests a feature that, if broken, would frustrate or block users. It validates real workflows - not implementation details. It catches regressions before users do.

## Coverage is a Tool, Not a Goal

Do NOT write tests just to increase coverage. Use coverage as a guide to find UNTESTED USER-FACING BEHAVIOR.

## Prioritization

**Test these first:**
- Error handling users will hit
- CLI commands and user-facing APIs
- Core operations (git, file parsing, network)
- Real workflows end-to-end

**Deprioritize:**
- Internal utilities
- Edge cases users won't encounter
- Boilerplate and glue code
- Implementation details that could change

## Process

1. Identify the most important USER-FACING FEATURE that lacks tests
2. Write meaningful tests that validate features work correctly for users
3. Coverage should increase as a side effect of testing real behavior
