---
name: nl-to-constraints
description: Translates natural-language requirements into formal constraints — logical predicates, schemas, or property-based test generators — that a machine can check. Use when turning a spec into validation code, when writing property tests, or as the bridge between requirements and formal verification.
license: Apache-2.0
metadata:
  category: "requirements"
  suite: "general-secure-coding-agent-skills"
  version: "0.3.0"
  related: "ambiguity-detector, req-to-test, requirement-to-tlaplus-property-generator"
---

# NL → Constraints

A requirement describes a property. A constraint *is* that property, stated formally enough to evaluate. The translation is lossless when done right — the constraint says exactly what the English said, no more, no less.

## Target formalisms

Pick based on what you're constraining:

| Constraining           | Target                                        | Example                                  |
| ---------------------- | --------------------------------------------- | ---------------------------------------- |
| Data shape             | JSON Schema / Protobuf / type definition      | "Email must contain @" → `pattern: ".*@.*"` |
| Single-state invariant | Boolean predicate (any language)              | "Balance never negative" → `balance >= 0` |
| Input/output relation  | Postcondition / property-based test           | "Output is sorted" → `all(a[i] <= a[i+1])` |
| Multi-step behavior    | Temporal logic                                | → `specification-to-temporal-logic-generator` |
| Relational (multiple entities) | First-order logic / SQL CHECK          | "Every order has a customer" → FK constraint |

## English → logic — the mapping table

| English                       | Logic                          | Notes                                    |
| ----------------------------- | ------------------------------ | ---------------------------------------- |
| "X must be Y"                 | `X = Y` or `Y(X)`              |                                          |
| "X must be at least N"        | `X >= N`                       |                                          |
| "X must be between A and B"   | `A <= X <= B`                  | Inclusive unless stated — ask            |
| "Every X has property P"      | `∀x ∈ X : P(x)`                |                                          |
| "At least one X has P"        | `∃x ∈ X : P(x)`                |                                          |
| "No X has P"                  | `∀x ∈ X : ¬P(x)`               | Same as `¬∃x : P(x)`                     |
| "If X then Y"                 | `X → Y`                        | Contrapositive: `¬Y → ¬X` also holds     |
| "X unless Y"                  | `¬Y → X`                       | Same as `X ∨ Y`                          |
| "X only if Y"                 | `X → Y`                        | **Not** `Y → X`! Common trap.            |
| "X if and only if Y"          | `X ↔ Y`                        |                                          |
| "Exactly one of X, Y, Z"      | `(X∧¬Y∧¬Z) ∨ (¬X∧Y∧¬Z) ∨ (¬X∧¬Y∧Z)` | Or: sum of indicators = 1     |
| "At most N of X satisfy P"    | `|{x ∈ X : P(x)}| <= N`        |                                          |

## Worked example — data constraint

**Requirement:** "A valid user record has a username (3–32 alphanumeric characters, starting with a letter), an email containing exactly one @, an age of at least 13, and an optional bio of at most 500 characters."

**Decompose:**

| Clause                               | Constraint                                        |
| ------------------------------------ | ------------------------------------------------- |
| username: 3–32 alphanumeric, letter-start | `match(username, /^[A-Za-z][A-Za-z0-9]{2,31}$/)` |
| email: exactly one @                 | `count(email, '@') == 1`                          |
| age: at least 13                     | `age >= 13`                                       |
| bio: optional, ≤ 500 chars           | `bio is null OR len(bio) <= 500`                  |

**As JSON Schema:**

```json
{
  "type": "object",
  "required": ["username", "email", "age"],
  "properties": {
    "username": { "type": "string", "pattern": "^[A-Za-z][A-Za-z0-9]{2,31}$" },
    "email":    { "type": "string", "pattern": "^[^@]*@[^@]*$" },
    "age":      { "type": "integer", "minimum": 13 },
    "bio":      { "type": ["string", "null"], "maxLength": 500 }
  },
  "additionalProperties": false
}
```

**As a property-based test generator (Hypothesis):**

```python
from hypothesis import strategies as st

valid_user = st.fixed_dictionaries({
    "username": st.from_regex(r"^[A-Za-z][A-Za-z0-9]{2,31}$", fullmatch=True),
    "email":    st.from_regex(r"^[^@]+@[^@]+$", fullmatch=True),
    "age":      st.integers(min_value=13),
    "bio":      st.one_of(st.none(), st.text(max_size=500)),
})
```

Same constraint, three formalisms. Pick the one your tooling consumes.

## Worked example — behavioral constraint

**Requirement:** "The `sort` function returns a permutation of its input in non-decreasing order."

**Two constraints:**
1. Permutation: `Counter(output) == Counter(input)` — same elements, same multiplicities.
2. Ordered: `all(output[i] <= output[i+1] for i in range(len(output)-1))`.

**As a property test:**

```python
from collections import Counter
from hypothesis import given, strategies as st

@given(st.lists(st.integers()))
def test_sort_is_ordered_permutation(xs):
    result = sort(xs)
    assert Counter(result) == Counter(xs)                            # permutation
    assert all(result[i] <= result[i+1] for i in range(len(result)-1))  # ordered
```

This is the requirement, executable. Any `sort` that passes this test satisfies the requirement. Any `sort` that fails violates it.

## The "only if" trap

English conditionals don't map to logic the way they sound:

| English           | Means                | Common mistranslation    |
| ----------------- | -------------------- | ------------------------ |
| "X if Y"          | `Y → X`              |                          |
| "X only if Y"     | `X → Y`              | ❌ `Y → X`               |
| "X unless Y"      | `¬Y → X` (= `X ∨ Y`) | ❌ `Y → ¬X`              |

"Access granted only if authenticated" = `access → authenticated`. It does NOT say authenticated users *get* access — that would be "access granted if authenticated."

## Do not

- **Do not** translate ambiguous English. If → `ambiguity-detector` would flag it, resolve the ambiguity *before* formalizing. A precise formalization of an ambiguous requirement is precisely wrong.
- **Do not** over-constrain. "Email must contain @" → `.*@.*` is right. `^[a-z]+@[a-z]+\.[a-z]+$` is wrong — the requirement didn't say any of that.
- **Do not** under-constrain. "Exactly one @" needs both "at least one" and "at most one." `.*@.*` gives you only the first.
- **Do not** mix "only if" with "if." They're converses. Wrong direction = the constraint says the opposite of the requirement.

## Output format

```
## Requirement (verbatim)
<text>

## Decomposition
| Clause | Constraint (logic) |
| ------ | ------------------ |

## Formalization
Target: <JSON Schema | predicate | property test | temporal logic>
<code>

## Faithfulness check
- Over-constrains? <anything the formalization forbids that the English allows>
- Under-constrains? <anything the English forbids that the formalization allows>
- Direction check: <for each conditional — if/only-if/unless resolved correctly>
```
