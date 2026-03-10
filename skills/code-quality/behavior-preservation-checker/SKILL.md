---
name: behavior-preservation-checker
description: Verifies that a refactoring or transformation preserved observable behavior by comparing before and after execution, differential testing, or I/O capture. Use after a refactoring, after automated code transformation, before merging a structural PR, or whenever the claim is that two code versions do the same thing.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "semantic-equivalence-verifier, code-refactoring-assistant, multi-version-behavior-comparator"
---

# Behavior Preservation Checker

"It's just a refactor" is a claim. This skill checks the claim: **does the new code produce the same observable behavior as the old code** on the inputs that matter?

## Approaches — cheapest to strongest

| Approach                    | Checks                                         | Cost    | Confidence                             |
| --------------------------- | ---------------------------------------------- | ------- | -------------------------------------- |
| Run the existing tests      | Whatever the tests assert                      | Free    | As good as your test suite — often not very |
| Differential testing        | Old and new produce same output on random/prod inputs | Low   | High where you can enumerate inputs    |
| Golden-master / snapshot    | Output matches a recorded baseline byte-for-byte | Low   | Very high for serialized output; brittle |
| Side-effect capture         | Same DB writes, same HTTP calls, same log lines | Medium | Catches effects tests usually miss     |
| Property-based equivalence  | `∀x. old(x) == new(x)` over generated inputs   | Medium | High for pure functions                |
| Formal equivalence proof    | Proven equal by construction                   | High   | Absolute — → `semantic-equivalence-verifier` |

Use the cheapest one that gives you the confidence you need. Differential testing covers 90% of cases.

## Differential testing — the workhorse

```
for each input in <sample>:
    old_out = old_version(input)
    new_out = new_version(input)
    if old_out != new_out:
        REPORT divergence
```

Where `<sample>` comes from:
- **Production replay:** Captured real inputs. Best signal.
- **Existing test inputs:** What your tests already feed in. Cheap but narrow.
- **Fuzz / property generation:** Random inputs in the valid domain. Broad but can miss real-world shapes.

## What counts as "same behavior"

Decide **before** comparing — not after you see a diff:

| Observable                  | Must match?                                                        |
| --------------------------- | ------------------------------------------------------------------ |
| Return value                | Yes — by definition                                                |
| Exception type + message    | Type yes; message… usually yes but debatable                       |
| Side effects (DB, files, network) | Yes — this is where refactors silently break                  |
| Side-effect **order**       | Depends — was order specified, or incidental?                      |
| Log output                  | Usually no — logs are diagnostics, not contract                    |
| Timing / performance        | Usually no — unless that's the contract                            |
| Iteration order             | Depends — was it `dict` (unordered pre-3.7) or `list` (ordered)?   |
| Float precision             | Equal within ε, not bit-exact — define ε upfront                   |

Write down the equivalence relation. "Same return value, same DB writes (order-insensitive), ignore logs, floats within 1e-9."

## Worked example

**Change:** Refactored `compute_tax(order)` — was a 60-line method, now calls three helpers.

**Setup:** Both versions available — old commit checked out in a sibling worktree.

```python
# differential_test.py
from old.tax import compute_tax as old_compute
from new.tax import compute_tax as new_compute

def test_equivalence(sample_orders):         # 500 orders from prod snapshot
    for order in sample_orders:
        old = old_compute(order)
        new = new_compute(order)
        assert abs(old - new) < 0.001, f"diverged on {order.id}: {old} != {new}"
```

**Run:** 498 match. 2 diverge:
- Order #44291: old=`12.50`, new=`12.49`. Off by a cent.
- Order #81007: old=`0.00`, new=`0.00`. Wait — match? Rerun: `old` raised `KeyError` (test swallowed it). `new` returned `0.00`.

**Findings:**
1. The penny diff — new code rounds at a different step. Accumulation order changed. **Real behavior change.** Either a latent bug in old code (fixed accidentally) or a regression — need domain judgment.
2. Order #81007 — old code crashed on orders with no tax jurisdiction set; new code returns 0. **Real behavior change.** Probably an improvement, but it's not "just a refactor."

## Side-effect capture

For non-pure functions, return value isn't enough. Capture effects:

```python
with capture_sql() as old_queries:
    old_fn(x)
with capture_sql() as new_queries:
    new_fn(x)
assert normalize(old_queries) == normalize(new_queries)
```

`normalize` = sort if order doesn't matter, strip timestamps, etc. — per your equivalence relation.

## Edge cases

- **Old version had a bug:** Differential testing will flag the fix as a divergence. That's correct — it IS a behavior change. Report it; let the human decide it's a *desired* change.
- **Nondeterminism** (threads, random, time): Both versions produce different outputs run-to-run. Seed the RNG; freeze the clock; serialize the threads. If you can't, you can only compare distributions, not values.
- **Inputs with side effects** (reading an iterator exhausts it): Can't feed the same input to both. Clone/tee the input, or record-replay.
- **The refactor changed the signature:** Write an adapter so both versions take the same shape. The adapter is part of the refactor.

## Do not

- **Do not** accept "tests pass" as sufficient evidence for a large refactor. Tests cover the paths someone thought of. Production inputs cover the paths that actually happen.
- **Do not** decide what "equivalent" means after you see the diffs. You'll rationalize every divergence. Write the relation first.
- **Do not** ignore side-effect divergence because return values match. An extra DB write is a behavior change.
- **Do not** treat a divergence as automatically a bug. Sometimes the old behavior was the bug. But it IS a change, and the PR shouldn't claim "no behavior change."

## Output format

```
## Equivalence relation
<return values | side effects | what's compared, what's ignored>

## Sample
<N> inputs from <source>

## Result
<N-k> equivalent
<k> divergent:
  input=<summary>  old=<val>  new=<val>
    <verdict: regression | latent-bug-fix | incidental | needs-review>

## Confidence
<high | medium | low — based on sample coverage>
```
