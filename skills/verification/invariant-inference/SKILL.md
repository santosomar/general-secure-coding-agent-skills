---
name: invariant-inference
description: Infers likely loop invariants and function contracts by observing execution traces, synthesizing candidates, and checking them inductively. Use when a verifier rejects a loop because the invariant is missing or too weak, when a Daikon-style tool is needed, or before translating code to a verification language.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "abstract-invariant-generator, python-to-dafny-translator"
---

# Invariant Inference

A loop invariant is **what stays true every time you hit the loop head.** Verifiers need it stated; programmers rarely state it. This skill guesses it from evidence.

## Two routes to an invariant

| Route              | How it works                                          | Good at                         | Bad at                          |
| ------------------ | ----------------------------------------------------- | ------------------------------- | ------------------------------- |
| Dynamic (Daikon-style) | Run the code, record state at loop head, generalize patterns | Concrete relationships in traces | Generalization — might be coincidence |
| Static (template-based) | Guess invariants of form `ax + by ≤ c`, solve for `a,b,c` | Sound by construction           | Limited to the template shapes |

Do both: dynamic finds candidates fast; static checks them.

## Dynamic — the observe-and-generalize loop

1. **Instrument.** At every loop head, dump all local variable values.
2. **Run** on a bunch of inputs. The more diverse, the better.
3. **Mine patterns** across the dumps. What's true at *every* observation?
4. **Filter for inductiveness.** A pattern true by coincidence won't survive the inductive check.

**Template library** (what to look for in the dumps):

| Template                    | Example                              |
| --------------------------- | ------------------------------------ |
| Constant                    | `n == 10` in every observation       |
| Range                       | `0 <= i <= n`                        |
| Linear relation             | `i + j == n`, `2*i == k`             |
| Ordering                    | `i < j`, `a[i] <= a[j]`              |
| Membership                  | `x in {0, 1, 2}`                     |
| Collection property         | `sorted(a[:i])`, `sum(a[:i]) == s`   |
| Implication                 | `flag => (x > 0)`                    |

## Step-by-step

1. **Collect traces.** Run with varied inputs; dump `(iter, locals)` at loop head.
2. **Enumerate candidates.** For each template, instantiate with the observed variables. Keep any that hold on *all* traces.
3. **Prune coincidences.** If `x == 5` in every trace but your test inputs all had `n == 5` — it's not invariant, it's your test bias. Vary `n`.
4. **Check inductiveness.** For each survivor `I`: does `I ∧ guard ∧ body ⟹ I'`? Use an SMT solver or just plug into the verifier.
5. **Check sufficiency.** Does `I ∧ ¬guard ⟹ postcondition`? If not, `I` is true but useless — you need a stronger one.

## Worked example

**Loop:**

```python
def find_max(a):
    m = a[0]
    i = 1
    while i < len(a):
        if a[i] > m:
            m = a[i]
        i += 1
    return m    # post: m == max(a)
```

**Traces** (input `a = [3, 1, 4, 1, 5]`):

| iter | i | m | a[:i]       |
| ---- | - | - | ----------- |
| 0    | 1 | 3 | [3]         |
| 1    | 2 | 3 | [3, 1]      |
| 2    | 3 | 4 | [3, 1, 4]   |
| 3    | 4 | 4 | [3, 1, 4, 1]|
| 4    | 5 | 5 | [3,1,4,1,5] |

**Candidate patterns** (mined):

- `1 <= i <= len(a)` ✓ all rows
- `m in a` ✓ all rows
- `m == max(a[:i])` ✓ all rows — **this is the one**
- `m >= a[0]` ✓ all rows — true but weaker than the above

**Inductive check on `m == max(a[:i])`:**
- Init: `i=1, m=a[0]` → `max(a[:1]) = a[0] = m`. ✓
- Preserved: Assume `m == max(a[:i])`. Body sets `m' = max(m, a[i])` and `i' = i+1`. Then `m' = max(max(a[:i]), a[i]) = max(a[:i+1]) = max(a[:i'])`. ✓
- Sufficient: Exit when `i == len(a)`. Then `m == max(a[:len(a)]) == max(a)`. ✓

**Invariant:** `1 <= i <= len(a) ∧ m == max(a[:i])`.

## When dynamic fails

- **Not enough trace diversity:** All your inputs hit the same branch → you miss the invariant for the other branch. Fix: coverage-guided input generation.
- **The invariant isn't in your template library:** `a[i] * a[j] < n` — nonlinear. → `abstract-invariant-generator` with a polynomial domain, or hand-write.
- **Ghost state needed:** The invariant mentions a quantity the code doesn't compute (e.g., "the number of swaps so far"). Add a ghost variable.

## Do not

- **Do not** trust a candidate that passes on traces but fails the inductive check. Traces are a finite sample. The inductive check is the proof.
- **Do not** stop at the first invariant that's inductive. Check it's *sufficient* for the postcondition. `true` is inductive; it proves nothing.
- **Do not** over-fit. `i in {1, 2, 3, 4, 5}` holds on your one trace — but the real invariant is `1 <= i <= len(a)`. Prefer the most general candidate that survives.

## Output format

```
## Candidates (from traces)
<template> : <instantiation>  — held on <N>/<N> observations

## Inductive check
<candidate>  — <✓ inductive | ✗ fails: <counterexample>>

## Sufficient for postcondition
<candidate>  — <✓ | ✗ too weak, missing: <what>>

## Recommended invariant
<the winner — conjunction of surviving, sufficient, minimal candidates>
```
