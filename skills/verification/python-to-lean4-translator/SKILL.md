---
name: python-to-lean4-translator
description: Translates Python into Lean 4 for interactive theorem proving, handling dynamic types and duck typing by specializing to the concrete types actually used. Use when proving correctness of a Python algorithm beyond what testing can establish, or when building a verified reference for numerical or combinatorial Python code.
license: Apache-2.0
metadata:
  category: "verification"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "c-cpp-to-lean4-translator, python-to-dafny-translator"
---

# Python → Lean 4 Translator

This is a **delta over → `c-cpp-to-lean4-translator`** — same target, easier memory model (no pointers), harder type discipline (dynamic → dependent).

## Python-specific mappings

| Python                        | Lean                                                               |
| ----------------------------- | ------------------------------------------------------------------ |
| `int`                         | `Int` — both unbounded. The easy case.                             |
| `list[T]`                     | `List T` — both immutable-by-default conceptually. Pythonic `.append` in a loop → build via `::` then reverse, or fold directly. |
| `dict[K,V]`                   | `Std.HashMap K V` or `K → Option V` function — the function form proves better |
| `tuple`                       | `Prod` (`×`) for pairs; anonymous structure for n-tuples           |
| `None` / `Optional[T]`        | `Option T` — direct                                                |
| `for x in xs:`                | `xs.foldl`, `xs.map`, or explicit recursion — same as C→Lean       |
| List comprehension            | `xs.filter p \|>.map f` — reads almost the same                    |
| `range(n)`                    | `List.range n : List Nat`                                          |
| `f-string` / `str`            | `String` — but if you're proving things about strings, use `List Char` for induction |
| `@dataclass`                  | `structure` — direct                                               |
| `raise` / `try`               | `Except ε α` monad — or precondition the error away                |

## The type-narrowing step

Python functions often accept "anything duck-typed like X." Lean can't verify "anything." Narrow:

1. Look at the actual call sites. What concrete types are passed?
2. Translate for *those* types.
3. If there are genuinely multiple types, either: (a) one Lean `def` per type, or (b) a typeclass capturing the shared interface.

```python
def sum_sq(xs):                    # accepts list of "numbers"
    return sum(x*x for x in xs)
```

Call sites only ever pass `list[int]` → translate for `Int`:

```lean
def sumSq (xs : List Int) : Int := (xs.map (· ^ 2)).sum
```

If floats also appear → either two defs, or `[Mul α] [Add α] [Zero α]` constraints.

## Worked example

**Python:**

```python
def is_palindrome(s: str) -> bool:
    i, j = 0, len(s) - 1
    while i < j:
        if s[i] != s[j]:
            return False
        i += 1; j -= 1
    return True
```

**Lean:**

```lean
def isPalindrome (s : List Char) : Bool :=
  s = s.reverse

-- The imperative two-pointer version, via recursion:
def isPalindrome' : List Char → Bool
  | []      => true
  | [_]     => true
  | c :: cs =>
    match cs.getLast? with
    | none   => true
    | some d => c = d && isPalindrome' cs.dropLast

-- Prove they agree (connects the efficient form to the spec form):
theorem isPalindrome'_eq (s : List Char) :
    isPalindrome' s = isPalindrome s := by
  induction s using List.strongRecOn with
  | _ s ih =>
    match s with
    | []      => rfl
    | [_]     => simp [isPalindrome, isPalindrome']
    | c :: d :: rest => sorry  -- unfold, use ih on dropLast, List.reverse lemmas
```

**Strategy:** Translate twice — once as the *obvious spec* (`s = s.reverse`), once as the *imperative algorithm* (the two-pointer walk). Then prove they agree. The spec is what you trust; the algorithm is what runs.

## What to skip

| Python feature                | Don't translate — instead                                  |
| ----------------------------- | ---------------------------------------------------------- |
| `print`, `open`, `requests`   | These are effects. Verify the pure core; trust the I/O.    |
| `*args`, `**kwargs`           | Fix the arity to what the call site uses.                  |
| Metaclass / `__getattr__`     | If the algorithm depends on this, Lean is the wrong tool.  |
| Float (`0.1 + 0.2`)           | Lean's `Float` is IEEE but proving things about it is painful. If the math is real-valued, use `ℝ` from Mathlib and accept the model gap. |

## Do not

- **Do not** use `partial def` to dodge termination. You lose `induction` and all the proof infrastructure. Find the decreasing measure.
- **Do not** translate `list.append` in a loop as `xs := xs ++ [x]` inside a fold — that's O(n²). Fold into a reversed list and `.reverse` once at the end, or use `Array` with `push`.
- **Do not** invent the spec. If the Python doesn't document what `normalize()` means, ask — a proof against a guessed spec is a proof of nothing.

## Output format

Same as → `c-cpp-to-lean4-translator`, plus:

```
## Type narrowing
Python accepts: <duck-typed description>
Translated for: <concrete type(s)> — justified by <call-site analysis | type hints>
```
