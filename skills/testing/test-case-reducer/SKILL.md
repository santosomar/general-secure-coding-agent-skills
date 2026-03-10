---
name: test-case-reducer
description: Shrinks a failing test input to its minimal form while preserving the failure — delta debugging and structured shrinking to find the smallest input that still triggers the bug. Use when a fuzzer or property test finds a failure with a huge input, when a bug report has an unwieldy reproduction, or when you need a minimal test case for a regression suite.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "bug-reproduction-test-generator, unit-test-generator"
---

# Test Case Reducer

Fuzzer found a crash with a 50KB JSON input. The bug is probably triggered by 3 bytes of it. Reduction finds those 3 bytes.

## The interestingness test

Everything hinges on a boolean oracle: **does this (smaller) input still trigger the bug?**

```python
def interesting(input_bytes):
    proc = subprocess.run(["./parser", "-"], input=input_bytes,
                          capture_output=True, timeout=5)
    return proc.returncode == -11   # SIGSEGV
```

Make it **specific**. "Crashes" is too loose — you might reduce to a *different* crash. Check for the same stack trace, same error message, same assertion.

## Delta debugging (ddmin) — the general algorithm

Works on any sequence (bytes, lines, list elements):

```
ddmin(input, n=2):
    Split input into n chunks.
    For each chunk: is input-minus-this-chunk still interesting?
        YES → recurse on the reduced input, reset n=2
    For each chunk: is this-chunk-alone interesting?
        YES → recurse on just this chunk, reset n=2
    No reduction at granularity n?
        n < len(input) → try n*=2 (finer chunks)
        else → done, input is 1-minimal
```

"1-minimal" means: removing any single element breaks the interestingness. Not globally minimal (that's NP-hard), but small enough.

## Structured shrinking — faster for known formats

Bytes-level ddmin on JSON wastes time producing syntactically invalid JSON. Shrink **at the grammar level:**

| Format   | Shrink operations                                           |
| -------- | ----------------------------------------------------------- |
| JSON     | Remove object keys; remove array elements; replace strings with `""`; replace numbers with `0`; replace objects with `null` |
| Source code | Remove statements; inline variables; simplify expressions (`a + b` → `a`); remove function params |
| Lists/trees | Remove children; collapse single-child nodes; replace subtrees with leaves |
| Integers | Try 0, 1, -1, value/2 — smaller magnitude                   |
| Strings  | Try `""`, first char, each half                             |

Hypothesis/QuickCheck do this automatically for inputs they generated. For external inputs, do it manually.

## Worked example

**Crash input (from fuzzer):**

```json
{"users": [
  {"id": 1, "name": "alice", "tags": ["a", "b", "c"], "meta": {"x": 1, "y": 2}},
  {"id": 2, "name": "bob", "tags": [], "meta": {"x": null}},
  {"id": 3, "name": "", "tags": ["d"], "meta": {}}
]}
```

**Interestingness:** `parser.parse(j)` raises `KeyError: 'y'`.

**Structured shrink:**

1. Remove users[0] → still crashes? **No.** Keep users[0].
2. Remove users[1] → still crashes? **Yes.** Drop it.
3. Remove users[2] → still crashes? **Yes.** Drop it.

Now: `{"users": [{"id": 1, "name": "alice", "tags": ["a","b","c"], "meta": {"x": 1, "y": 2}}]}`

4. Remove "tags" from users[0] → **Yes.** Drop.
5. Remove "name" → **Yes.** Drop.
6. Remove "id" → **Yes.** Drop.
7. Remove "meta" → **No.** Keep. ← *aha*
8. Remove "x" from meta → **No.** Keep.
9. Remove "y" from meta → **Yes.** Drop.

Now: `{"users": [{"meta": {"x": 1}}]}`

10. Replace `1` with `0` → **Yes.** `{"users": [{"meta": {"x": 0}}]}`
11. Can we simplify further? Remove the outer "users"? **No** — parser expects it. Done.

**Minimal:** `{"users": [{"meta": {"x": 0}}]}`

The bug: parser assumes `meta` has `"y"` when `"x"` is present. 50KB → 30 bytes. The bug is now *obvious.*

## Tools

| Target                   | Tool                                                  |
| ------------------------ | ----------------------------------------------------- |
| C/C++ source (compiler bugs) | `creduce`, `cvise`                                |
| Arbitrary text           | `halfempty`, `delta`                                  |
| Hypothesis-generated     | Built-in — `hypothesis.find` or just `@given` shrinking |
| AFL crashes              | `afl-tmin`                                            |
| Custom formats           | Write ddmin yourself — it's ~30 lines                 |

## Flaky interestingness

If the bug is non-deterministic (race condition, uninitialized memory), `interesting()` flips between True and False on the same input. Reduction goes haywire.

**Fix:** require N-out-of-M successes:

```python
def interesting(inp):
    crashes = sum(1 for _ in range(10) if _run_once(inp) == CRASH)
    return crashes >= 3   # crashes at least 30% of the time
```

Slower (10× the runs), but stable.

## Do not

- **Do not** use "any error" as the interestingness test. You'll reduce to a trivially malformed input that triggers a different (boring) error.
- **Do not** skip verifying the final reduced input. Run it manually, confirm it's the *same* bug.
- **Do not** reduce on a machine different from where the bug was found. Environment-dependent bugs (locale, filesystem, libc version) might not reproduce.
- **Do not** stop at "small enough to read." Keep going to 1-minimal — the last few removals often reveal *exactly* which field matters.
- **Do not** throw away intermediate reductions. If the full reduction takes hours and the machine dies, you want the 80%-reduced checkpoint.

## Output format

```
## Original input
Size: <bytes/lines/elements>
<truncated sample>

## Interestingness test
<exact predicate — error type, stack frame, message pattern>
Stability: <deterministic | N-of-M with N, M>

## Reduction trace
| Step | Operation | Size after | Still interesting |
| ---- | --------- | ---------- | ----------------- |

## Minimal input
Size: <bytes/lines/elements>  (reduction: <%>  from original)
<full input>

## Verified
<ran minimal input N times, triggered the bug M times>

## Bug hypothesis
<what the minimal input reveals about the bug>
```
