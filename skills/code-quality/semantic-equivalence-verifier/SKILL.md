---
name: semantic-equivalence-verifier
description: Proves two program fragments semantically equivalent using symbolic reasoning — stronger than testing, applicable when differential testing is insufficient or impossible. Use when behavior preservation must be proven rather than sampled, when the input space is too large to enumerate, or when a transformation needs a correctness argument.
license: Apache-2.0
metadata:
  category: "code-quality"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "behavior-preservation-checker, python-to-dafny-translator"
---

# Semantic Equivalence Verifier

→ `behavior-preservation-checker` tests on a sample; this skill **proves** over the full input space. Use when "we ran it on 500 inputs" isn't enough.

## When proof beats testing

| Situation                                              | Why testing fails                                   |
| ------------------------------------------------------ | --------------------------------------------------- |
| Input space is infinite and adversarial                | Attacker picks the input you didn't test (crypto, parsers, sanitizers) |
| Rare edge case matters (overflow, boundary)            | 1-in-2³² inputs — fuzzing takes forever             |
| Regulatory / safety requirement                        | "We tested it" isn't certifiable evidence           |
| The transformation is algebraic                        | `x*2` ↔ `x<<1` — easy to prove, tedious to test     |

## Proof strategies — pick by code shape

| Shape                                  | Strategy                                                              |
| -------------------------------------- | --------------------------------------------------------------------- |
| Straight-line, no loops                | Symbolic execution → discharge `old_out == new_out` with an SMT solver |
| Loop, both versions have same structure| Show loop bodies equivalent + same termination → induction            |
| Loop, restructured (unrolled, fused)   | Find a *simulation relation* between iteration states                 |
| Recursive                              | Structural induction on the recursion argument                        |
| Different algorithms, same spec        | Can't prove A≡B directly. Prove A⊨spec ∧ B⊨spec separately.           |

## Step-by-step — SMT-backed equivalence (common case)

1. **Translate** both fragments to a common logical form. Same input variable names, same state model.
2. **Assert** the preconditions: `assume(pre(x))`.
3. **Assert inequivalence** of outputs: `assert(old_out != new_out)`.
4. **Solve.** If UNSAT → no input makes them differ → equivalent. If SAT → the model is a counterexample.

For loops: replace the loop with its invariant. You need the invariant to be the *same* for both versions (or for one to imply the other).

## Worked example

**Old:**

```c
int abs_old(int x) {
    if (x < 0) return -x;
    return x;
}
```

**New (branchless):**

```c
int abs_new(int x) {
    int mask = x >> 31;
    return (x + mask) ^ mask;
}
```

**Encode (SMT-LIB, bitvector theory):**

```
(declare-const x (_ BitVec 32))
(define-fun old () (_ BitVec 32)
  (ite (bvslt x #x00000000) (bvneg x) x))
(define-fun new () (_ BitVec 32)
  (let ((mask (bvashr x #x0000001f)))
    (bvxor (bvadd x mask) mask)))
(assert (not (= old new)))
(check-sat)
```

**Solve:** `unsat`. Equivalent for all 32-bit inputs. ∎

**But wait** — what about `INT_MIN`? `-INT_MIN` overflows in C (UB). The SMT encoding uses modular arithmetic, so `bvneg(INT_MIN) = INT_MIN`, and both versions return `INT_MIN`. *Equivalent, including at the UB point* — they both return the wrong thing. The proof shows equivalence, not correctness. If you need correctness, add `(assume (not (= x INT_MIN)))` or prove against the spec `result ≥ 0`.

## When proof fails

| Failure                                    | What it means                                      | Next move                                  |
| ------------------------------------------ | -------------------------------------------------- | ------------------------------------------ |
| SAT — solver found a counterexample        | They're NOT equivalent                             | Look at the model. Is it a real input, or outside the precondition you forgot to state? |
| Unknown / timeout                          | Solver couldn't decide                             | Add lemmas; break into smaller pieces; try a different solver |
| Can't encode the construct                 | Heap, I/O, unbounded recursion                     | Abstract it — model the heap as an uninterpreted function; axiomatize I/O |

## Limits — when to fall back to testing

- Heavy heap mutation → state space explodes
- Floating point → solvers handle it but slowly and with surprising edge cases (NaN, -0.0)
- External calls → can't symbolically execute what you don't have the body of
- Code > ~500 lines → decompose or give up on full proof; → `behavior-preservation-checker` with heavy fuzzing

## Do not

- **Do not** claim equivalence without stating the precondition. "Equivalent for all inputs" and "equivalent for non-null inputs" are different theorems.
- **Do not** confuse "equivalent" with "correct." Two functions can be identically wrong. Equivalence is A↔B; correctness is A↔spec.
- **Do not** trust a proof of a hand-translation. If you translated the code to SMT-LIB by hand, the bug might be in the translation. Use a trusted frontend (→ `python-to-dafny-translator`, CBMC's C frontend) when possible.

## Output format

```
## Claim
old: <fragment>
new: <fragment>
Equivalent under precondition: <P(x)>

## Proof strategy
<SMT | induction | simulation>

## Result
<PROVEN EQUIVALENT | COUNTEREXAMPLE: x=<val>, old=<o>, new=<n> | UNKNOWN — <why>>

## Trust base
<what the proof relies on: solver correctness, translation fidelity, axioms assumed>
```
