---
name: test-deduplicator
description: Finds and removes redundant tests — tests that cover the same code, kill the same mutants, or assert the same behavior — to shrink suite runtime without losing coverage. Use when the test suite is slow, when tests have accumulated over years of copy-paste, or when CI costs are too high.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "test-suite-prioritizer, mutation-test-suite-optimizer"
---

# Test Deduplicator

Ten tests cover the same branch. Nine of them can go. But which nine? The one you keep is the one that covers something the others don't.

## What makes two tests "the same"

| Sameness level                | Evidence                                                     | Deletion confidence |
| ----------------------------- | ------------------------------------------------------------ | ------------------- |
| **Textual clone**             | Near-identical code, one value different                     | Low — might test different boundaries |
| **Same coverage**             | Per-test coverage sets are equal                             | Medium              |
| **Same coverage, subsumes**   | Test A's coverage ⊇ Test B's                                 | High — B is redundant |
| **Same mutants killed**       | Mutation testing: A and B kill identical mutant sets         | High                |
| **Same assertions, same inputs** | Literally the same test, different name                    | Very high — copy-paste accident |

Coverage subsumption is the key signal. If test A covers everything test B covers and more, B adds nothing.

## Step 1 — Build per-test coverage

```bash
# Python — run each test in isolation, collect coverage
pytest --cov=src --cov-context=test --cov-report=
coverage json -o coverage.json
# coverage.json now has per-context (per-test) line data
```

```bash
# Java — JaCoCo per-test is harder; use test-impact tools or:
# Run suite once, instrument to log (test_name, covered_line) pairs
```

Output: `{test_name: set(covered_lines)}`

## Step 2 — Find subsumption

```python
# A subsumes B if coverage[A] >= coverage[B]  (superset)
subsumed = []
tests = sorted(coverage, key=lambda t: len(coverage[t]), reverse=True)
for i, a in enumerate(tests):
    for b in tests[i+1:]:
        if coverage[a] >= coverage[b]:   # a's coverage is a superset
            subsumed.append((b, a))      # b is subsumed by a
```

This is O(n²) set comparisons. Fine for a few thousand tests. For tens of thousands, sort by size and prune.

## Step 3 — Greedy minimum cover

Subsumption alone misses partial overlap. For the **minimum** set of tests with the same total coverage, use greedy set cover:

```python
covered = set()
keep = []
remaining = sorted(coverage, key=lambda t: len(coverage[t]), reverse=True)
total = set().union(*coverage.values())

for t in remaining:
    new_lines = coverage[t] - covered
    if new_lines:
        keep.append(t)
        covered |= coverage[t]
    if covered == total:
        break

delete_candidates = set(coverage) - set(keep)
```

Greedy isn't optimal (set cover is NP-hard) but it's within ln(n) of optimal and runs in seconds.

## Step 4 — Don't just delete — verify

Coverage equivalence ≠ behavioral equivalence. Two tests can cover the same lines but assert different things:

```python
def test_parse_int():
    assert parse("42") == 42         # covers lines 10-15

def test_parse_int_leading_zero():
    assert parse("042") == 42        # covers lines 10-15 — same coverage!
```

Same coverage. Different inputs. The second might catch a bug the first doesn't (octal interpretation, anyone?).

**Before deleting, check mutation-kill equivalence:** run mutation testing (→ `mutation-test-suite-optimizer`) on the delete candidates. If a candidate kills a mutant no keeper kills, it's not redundant.

## Parametrize instead of delete

Often the "duplicates" are value variations:

```python
def test_discount_gold():    assert discount(100, "gold") == 80
def test_discount_silver():  assert discount(100, "silver") == 90
def test_discount_bronze():  assert discount(100, "bronze") == 95
def test_discount_none():    assert discount(100, "none") == 100
```

Four tests, near-identical structure. Don't delete three — **parametrize:**

```python
@pytest.mark.parametrize("tier,expected", [
    ("gold", 80), ("silver", 90), ("bronze", 95), ("none", 100),
])
def test_discount(tier, expected):
    assert discount(100, tier) == expected
```

Same coverage. Same assertions. One-quarter the code. Still four test cases.

## Tests that look redundant but aren't

| Pattern                                            | Why not redundant                         |
| -------------------------------------------------- | ----------------------------------------- |
| Same function, one mocks DB, one hits real DB      | Unit vs integration — different failure modes |
| Same logic, different input sizes (1 vs 10000)     | The big one catches O(n²) performance bugs |
| Same coverage, one asserts exception message       | Error-path granularity                    |
| Flaky test + reliable test covering same thing     | Delete the flaky one, keep the reliable one — but this is a flake fix, not dedup |

## Do not

- **Do not** trust coverage equivalence alone. Same lines ≠ same assertions. Mutation-kill equivalence is the real test.
- **Do not** delete tests in a single commit without a revert plan. Batch deletions, run full suite between batches, easy rollback.
- **Do not** delete slow tests just because a fast test has the same coverage. If the slow test is an integration test, it catches integration bugs the fast one can't.
- **Do not** delete tests that *document* behavior even if they're covered elsewhere. `test_empty_list_returns_empty` is worth keeping for the *name*, even if `test_various_inputs` also covers it.
- **Do not** skip the before/after coverage diff. If coverage dropped, you deleted something that wasn't redundant.

## Output format

```
## Suite before
Tests: <N>  Runtime: <s>  Coverage: <%> line, <%> branch

## Subsumption
| Subsumed test | Subsumed by | Coverage delta |
| ------------- | ----------- | -------------- |

## Greedy minimum cover
Keep: <N> tests → same total coverage
Delete candidates: <M> tests

## Mutation verification
| Candidate | Mutants only this test kills | Keep? |
| --------- | ---------------------------- | ----- |

## Parametrization opportunities
| Tests | Merged into | Cases |
| ----- | ----------- | ----- |

## Final
| Action | Count |
| ------ | ----- |
| Delete (fully subsumed, no unique mutants) | <N> |
| Parametrize | <N> groups → <M> tests |
| Keep (looked redundant, isn't) | <N> |

## After
Tests: <N>  Runtime: <s>  Coverage: <%> (unchanged)  Mutation score: <%> (unchanged)
```
