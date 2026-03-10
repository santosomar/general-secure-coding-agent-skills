---
name: code-translation
description: Translates a single function or small code unit between programming languages, mapping idioms and preserving observable behavior. Use when porting one function, when the user pastes code and asks for it in another language, or as the per-unit primitive for larger migrations.
license: Apache-2.0
metadata:
  category: "code-analysis"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "module-level-code-translator, behavior-preservation-checker, multi-version-behavior-comparator"
---

# Code Translation

Function-level translation between languages. The goal is **observably equivalent** behavior — same inputs → same outputs, same side effects, same error behavior. Not line-by-line transliteration; idiomatic code in the target.

For whole modules/packages, → `module-level-code-translator` (handles imports, file structure, cross-function concerns).

## The three layers

1. **Syntax** — the easy part. `for x in xs` ↔ `for (auto& x : xs)`. Mechanical.
2. **Semantics** — the dangerous part. `3 / 2` is `1` in Python 2, `1.5` in Python 3, `1` in C. Integer overflow, null semantics, string encoding.
3. **Idiom** — the quality part. `[x*2 for x in xs if x > 0]` → Java stream, not a for-loop with an ArrayList. Optional but expected.

## Semantic minefields — per language pair

| Source → Target  | What silently changes                                            |
| ---------------- | ---------------------------------------------------------------- |
| Python → Java    | `int` unbounded → `int` 32-bit overflow. `/` float vs int. Dicts ordered since 3.7 — HashMap isn't. `None` vs checked exceptions. |
| Java → Python    | `int` overflow at 2³¹ → Python never overflows. `==` on objects is identity, not equality. Checked exceptions disappear. |
| C++ → Rust       | UB becomes panic or won't compile. Raw pointers need lifetime decisions. Move semantics are default, not opt-in. Implicit conversions gone. |
| JS → TypeScript  | Mostly additive (types) — but `==` coercion stays, strictNullChecks changes nullability. |
| Python → Go      | Exceptions → explicit `error` returns. Duck typing → interfaces. No default args. `**kwargs` has no equivalent. |
| C → Go           | Manual memory → GC. Pointer arithmetic gone. `goto` cleanup → `defer`. No unions. |

**Before translating, write down the three semantic mismatches that matter for this function.** Check them in the output.

## Idiom mapping — common patterns

| Pattern (Python)                    | Java                                     | Go                                   | Rust                                  |
| ----------------------------------- | ---------------------------------------- | ------------------------------------ | ------------------------------------- |
| `[f(x) for x in xs]`                | `xs.stream().map(f).toList()`            | for-append loop                      | `xs.iter().map(f).collect()`          |
| `{k: v for k, v in ...}`            | `Collectors.toMap(...)`                  | make + loop                          | `HashMap::from_iter(...)`             |
| `try/except` for control flow       | Anti-pattern — check first               | `if err != nil`                      | `Result<T, E>` / `?`                  |
| `with open(...) as f:`              | try-with-resources                       | `defer f.Close()`                    | `Drop` (automatic)                    |
| `*args, **kwargs`                   | Overloads or a params object             | variadic (`...T`) — no kwargs        | No direct — builder or options struct |
| Multiple return values              | Record/Pair (awkward)                    | Native — `(T, error)`                | Tuple or struct                       |

## Worked example

**Python:**

```python
def top_k(items, k, key=lambda x: x):
    """Return the k largest items by key."""
    if k <= 0:
        return []
    if k >= len(items):
        return sorted(items, key=key, reverse=True)
    import heapq
    return heapq.nlargest(k, items, key=key)
```

**Go translation:**

```go
import "sort"

// TopK returns the k largest items by the given key function.
// If k <= 0, returns an empty slice. If k >= len(items), returns all items sorted descending.
func TopK[T any](items []T, k int, key func(T) int) []T {
    if k <= 0 {
        return nil
    }
    // Go has no heapq.nlargest — full sort then slice.
    // (O(n log n) vs Python's O(n log k). Acceptable unless n is huge and k is small.)
    sorted := make([]T, len(items))
    copy(sorted, items)
    sort.Slice(sorted, func(i, j int) bool {
        return key(sorted[i]) > key(sorted[j])  // descending
    })
    if k >= len(sorted) {
        return sorted
    }
    return sorted[:k]
}
```

**Semantic deltas noted:**
- No default arg — caller must pass `key` explicitly (or make a `TopKInt` wrapper).
- Complexity: O(n log n) vs Python's O(n log k). For large n / small k, consider `container/heap`. Flagged, not fixed — premature optimization.
- `nil` vs `[]` — both are zero-length in Go, but `nil` is more idiomatic for "empty result."
- Copies input (Python's `heapq.nlargest` doesn't mutate either). Matches.

## Verify the translation

After translating, → `behavior-preservation-checker` or → `multi-version-behavior-comparator`: run both versions on the same inputs, diff outputs. Pay attention to edge cases — empty input, k=0, k=len, duplicate keys.

## Do not

- **Do not** transliterate control flow when the target has a better idiom. `for i in range(len(xs)): f(xs[i])` → don't index in Go/Rust, use `for _, x := range xs`.
- **Do not** silently swallow error-handling differences. If the source throws and the target returns errors, *every call site* changes.
- **Do not** assume library parity. Python's `heapq.nlargest` has no exact Go equivalent — note the complexity tradeoff.
- **Do not** change mutability behavior. If the source doesn't mutate inputs, neither should the translation. Copy if you have to.
- **Do not** translate into deprecated/unsafe target idioms. Python `str.format` → Java `String.format`, not concatenation. No `strcpy` in C targets.

## Output format

```
## Target
<language + version>

## Translation
<code block>

## Semantic deltas
- <what behaves differently, and why it's acceptable or flagged>

## Idiom choices
- <non-obvious mappings — why this Go/Rust/Java pattern and not the literal one>

## Verify with
<example inputs to diff against the source — especially the edge cases>
```
