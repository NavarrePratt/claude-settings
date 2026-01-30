---
description: Grug brain philosophy - complexity is the enemy, embrace simplicity and pragmatism
---

# Grug Brain Development Philosophy

Complexity very, very bad. Fight complexity demon always.

## Quick Decision Framework

| Situation | Decision |
|-----------|----------|
| "Should I extract this to a function?" | Only after 3+ occurrences with clear pattern |
| "Should I add a helper class?" | No. Write procedurally first, extract later |
| "Should I add generics?" | Only for containers. Otherwise: no. |
| "Should I refactor this module?" | Only with clear pain point and small increments |
| "Should I optimize this?" | Only with profiler data showing specific problem |
| "Should I add an abstraction layer?" | Default no. Let patterns emerge first. |

## Core Principles

### Say No to Complexity
- Default answer to new abstraction: "no"
- When forced, pursue 80/20 solutions - most value, minimal code
- Ask forgiveness rather than permission when cutting scope

### Code Factoring
- No premature abstraction - let patterns emerge naturally
- Wait for clear "cut points" before extracting
- Good abstractions have narrow interfaces trapping complexity inside

```python
# No: premature abstraction
class DataProcessor(ABC):
    @abstractmethod
    def process(self, data): ...

class JSONProcessor(DataProcessor): ...
class XMLProcessor(DataProcessor): ...

# Yes: just write the code, abstract when pattern is clear
def process_json(data):
    return json.loads(data)
```

### Code Style
- Break complex expressions into simple statements with clear names
- Explicit readable code > compact clever code
- Debuggability > line count

```python
# No: clever but hard to debug
valid = all(x > 0 for x in data) and len(data) > 1

# Yes: explicit, debuggable
has_positive_values = all(x > 0 for x in data)
has_multiple_items = len(data) > 1
valid = has_positive_values and has_multiple_items
```

### Locality of Behavior
- Related code belongs together
- Question aggressive separation of concerns
- When maintaining a feature, all relevant code findable in one place

### Refactoring
- Small incremental changes only
- Keep code working throughout
- Understand why code exists before removing (Chesterton's Fence)
- Large refactors fail - avoid them

### Testing
- Integration tests are the sweet spot
- Minimal mocking - only at system boundaries
- Reproduce bugs with regression tests before fixing

### Performance
- Never optimize without profiler data showing specific problems
- Network calls usually the real bottleneck, not CPU
- Big-O thinking without measurement leads astray

### Concurrency
- Fear and minimize concurrency
- Prefer simple models: stateless handlers, independent job queues

## Anti-Patterns to Avoid

- Over-engineering "for future flexibility" - write for today's needs
- Extracting abstractions before patterns are clear
- Complex generics or type hierarchures beyond simple containers
- Premature performance optimization without profiler data
- Aggressive separation of concerns that splits related logic
- Adding "just in case" error handling for impossible scenarios

## See Also

This philosophy underlies several specific rules:
- **comments.md** - code should self-document, not need explaining
- **testing.md** - integration tests > unit test coverage chasing
- **python.md** - avoid complexity, keep it simple
