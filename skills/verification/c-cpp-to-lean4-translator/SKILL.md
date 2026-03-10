---
name: c-cpp-to-lean4-translator
description: Translates C/C++ into Lean 4 for interactive theorem proving — deep verification where automated tools fail. Use when Dafny's automation isn't enough, when proving mathematical properties of an algorithm, or when building a machine-checked proof for publication or certification.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "python-to-lean4-translator, cpp-to-dafny-translator"
---

# C/C++ → Lean 4 Translator

Lean is for when you need a **proof, not a check.** Dafny asks an SMT solver "is this true?"; Lean asks *you* to construct the proof term. More work, more control, no timeouts.

## When Lean over Dafny

| Situation                                             | Why Lean                                                    |
| ----------------------------------------------------- | ----------------------------------------------------------- |
| SMT solver times out or returns `unknown`             | You write the proof steps; no solver to time out            |
| Proving mathematical properties (complexity bounds, algebraic laws) | Lean has Mathlib; Dafny has arithmetic           |
| The proof itself is the artifact (paper, certification) | Lean proofs are inspectable; Dafny's are solver-internal |
| You need custom proof tactics                         | Lean is a tactic language; Dafny's hints are fixed          |

## The functional lift

Lean is purely functional. The translation path is: **imperative C → functional model → proof about the model**.

| C construct                   | Lean model                                                       |
| ----------------------------- | ---------------------------------------------------------------- |
| `int32_t`                     | `Int32` (modular) or `Int` (unbounded) — same choice as Dafny    |
| Loop with accumulator         | Fold: `xs.foldl f init` — or structural recursion                |
| Loop with early exit          | Recursion with a sum type result: `Option α` or `Except ε α`     |
| Mutable local var             | Let-bind a new name each "mutation": `let x₁ := ...; let x₂ := f x₁` |
| Array `a[i]`                  | `a.get i` with `(h : i < a.size)` proof, or `a.get? i : Option α`|
| Pointer into array            | `(a : Array α) × (i : Fin a.size)` — a dependent pair            |
| `struct`                      | `structure` — direct                                             |
| `malloc`/pointer to object    | Just the object. No heap in the pure fragment.                   |

## The state monad escape hatch

If the imperative structure is too tangled to lift functionally, use `StateM`:

```lean
def impStep : StateM (Array Int) Unit := do
  let a ← get
  if h : 0 < a.size then
    set (a.set ⟨0, h⟩ 42)

-- Reason about the result:
theorem impStep_sets_zero (a : Array Int) (h : 0 < a.size) :
  (impStep.run a).2.get ⟨0, h⟩ = 42 := by simp [impStep, StateT.run]
```

But prefer the functional lift — proofs over folds are easier than proofs over monadic runs.

## Worked example

**C:**

```c
// Count elements equal to target
size_t count(const int* a, size_t n, int target) {
    size_t c = 0;
    for (size_t i = 0; i < n; ++i)
        if (a[i] == target) ++c;
    return c;
}
```

**Lean — functional lift:**

```lean
def count (a : List Int) (target : Int) : Nat :=
  a.foldl (fun c x => if x = target then c + 1 else c) 0

-- Spec via List.countP from Mathlib
theorem count_eq_countP (a : List Int) (target : Int) :
    count a target = a.countP (· = target) := by
  induction a with
  | nil => rfl
  | cons x xs ih =>
    simp only [count, List.foldl_cons, List.countP_cons]
    split <;> simp [ih, count]
```

**Notes:**
- `List` instead of `Array` — easier induction. If you need `Array` for the performance story, prove on `List` and transport via `Array.toList`.
- The theorem connects our `count` to Mathlib's `countP` — once connected, all of Mathlib's lemmas about `countP` apply to us for free.
- `size_t` → `Nat` (unbounded). If we cared about `SIZE_MAX`, we'd add a separate bound theorem.

## Proof patterns you'll need

| Goal shape                          | Tactic                                                    |
| ----------------------------------- | --------------------------------------------------------- |
| Property of a fold                  | `induction xs` + unfold the fold one step                 |
| Property of recursion on `Nat`      | `induction n` or `Nat.rec`                                |
| Array index in bounds               | `omega` / `simp_arith` — or carry `Fin n` so it's by-type |
| Decidable equality case split       | `split` after `if` / `by_cases h : P`                     |
| "Obvious" arithmetic                | `omega`, `ring`, `simp_arith`                             |
| Reuse a library lemma               | `exact?` / `apply?` — let Lean search Mathlib             |

## Do not

- **Do not** translate to Lean when Dafny would verify automatically. Lean proofs are manual labor; only pay for what you need.
- **Do not** state the theorem as "equals the C code." State it as "satisfies this mathematical property." The C is an implementation; the theorem is about the spec.
- **Do not** fight Lean's termination checker with `partial def`. `partial` means "trust me" — you lose proofs-by-induction. Find the structural argument.
- **Do not** reinvent Mathlib. `List.countP`, `List.Sorted`, `Nat.gcd` — they exist, with lemma libraries. Connect to them.

## Output format

```
## Functional model
<Lean def — the translated algorithm>

## Theorem
<what property this proves — connected to Mathlib where possible>

## Proof
<tactic script — or `sorry` placeholders with notes on what's left>

## Model fidelity
<where the Lean model diverges from the C: unbounded ints, List vs Array, etc.>
```
