---
name: abstract-invariant-generator
description: Generates abstract invariants using domain abstraction — intervals, octagons, polyhedra, sign domains — to find invariants that concrete reasoning misses. Use when standard invariant inference fails, when the invariant involves relationships between multiple variables, or when verifying numerical code.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "invariant-inference, python-to-dafny-translator"
---

# Abstract Invariant Generator

When → `invariant-inference` (which works from concrete traces) can't find the invariant, abstract interpretation can. It over-approximates: instead of tracking exact values, track which *region* of a chosen domain the values are in.

## Abstract domains — pick by invariant shape

| Domain         | Can express                            | Example invariant              | Cost      |
| -------------- | -------------------------------------- | ------------------------------ | --------- |
| Sign           | `x > 0`, `x = 0`, `x < 0`              | `n > 0`                        | Trivial   |
| Interval       | `lo ≤ x ≤ hi`                          | `0 ≤ i ≤ n`                    | Cheap     |
| Octagon        | `±x ± y ≤ c` (two-var bounds)          | `i ≤ j`, `j - i ≤ n`           | Moderate  |
| Polyhedra      | Arbitrary linear inequalities          | `2i + 3j ≤ n + 5`              | Expensive |
| Congruence     | `x ≡ k (mod m)`                        | `i is even`                    | Cheap     |

**Start cheap.** Intervals catch 80% of array-bounds invariants. Octagons catch most two-pointer patterns. Polyhedra are for when you actually need a weird linear relationship.

## The algorithm (fixpoint iteration)

1. **Abstract the initial state.** `i = 0` → in intervals: `i ∈ [0, 0]`.
2. **Abstract the transition.** `i := i + 1` on `[a, b]` → `[a+1, b+1]`.
3. **Join at loop head.** Union the incoming abstract states: `[0,0] ⊔ [1,1] = [0,1]`.
4. **Iterate until fixpoint.** The loop stabilizes → that's your invariant.
5. **Widen if it doesn't stabilize.** `[0,1] ⊔ [0,2] ⊔ [0,3] ⊔ ...` → widen to `[0, +∞)`. Then narrow: apply the loop guard `i < n` to get `[0, n-1]`.

## Worked example — interval domain

**Code:**

```c
int i = 0, j = n;
while (i < j) {
    i++;
    j--;
}
// invariant at loop head? need: 0 ≤ i, j ≤ n, and relationship between i and j
```

**Interval abstraction:**

| Iteration | i         | j           |
| --------- | --------- | ----------- |
| Init      | [0, 0]    | [n, n]      |
| After 1   | [1, 1]    | [n-1, n-1]  |
| Join      | [0, 1]    | [n-1, n]    |
| After 2   | [1, 2]    | [n-2, n-1]  |
| Join      | [0, 2]    | [n-2, n]    |
| …widen    | [0, +∞)   | (-∞, n]     |
| Narrow (guard `i < j`) | [0, n] | [0, n] |

**Interval invariant:** `0 ≤ i ≤ n ∧ 0 ≤ j ≤ n`. True but weak — doesn't capture `i + j = n`.

**Octagon domain** (tracks `±x ± y ≤ c`):

Initial: `i + j = n` (since `i=0, j=n`). Transition: `i' = i+1, j' = j-1` → `i' + j' = i + j = n`. **Preserved.** Octagon finds `i + j = n` (as `i+j ≤ n ∧ -(i+j) ≤ -n`).

**Lesson:** Intervals are per-variable; they can't see `i + j = n`. For relational invariants, you need a relational domain.

## Choosing the domain

1. Look at the postcondition you're trying to prove. What shape is it?
   - Single-var bound → intervals suffice.
   - Two-var comparison (`i ≤ j`) → octagons.
   - Three+ var linear → polyhedra.
   - Divisibility → congruence.
2. Run with the cheapest domain that could express it.
3. If the result is too weak (e.g., `[-∞, +∞]`), the domain can't express the invariant — upgrade.

## Do not

- **Do not** start with polyhedra. They're exponentially expensive. Intervals → octagons → polyhedra.
- **Do not** forget to narrow after widening. Widening alone gives `[0, +∞)` — useless. The guard narrows it back.
- **Do not** expect abstract interpretation to find *nonlinear* invariants. `i*i ≤ n` — none of these domains can express it. You need a polynomial domain or template-based search.
- **Do not** trust an invariant without checking it's inductive. Abstract interpretation is sound (no false invariants) — but double-check by plugging it back into the verifier.

## Output format

```
## Domain
<interval | octagon | polyhedra | congruence> — <why this domain>

## Fixpoint iteration
<table: iteration → abstract state>

## Invariant
<the stabilized abstract state, written as a logical formula>

## Strength check
Sufficient for the postcondition? <yes | no — need stronger domain: which>
```
