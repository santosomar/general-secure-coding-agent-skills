---
name: test-oracle-generator
description: Generates test oracles — the "expected output" part of a test — by choosing among reference implementations, invariants, inverse functions, or differential comparison when the correct answer isn't obvious. Use when the hard part of testing is knowing what the right answer is, not generating inputs.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "unit-test-generator, metamorphic-test-generator"
---

# Test Oracle Generator

An oracle answers: "is this output correct?" For `add(2, 3)`, the oracle is trivial: `== 5`. For `optimize_route(cities)`, what's the right answer? The oracle is the hard part.

## Oracle types — pick the cheapest that works

| Oracle type              | How it works                                          | When available                             |
| ------------------------ | ----------------------------------------------------- | ------------------------------------------ |
| **Known value**          | Hardcoded expected output                             | Small inputs you can compute by hand       |
| **Reference implementation** | Compare against a trusted other implementation    | A slow/simple version exists               |
| **Inverse function**     | `decode(encode(x)) == x`                              | There's a round-trip                       |
| **Invariant / property** | Output satisfies a predicate                          | You know properties, not values            |
| **Metamorphic relation** | Multiple runs relate in a known way                   | → `metamorphic-test-generator`             |
| **Differential**         | N implementations should agree                        | Multiple implementations exist             |
| **Regression (golden)**  | Output matches a saved previous output                | You trust the current behavior             |

## Decision flow

```
Can you compute the answer by hand for small inputs?
│
├─ YES → Known-value oracle for those inputs.
│        Still need something for random/large inputs — continue ↓
│
Is there a simpler/slower implementation that's obviously correct?
│
├─ YES → Reference implementation oracle.
│        `assert fast(x) == slow_obvious(x)` for random x.
│
Is there an inverse?  (encode/decode, compress/decompress, serialize/parse)
│
├─ YES → Round-trip oracle.  `assert decode(encode(x)) == x`
│
Do you know properties the output must have, even if not the exact value?
│
├─ YES → Invariant oracle.
│        `result = sort(x); assert is_sorted(result) and is_permutation(result, x)`
│
None of the above?
│
└─ Regression oracle (golden files) as a last resort.
   Captures current behavior — not correctness.
```

## Worked example — reference implementation

**Under test:** `fast_median(nums)` — O(n) quickselect-based.

**Oracle:** The obvious O(n log n) version:

```python
def slow_median(nums):
    s = sorted(nums)
    n = len(s)
    return s[n // 2] if n % 2 else (s[n//2 - 1] + s[n//2]) / 2

from hypothesis import given, strategies as st

@given(st.lists(st.integers(), min_size=1))
def test_fast_matches_slow(nums):
    assert fast_median(nums) == slow_median(nums)
```

`slow_median` is three lines and obviously correct. `fast_median` is 40 lines of partitioning. Any disagreement is a bug in `fast_median`.

## Worked example — invariant oracle

**Under test:** `schedule(tasks, workers)` — assigns tasks to workers, minimizing makespan. NP-hard; you can't compute the optimal answer.

**What you *can* check:**

```python
@given(tasks=task_lists(), workers=st.integers(1, 10))
def test_schedule_is_valid(tasks, workers):
    assignment = schedule(tasks, workers)

    # Every task assigned exactly once
    assigned = [t for w in assignment.values() for t in w]
    assert sorted(assigned) == sorted(tasks)

    # No worker index out of range
    assert all(0 <= w < workers for w in assignment)

    # Makespan is no worse than the trivial round-robin
    # (not optimal — but if we're worse than round-robin, something's very wrong)
    trivial = makespan(round_robin(tasks, workers))
    assert makespan(assignment) <= trivial
```

Three invariants. None of them is "the answer is X." All of them catch real bugs.

## Worked example — inverse

**Under test:** `serialize(obj) -> bytes` and `deserialize(bytes) -> obj`.

```python
@given(arbitrary_objects())
def test_roundtrip(obj):
    assert deserialize(serialize(obj)) == obj
```

One line. Tests both functions against each other. Catches: field dropped in serialize, wrong type on deserialize, encoding mismatches.

## Regression (golden) — the weakest oracle

When nothing else works: run once, save the output, assert future runs match.

```python
def test_render_matches_golden():
    output = render(template, data)
    golden_path = Path("test/golden/render.txt")
    if UPDATE_GOLDEN:
        golden_path.write_text(output)
    assert output == golden_path.read_text()
```

**This tests stability, not correctness.** The first golden might be wrong. Use only when:
- You've manually verified the golden is correct, once.
- The function is mostly-frozen (change = deliberate).
- Better oracles are genuinely unavailable.

## Do not

- **Do not** use the code-under-test as its own oracle. `assert f(x) == f(x)` tests nothing. Even subtler: reference implementations that share a buggy helper with the fast version.
- **Do not** default to golden files. They test "same as yesterday," not "correct." A bug that was always there stays there.
- **Do not** write invariants so weak they never fail. `assert len(result) >= 0` — always true, useless.
- **Do not** forget that reference implementations can be wrong too. `slow_median` above — does it handle the even-length case right? (It does. But check.)

## Output format

```
## Under test
<function — and why the oracle is non-trivial>

## Oracle type
<known-value | reference | inverse | invariant | metamorphic | differential | golden>

## Why this oracle
<decision-flow reasoning — what cheaper oracles were unavailable>

## Oracle implementation
<code — the reference impl, the invariant predicates, the round-trip, etc>

## Test
<code — uses the oracle>

## Oracle validity
<why you trust the oracle — it's simpler, it's from a different codebase, the invariant is from the spec>
```
