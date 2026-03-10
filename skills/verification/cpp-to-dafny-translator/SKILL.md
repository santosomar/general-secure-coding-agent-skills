---
name: cpp-to-dafny-translator
description: Translates C++ functions into Dafny for formal verification, modeling pointers, fixed-width integers, and manual memory as Dafny heap objects and bitvectors. Use when verifying a C++ algorithm, when proving absence of overflow or out-of-bounds access, or when building a verified reference for safety-critical C++ code.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "python-to-dafny-translator, c-cpp-to-lean4-translator, invariant-inference"
---

# C++ → Dafny Translator

This is a **delta over → `python-to-dafny-translator`** — the Dafny target is the same; the source is harder. C++ brings three problems Python doesn't: fixed-width integers (overflow), pointers (aliasing), and manual memory (lifetime).

## The C++-specific impedance mismatch

| C++ has                        | Dafny needs                        | Translation                                                   |
| ------------------------------ | ---------------------------------- | ------------------------------------------------------------- |
| `int32_t`, `uint64_t` — wraps  | `bv32`, `bv64` — modular; or `int` — unbounded | Choose: proving no-overflow → `int` + `ensures` in-range. Modeling actual wrap → `bv`. |
| `T*` pointer                   | `array<T>` + index, or class ref   | Decompose: a pointer is (base array, offset). Verify offset in bounds. |
| `T& ref`                       | `inout` param or direct object access | Dafny params are value by default — use class fields for mutation |
| `new`/`delete`                 | `new` (garbage-collected, no delete) | Drop `delete`. If use-after-free is the property you're proving, you need a ghost lifetime tracker. |
| Pointer arithmetic `p + n`     | `a, i+n` on the (array, index) pair | Stays in bounds? That's a proof obligation. |
| `reinterpret_cast`             | **Nothing**                        | Refuse. If the algorithm relies on type punning, Dafny is the wrong tool. |
| Templates                      | Generics `<T>`                     | Usually direct; constraints become `T extends Trait` |
| `std::vector<T>`               | `seq<T>` (value semantics) or `array<T>` (ref) | `seq` if you're not mutating through aliases |
| Undefined behavior             | Dafny has none                     | Every UB in C++ becomes a `requires` clause — that's the point |

## The overflow question — decide upfront

This is the most important modeling choice:

**Option A — Prove no overflow:** Use Dafny `int` (unbounded). Add `ensures result fits in int32`. The proof obligation is that your arithmetic never exceeds bounds. This is what you want for *verifying the algorithm is overflow-safe*.

**Option B — Model wraparound:** Use `bv32`. Arithmetic wraps silently. Use this only if the C++ *intentionally relies on wrap* (hash functions, checksums).

Most of the time you want Option A. If Dafny refuses to verify because `a + b` might overflow — that's the bug you were looking for.

## Pointer modeling

A C++ pointer `int* p` pointing into array `arr` at offset `k` becomes two Dafny variables:

```dafny
var arr: array<int>;
var k: nat;               // requires k <= arr.Length
// *p       ↦  arr[k]
// p[i]     ↦  arr[k + i]    with proof obligation k + i < arr.Length
// p++      ↦  k := k + 1
```

Aliasing: if C++ has `int* p, *q` that might point into the same array, Dafny needs `modifies` clauses and frame reasoning. If they're known-disjoint, say so: `requires arr_p != arr_q`.

## Worked example

**C++:**

```cpp
// Returns index of minimum element, or -1 if empty
int argmin(const int* a, size_t n) {
    if (n == 0) return -1;
    size_t best = 0;
    for (size_t i = 1; i < n; ++i) {
        if (a[i] < a[best]) best = i;
    }
    return (int)best;    // UB if best > INT_MAX
}
```

**Dafny:**

```dafny
method ArgMin(a: array<int>) returns (r: int)
  requires a.Length <= 0x7fffffff              // ← UB guard: result fits in int
  ensures a.Length == 0 ==> r == -1
  ensures a.Length > 0  ==>
          0 <= r < a.Length &&
          forall j :: 0 <= j < a.Length ==> a[r] <= a[j]
{
  if a.Length == 0 { return -1; }
  var best := 0;
  var i := 1;
  while i < a.Length
    invariant 1 <= i <= a.Length
    invariant 0 <= best < i
    invariant forall j :: 0 <= j < i ==> a[best] <= a[j]   // best is min of prefix
    decreases a.Length - i
  {
    if a[i] < a[best] { best := i; }
    i := i + 1;
  }
  return best;
}
```

Note the `requires a.Length <= INT_MAX` — this is the translation of the C++ cast's UB into an explicit precondition. If the caller can't prove it, the cast was a real bug.

## What doesn't translate

| C++ construct                 | Why                                               | Workaround                                      |
| ----------------------------- | ------------------------------------------------- | ----------------------------------------------- |
| `reinterpret_cast` / type punning | Dafny's type system is sound by design         | Model as uninterpreted function; verify around it |
| Inline assembly               | Obviously                                         | Spec the asm's effect; `assume` it; note the hole |
| Virtual dispatch with unknown overrides | Don't know what code runs                | Verify each concrete override separately        |
| `volatile` / memory-mapped I/O | Dafny has no memory model for hardware          | Model as external method with `modifies` frame  |
| Concurrency (`std::thread`, atomics) | Dafny is sequential                         | → `program-to-tlaplus-spec-generator` for concurrency |

## Do not

- **Do not** use `bv32` by default "because C++ ints are 32-bit." Use `int` and prove no-overflow — that's usually the property you want.
- **Do not** model every `*p` as `arr[0]`. The offset matters. `p++; *p` is `arr[1]`, not `arr[0]`.
- **Do not** drop `const` without thinking. C++ `const int*` means the function doesn't write through it — that's a frame condition: the array isn't in `modifies`.
- **Do not** pretend the cast away. `(int)size_t_val` has a precondition; state it.

## Output format

Same as → `python-to-dafny-translator`, plus:

```
## Integer model
<int (prove-no-overflow) | bvN (model-wrap)> — <why>

## Pointer model
<pointer> ↦ (<array>, <index>)  — aliasing: <disjoint | may-alias>

## UB → preconditions
<each C++ UB that became a requires clause>
```
