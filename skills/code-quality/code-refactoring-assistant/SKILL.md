---
name: code-refactoring-assistant
description: Executes refactorings — extract method, inline, rename, move — in small, behavior-preserving steps with a test between each. Use when the user wants to restructure working code, when cleaning up after a feature lands, or when a smell has been identified and needs fixing.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "code-smell-detector, behavior-preservation-checker, dead-code-eliminator"
---

# Code Refactoring Assistant

Refactoring is **changing structure without changing behavior.** The discipline: every step is small enough that if the tests break, you know exactly which change did it.

## Preconditions

| Precondition           | Why                                                  | If missing                           |
| ---------------------- | ---------------------------------------------------- | ------------------------------------ |
| Tests pass             | You need a green baseline to detect breakage         | Fix the tests first; don't refactor red |
| Tests cover the code   | Uncovered code = undetected breakage                 | Add characterization tests first     |
| You know the target    | Refactoring without a goal is churn                  | → `code-smell-detector` to pick a target |

## The catalog — mechanics per refactoring

| Refactoring         | When                                     | Mechanics                                                             | Gotcha                                        |
| ------------------- | ---------------------------------------- | --------------------------------------------------------------------- | --------------------------------------------- |
| Extract method      | Chunk of a long method has one job       | Copy out → add params for free vars → replace original with call      | Free vars: everything the chunk reads but doesn't define |
| Inline method       | Method body is clearer than its name     | Replace all calls with body → delete method                           | Only if callers ≤ 3 and no polymorphism       |
| Rename              | Name lies about what the thing does      | IDE rename (finds all refs). Manual rename = missed callers           | Strings, reflection, config files — IDE misses these |
| Move method         | Method uses another class's data more    | Copy to target → delegate from old → update callers → delete delegate | Do it in two commits: add+delegate, then remove delegate |
| Introduce parameter object | 4+ params that travel together    | New class with those fields → one call site at a time                 | Don't convert all call sites at once — one per commit |
| Replace conditional with polymorphism | Switch on type code     | One subclass per case → move each branch into its subclass            | Only if the switch repeats 3+ times — otherwise the switch is simpler |
| Extract class       | Class has two responsibilities           | New class → move one responsibility's fields+methods → delegate       | Don't move everything at once; one method per step |

## The rhythm

```
  run tests (green) → one small mechanical step → run tests → commit
                                   ↓ red?
                           revert that one step. rethink.
```

Every commit is a safe point. If you get interrupted, you're never more than one step from green.

## Worked example — Extract Method

**Before (`process_order`, 50 lines):**

```python
def process_order(order, user):
    # ... 20 lines ...
    subtotal = sum(item.price * item.qty for item in order.items)
    discount = 0
    if user.tier == "gold":
        discount = subtotal * 0.1
    elif user.tier == "silver":
        discount = subtotal * 0.05
    total = subtotal - discount
    # ... 20 more lines using `total` ...
```

**Step 1 — identify the chunk.** Lines computing `total` from `order.items` and `user.tier`. Free variables read: `order.items`, `user.tier`. Output: `total`.

**Step 2 — copy out, don't cut.**

```python
def _compute_total(items, tier):
    subtotal = sum(item.price * item.qty for item in items)
    discount = 0
    if tier == "gold":
        discount = subtotal * 0.1
    elif tier == "silver":
        discount = subtotal * 0.05
    return subtotal - discount
```

**Step 3 — test.** Run. Green? Good. (We haven't changed `process_order` yet — this is a pure addition.)

**Step 4 — replace in place.**

```python
def process_order(order, user):
    # ... 20 lines ...
    total = _compute_total(order.items, user.tier)
    # ... 20 more lines using `total` ...
```

**Step 5 — test.** Green → commit. Red → revert step 4, check: did you miss a free variable? Does `_compute_total` have a side effect the original inline code had?

## Pitfalls

- **Extract + rename in one step.** The extract might fail; the rename might fail. If both are in one step, you don't know which. One at a time.
- **"While I'm here" feature creep.** You're extracting a method and you notice a bug. Stop. Note the bug. Finish the refactor. Fix the bug in a *separate* commit.
- **Refactoring across a merge.** Big structural changes while a teammate has a feature branch open → merge hell. Coordinate.
- **Extracting a method that closes over mutable state.** The extracted function looks pure but it's reading `self.cache` — and the original code mutated `self.cache` three lines earlier. You just changed the ordering.

## Do not

- **Do not** refactor and change behavior in the same commit. Reviewers can verify "same behavior" or "correct new behavior" — not both at once.
- **Do not** refactor without tests. You have no way to know you preserved behavior. → `behavior-preservation-checker` if tests are thin.
- **Do not** do a "big bang" refactor — 15 extractions in one commit. When it breaks, bisecting inside one commit is impossible.
- **Do not** rename across module boundaries in languages with dynamic dispatch / duck typing without a grep pass. The IDE will miss call sites.

## Output format

When executing a refactor:

```
## Target
<smell/goal — why this refactor>

## Steps
1. <mechanical step>        → test → commit <sha>
2. <mechanical step>        → test → commit <sha>
...

## Safe points
Every commit above is independently revertible.

## Behavior check
<→ behavior-preservation-checker result, or: "tests green, N tests cover the changed region">
```
