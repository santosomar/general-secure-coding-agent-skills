---
name: test-driven-generation
description: Generates code test-first — writes a failing test from a requirement, then generates the minimal code to pass it, then refactors, in strict red-green-refactor cycles. Use when building new features where the spec is clear, when the design is uncertain and you want tests to drive it, or when you need high confidence in coverage from the start.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "unit-test-generator, scenario-generator"
---

# Test-Driven Generation

TDD isn't "write tests too." It's **test before code, minimal code to pass, repeat.** Each cycle is tiny. The tests design the API.

## The cycle — do not skip steps

```
┌─────────────────────────────────────────────────┐
│  RED: Write ONE failing test.                   │
│       Run it. Confirm it fails for the RIGHT    │
│       reason (assertion, not ImportError).      │
└──────────────────┬──────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────┐
│  GREEN: Write the MINIMUM code to pass.         │
│         Hardcoding is fine. Duplication is fine.│
│         Just make it green.                     │
└──────────────────┬──────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────┐
│  REFACTOR: Clean up. Tests stay green.          │
│            NOW remove duplication.              │
│            NOW generalize the hardcoded value.  │
└──────────────────┬──────────────────────────────┘
                   │
                   └──► back to RED with next test
```

One test at a time. One small step of code. The discipline is the point.

## Red — the test drives the API

Write the test as if the code exists. The test is the first **user** of the API:

```python
def test_cart_empty_total_is_zero():
    cart = Cart()
    assert cart.total() == 0
```

`Cart` doesn't exist. `total()` doesn't exist. That's fine — the test just **decided** that `Cart()` takes no args and `total()` returns a number. Design through usage.

**Run it.** Fails: `NameError: Cart`. Wrong failure — we want `AssertionError`, not `NameError`. Create the shell:

```python
class Cart:
    def total(self):
        pass
```

Run again: `assert None == 0` — fails. **Right failure.** Now we're red.

## Green — embarrassingly minimal

```python
class Cart:
    def total(self):
        return 0
```

`return 0`. Hardcoded. Passes. **Resist** the urge to write `return sum(item.price for item in self.items)`. You have one test. It wants 0. Give it 0.

Why? Because the *next* test forces the generalization, and you end up with exactly as much code as the tests demand — no more.

## Triangulation — the next test forces generality

```python
def test_cart_with_one_item_total_is_item_price():
    cart = Cart()
    cart.add(Item(price=10))
    assert cart.total() == 10
```

Now `return 0` fails. Now we need `items`. Now we generalize:

```python
class Cart:
    def __init__(self):
        self._items = []
    def add(self, item):
        self._items.append(item)
    def total(self):
        return sum(i.price for i in self._items)
```

Both tests green. The generalization was **forced** by the second test, not guessed from the first.

## Refactor — with a safety net

Green means refactor is safe — tests will catch breakage. Extract methods, rename, restructure. Run tests after each micro-change.

```python
# Maybe later: total grows complex with discounts, tax.
# Refactor when green:
def total(self):
    subtotal = self._subtotal()
    return subtotal - self._discount(subtotal) + self._tax(subtotal)
```

Tests still green → refactor is safe. Test went red → revert, try smaller step.

## Worked cycle transcript

**Requirement:** Rate limiter — max N calls per window, returns True if allowed.

```
RED   test_first_call_allowed
      → assert limiter.allow("user1") is True
      → fails: RateLimiter doesn't exist

GREEN class RateLimiter: def allow(self, k): return True
      → passes (always True is enough for one test)

RED   test_call_over_limit_denied
      → limiter = RateLimiter(max_calls=2, window_s=60)
      → limiter.allow("u"); limiter.allow("u"); assert limiter.allow("u") is False
      → fails: returns True (hardcoded)

GREEN Add self._counts dict. Track. return count < max.
      → passes. Two tests green.

RED   test_different_keys_independent
      → limiter.allow("a"); limiter.allow("a"); assert limiter.allow("b") is True
      → already passes! (dict is per-key). Skip — not a red test.

RED   test_window_resets
      → fill limit at t=0. At t=61, allow again.
      → fails: no time logic.

GREEN Inject clock. Store timestamps. Prune old.
      → passes.

REFACTOR  _counts is getting messy. Extract _prune_expired().
          Tests green throughout.
```

Four tests. Each added exactly the code needed. No `window_resets` logic existed until a test demanded it.

## Anti-patterns

| Anti-pattern                                  | Why bad                                           | Instead                                    |
| --------------------------------------------- | ------------------------------------------------- | ------------------------------------------ |
| Write 10 tests, then all the code             | No red-green feedback. Tests might all be wrong.  | One at a time.                             |
| Write code, then tests that pass              | That's test-after. Tests fit the code, not spec.  | Test first. Always.                        |
| Skip running the red test                     | You don't know if it can fail.                    | Run it. See it fail. See *why* it fails.   |
| Over-implement on green                       | Untested code. Speculative generality.            | Minimum to pass. Next test generalizes.    |
| Refactor on red                               | No safety net. Can't tell refactor bugs from feature bugs. | Only refactor on green.          |
| Write a test you can't make fail              | Test is tautological or already covered.          | Skip it or delete it.                      |

## Do not

- **Do not** write the test and the code in the same step. The red confirmation is non-negotiable — it proves the test *can* fail.
- **Do not** generalize before triangulation. `return 0` passing one test is correct TDD. `return sum(...)` passing one test is speculation.
- **Do not** let the test tell you *how* to implement. `assert cart._items == [item]` tests structure. `assert cart.total() == 10` tests behavior. Test behavior.
- **Do not** skip refactor because "it works." Green without refactor accumulates cruft. Refactor is one-third of the cycle.
- **Do not** TDD through a big design problem without sketching first. TDD discovers design through small steps; it doesn't replace thinking about the large shape.

## Output format

```
## Requirement
<feature being built>

## Test list (planned — gets revised as you go)
- [ ] <test 1 — simplest case>
- [ ] <test 2 — next simplest>
- [ ] <test 3 — first interesting constraint>
- ...

## Cycle log
### Cycle 1
RED:   <test code>  — fails with: <failure>
GREEN: <minimal impl>  — passes
REFACTOR: <what changed, or "none">

### Cycle 2
...

## Final
Tests: <N>  All green.
Coverage: <%>  (should be ~100% — every line was demanded by a test)

## Deferred
<tests on the list that turned out unnecessary or out of scope>
```
