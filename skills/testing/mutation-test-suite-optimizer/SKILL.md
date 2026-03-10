---
name: mutation-test-suite-optimizer
description: Uses mutation testing to find weak assertions and missing tests — injects small bugs and checks if the suite catches them, then generates tests targeting the surviving mutants. Use when coverage is high but bugs still ship, when auditing test quality, or when deciding if the suite is good enough.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "smart-mutation-operator-generator, coverage-enhancer, test-oracle-generator"
---

# Mutation Test Suite Optimizer

Coverage says "this line ran." Mutation testing says "this line ran AND if it were wrong, a test would catch it." Surviving mutants are lines where the test suite is **blind.**

## How it works

1. **Mutate:** Apply a small change to the code. `>` → `>=`. `+` → `-`. `return x` → `return None`.
2. **Run tests:** Does any test fail?
3. **Killed** if a test fails — the suite noticed the bug. **Survived** if all pass — the suite is blind here.
4. **Mutation score** = killed / total. Higher is better. > 80% is good.

## Standard mutation operators

| Operator       | Mutation                              | Exposes                               |
| -------------- | ------------------------------------- | ------------------------------------- |
| AOR (arithmetic) | `+` ↔ `-`, `*` ↔ `/`                 | Tests that don't check actual values  |
| ROR (relational) | `<` ↔ `<=` ↔ `==` ↔ `!=` ↔ `>=` ↔ `>` | Off-by-one, boundary untested       |
| COR (conditional) | `and` ↔ `or`, negate condition      | Branches where only one arm matters   |
| LVR (literal value) | `0` → `1`, `1` → `0`, `"x"` → `""` | Magic numbers with no assertion     |
| SDL (statement delete) | Remove a line                   | Dead code, or unchecked side effects  |
| RVR (return value) | `return x` → `return None`, `return 0` | Caller ignores return value    |

→ `smart-mutation-operator-generator` for domain-specific mutations beyond these.

## Step 1 — Run the mutation tool

| Ecosystem | Tool                                                |
| --------- | --------------------------------------------------- |
| Python    | `mutmut`, `cosmic-ray`                              |
| Java      | `pitest`                                            |
| JS/TS     | `stryker`                                           |
| Ruby      | `mutant`                                            |
| C/C++     | `mull`, `dextool mutate`                            |

These take hours on big codebases. Scope to changed files: `mutmut run --paths-to-mutate src/pricing.py`.

## Step 2 — Triage survivors

Not every survivor needs a test. Classify:

| Survivor type                        | Action                                                |
| ------------------------------------ | ----------------------------------------------------- |
| **Equivalent mutant**                | `x = x * 1` → `x = x / 1` — same behavior. Ignore.   |
| **Dead code mutant**                 | Mutated line never runs. → `dead-code-eliminator`    |
| **Weak assertion**                   | Test ran the line but didn't check the result. **Fix the test.** |
| **Missing boundary**                 | `<` vs `<=` both pass — never tested the boundary. **Add test.** |
| **Unchecked side effect**            | Mutant deletes a log call, nothing notices. **Decide: is this worth testing?** |

## Worked example — killing a survivor

**Code:**

```python
def discount(price, tier):
    if tier == "gold":
        return price * 0.8
    return price
```

**Mutation report:**

```
SURVIVED: discount.py:3 — `price * 0.8` → `price * 0.9`
```

**Existing test:**

```python
def test_gold_discount():
    assert discount(100, "gold") < 100   # ← too weak
```

Both `0.8` and `0.9` give something `< 100`. The test is **imprecise.**

**Fix — strengthen the assertion:**

```python
def test_gold_discount():
    assert discount(100, "gold") == 80   # 20% off, exactly
```

Now `* 0.9` → `90 != 80` → test fails → mutant killed.

**Another survivor:**

```
SURVIVED: discount.py:2 — `tier == "gold"` → `tier != "gold"`
```

Only test is gold. The non-gold path is covered (return price) but the *condition* isn't — both `==` and `!=` give the right answer *for this one input.* Need a second input:

```python
def test_non_gold_no_discount():
    assert discount(100, "silver") == 100
```

Now `!=` would give silver → `0.8 * 100 = 80 != 100` → killed.

## Equivalent mutants — don't fight them

Some mutants can't be killed because they're **behaviorally identical:**

```python
for i in range(len(xs)):   # mutant: range(len(xs)) → range(0, len(xs))
```

`range(n)` and `range(0, n)` are the same. No test can distinguish them. Mark as equivalent and move on.

Detecting equivalence is undecidable in general. Heuristics: if you've spent 5 minutes trying to kill a mutant and every test you write passes on both, it's probably equivalent.

## Budget — mutation testing is slow

Full mutation on a big codebase: hours to days. Scope it:

- **Changed files only** on every PR.
- **Weekly full run** on the whole codebase.
- **Sample:** mutate 10% of operators randomly — statistical estimate of mutation score.
- **Timeout per mutant:** if a mutant makes tests hang, kill it (counts as killed — it changed behavior, even if the change is "infinite loop").

## Do not

- **Do not** chase 100% mutation score. 20–30% of survivors are equivalent mutants. 80% is excellent.
- **Do not** kill mutants with assertions that test the implementation. `assert discount.__code__.co_consts[1] == 0.8` kills the mutant and is a terrible test.
- **Do not** mutate test code. Mutate source; run tests. Mutating tests is circular.
- **Do not** treat SDL (statement delete) survivors on logging/metrics as bugs. Yes, deleting `logger.info(...)` doesn't fail any test. No, you probably don't want to assert on every log line.

## Output format

```
## Mutation run
Tool: <mutmut/pitest/stryker>  Scope: <files>
Mutants: <total>  Killed: <N>  Survived: <M>  Timeout: <T>  Score: <%>

## Survivors — triaged
### Weak assertions (fix the test)
| Mutant | Location | Existing test | Why it survived | Fixed assertion |
| ------ | -------- | ------------- | --------------- | --------------- |

### Missing tests (add a test)
| Mutant | Location | Missing case | New test |
| ------ | -------- | ------------ | -------- |

### Equivalent (ignore)
| Mutant | Why equivalent |
| ------ | -------------- |

### Dead code (remove)
| Mutant | Evidence |
| ------ | -------- |

## After fixes
Projected score: <%>  (killed +<N>, marked equivalent +<M>)
```
