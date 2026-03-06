# Performance Review Brief

You are looking for code that will be slow, waste memory, or degrade under load. Focus on problems with measurable impact, not micro-optimizations.

## What to Look For

**Algorithmic complexity**
- Nested loops over data that could grow large (O(n^2) or worse)
- Linear scans where a map/set lookup would work
- Sorting when only min/max is needed
- Repeated work that could be cached or memoized
- String concatenation in loops (quadratic in many languages)

**Database and I/O**
- N+1 query patterns: loading a list then querying for each item individually
- Missing indexes for new query patterns
- Large result sets loaded into memory when pagination or streaming would work
- Sequential I/O operations that could be parallelized or batched
- File handles, connections, or cursors not closed on all paths

**Memory**
- Large allocations in hot paths (inside loops, per-request handlers)
- Unbounded caches or buffers that grow without limits
- Retaining references to large objects longer than needed
- Copying large data structures where references or slices would work
- Loading entire files into memory when streaming would suffice

**Resource management**
- Connection pools sized incorrectly (too small causes contention, too large wastes resources)
- Missing timeouts on network calls, database queries, or external API requests
- Goroutine/thread spawning without backpressure or limits
- Retry logic without backoff (thundering herd on failure)

**Serialization and data transfer**
- Serializing large objects when only a subset of fields is needed
- Missing compression for large payloads
- Chatty protocols: many small messages where one batch would work
- Redundant marshaling/unmarshaling in a pipeline

## How to Think

For each change, ask:
1. What is the expected data size? What happens at 10x or 100x that size?
2. How many times per second will this code path execute?
3. What external resources does this touch? What happens when they are slow or unavailable?
4. Is there an allocation or I/O operation that could be avoided or batched?

Focus on hot paths: request handlers, loop bodies, frequently called functions. Do not optimize cold paths (startup, config loading, one-time setup).

Do NOT flag hypothetical performance issues without considering actual usage patterns. "This could be slow if the list has millions of items" is only valid if the list could actually have millions of items. Check the context.

## Severity Guidance

- **Critical**: Will cause outages or severe degradation under normal production load
- **High**: Measurable performance impact under expected load, resource leaks
- **Medium**: Suboptimal patterns that matter at scale, missing limits or timeouts
- **Low**: Minor optimization opportunities, allocations that could be pooled
