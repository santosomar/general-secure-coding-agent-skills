---
name: semantic-bug-detector
description: Detects logical and semantic bugs by understanding program intent — catches issues that syntax-only tools miss. Use when static analysis has already run and found nothing, when the user reports incorrect behavior but no crash, or when reviewing algorithmic code for correctness.
license: Apache-2.0
metadata:
  category: "debugging"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "static-bug-detector, test-guided-bug-detector, bug-localization"
---

# Semantic Bug Detector

Find bugs where the code is syntactically fine and type-correct but **does the wrong thing**. → `static-bug-detector` catches provable errors; this skill catches violated intent.

## The intent question

You cannot find a semantic bug without a model of intent. For each function, establish intent from (in order of reliability):

1. A failing test (the assertion *is* the intent)
2. A docstring / type signature / name that makes a contract (`sort` should sort; `retry(n)` should try at most n+1 times)
3. An algorithm name (if the function is called `binarySearch`, you know what invariants it should hold)
4. Surrounding usage (what do callers assume about the return value?)

If you have none of these, you are not detecting bugs — you are generating hypotheses.

## Signal catalog — semantic smells that correlate with real bugs

| Smell                           | What it looks like                                           | Why it's suspicious                                             |
| ------------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------- |
| Copy-paste-mutate asymmetry     | Two near-identical blocks differ in one token                | The one token was supposed to change in *both* — or neither     |
| Condition-action mismatch       | `if x > limit: ... x`  — the condition checks one variable, the body mutates another | The variable in the condition is probably the one the author meant to modify |
| Loop variable ignored           | `for x in items:` but body never uses `x`                    | Either the loop is a `repeat n times` — or author meant to use `x` |
| Inverted boundary               | Sorted-collection logic that uses `<` where the surrounding comparisons use `<=` (or vice versa) | Boundary consistency is algorithmic; inconsistency is usually a bug |
| Mutation during iteration       | Modifying the collection being iterated                      | Skipped elements / ConcurrentModificationException — correct use is rare |
| State set, never read           | Field written in 3 places, read in 0                         | Either dead state — or the *read* is the bug (someone forgot to check it) |
| Integer division in float ctx   | `a / b` where `a`, `b` are ints but result feeds a float op  | Python 2 habits, C integer division — silent truncation         |
| Exception changes control flow silently | `try: ...; except: pass` followed by code that assumes `try` succeeded | The swallow masks the failure; the following code operates on garbage |

## Ranking

Unlike static bugs, semantic findings are **hypotheses** until confirmed. Rank by how cheaply they can be confirmed:

1. Can write a 3-line test → write it, then it's confirmed or killed
2. Named algorithm with known invariant → check invariant inline
3. Only evidence is asymmetry → flag as `needs-review`, don't assert

## FP suppression

- **Copy-paste asymmetry:** check git blame. If both blocks were written in the same commit, high confidence. If they diverged over time via separate edits, low — probably intentional.
- **Loop variable ignored:** check if `_` naming convention would apply. `for _ in range(n)` is intentional; `for item in items` with no `item` reference is suspicious.
- **Inverted boundary:** check the domain. Half-open intervals `[a, b)` legitimately mix `<=` and `<`.

## Worked example

**Code:**

```python
def apply_discounts(cart, coupons):
    total = sum(item.price for item in cart)
    for coupon in coupons:
        if coupon.type == "percent":
            total -= total * coupon.value / 100
        elif coupon.type == "fixed":
            total -= total * coupon.value / 100     # line 7
    return max(total, 0)
```

**Detection:** Copy-paste-mutate asymmetry. Lines 5 and 7 are identical, but the branch conditions differ. The `fixed` branch should subtract `coupon.value`, not `total * coupon.value / 100`.

**Confirmation:** Write a test — `cart total 100, one fixed coupon value 10 → expect 90, get 90`. Wait — it passes? Re-check: `100 - 100 * 10 / 100 = 90`. The bug is *masked* at `total=100`. Try `total=50`: expect `40`, get `45`. **Confirmed.**

**Finding:**

```
pricing.py:7  COPY_PASTE_ASYMMETRY  confidence=high (confirmed by test)
  Fixed-coupon branch applies percent formula. At total=50, fixed coupon of
  $10 discounts $5 instead of $10.

  Fix: `total -= coupon.value`

  Confirming test:
    cart(total=50), fixed_coupon(10) → assert total == 40
```

## Edge cases

- **The smell is the spec:** Sometimes the "asymmetry" is intentional domain logic you don't understand. If git blame shows different authors and the code has survived multiple releases, the prior is that it's correct.
- **Performance code:** `state set, never read` in a hot loop might be cache prefetching. `mutation during iteration` might be a carefully designed in-place algorithm. Lower confidence for anything marked `// PERF`.
- **Generated code:** Often full of semantic smells that are safe by construction. Skip.

## Do not

- **Do not** report a semantic finding with the same certainty as a static finding. Say "suspicious," not "bug," until confirmed.
- **Do not** assume the intent from the function name alone. `normalize()` means fifteen different things.
- **Do not** report the same underlying bug in every function it propagates to. One root, one finding.

## Output format

```
<file>:<line>  <SMELL_NAME>  confidence=<hypothesis|confirmed>
  <what the code does vs. what intent suggests it should do>
  <confirming evidence or test, or "needs manual review">
  <proposed fix if confident>
```
