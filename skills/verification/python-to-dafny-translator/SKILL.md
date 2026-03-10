---
name: python-to-dafny-translator
description: Translates Python functions into Dafny, adding types, pre/postconditions, and loop invariants sufficient for Dafny to verify. Use when formally verifying a Python algorithm, when the user wants machine-checked correctness for a critical function, or when building a verified reference implementation.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "cpp-to-dafny-translator, python-to-lean4-translator, invariant-inference"
---

# Python → Dafny Translator

Dafny is a verification-aware language: you write code + spec together, and the verifier checks the code meets the spec. Translating from Python means **adding everything Dafny needs that Python omits**: types, termination measures, loop invariants, and a precise contract.

## The impedance mismatch

| Python has                    | Dafny needs                         | You must supply                                          |
| ----------------------------- | ----------------------------------- | -------------------------------------------------------- |
| Duck typing                   | Static types                        | Type annotations — infer from usage if not annotated     |
| Unbounded `int`               | Unbounded `int` — ✅ matches         | Nothing. This is the easy one.                           |
| `list` (mutable, heterogeneous) | `seq<T>` (immutable, homogeneous) or `array<T>` (mutable, heap) | Choose: algorithm is functional → `seq`. Mutates in place → `array` + frame conditions. |
| `dict`                        | `map<K,V>` (immutable)              | Usually fine — most dict uses are read-mostly            |
| `for x in xs`                 | `while i < |xs|` with index         | Rewrite as indexed loop; Dafny's `forall` is a quantifier, not a loop |
| Exceptions                    | No exceptions                       | Encode as `Result` datatype, or as a precondition that rules out the failing case |
| Implicit termination          | Explicit `decreases` clause         | Find the measure — usually the loop bound or recursion arg size |
| No spec                       | `requires`/`ensures`                | The hard part. See below.                                |

## Step 1 — Extract the spec

The contract has to come from somewhere. In priority order:

1. Docstring says what the function returns → that's your `ensures`.
2. Function name is a known algorithm (`binary_search`, `sort`, `gcd`) → use the textbook postcondition.
3. Assertions in the code → promote to `ensures`/`requires`.
4. Test cases → each test is a ground instance; generalize.
5. Nothing → ask the user. You can't verify against an unknown spec.

## Step 2 — Translate the body

| Python construct              | Dafny translation                                                  |
| ----------------------------- | ------------------------------------------------------------------ |
| `def f(x): ... return y`      | `method f(x: T) returns (y: U)` or `function f(x: T): U` if pure   |
| `x = []; x.append(...)`       | `var x: seq<T> := []; x := x + [elem];`                            |
| `for i in range(n):`          | `var i := 0; while i < n { ...; i := i + 1; }`                     |
| `for x in xs:`                | `var i := 0; while i < \|xs\| { var x := xs[i]; ...; i := i + 1; }` |
| `if c: return a; return b`    | `if c { y := a; } else { y := b; }` (in a `method`)                |
| `x[i] = v` (list)             | If `array`: `x[i] := v;`. If `seq`: `x := x[..i] + [v] + x[i+1..];` |
| `raise ValueError`            | Precondition `requires <not-that-case>` — or return `Failure(msg)` |
| `a // b`                      | `a / b` (Dafny `/` on int is floor for positives — sign semantics differ for negatives, add `requires b > 0`) |

## Step 3 — Add invariants (the hard part)

Every `while` needs invariants strong enough that `invariant ∧ ¬guard ⟹ postcondition`.

**Pattern — accumulator loops:** The invariant says "the accumulator holds the result for the prefix processed so far."

```
// Python: total = sum(xs)
var i, total := 0, 0;
while i < |xs|
  invariant 0 <= i <= |xs|
  invariant total == Sum(xs[..i])     // "correct so far"
  decreases |xs| - i
{ total := total + xs[i]; i := i + 1; }
```

**Pattern — search loops:** The invariant says "the thing isn't in the part we've already looked at."

**Pattern — two-pointer:** Two index-bound invariants plus the relationship between them.

If you can't find the invariant: → `invariant-inference`.

## Worked example

**Python:**

```python
def index_of(xs: list[int], target: int) -> int:
    """Return index of target in xs, or -1."""
    for i, x in enumerate(xs):
        if x == target:
            return i
    return -1
```

**Dafny:**

```dafny
method IndexOf(xs: seq<int>, target: int) returns (r: int)
  ensures r == -1 ==> target !in xs
  ensures r >= 0  ==> 0 <= r < |xs| && xs[r] == target
{
  var i := 0;
  while i < |xs|
    invariant 0 <= i <= |xs|
    invariant target !in xs[..i]      // not found in the prefix
    decreases |xs| - i
  {
    if xs[i] == target { return i; }
    i := i + 1;
  }
  return -1;
}
```

Verifies. The `target !in xs[..i]` invariant plus loop exit (`i == |xs|`) gives `target !in xs`, which discharges the `r == -1` postcondition.

## What doesn't translate

| Python feature                | Why it's hard                                        | Workaround                                  |
| ----------------------------- | ---------------------------------------------------- | ------------------------------------------- |
| Deep mutation of aliased refs | Dafny's heap model needs `modifies` frames; aliasing blows them up | Rewrite to functional style with `seq` |
| `**kwargs`, `*args`           | No variadic in Dafny                                 | Fix arity for the specific call you're verifying |
| Dynamic dispatch / `isinstance` | Dafny has traits but dispatch is resolved at verification | One translated version per concrete type |
| I/O, network, filesystem      | Dafny has no I/O model                               | Model as inputs/outputs; verify the pure core |
| Float                         | Dafny has `real` (mathematical), not IEEE 754        | If IEEE semantics matter, this is the wrong tool |

Flag these. A translation with `assume false` hidden where the hard part is is worse than no translation.

## Do not

- **Do not** weaken the postcondition until it verifies. `ensures true` always verifies and proves nothing.
- **Do not** use `assume` to paper over a hard invariant. `assume` is a trust hole — mark it LOUDLY and justify it.
- **Do not** translate the *bug*. If the Python has an off-by-one, Dafny will refuse to verify — good. Don't "fix" the spec to match the bug.
- **Do not** skip the termination clause. Dafny requires it; `decreases *` means "I gave up on proving this terminates" — only acceptable if the function genuinely might not.

## Output format

```
## Spec (extracted from)
<docstring | algorithm name | tests | user-provided>

## Dafny
<code block — must verify clean or failures must be annotated>

## Trust holes
<every `assume` with justification, or "none">

## Untranslated
<Python constructs with no Dafny equivalent — and what you did instead>
```
