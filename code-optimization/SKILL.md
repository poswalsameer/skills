---
name: code-optimization
description: >-
  Review code for unnecessary work, inefficient algorithms, excessive I/O,
  memory waste, and scalability issues. Use when the user says slow, optimize,
  performance, perf, efficiency, latency, scaling, reduce cost, speed up, or
  runs /code-optimization. Assumes AI-generated code may be functionally
  correct but inefficient.
---

# Code Optimization Review

Review code for unnecessary work, inefficient algorithms, excessive I/O, memory waste, and scalability issues. Assume AI-generated code may be functionally correct but inefficient.

## Review Process

1. **Trace execution:** Identify DB/API calls, loops, transformations, data dependencies, and expected data sizes.
2. **Audit I/O:** Find N+1 queries, duplicate calls, unnecessary reads/writes, missing batching, excessive sequential awaits, and redundant network requests.
3. **Analyze algorithms:** Detect nested loops, repeated linear searches, unnecessary sorting, redundant iterations, and avoidable expensive computation.
4. **Evaluate data structures:** Check whether `Map`, `Set`, or other structures are more suitable than repeated array scans. Identify unnecessary copies and intermediate arrays.
5. **Check async operations:** Identify accidental serialization, unsafe unlimited concurrency, redundant requests, and missing bounded concurrency.
6. **Check memory and scalability:** Find unbounded data loading, missing pagination, unnecessary allocations, and opportunities for streaming or chunking.
7. **Check infrastructure costs:** Identify excessive database reads/writes, API calls, serverless invocations, logging, retries, and unnecessary compute.
8. **Detect AI-code smells:** Look for redundant abstractions, duplicate validation, repeated transformations, hidden I/O in helpers, and over-engineering.

## Finding Severity

Classify every finding into exactly one category:

- **CRITICAL:** Immediate, severe performance or cost risk. Examples: unbounded operations likely to exhaust memory, runaway queries or loops, catastrophic algorithmic complexity on realistic workloads, or retry storms capable of destabilizing production.
- **HIGH:** Significant latency, scalability, or infrastructure-cost issue. Examples: N+1 queries on large datasets, expensive nested loops, repeated DB calls, unbounded concurrency, or fetching entire collections unnecessarily.
- **MEDIUM:** Meaningful inefficiency with limited or conditional impact. Examples: repeated computations, avoidable array passes, unnecessary serialization, inefficient lookups on moderate datasets, or sequential independent operations.
- **LOW:** Minor optimization with negligible current impact. Examples: small avoidable allocations, trivial redundant operations, or micro-optimizations without demonstrated bottlenecks.

**Severity rules:**
- Assess realistic data sizes, execution frequency, production impact, and resource costs.
- Do not classify an issue as CRITICAL merely because its theoretical complexity is poor.
- Do not report stylistic preferences as performance findings.
- Avoid duplicate findings for the same underlying problem.
- If evidence is insufficient, state the uncertainty rather than exaggerating severity.

## Required Actions

For each finding, provide:

- **Severity:** CRITICAL / HIGH / MEDIUM / LOW
- **Location:** File and relevant function or line
- **Problem:** What is inefficient
- **Impact:** Why it matters, including complexity or estimated I/O when possible
- **Fix:** A concrete, actionable improvement

Prioritize findings from CRITICAL to LOW. Fix significant issues when authorized; do not modify unrelated behavior.

## Verification

After making changes:

1. Run relevant tests.
2. Confirm behavior, ordering, error handling, and consistency remain correct.
3. Recheck DB/API call counts, algorithmic complexity, and memory behavior.
4. Run benchmarks or profiling when available.
5. Report changes and measured or estimated before/after impact. Never claim unmeasured improvements as proven.

**Golden rule:** Eliminate unnecessary work before optimizing necessary work. Preserve correctness, readability, and maintainability.
