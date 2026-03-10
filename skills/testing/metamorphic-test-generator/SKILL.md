---
name: metamorphic-test-generator
description: Generates metamorphic tests — tests that check relationships between multiple runs instead of checking exact outputs, useful when the correct output is unknown or expensive to compute. Use when there's no oracle, when testing ML/numerical/search code, or when the spec describes properties rather than values.
license: Apache-2.0
metadata:
  category: "testing"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "test-oracle-generator"
---

# Metamorphic Test Generator

Normal test: "for input X, output should be Y." But what if you don't know Y? Metamorphic test: "for inputs X and X', the outputs should *relate* in a known way." You don't need an oracle — you need a **metamorphic relation.**

## The core idea

```
         f
  X ─────────► Y           You don't know Y.
  │            │
  │ transform  │ relation  But you know: if X' = transform(X),
  ▼            ▼           then f(X') should relate to f(X).
  X' ────────► Y'
         f
```

If the relation fails, `f` is wrong. You found a bug without knowing the right answer.

## Metamorphic relation catalog

| Domain              | Relation                                                        | Transform → expected output change       |
| ------------------- | --------------------------------------------------------------- | ---------------------------------------- |
| Sorting             | Permutation invariance                                          | shuffle(X) → same output                 |
| Search              | Adding irrelevant docs doesn't change top-k for a fixed query   | X ∪ {irrelevant} → same top-k            |
| Search              | Subset monotonicity                                             | results(A∪B) ⊇ results(A) for query q    |
| Numerical           | Scaling                                                         | f(k·X) = k·f(X) (if f is linear)         |
| Numerical           | Additivity                                                      | f(X+Y) = f(X) + f(Y) (if linear)         |
| Pathfinding         | Adding an edge can't make shortest path longer                  | G + edge → dist ≤ old dist               |
| Compression         | Round-trip                                                      | decompress(compress(X)) = X              |
| Parsing             | Pretty-print invariance                                         | parse(print(parse(X))) = parse(X)        |
| ML classifier       | Semantic-preserving input → same class                          | image + tiny noise → same prediction     |
| Aggregation (sum)   | Partition invariance                                            | sum(A∪B) = sum(A) + sum(B) (disjoint)    |
| Caching             | Idempotence                                                     | f(X); f(X) → same result, second is fast |

## Finding relations for your function

Ask: **what transformation of the input has a predictable effect on the output?**

1. **Does adding something irrelevant change nothing?** (search, filter, max)
2. **Does reordering change nothing?** (set operations, commutative aggregates)
3. **Does scaling input scale output?** (linear functions)
4. **Is there a round-trip?** (encode/decode, serialize/deserialize)
5. **Is there a known inverse?** (`f(g(x)) = x`)
6. **Is there monotonicity?** (more input → more-or-equal output)
7. **Is the operation idempotent?** (`f(f(x)) = f(x)`)

Each "yes" is a metamorphic relation.

## Worked example — testing a search engine without knowing right answers

**System:** `search(query, corpus) -> list[doc]` ranked by relevance. You don't know what's "correct" — relevance is subjective.

**Relations:**

```python
from hypothesis import given, strategies as st

# MR1: Permuting the corpus doesn't change results (order-independence of index)
@given(query=queries(), corpus=corpora())
def test_mr_corpus_order_invariant(query, corpus):
    r1 = search(query, corpus)
    r2 = search(query, shuffled(corpus))
    assert r1 == r2

# MR2: Adding a document irrelevant to the query doesn't change top-k
@given(query=queries(), corpus=corpora(), k=st.integers(1, 10))
def test_mr_irrelevant_addition(query, corpus, k):
    r1 = search(query, corpus)[:k]
    irrelevant = make_doc_with_no_overlap(query)   # no shared terms
    r2 = search(query, corpus + [irrelevant])[:k]
    assert r1 == r2

# MR3: Duplicating a top result keeps it in top results
@given(query=queries(), corpus=corpora())
def test_mr_duplicate_stays_top(query, corpus):
    r1 = search(query, corpus)
    if not r1:
        return
    top = r1[0]
    r2 = search(query, corpus + [top])   # add another copy
    assert top in r2[:2]   # original or copy should be in top-2

# MR4: Query with more terms → results are a subset (conjunctive search)
@given(base_query=queries(), extra_term=terms(), corpus=corpora())
def test_mr_conjunction_narrows(base_query, extra_term, corpus):
    r_broad = set(search(base_query, corpus))
    r_narrow = set(search(base_query + " " + extra_term, corpus))
    assert r_narrow <= r_broad
```

MR4 assumes conjunctive (AND) semantics. If search is disjunctive (OR), the relation flips. **The relation encodes a spec claim** — if it fails, either the relation is wrong (you misunderstood the spec) or the code is wrong.

## Worked example — numerical code

**Function:** `std_dev(samples: list[float]) -> float`. You could compute the expected value by hand for each test input. Or:

```python
# MR: std dev is shift-invariant — adding a constant doesn't change spread
@given(st.lists(st.floats(allow_nan=False, allow_infinity=False), min_size=2),
       st.floats(allow_nan=False, allow_infinity=False))
def test_mr_shift_invariant(samples, c):
    assert abs(std_dev(samples) - std_dev([s + c for s in samples])) < 1e-9

# MR: std dev scales linearly with the data
@given(st.lists(st.floats(1, 100), min_size=2), st.floats(0.1, 10))
def test_mr_scale(samples, k):
    assert abs(std_dev([s * k for s in samples]) - k * std_dev(samples)) < 1e-9
```

These test the *algebra* of std dev. A buggy implementation (e.g., forgot the square root, or divides by N instead of N-1) will break at least one.

## Do not

- **Do not** use a relation you're not sure holds. `test_mr_conjunction_narrows` above is wrong for OR-search. Verify the relation against the spec before encoding it.
- **Do not** make the relation so weak it never fails. `assert len(r1) >= 0` is always true. The relation has to be tight enough to catch bugs.
- **Do not** forget floating-point tolerance in numerical MRs. `==` on floats → flaky test. Use `abs(a - b) < ε`.
- **Do not** skip metamorphic testing because you *can* write oracles. MRs find different bugs — they test *algebraic properties*, which unit tests with fixed I/O pairs don't.

## Output format

```
## System under test
<function — and why an oracle is hard>

## Metamorphic relations
| # | Relation | Transform | Expected output relation | Spec basis |
| - | -------- | --------- | ------------------------ | ---------- |

## Tests
### MR-<N>: <name>
<code — property-test style, Hypothesis/QuickCheck>
Bug classes this catches: <what would violate this relation>

## Relation validity
<for each MR: why you believe this relation holds — cite the spec or the math>
```
