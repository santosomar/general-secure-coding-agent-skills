---
name: verified-pseudocode-extractor
description: Extracts human-readable pseudocode from a verified formal artifact (Dafny, Lean, TLA+) while preserving the verified properties as annotations, so the proof-carrying logic can be reimplemented in a production language. Use when porting verified code to an unverified target, when documenting what a formal spec actually does, or when handing a verified algorithm to an implementer.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "pseudocode-extractor, python-to-dafny-translator"
---

# Verified Pseudocode Extractor

The reverse of → `python-to-dafny-translator`. You have verified code; you want to carry the *verification obligations* into a reimplementation without carrying the verification language.

The output is pseudocode annotated with **preconditions, postconditions, and invariants** — so whoever implements it knows exactly what the verifier was relying on.

## What to keep vs drop

| Dafny/Lean/TLA+ construct       | In pseudocode                                          |
| ------------------------------- | ------------------------------------------------------ |
| `requires P`                    | KEEP — `// PRE: P`                                     |
| `ensures Q`                     | KEEP — `// POST: Q`                                    |
| `invariant I` (loop)            | KEEP — `// INV: I` at loop head                        |
| `decreases e`                   | KEEP — `// TERMINATES BY: e decreases each iteration`  |
| `ghost var`                     | DROP from code, KEEP as comment explaining what it tracks |
| `assert P` (proof hint)         | Usually DROP — it's a verifier hint, not a runtime check |
| `lemma` calls                   | DROP — they have no runtime effect                     |
| `modifies` clauses              | KEEP as comment — tells implementer what's mutated     |

## The annotation rule

Every dropped verification construct becomes a comment **at the point where the verifier used it.** Don't aggregate them into a docblock at the top — put the invariant comment *at the loop head*, where it's checked.

## Worked example — Dafny → annotated pseudocode

**Dafny (verified):**

```dafny
method BinarySearch(a: array<int>, key: int) returns (idx: int)
  requires forall i, j :: 0 <= i < j < a.Length ==> a[i] <= a[j]   // sorted
  ensures 0 <= idx ==> idx < a.Length && a[idx] == key
  ensures idx < 0  ==> forall i :: 0 <= i < a.Length ==> a[i] != key
{
  var lo, hi := 0, a.Length;
  while lo < hi
    invariant 0 <= lo <= hi <= a.Length
    invariant forall i :: 0 <= i < lo ==> a[i] < key
    invariant forall i :: hi <= i < a.Length ==> a[i] > key
    decreases hi - lo
  {
    var mid := lo + (hi - lo) / 2;
    if a[mid] < key      { lo := mid + 1; }
    else if a[mid] > key { hi := mid; }
    else                 { return mid; }
  }
  return -1;
}
```

**Extracted pseudocode:**

```
function binary_search(a: sorted array of int, key: int) -> int
    // PRE:  a is sorted ascending (a[i] <= a[j] for i < j)
    // POST: if result >= 0 then a[result] == key
    // POST: if result < 0  then key is not in a

    lo ← 0
    hi ← length(a)

    while lo < hi:
        // INV: 0 <= lo <= hi <= length(a)
        // INV: everything left of lo is < key   (a[0..lo) all < key)
        // INV: everything at/right of hi is > key   (a[hi..) all > key)
        // TERMINATES BY: (hi - lo) strictly decreases each iteration

        mid ← lo + (hi - lo) / 2        // no overflow — NOT (lo+hi)/2

        if a[mid] < key:
            lo ← mid + 1
        else if a[mid] > key:
            hi ← mid
        else:
            return mid

    return -1
```

**What got preserved:** The three loop invariants are the *essence* of why binary search works. An implementer who maintains them can't get the algorithm wrong. The overflow note on `mid` — that's from the verified `lo + (hi - lo) / 2`, not the naive `(lo + hi) / 2`.

**What got dropped:** Nothing here — this Dafny had no ghost state or lemmas. If it had `ghost var numCompares`, that'd become a comment: `// (verifier tracked comparison count for complexity proof — not needed at runtime)`.

## From TLA+

TLA+ → pseudocode is different: TLA+ actions become pseudocode *steps*, and the invariant becomes a global assertion.

```
// GLOBAL INVARIANT (must hold after every step):
//   at most one process in critical section

step AcquireLock(p):
    // ENABLED WHEN: lock is free
    wait until lock == FREE
    lock ← p

step ReleaseLock(p):
    // ENABLED WHEN: p holds the lock
    assert lock == p
    lock ← FREE
```

The `ENABLED WHEN` annotations are the TLA+ guards. They tell the implementer: this operation blocks (or fails) if the condition isn't met.

## Do not

- **Do not** drop the invariants. They're the whole point. Pseudocode without the invariants is just unverified pseudocode.
- **Do not** translate ghost variables into runtime variables. They existed only to state the proof. Mentioning them in comments is fine; computing them is waste.
- **Do not** simplify the pseudocode in ways the verifier would reject. `(lo + hi) / 2` is simpler than `lo + (hi - lo) / 2` and also wrong at scale. Keep what the verifier verified.
- **Do not** lose the `decreases` clause. Termination is a correctness property. "This loop terminates because X decreases" belongs in the output.

## Output format

```
## Source
<Dafny | Lean | TLA+> — <file/module>

## Verified properties
PRE:  <list>
POST: <list>
INV:  <per-loop list>
TERM: <decreases measures>

## Pseudocode
<annotated pseudocode — annotations inline at point-of-use>

## Dropped constructs
<ghost vars, lemmas, proof hints — what they were for>

## Implementation warnings
<anything the verifier relied on that a naive implementer might get wrong — overflow, aliasing, atomicity>
```
