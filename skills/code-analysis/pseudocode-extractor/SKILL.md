---
name: pseudocode-extractor
description: Extracts language-agnostic pseudocode from real code, stripping syntax noise and language-specific machinery while preserving the algorithmic structure. Use when documenting an algorithm for a paper or spec, when porting and wanting a neutral intermediate, or when explaining code to someone who doesn't know the source language.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "verified-pseudocode-extractor, pseudocode-to-python-code, pseudocode-to-java-code"
---

# Pseudocode Extractor

Real code has type annotations, error handling, logging, defensive checks, and language-specific idioms. Pseudocode has **the algorithm**. Extraction is deciding what's essential vs. what's machinery.

Compare → `verified-pseudocode-extractor`: that one preserves pre/post/invariants from verified code. This one works on any code and focuses on algorithmic clarity.

## Keep vs. strip

| Source code element                | In pseudocode                                          |
| ---------------------------------- | ------------------------------------------------------ |
| Core control flow (if/while/for)   | KEEP — this is the algorithm                           |
| Data transformations               | KEEP — the what                                        |
| Variable types                     | STRIP usually; KEEP if the type is the point (e.g., "priority queue") |
| Error handling (try/catch)         | STRIP unless the error *path* is algorithmic           |
| Logging, metrics, tracing          | STRIP                                                  |
| Defensive null checks              | STRIP (or note as precondition)                        |
| Resource management (open/close)   | STRIP (becomes "with resource R: ...")                 |
| Memory management                  | STRIP entirely                                         |
| Language idioms (`__enter__`, RAII)| STRIP — translate to intent                            |

**Rule of thumb:** if changing this line changes the *output* for some input, keep it. If it changes the *robustness* or *performance* without changing the output, strip it.

## Abstraction levels

Pick a level and stay there:

| Level          | Example                                                  | Use when                              |
| -------------- | -------------------------------------------------------- | ------------------------------------- |
| High (prose-y) | `sort items by score, descending`                        | Explaining to non-programmers         |
| Medium         | `sort(items, key=score, descending)`                     | Algorithm documentation, paper specs  |
| Low (near-code)| `for i ← 1 to n: for j ← i+1 to n: if items[i].score < items[j].score: swap` | Direct reimplementation target |

Mixing levels reads badly. `sort items by score` followed by `for i ← 0 to n-1` is jarring.

## Worked example

**Source (Python, realistic mess):**

```python
def merge_intervals(intervals: list[tuple[int, int]]) -> list[tuple[int, int]]:
    """Merge overlapping intervals."""
    if not intervals:
        return []

    logger.debug(f"merging {len(intervals)} intervals")

    try:
        sorted_ivs = sorted(intervals, key=lambda iv: iv[0])
    except TypeError as e:
        raise ValueError(f"intervals must be comparable: {e}") from e

    result = [sorted_ivs[0]]
    for current in sorted_ivs[1:]:
        last_start, last_end = result[-1]
        cur_start, cur_end = current
        assert cur_start >= last_start, "sort invariant broken"  # sanity

        if cur_start <= last_end:
            result[-1] = (last_start, max(last_end, cur_end))
        else:
            result.append(current)

    metrics.increment("intervals.merged", len(intervals) - len(result))
    return result
```

**Pseudocode (medium level):**

```
function merge_intervals(intervals):
    // PRE: intervals is a list of (start, end) pairs with start ≤ end

    if intervals is empty:
        return empty list

    sort intervals by start

    result ← [first interval]
    for each interval (s, e) in remaining intervals:
        (ls, le) ← last interval in result
        if s ≤ le:                            // overlaps or touches the last one
            replace last in result with (ls, max(le, e))
        else:
            append (s, e) to result

    return result
```

**What got stripped:**
- Logging (`logger.debug`, `metrics.increment`) — observability, not algorithm.
- `try/except TypeError` — defensive; becomes a precondition comment.
- `assert cur_start >= last_start` — sanity check on the sort; true by construction.
- Type annotations — the `(start, end)` structure is noted in the PRE.
- Tuple destructuring syntax — replaced with "(s, e)" notation.

**What got kept:**
- Empty check — it's a real edge case that affects output.
- The sort — essential; the algorithm depends on it.
- The overlap condition `s ≤ le` — the heart of the algorithm. With the `≤` preserved (touching intervals merge too).

## Common source idioms → pseudocode

| Source idiom                              | Pseudocode                                          |
| ----------------------------------------- | --------------------------------------------------- |
| `for i, x in enumerate(xs):`              | `for i ← 0 to |xs|-1:` + `x ← xs[i]`, or drop `i` if unused |
| `while True: ... if cond: break`          | `repeat: ... until cond`                            |
| List comprehension `[f(x) for x in xs if p(x)]` | `for each x in xs where p(x): collect f(x)`   |
| `defaultdict(list)`                       | `map with default empty-list`                       |
| `heapq.heappop`                           | `extract-min from priority queue`                   |
| Exception-for-control-flow (StopIteration)| Explicit condition — exceptions are a Python-ism   |
| `with lock:`                              | `atomically:` or `under lock L:`                    |

## Do not

- **Do not** mix abstraction levels. Pick high/medium/low and stay consistent.
- **Do not** strip something just because it's boilerplate in the source language. `if not intervals: return []` is one line of Python but it's a *real* edge case.
- **Do not** keep error handling that's just "propagate the error." Keep error handling that's *algorithmic* — e.g., "on parse failure, skip this record and continue."
- **Do not** invent structure that isn't there. If the source is a 200-line straight-line function, the pseudocode is also long. Don't impose phases that don't exist.

## Output format

```
## Source
<language, file/function>

## Abstraction level
<high | medium | low> — <intended audience>

## Pseudocode
<block>

## Stripped
<what was removed — logging, defensive checks, language machinery — and why it's not algorithmic>

## Preserved subtleties
<edge cases, ≤ vs <, off-by-one choices — things that look minor but matter>
```
