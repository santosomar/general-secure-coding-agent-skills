---
name: code-smell-detector
description: Identifies code smells — structural patterns that correlate with maintainability problems — and explains why each matters in context. Use when reviewing a PR for structural quality, when the user asks what's wrong with a piece of code that isn't buggy, or when prioritizing refactoring targets.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "technical-debt-analyzer, code-refactoring-assistant, design-pattern-suggestor"
---

# Code Smell Detector

A smell is not a bug. It's a **structural property that makes bugs more likely later.** The code works today; the smell is about tomorrow.

## Smell catalog — ranked by how often they actually matter

| Smell                    | Detection heuristic                                       | Why it matters                                          | Fix                                  |
| ------------------------ | --------------------------------------------------------- | ------------------------------------------------------- | ------------------------------------ |
| Long method              | > 40 lines, > 3 levels of nesting                         | Can't hold it in your head; every change risks the whole thing | Extract method                  |
| God class                | > 15 methods, > 10 fields, or depends on > 8 other classes | Every change touches this file; merge conflicts; nobody understands it all | Split by responsibility         |
| Feature envy             | Method reads 3+ fields of another object, 0 of its own     | The method is in the wrong class                        | Move method                          |
| Shotgun surgery          | One conceptual change → edits in 5+ files                 | High coupling; change is expensive and error-prone      | Consolidate the scattered concern    |
| Primitive obsession      | Three strings passed together (`street, city, zip`) everywhere | The missing type *is* the abstraction you need        | Introduce parameter object / value type |
| Boolean blindness        | Method takes 3+ booleans; call sites look like `f(true, false, true)` | Nobody knows what `true` means at the call site    | Named parameters, enum, or config object |
| Speculative generality   | Abstract class with one concrete subclass; interface with one impl; hook method nobody overrides | Complexity paid for, never used    | Inline it → `dead-code-eliminator`   |
| Data clump               | Same 3+ variables always appear together in signatures    | Missing type (same as primitive obsession, across methods) | Introduce a class                |
| Message chain            | `a.getB().getC().getD().doThing()`                        | Caller knows the entire object graph; any link change breaks it | Tell-don't-ask; hide the chain  |
| Duplicated code          | Two regions > 6 lines, > 80% similar                      | Bug fixed in one, not the other                         | Extract; but see FP note below       |

## Ranking — which smells to surface first

Don't dump all findings. Rank by **change frequency × smell severity**:

1. God class that's edited weekly — urgent. Every week of delay costs more conflicts.
2. Long method in a file untouched for 2 years — ignore. It's ugly but stable.
3. Duplicated code where both copies are in hot files — high. The divergence-bug will happen.

Use `git log --format= --name-only | sort | uniq -c | sort -rn` to get change frequency. Smells in cold code are academic.

## False-positive suppression

| Smell flagged            | But actually fine when…                                                    |
| ------------------------ | -------------------------------------------------------------------------- |
| Long method              | It's a flat sequence of independent steps (setup, setup, setup — a script) |
| God class                | It's the app's composition root / DI container — by design, it knows everything |
| Duplicated code          | The two copies have different *reasons to change*. "Coincidental duplication" should NOT be extracted — you'll couple two unrelated concerns |
| Speculative generality   | It's a plugin interface and the second impl is on the roadmap this quarter |
| Message chain            | The chain is navigating a well-known, stable schema (`config.database.pool.size`) |

## Worked example

**Code:**

```python
class OrderProcessor:
    def process(self, order, user, warehouse, shipper, payment, notifier,
                audit, retry, dry_run):                              # 9 params
        if not dry_run:
            if payment.charge(user.card, order.total):
                if warehouse.has_stock(order.items):
                    warehouse.reserve(order.items)
                    if shipper.can_deliver(user.address.zip):        # message chain
                        label = shipper.create_label(user.address.street,
                                                     user.address.city,
                                                     user.address.zip)  # data clump
                        # ... 40 more lines ...
```

**Findings:**

1. **Long method** (60 lines, 4 nesting levels) — `process()` does validation, charging, reservation, shipping, notification, auditing. Six responsibilities.
2. **Boolean blindness** — `retry, dry_run` as positional booleans. Call sites: `processor.process(o, u, w, s, p, n, a, True, False)`. Which one is `dry_run`?
3. **Data clump** — `street, city, zip` always travel together. There's an `Address` object right there (`user.address`) — pass it whole.
4. **Message chain** — `user.address.zip`. Minor here (Address is stable), but if `shipper` took an `Address`, this goes away with fix #3.

**Prioritization:** `git log` shows `OrderProcessor` edited 14 times last quarter. Long method + high churn → extract first.

## Do not

- **Do not** report a smell without saying *why it matters here*. "This is a long method" is lint output. "This is a long method that 4 people edited last month and each edit touched a different 10-line region" is a finding.
- **Do not** extract coincidental duplication. If two copies look similar but serve different domains, they'll diverge — and then your shared helper has a boolean parameter and you're worse off.
- **Do not** flag smells in generated code, test fixtures, or migration scripts. Different rules.
- **Do not** recommend the fix without → `code-refactoring-assistant` for the mechanics. This skill finds; that one fixes.

## Output format

```
<file>:<line>  <SMELL>  churn=<commits-last-90d>
  <what the structure is>
  <why it will cost you — specific to this instance>
  <suggested refactoring — one sentence>
  <false-positive check: does the suppression rule apply?>

---
Ranked by (churn × severity). Fix the top 3; ignore the rest until they climb.
```
