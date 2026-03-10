---
name: code-optimizer
description: Optimizes code for performance by identifying the actual bottleneck, choosing the right optimization lever, and measuring the result. Use when a specific operation is too slow, when a profiler has pointed at a hot path, or when the user asks to make something faster.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "behavior-preservation-checker, code-refactoring-assistant"
---

# Code Optimizer

Making code fast is a **measurement discipline**, not a coding style. The first rule: you don't know where the time goes until you measure. The second rule: you're usually wrong about where you think it goes.

## Step 0 — Do you actually need to optimize?

| Question                                       | If no → stop                                     |
| ---------------------------------------------- | ------------------------------------------------ |
| Is there a concrete, measured slowness?        | "It feels slow" is not a measurement             |
| Is the slow path on a hot path?                | A 10s function called once at startup is fine    |
| Is there a target? ("under 100ms p99")         | Without a target, you don't know when to stop    |

## Step 1 — Profile. Always. First.

| What's slow                | Tool                                                           |
| -------------------------- | -------------------------------------------------------------- |
| CPU-bound Python           | `py-spy`, `cProfile` + snakeviz                                |
| CPU-bound JVM              | `async-profiler`, JFR                                          |
| CPU-bound native           | `perf`, `Instruments`, `vtune`                                 |
| Memory pressure / GC       | Heap profiler (`tracemalloc`, `jmap`, `heaptrack`)             |
| I/O-bound (DB, network)    | Query logs, `EXPLAIN ANALYZE`, trace spans                     |
| Unclear                    | Flame graph first — it'll tell you which category              |

Profile the **real workload**, not a toy. Micro-benchmarks lie.

## Step 2 — Pick the lever

Optimizations, ranked by typical payoff-to-effort:

| Lever                        | When it applies                                     | Typical speedup | Effort |
| ---------------------------- | --------------------------------------------------- | --------------- | ------ |
| Do less work                 | You're computing things nobody uses                 | 10–100×         | Low    |
| Fix the algorithm            | O(n²) where O(n log n) exists; nested loops over the same collection | 10–1000×  | Medium |
| Cache / memoize              | Same expensive call, same inputs, repeatedly        | 2–100×          | Low    |
| Batch                        | N round-trips to a service → 1 round-trip           | N×              | Medium |
| Move out of the loop         | Invariant computation inside a loop                 | iterations×     | Trivial|
| Use the right data structure | `list` where you need `set` lookup; linear scan where you need index | 2–1000×   | Low    |
| Parallelize                  | Embarrassingly parallel work on a multi-core box    | cores×          | High   |
| Go native / use SIMD         | Tight numeric loop in an interpreted language       | 10–100×         | High   |
| Micro-optimize               | Unroll, inline, avoid allocations                   | 1.1–2×          | High   |

**Start at the top.** Micro-optimization is the last resort, not the first instinct.

## Step 3 — Change one thing, measure again

Benchmark before → one change → benchmark after → record the delta. Every time. If you make three changes and it's faster, you don't know which one did it — and one of them probably made it slower.

## Worked example

**Complaint:** "Exporting the report takes 40 seconds."

**Profile (`py-spy top`):**

```
 84%  _lookup_user_name   (report.py:67)
 11%  _format_row         (report.py:80)
  3%  csv.writer.writerow
```

84% in one function. Look at it:

```python
def _lookup_user_name(user_id):
    return db.query("SELECT name FROM users WHERE id = ?", user_id).one()

def export(rows):
    for row in rows:                        # 10,000 rows
        row.user_name = _lookup_user_name(row.user_id)
        writer.writerow(_format_row(row))
```

**Diagnosis:** N+1 query. 10,000 rows → 10,000 round-trips. **Lever: batch.**

```python
def export(rows):
    user_ids = {row.user_id for row in rows}
    names = dict(db.query("SELECT id, name FROM users WHERE id IN ?", list(user_ids)))
    for row in rows:
        row.user_name = names[row.user_id]
        writer.writerow(_format_row(row))
```

**Measure:** 40s → 0.6s. 67× speedup, one query instead of 10,000. No data structure changed, no parallelism, no C extension. Just: do less work.

→ `behavior-preservation-checker` — the `IN` query with a set dedupes `user_id`s; make sure that's equivalent (it is — we're populating a dict, dupes were redundant anyway).

## Common traps

- **Optimizing the wrong thing.** You made `_format_row` 2× faster. It was 11% of runtime. Total speedup: 1.06×. The profiler told you to look at `_lookup_user_name`.
- **Micro-benchmark lies.** Your loop is 3× faster in isolation. In production, it's memory-bandwidth-bound and the "optimization" does nothing. Benchmark the real path.
- **Caching without eviction.** Memoization sped up the hot path; three days later you're OOM because the cache never forgets.
- **Premature parallelism.** Threading added 20% overhead and the GIL means you got zero speedup. Profile says you're CPU-bound in Python → `multiprocessing`, not `threading`.

## Do not

- **Do not** optimize without profiling. Your intuition is wrong. Everyone's is.
- **Do not** optimize before the code is correct. A fast wrong answer is worthless.
- **Do not** change the algorithm and micro-optimize in the same pass. You won't know which helped.
- **Do not** leave the benchmark out of the PR. "It's faster" is a claim; the before/after numbers are the evidence.
- **Do not** sacrifice readability for a 5% gain in cold code. 5% of nothing is nothing.

## Output format

```
## Baseline
<metric> = <value>  (measured with: <tool/command>)

## Bottleneck
<file>:<line>  — <N>% of runtime
<why it's slow — the diagnosis>

## Change
<lever from table> — <one-sentence what>
<diff>

## Result
<metric> = <value>  (<N>× speedup)

## Behavior check
<→ behavior-preservation-checker, or: tests green>
```
